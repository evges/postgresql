Описание/Пошаговая инструкция выполнения домашнего задания:
**1 создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)**

gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-13

**2 зайдите в созданный кластер под пользователем postgres**

gcloud compute ssh postgres

**3 создайте новую базу данных testdb**

sudo -u postgres psql
CREATE DATABASE testdb;

**4 зайдите в созданную базу данных под пользователем postgres**

\c testdb

**5 создайте новую схему testnm**

CREATE SCHEMA testnm;

**6 создайте новую таблицу t1 с одной колонкой c1 типа integer**

CREATE TABLE T1 (c1 integer);

**7 вставьте строку со значением c1=1**

INSERT INTO T1 values(1);

**8 создайте новую роль readonly**

CREATE role readonly;

**9 дайте новой роли право на подключение к базе данных testdb**

GRANT CONNECT on DATABASE testdb TO readonly;

**10 дайте новой роли право на использование схемы testnm**

GRANT USAGE on SCHEMA testnm to readonly;

**11 дайте новой роли право на select для всех таблиц схемы testnm**

GRANT SELECT on all TABLEs in SCHEMA testnm TO readonly;

**12 создайте пользователя testread с паролем test123**

CREATE USER testread with password 'test123';

**13 дайте роль readonly пользователю testread**

GRANT readonly TO testread;

**14 зайдите под пользователем testread в базу данных testdb**

psql -h 127.0.0.1 -U testread -d testdb -W
\c testdb testread

**15 сделайте select * from t1;**

ERROR:  permission denied for table t1

**16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)**

Нет

**17 напишите что именно произошло в тексте домашнего задания**

Подключить не удалось

**18 у вас есть идеи почему? ведь права то дали?**

табилица была создана в схеме public, а не в новой testnm, переключение в схему не было произведено

19 посмотрите на список таблиц

**public | t1   | table | postgres**

**20 подсказка в шпаргалке под пунктом 20**
**21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)**

Owner = postgres, не был настроен search_path

**22 вернитесь в базу данных testdb под пользователем postgres**

sudo -u postgres psql
\c testdb postgres

**23 удалите таблицу t1**

DROP TABLE T1;

**24 создайте ее заново но уже с явным указанием имени схемы testnm**

CREATE TABLE testnm.T1 (c1 integer);
GRANT SELECT on all TABLEs in SCHEMA testnm TO readonly;

**25 вставьте строку со значением c1=1**

INSERT INTO testnm.T1 values(1);

**26 зайдите под пользователем testread в базу данных testdb**

psql -h 127.0.0.1 -U testread -d testdb -W

**27 сделайте select * from testnm.t1;**
 
 c1

  1

(1 row)


**28 получилось?**

Да, обновил гранты пользователю testread

**29 есть идеи почему? если нет - смотрите шпаргалку**
**30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку**

ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly; 

**31 сделайте select * from testnm.t1;**
**32 получилось?**
**33 есть идеи почему? если нет - смотрите шпаргалку**
**31 сделайте select * from testnm.t1;**
**32 получилось?**
**33 ура!**

**34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);**

CREATE TABLE

INSERT 0 1

**35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?**

Создали таблицу в схеме public

**36 есть идеи как убрать эти права? если нет - смотрите шпаргалку**

Отозвать права

revoke CREATE on SCHEMA public FROM public; 

revoke all on DATABASE testdb FROM public; 

psql -h 127.0.0.1 -U testread -d testdb -W

**37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды**
**38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);**

**39 расскажите что получилось и почему**

Таблица не создалась
ERROR:  permission denied for schema public

Инсрерт отработал, т.к. пользователь testread создал эту таблицу, у отобрать права у владельца не получится