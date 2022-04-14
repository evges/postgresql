# развернуть виртуальную машину любым удобным способом

```bash
# && sudo apt-get upgrade -y
gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

```bash
gcloud compute ssh postgres
```

# поставить на неё PostgreSQL 14 из пакетов собираемых postgres.org

```bash
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
```

# настроить кластер PostgreSQL 14 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

```bash
sudo -u postgres psql
```

```SQL

--Посмотрим в pg_settings какие пареметры установлены
select * from pg_settings where name = 'shared_buffers' \gx
select * from pg_settings where name = 'wal_buffers' \gx
select * from pg_settings where name = 'work_mem' \gx
select * from pg_settings where name = 'maintenance_work_mem' \gx
select * from pg_settings where name = 'effective_cache_size' \gx
select * from pg_settings where name = 'autovacuum' \gx
select * from pg_settings where name = 'synchronous_commit' \gx


/*DB Version: 14
OS Type: linux
DB Type: web
Total Memory (RAM): 4 GB
CPUs num: 2
Data Storage: ssd*/


ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET work_mem = '5242kB';
ALTER SYSTEM SET maintenance_work_mem = '256MB';
ALTER SYSTEM SET effective_cache_size = '3GB';
ALTER SYSTEM SET autovacuum = 'off';
ALTER SYSTEM SET synchronous_commit = 'off';

select pg_reload_conf();
```

# нагрузить кластер через утилиту
https://github.com/Percona-Lab/sysbench-tpcc (требует установки
https://github.com/akopytov/sysbench) или через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)

Выполнил нагрузку через утилиту pgbench

sudo su postgres

pgbench -i postgres

pgbench -c 50 -j 2 -P 60 -T 600 -U postgres postgres


# написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему

До изменения параметров pgbench 

```bash
initial connection time = 115.795 ms
tps = 680.515396 (without initial connection time)
```

После изменения параметров pgbench

```bash
initial connection time = 95.456 ms
tps = 936.426681 (without initial connection time)
```

Изменены параметры

```sql
--параметры выставлены согласно предложенных pgtune
    
    --для кэширования данных
    select * from pg_settings where name = 'shared_buffers' \gx
    setting         | 16384

    --Объём разделяемой памяти
    select * from pg_settings where name = 'wal_buffers' \gx
    setting         | 512

    --Используется для сортировок, построения hash таблиц
    select * from pg_settings where name = 'work_mem' \gx
    setting         | 5242

    --Определяет максимальное количество ОП для операций типа VACUUM, CREATE INDEX, CREATE FOREIGN KEY.
    select * from pg_settings where name = 'maintenance_work_mem' \gx
    setting         | 262144

    --Служит подсказкой для планировщика, сколько ОП у него в запасе
    select * from pg_settings where name = 'effective_cache_size' \gx
    setting         | 393216
--

--отключил автоочистку
select * from pg_settings where name = 'autovacuum' \gx
setting         | off 

-- отключил сбрасывание wal файлов
select * from pg_settings where name = 'synchronous_commit' \gx
setting         | off

```