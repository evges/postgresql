**создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a**

gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

postgres  us-central1-a  e2-medium                  10.128.0.7   34.135.196.6  RUNNING

**поставьте на нее PostgreSQL 14 через sudo apt**

gcloud compute ssh postgres

sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

**проверьте что кластер запущен через sudo -u postgres pg_lsclusters**

14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

**зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым**
**postgres=# create table test(c1 text); postgres=# insert into test values('1'); \q**

sudo -u postgres psql

`postgres=# select * from test;`
 `c1`
 `1`
`(1 row)`


**остановите postgres например через sudo -u postgres pg_ctlcluster 14 main stop**

sudo pg_ctlcluster 14 main stop

**создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB**

ok

**добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk**

ok

**проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux**

lsblk

sdb       8:16   0    10G  0 disk

sudo parted /dev/sdb mklabel gpt

sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%

sudo mkfs.ext4 -L datapartition /dev/sdb1

sudo e2label /dev/sdb1 newlabel

sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT

sudo mkdir -p /mnt/data

sudo mount -o defaults /dev/sdb1 /mnt/data

sudo nano /etc/fstab

`LABEL=newlabel /mnt/data ext4 defaults 0 2`

sudo mount -a

df -h -x tmpfs -x devtmpfs

ls -l /mnt/data

`drwx------ 2 root root 16384 Mar 24 04:51 lost+found`

echo "success" | sudo tee /mnt/data/test_file

`success`

**перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)**

df -h -x tmpfs -x devtmpfs

`/dev/sdb1       9.8G   37M  9.3G   1% /mnt/data`

**сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/**

sudo chown -R postgres:postgres /mnt/data/

**перенесите содержимое /var/lib/postgres/14 в /mnt/data - mv /var/lib/postgresql/14 /mnt/data**

sudo mv /var/lib/postgresql/14 /mnt/data

**попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start**

`Error: /var/lib/postgresql/14/main is not accessible or does not exist`

**напишите получилось или нет и почему**

Кластер не был запущен т.к. файлы были перенесены

**задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/10/main который надо поменять и поменяйте его**

sudo nano /etc/postgresql/14/main/postgresql.conf

**напишите что и почему поменяли**

#data_directory = '/var/lib/postgresql/14/main'

data_directory = /mnt/data/14/main

Изменил путь к файлам, т.к. они были перенесены

**попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start**

**напишите получилось или нет и почему**

Кластер запущен

**зайдите через через psql и проверьте содержимое ранее созданной таблицы**

sudo -u postgres psql

`postgres=# select * from test;`
 `c1`
 `1`
`(1 row)`

**задание со звездочкой *: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.**

gcloud compute instances create postgres2 --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

postgres2  us-central1-a  e2-medium                  10.128.0.8   34.132.38.147  RUNNING

gcloud compute ssh postgres

sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

sudo rm -Rf /var/lib/postgres

Отвязал диск от первого инстанса и привязал ко второму

Диск не форматировал, т.к. нужно поключить с данными

lsblk

sdb       8:16   0    10G  0 disk
└─sdb1    8:17   0    10G  0 part

sudo e2label /dev/sdb1 newlabel

sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT

`sdb`
`└─sdb1  ext4     newlabel        21c559fc-500a-4d7a-9744-8064dd9227ff`

sudo mkdir -p /mnt/data

sudo mount -o defaults /dev/sdb1 /mnt/data

`LABEL=newlabel /mnt/data ext4 defaults 0 2`

sudo mount -a

df -h -x tmpfs -x devtmpfs

ls -l /mnt/data

`total 20`

`drwxr-xr-x 3 postgres postgres  4096 Mar 24 04:28 14`

`drwx------ 2 postgres postgres 16384 Mar 24 04:51 lost+found`

sudo chown -R postgres:postgres /mnt/data/

sudo -u postgres pg_ctlcluster 14 main stop

sudo nano /etc/postgresql/14/main/postgresql.conf

data_directory = /mnt/data/14/main

sudo -u postgres pg_ctlcluster 14 main start

sudo -u postgres psql

`postgres=# select * from test;`
 `c1`
 `1`
`(1 row)`


По итогу примониторовался существующий диск, данные таблицы созданной на первом инстансе видны.