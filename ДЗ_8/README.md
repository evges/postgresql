**создать GCE инстанс типа e2-medium и standard disk 10GB**

gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

**установить на него PostgreSQL 13 с дефолтными настройками**

gcloud compute ssh postgres

sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-13

**применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла**

max_connections = 40 +

shared_buffers = 1GB +

effective_cache_size = 3GB +

maintenance_work_mem = 512MB +

checkpoint_completion_target = 0.9 +

wal_buffers = 16MB +

default_statistics_target = 500 +

random_page_cost = 4 +

effective_io_concurrency = 2 +

work_mem = 6553kB +

min_wal_size = 4GB +

max_wal_size = 16GB +

sudo nano /etc/postgresql/13/main/postgresql.conf

sudo service postgresql restart

**выполнить pgbench -i postgres**

sudo su postgres
pgbench -i postgres

**запустить pgbench -c8 -P 60 -T 3600 -U postgres postgres**

pgbench -c8 -P 60 -T 3600 -U postgres postgres

**дать отработать до конца**
**зафиксировать среднее значение tps в последней ⅙ части работы**

Среднее значение = 484 tps

**а дальше настроить autovacuum максимально эффективно**
**так чтобы получить максимально ровное значение tps на горизонте часа**

SELECT name, setting, context, short_desc FROM pg_settings WHERE category like '%Autovacuum%';

ALTER SYSTEM SET autovacuum_max_workers TO 10;

ALTER SYSTEM SET autovacuum_vacuum_threshold TO 5000;

ALTER SYSTEM SET autovacuum_vacuum_scale_factor TO 0.05;

ALTER SYSTEM SET autovacuum_vacuum_cost_limit TO 1000;

ALTER SYSTEM SET autovacuum_vacuum_cost_delay TO 10;

ALTER SYSTEM SET autovacuum_naptime TO 15;

sudo service postgresql restart

progress: 2820.0 s, 470.2 tps, lat 17.013 ms stddev 26.225

progress: 2880.0 s, 471.6 tps, lat 16.954 ms stddev 25.924

progress: 2940.0 s, 483.6 tps, lat 16.537 ms stddev 25.264

progress: 3000.0 s, 464.2 tps, lat 17.233 ms stddev 25.675

progress: 3060.0 s, 476.2 tps, lat 16.794 ms stddev 25.740

progress: 3120.0 s, 483.0 tps, lat 16.561 ms stddev 26.278

progress: 3180.0 s, 483.2 tps, lat 16.557 ms stddev 25.387

progress: 3240.0 s, 477.7 tps, lat 16.743 ms stddev 26.002

progress: 3300.0 s, 479.9 tps, lat 16.668 ms stddev 26.075

progress: 3360.0 s, 469.9 tps, lat 17.026 ms stddev 25.850

progress: 3420.0 s, 480.6 tps, lat 16.642 ms stddev 26.031

progress: 3480.0 s, 480.0 tps, lat 16.666 ms stddev 26.133

progress: 3540.0 s, 476.4 tps, lat 16.789 ms stddev 25.699

progress: 3600.0 s, 470.4 tps, lat 16.994 ms stddev 26.341

Значение tps более-менее выравнялось 