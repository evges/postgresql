**Настройте выполнение контрольной точки раз в 30 секунд.**

gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

gcloud compute ssh postgres

sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

sudo -u postgres psql

select * from pg_settings where name = 'checkpoint_timeout' \gx

ALTER SYSTEM SET checkpoint_timeout = 30;

select pg_reload_conf();

**10 минут c помощью утилиты pgbench подавайте нагрузку.**

SELECT pg_current_wal_insert_lsn();  --0/3E922100

SELECT pg_walfile_name('0/3E922100');  --00000001000000000000003E

sudo su postgres

pgbench -i postgres

pgbench -c8 -P 60 -T 600 -U postgres postgres

**Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.**

SELECT pg_current_wal_insert_lsn();  --0/5C540F90

SELECT pg_walfile_name('0/5C540F90'); 00000001000000000000005C

SELECT pg_size_pretty('0/5C540F90'::pg_lsn - '0/3E922100'::pg_lsn);  --  476 MB

`Размер журанальных файлов 499248784`

SELECT * FROM pg_ls_waldir() LIMIT 10;
 00000001000000000000005E | 16777216 | 2022-04-02 09:09:19+00

 00000001000000000000005F | 16777216 | 2022-04-02 09:09:37+00

 000000010000000000000060 | 16777216 | 2022-04-02 09:09:53+00

 00000001000000000000005D | 16777216 | 2022-04-02 09:08:52+00

 00000001000000000000005C | 16777216 | 2022-04-02 09:10:54+00

`В среднем на одну контрольную точку получилось 95 мб`

**Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?**

SELECT * FROM pg_stat_bgwriter \gx

checkpoints_timed     | 517
checkpoints_req       | 1
checkpoint_write_time | 1702931
checkpoint_sync_time  | 485
buffers_checkpoint    | 126431
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 10203
buffers_backend_fsync | 0
buffers_alloc         | 10875
stats_reset           | 2022-04-02 05:04:47.269556+00

`Контрольные точки выполнялись не по расписнию, выполнилась одна, т.к. контрольная точка не успела завершиться.`

**Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.**

SHOW synchronous_commit; --on

        pgbench -c8 -P 10 -T 60 -U postgres postgres

        progress: 10.0 s, 737.6 tps, lat 10.820 ms stddev 7.202

        progress: 20.0 s, 763.0 tps, lat 10.483 ms stddev 6.929

        progress: 30.0 s, 751.7 tps, lat 10.642 ms stddev 6.675

        progress: 40.0 s, 740.2 tps, lat 10.806 ms stddev 6.738

        progress: 50.0 s, 740.5 tps, lat 10.800 ms stddev 6.461

        progress: 60.0 s, 748.1 tps, lat 10.692 ms stddev 6.800

ALTER SYSTEM SET synchronous_commit = off;

SELECT pg_reload_conf();

pgbench -c8 -P 10 -T 60 -U postgres postgres

        progress: 10.0 s, 2594.3 tps, lat 3.078 ms stddev 0.997

        progress: 20.0 s, 2587.4 tps, lat 3.091 ms stddev 1.020

        progress: 30.0 s, 2579.3 tps, lat 3.100 ms stddev 1.025

        progress: 40.0 s, 2594.0 tps, lat 3.084 ms stddev 1.007

        progress: 50.0 s, 2461.3 tps, lat 3.249 ms stddev 1.213

        progress: 60.0 s, 2543.4 tps, lat 3.145 ms stddev 1.112


`В асинхронном режиме производительность выше, фиксация изменений ничего не ждёт`

**Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?**

sudo pg_ctlcluster 14 main stop

sudo pg_dropcluster 14 main

sudo -u pg_createcluster 14 main --  --data-checksums

sudo pg_ctlcluster 14 main start

sudo -u postgres psql

create table SEV (id int);

insert into sev values (1);

select * from sev;

 id
  1
  2
  3
(3 rows)


SELECT pg_relation_filepath('sev');

 base/13726/16384

sudo pg_ctlcluster 14 main stop

dd if=/dev/zero of=/var/lib/pgsql/postgresql/14/base/13726/16384 oflag=dsync conv=notrunc bs=1 count=8

sudo pg_ctlcluster 14 main start

select * from sev;

WARNING:  page verification failed, calculated checksum 9010 but expected 79
ERROR:  invalid page in block 0 of relation base/13726/16384

SET ignore_checksum_failure = on; 

`ignore_checksum_failure` позволяет прочитать таблицу, с риском получить искажение

select * from sev;

WARNING:  page verification failed, calculated checksum 9010 but expected 79

id
  1
  2
  3
(3 rows)