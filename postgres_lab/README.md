# ЛР 3. HA Postgres Cluster

**Марковник Полина 466635**

## Состав репозитория

| Файл | Назначение |
|---|---|
| `Dockerfile` | образ Postgres 15 + Patroni |
| `docker-compose.yml` | сервисы: pg-master, pg-slave, zoo, haproxy |
| `postgres0.yml`, `postgres1.yml` | конфигурации Patroni для первой и второй ноды |
| `haproxy.cfg` | конфигурация балансировщика |
| `screenshots/` | скриншоты хода работы |

---

## Часть 1. Поднимаем Postgres

Запуск:

```
docker compose up -d --build
```

![сборка и запуск](screenshots/02_compose_up.png)

Проверяем, что все контейнеры запущены:

```
docker ps
```

![docker ps](screenshots/03_docker_ps.png)

Логи Zookeeper (сервис запустился):

```
docker compose logs zoo
```

![логи zookeeper](screenshots/04_zoo_logs.png)

Логи нод Postgres (одна нода стала лидером, вторая — репликой):

```
docker compose logs pg-master
docker compose logs pg-slave
```

![логи pg-master](screenshots/05_logs_pg_master.png)
![логи pg-slave](screenshots/06_logs_pg_slave.png)

Состав кластера и роли нод:

```
docker exec pg-master patronictl -c /postgres0.yml list
```

![patronictl list](screenshots/07_patronictl_list.png)

**Лидером кластера стала нода: pg-master**

---

## Часть 2. Проверяем репликацию

Создаём таблицу и добавляем строку на лидере:

```
docker exec -e PGPASSWORD=postgres pg-master psql -h localhost -U postgres -c "CREATE TABLE my_first_replication (id int, my_data varchar, my_comment varchar);"
docker exec -e PGPASSWORD=postgres pg-master psql -h localhost -U postgres -c "INSERT INTO my_first_replication VALUES (1, 'my important data', 'is this line replicated?');"
```

![создание таблицы на лидере](screenshots/08_create_table_leader.png)

Читаем данные на второй ноде — таблица и строка появились автоматически (streaming replication):

```
docker exec -e PGPASSWORD=postgres pg-slave psql -h localhost -U postgres -c "SELECT * FROM my_first_replication;"
```

![данные на реплике](screenshots/09_select_replica.png)

Пробуем записать данные на реплике — получаем ошибку, так как реплика работает в режиме read-only:

```
docker exec -e PGPASSWORD=postgres pg-slave psql -h localhost -U postgres -c "INSERT INTO my_first_replication VALUES (2, 'new important data', 'is this gonna work?');"
```

![ошибка записи на реплике](screenshots/10_readonly_error.png)

---

## Часть 3. HAProxy

HAProxy добавлен в `docker-compose.yml`, конфигурация — в `haproxy.cfg`. Он опрашивает REST API Patroni на порту 8008 обеих нод и направляет трафик только на ту ноду, которая отвечает кодом 200, то есть на текущего лидера.

Логи HAProxy и состояние контейнеров:

```
docker compose logs haproxy
docker ps
```

![логи haproxy](screenshots/11_haproxy_logs.png)

Подключение через единую точку входа (`postgres_entrypoint:5432`):

```
docker exec -e PGPASSWORD=postgres pg-master psql -h postgres_entrypoint -p 5432 -U postgres -c "SELECT * FROM my_first_replication;"
```

![подключение через entrypoint](screenshots/12_entrypoint_select.png)

Страница статистики HAProxy (http://localhost:7000): лидер зелёный, реплика красная.

![haproxy stats](screenshots/13_haproxy_stats.png)

---

## Задание: отключение мастера

### 1. Останавливаем лидера

```
docker stop pg-master
```

![docker stop](screenshots/14_docker_stop.png)

Ждём около 40 секунд и смотрим состояние кластера со второй ноды:

```
docker exec pg-slave patronictl -c /postgres1.yml list
```

![состояние кластера после остановки](screenshots/15_patronictl_after_stop.png)

Логи оставшейся ноды (видно, что она стала лидером):

```
docker compose logs pg-slave
```

![логи после остановки](screenshots/16_logs_after_stop.png)

### 2. Читаем и пишем через entrypoint

```
docker exec -e PGPASSWORD=postgres pg-slave psql -h postgres_entrypoint -p 5432 -U postgres -c "SELECT * FROM my_first_replication;"
docker exec -e PGPASSWORD=postgres pg-slave psql -h postgres_entrypoint -p 5432 -U postgres -c "INSERT INTO my_first_replication VALUES (3, 'written while master is down', 'failover test');"
```

![чтение и запись через entrypoint](screenshots/17_entrypoint_rw_after_stop.png)

Статистика HAProxy после остановки:

![haproxy stats после остановки](screenshots/18_haproxy_stats_after_stop.png)

**Получилось прочитать и записать**
### 3. Что произошло

1. После остановки контейнера лидер перестал обновлять свой ключ в Zookeeper. Через время жизни ключа Zookeeper считает лидера недоступным.
2. Patroni на оставшейся ноде увидела, что лидера нет, и повысила свою ноду до primary (promote). В кластере включён `synchronous_mode`, поэтому реплика была синхронной и содержала все подтверждённые данные: потери данных нет.
3. HAProxy каждые 3 секунды проверяет порт 8008 (`inter 3s fall 3`), и после трёх неудачных проверок помечает остановленную ноду как DOWN. Новый лидер отвечает кодом 200, поэтому трафик идёт на него. Поэтому в первые секунды после остановки подключение через entrypoint может не пройти, после переключения всё работает.
4. Запись через entrypoint стала возможной, потому что HAProxy направил её на новую primary-ноду.

### 4. Возвращаем ноду

```
docker start pg-master
docker exec pg-slave patronictl -c /postgres1.yml list
docker exec -e PGPASSWORD=postgres pg-master psql -h localhost -U postgres -c "SELECT * FROM my_first_replication;"
```

![нода вернулась в кластер](screenshots/19_patronictl_after_return.png)
![данные на вернувшейся ноде](screenshots/20_data_on_returned_node.png)

Вернувшаяся нода автоматически подключилась к кластеру как реплика (в конфигурации включён `use_pg_rewind: true`) и получила данные, записанные в её отсутствие.

**Результат: есть.**
