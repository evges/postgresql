# Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

```bash
# && sudo apt-get upgrade -y
gcloud compute instances create postgres --project=postgres2022-19860123 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=945627251304-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220303a,mode=rw,size=10,type=projects/postgres2022-19860123/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

```bash
gcloud compute ssh postgres
```

```bash
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14
```

```bash
sudo -u postgres psql
```

SHOW log_lock_waits;

ALTER SYSTEM SET log_lock_waits = on;

SHOW deadlock_timeout;

ALTER SYSTEM SET deadlock_timeout = 200s;

select pg_reload_conf();

create database locks;

\c locks

CREATE TABLE test(
  id integer PRIMARY KEY,
  count numeric
);

INSERT INTO test VALUES (1,100), (2,200), (3,300);

--SS1

\set AUTOCOMMIT OFF

UPDATE test set count = 1111 where id = 1;

--

--SS2

\set AUTOCOMMIT OFF

UPDATE test set count = 7777 where id = 1;

--

tail -n 10 /var/log/postgresql/postgresql-14-main.log

`postgres@locks LOG:  process 16070 still waiting for ShareLock on transaction 736 after 200.235 ms`

--SS1 COMMIT;

 `postgres@locks LOG:  process 16070 acquired ShareLock on transaction 736 after 202391.253 ms` --освободил

# Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

--SS1
BEGIN;
UPDATE test set count = 1111 where id = 3;
 txid_current | pg_backend_pid
--------------+----------------
          744 |          15947

--SS2
BEGIN;
UPDATE test set count = 2222 where id = 3;
txid_current | pg_backend_pid
--------------+----------------
          745 |          16070

--SS3
BEGIN;
UPDATE test set count = 3333 where id = 3;
 txid_current | pg_backend_pid
--------------+----------------
          746 |          16468

----

--SS4
\c locks

--создадим представление
CREATE VIEW locks_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 'test'::regclass);

------------------SS1

SELECT * FROM locks_v WHERE pid = 15947; --744

 pid  |   locktype    | lockid |       mode       | granted

-------+---------------+--------+------------------+---------

 15947 | relation      | test   | RowExclusiveLock | t         --блокировка отношений, устанавливается на изменяемые строки

 15947 | transactionid | 744    | ExclusiveLock    | t         --удерживается транзация для себя 

-----------------SS2

SELECT * FROM locks_v WHERE pid = 16070; --745

  pid  |   locktype    | lockid |       mode       | granted

-------+---------------+--------+------------------+---------

 16070 | relation      | test   | RowExclusiveLock | t          --блокировка отношений, устанавливается на изменяемые строки

 16070 | tuple         | test:3 | ExclusiveLock    | t          --блокировка версии строки

 16070 | transactionid | 744    | ShareLock        | f          --ожидание получения блокировки ShareLock для SS1

 16070 | transactionid | 745    | ExclusiveLock    | t          --удерживается транзакция для себя


-----------------SS3

SELECT * FROM locks_v WHERE pid = 16468; --746

  pid  |   locktype    | lockid |       mode       | granted

-------+---------------+--------+------------------+---------

 16468 | relation      | test   | RowExclusiveLock | t          --блокировка отношений, устанавливается на изменяемые строки

 16468 | transactionid | 746    | ExclusiveLock    | t          --удерживается транзакция для себя

 16468 | tuple         | test:3 | ExclusiveLock    | f          --ожидание получения блокировки на изменяемые строки


# Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

```bash
sudo -u postgres psql
```

```psql
\c locks
```

```sql
BEGIN;
UPDATE test set count = 1111 where id = 1; --SS1
UPDATE test set count = 33333333 where id = 3; --SS3

BEGIN;
UPDATE test set count = 2222 where id = 2; --SS2
UPDATE test set count = 11111111 where id = 1; --SS1

BEGIN;
UPDATE test set count = 3333 where id = 3; --SS3
UPDATE test set count = 22222222 where id = 2; --SS2
```

`ERROR:  deadlock detected`
`DETAIL:  Process 46378 waits for ShareLock on transaction 749; blocked by process 46376.`
`Process 46376 waits for ShareLock on transaction 748; blocked by process 46375.`
`Process 46375 waits for ShareLock on transaction 750; blocked by process 46378.`
`HINT:  See server log for query details.`
`CONTEXT:  while updating tuple (0,15) in relation "test"`


COMMIT; --SS1, SS2, SS3 

locks=# select * from test;
 id |  count
----+----------
  2 |     2222
  3 | 33333333
  1 | 11111111
(3 rows)

Разобраться d ситуации можно изучив журнал:

tail -n 10 /var/log/postgresql/postgresql-14-main.log

2022-04-10 09:41:08.169 UTC [46378] postgres@locks LOG:  process 46378 detected deadlock while waiting for ShareLock on transaction 752 after 200.156 ms
2022-04-10 09:41:08.169 UTC [46378] postgres@locks DETAIL:  Process holding the lock: 46376. Wait queue: .
2022-04-10 09:41:08.169 UTC [46378] postgres@locks CONTEXT:  while updating tuple (0,15) in relation "test"
2022-04-10 09:41:08.169 UTC [46378] postgres@locks STATEMENT:  UPDATE test set count = 22222222 where id = 2;
2022-04-10 09:41:08.169 UTC [46378] postgres@locks ERROR:  deadlock detected
2022-04-10 09:41:08.169 UTC [46378] postgres@locks DETAIL:  Process 46378 waits for ShareLock on transaction 752; blocked by process 46376.
        Process 46376 waits for ShareLock on transaction 751; blocked by process 46375.
        Process 46375 waits for ShareLock on transaction 753; blocked by process 46378.
        Process 46378: UPDATE test set count = 22222222 where id = 2;
        Process 46376: UPDATE test set count = 11111111 where id = 1;
        Process 46375: UPDATE test set count = 33333333 where id = 3;


# Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Транзакции могут заблокировать друг друга, если обращение к записи таблицы будет через разные планы запроса.

Попробуйте воспроизвести такую ситуацию.