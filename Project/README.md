Темы для работы:
Сравнение производительности от 2 PostgreSQL кластеров на большом обьеме данных

0. Подготовительные работы

Ограничил работу WSL2 4GB оперативной памяти и двумя процессорами

```bash
wsl --shutdown
notepad "$env:USERPROFILE/.wslconfig"

[wsl2] 
memory=4GB # Ограничивает память виртуальной машины в WSL 2 
processors=4 # Заставляет виртуальную машину WSL 2 использовать два виртуальных процессора
```

1. Создаём две машины в докере
```bash

sudo gpasswd -a $USER docker 

docker ps -a

docker network create pg-net

sudo mkdir /var/lib/postgres
sudo mkdir /var/lib/postgres2

#ВМ1
docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

#ВМ2
docker run --name pg-docker2 --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5433:5432 -v /var/lib/postgres2:/var/lib/postgresql/data postgres:14

docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS       PORTS                    NAMES
19e954f9a5aa   postgres:14   "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:5433->5432/tcp   pg-docker2
6eb0ff7e5920   postgres:14   "docker-entrypoint.s…"   3 minutes ago        Up 3 minutes        0.0.0.0:5432->5432/tcp   pg-docker


#Создадим клиента

docker run -it -d --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
```

2. Выполним ВМ1 настройку параметров Postgresql для оптимальной производительности (https://pgtune.leopard.in.ua/)

# DB Version: 14
# OS Type: linux
# DB Type: dw
# Total Memory (RAM): 4 GB
# CPUs num: 2
# Data Storage: ssd

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 13107kB
min_wal_size = 4GB
max_wal_size = 16GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1


Контейнер PostgtreSQL использует конфигурацию PostgreSQL по умолчанию, которая неэффективно использует ядра процессора или память. Рекомендуется настроить конфигурацию для повышения производительности.
