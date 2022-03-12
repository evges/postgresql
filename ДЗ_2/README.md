сделать в GCE инстанс с Ubuntu 20.04

```gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any

• поставить на нем Docker Engine
• сделать каталог /var/lib/postgres
• развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres
• развернуть контейнер с клиентом postgres
• подключится из контейнера с клиентом к контейнеру с сервером и сделать
таблицу с парой строк
• подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP
• удалить контейнер с сервером
• создать его заново
• подключится снова из контейнера с клиентом к контейнеру с сервером
• проверить, что данные остались на месте
• оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами