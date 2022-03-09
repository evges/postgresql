**Описание/Пошаговая инструкция выполнения домашнего задания:**


**создать новый проект в Google Cloud Platform, например postgres2022-, где yyyymmdd год, месяц и день вашего рождения (имя проекта должно быть уникально на уровне GCP)**

postgres2022-19860123

**дать возможность доступа к этому проекту пользователю ifti@yandex.ru с ролью Project Editor**

**далее создать инстанс виртуальной машины Compute Engine с дефолтными параметрами**

gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

**добавить свой ssh ключ в GCE metadata**

ssh-keygen -t rsa
cat .ssh/id_rsa.pub
eval `ssh-agent -s`
ssh-add .ssh/id_rsa

**зайти удаленным ssh (первая сессия), не забывайте про ssh-add**

gcloud compute instances list
ssh evges@34.121.10.16

**поставить PostgreSQL**
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

pg_lsclusters
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log


**зайти вторым ssh (вторая сессия)**

ssh evges@34.121.10.16

**запустить везде psql из под пользователя postgres**

sudo -u postgres psql


**выключить auto commit**

\echo :AUTOCOMMIT
\set AUTOCOMMIT OFF

**сделать в первой сессии новую таблицу и наполнить ее данными**

create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
commit;

**посмотреть текущий уровень изоляции: show transaction isolation level**

show transaction isolation level;
 transaction_isolation
 read committed
(1 row)

**начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции**
**в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');**
**сделать select * from persons во второй сессии**
**видите ли вы новую запись и если да то почему?**

*Новая запись во второй сесии не видна, т.к. уровень изоляции выбран read_commited, не допускаестся грязное чтение, изменнения будут видны после выполнения коммита*

**завершить первую транзакцию - commit;**
**сделать select * from persons; во второй сессии**
**видите ли вы новую запись и если да то почему?**

*Новая запись видна, т.к. выполнен коммит в первой сессии*

**завершите транзакцию во второй сессии**
**начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;**

show transaction isolation level;
 transaction_isolation
 repeatable read
(1 row)

**в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');**
**сделать select * from persons; во второй сессии**
**видите ли вы новую запись и если да то почему?**

*Новыя запись не видна*

**завершить первую транзакцию - commit;**
**сделать select * from persons; во второй сессии**
**видите ли вы новую запись и если да то почему?**

*Новая запись не видна*

**завершить вторую транзакцию**
**сделать select * from persons; во второй сессии**
**видите ли вы новую запись и если да то почему?**

*Новая запись видна, т.к. завершили транзакции в первой и во второй сесии, изменения из первой сесии стали видны, в repeatable read видны данные которые были зафиксированы до начала транзакции*