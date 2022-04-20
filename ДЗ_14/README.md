```bash
# && sudo apt-get upgrade -y
gcloud compute instances create postgres1 --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

10.128.0.16
```

```bash
# && sudo apt-get upgrade -y
gcloud compute instances create postgres2 --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

10.128.0.17
```

```bash
# && sudo apt-get upgrade -y
gcloud compute instances create postgres3 --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

10.128.0.18
```

```bash
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
```

# На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. 

```bash
gcloud compute ssh postgres1
sudo -u postgres psql
```

```sql
create database otus;	
\c otus

--создаём таблицы
create table test as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;

--для реплики
create table test2(idt int, fiot char(10));

```

# Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. 

```bash
gcloud compute ssh postgres2
sudo -u postgres psql
```
```sql
ALTER SYSTEM SET wal_level = logical;
```
ВМ1
```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
host    all             all             10.128.0.0/24           md5
sudo nano /etc/postgresql/14/main/postgresql.conf
listen_addresses = '*';

sudo pg_ctlcluster 14 main restart
```

```sql
--создаём публикацию для таблицы test
CREATE PUBLICATION test_pub FOR TABLE test;
\dRp+
```
Не стал создавать отдельного пользователя
\password 
otus123

\dRs
SELECT * FROM pg_stat_subscription \gx

# На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. 

ВМ2
```sql
--таблица для реплики с ВМ1
create table test(id int, fio char(10));

--подписываемся на test_sub ВМ1
CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=10.128.0.17 port=5432 user=postgres password=otus123 dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);
```

ВМ2
```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
host    all             all             10.128.0.0/24           md5

sudo nano /etc/postgresql/14/main/postgresql.conf
listen_addresses = '*';
```

```sql
ALTER SYSTEM SET wal_level = logical;
```

```bash
sudo pg_ctlcluster 14 main restart
```

Не стал создавать отдельного пользователя
```sql
\password 
otus123

create table test2 as 
select 
  generate_series(1,10) as idt,
  md5(random()::text)::char(10) as fiot;

\c otus
CREATE PUBLICATION test_pub FOR TABLE test2;
\dRp+
```

ВМ1
```sql
--подписываемся на test_sub ВМ2
CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=10.128.0.17 port=5432 user=postgres password=otus123 dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);
```

# 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). Небольшое описание, того, что получилось.

```bash
gcloud compute ssh postgres3
sudo -u postgres psql
```

```sql
create database otus;	
\c otus

create table test(id int, fio char(10));
create table test2(idt int, fiot char(10));

```

ВМ1
```sql

--подписываемся на test_pub ВМ1
CREATE SUBSCRIPTION test_sub1 
CONNECTION 'host=10.128.0.16 port=5432 user=postgres password=otus123 dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);
```

ВМ2
```sql
--подписываемся на test_pub ВМ2
CREATE SUBSCRIPTION test_sub2 
CONNECTION 'host=10.128.0.17 port=5432 user=postgres password=otus123 dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);


select * from test;
select * from test2;
```

Подписались на таблицы из ВМ1 и ВМ2, через логическое реплицирование

Задание со звездочкой*
реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.