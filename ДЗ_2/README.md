**сделать в GCE инстанс с Ubuntu 20.04**

gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

**• поставить на нем Docker Engine**

curl -fsSL https://get.docker.com -o get-docker.sh

sudo sh get-docker.sh

sudo gpasswd -a $USER docker 

docker ps -a

**• сделать каталог /var/lib/postgres**

sudo mkdir /var/lib/postgres

**• развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres**

docker network create pg-net

docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

docker ps -a

``15cab7f89a22   postgres:14   "docker-entrypoint.s…"   18 seconds ago   Up 16 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker``

**• развернуть контейнер с клиентом postgres**

docker run -it -d --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres

docker ps -a

``d87b26fe0dbe   postgres:14   "docker-entrypoint.s…"   4 seconds ago    Up 3 seconds    5432/tcp                                    pg-client``

``15cab7f89a22   postgres:14   "docker-entrypoint.s…"   15 minutes ago   Up 15 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker``

Клиента демонизировал

**• подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк**

docker exec -it pg-client psql -h pg-docker -U postgres

create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
commit;

**• подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP**

psql -p 5432 -U postgres -h 104.198.229.23 -d postgres -W

**• удалить контейнер с сервером**

docker stop 15cab7f89a22
docker rm 15cab7f89a22

``d87b26fe0dbe   postgres:14   "docker-entrypoint.s…"   22 minutes ago   Up 22 minutes   5432/tcp   pg-client``

Остался контейнер с клиентом

**• создать его заново**

``34571f405f13   postgres:14   "docker-entrypoint.s…"   2 seconds ago    Up 2 seconds    0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-docker``

``d87b26fe0dbe   postgres:14   "docker-entrypoint.s…"   23 minutes ago   Up 23 minutes   5432/tcp                                    pg-client``

**• подключится снова из контейнера с клиентом к контейнеру с сервером**

docker exec -it pg-client psql -h pg-docker -U postgres

• проверить, что данные остались на месте

 select * from persons;

 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

