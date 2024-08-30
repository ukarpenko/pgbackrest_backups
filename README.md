# VARIABLES:

Для запуска восстановления запустить джобу restore_download_backup вручную, задав переменные:

TARGET_TIME: yyyy-mm-dd hh:mm:ss - время восстанавливаемого бэкапа
## 
PG_BACKREST_STANZA: dev - целевой сервер (dev или demo)



# Redmine_backups



## Описание проекта

Данный проект нужен для создания регулярных бэкапов базы данных **PostgreSQL** таск-трекера **Redmine** средствами программы **PgBackRest** и автоматизацией для создания инкрементальных бэкапов в будние дни, полных бэкапов в субботу и отправку на **S3-сервер minIO** в воскресенье.

## Как это работает?

Сервер с **PgBackRest** (10.177.132.148) общается в сервером баз данных **PostgreSQL** (10.177.132.235) для создания бэкапов и кладет их в хранилище **main**. **PgBackRest** должен быть установлен на обоих серверах, 
должны быть созданы SSH-ключи:

```
пользователь pgbackrest@10.177.132.148 -> 10.177.132.235
пользователь postgres@10.177.132.235 -> 10.177.132.148
```

Для создания бэкапов должна быть настроена конфигурация **PgBackRest** на каждом из серверов (`/etc/pgbackrest/pgbackrest.conf`), настроены разрешения для подключения к БД PostgreSQL (`/etc/postgresql/14/main/postgresql.conf`; `/etc/postgresql/14/main/pg_hba.conf`) и создано хранилище **main**.

## Установка и настройка

1. **(Оба сервера)** Добавляем репозитории **PostgreSQL** для установки **PgBackRest версии 2.50**:

```
sudo apt update
sudo apt update && sudo apt install -y wget ca-certificates && wget --quiet -O - https://www postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc apt/sources.list.d/pgdg.list'
sudo apt update
sudo apt install postgresql-14
```
___

2. **(Сервер бэкапов)** Создаем директории для логов, даем им права:

```
sudo mkdir -p -m 770 /var/log/pgbackrest
sudo chown pgbackrest:pgbackrest /var/log/pgbackrest #На сервере с бд меняем юзера:группу на postgres
sudo mkdir -p /etc/pgbackrest
sudo mkdir -p /etc/pgbackrest/conf.d
sudo touch /etc/pgbackrest/pgbackrest.conf
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
sudo chown pgbackrest:pgbackrest /etc/pgbackrest/pgbackrest.conf #На сервере с бд меняем юзера:группу на postgres
```
___

3. **(Оба сервера)** Генерируем **SSH-ключи** на обоих серверах:
На сервере бэкапов генерируем **.pub** из под юзера **pgbackrest** и передаем на сервер бд.
На сервере базы данных генерируем **.pub** из под юзера **postgres** и передаем на сервер бэкапов.

4. **(Сервер БД)** Настраиваем доступы для PostgreSQL:

В конфиге /etc/postgresql/14/main/postgresql.conf добавляем:
```
listen_addresses = '*'
```

В конце этого же конфига добавляем: 

```
archive_command = 'pgbackrest --stanza=main archive-push %p'
archive_mode = on
max_wal_senders = 3
wal_level = replica

```
В конфиге /etc/postgresql/14/main/pg_hba.conf добавляем:
```
host  all  all  0.0.0.0/0  md5
```
___

5. **(Сервер БД)** Настраиваем конфигурацию **PgBackRest**:

В конфиге `/etc/pgbackrest/pgbackrest.conf` добавляем:
```
[main]
pg1-path=/var/lib/postgresql/14/main

[global]
log-level-file=detail
repo1-host=<repository_server_ip>
```
Затем перезагружаем **PostgreSQL**
```
systemctl restart postgresql
```
___

6. **(Сервер бэкапов)** Настраиваем конфиг **PgBackRest**:

В конфиге `/etc/pgbackrest/pgbackrest.conf` добавляем:

```
[main]
pg1-host=<postgres_server_ip>
pg1-path=/var/lib/postgresql/14/main

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
start-fast=y
```
___

7. **(Сервер бэкапов)** Создаем хранилище **PgBackRest**:

```
sudo mkdir -m 770 /var/lib/pgbackrest
sudo chown -R pgbackrest /var/lib/pgbackrest/
sudo -u pgbackrest pgbackrest --stanza=main stanza-create
```
Проверяем на обоих серверах
```
sudo -u pgbackrest/postgres pgbackrest --stanza=main --log-level-console=info check
```
___

## Ручной запуск резервного копирования

1. Если по какой-то причине потребовалось запустить создание бэкапа в ручном режиме, то из под пользователя **pgbackrest на сервере 10.177.132.148** необходимо выполнить данную команду: `sudo -u pgbackrest pgbackrest --stanza=main backup` (для выбора типа бэкапа нужно добавить ключ `--type=full backup` или `--type=incr`).
2. Для подробного вывода в консоль необходимо добавить ключ: `--log-level-console=info`.
3. При успешном завершении бэкапа его можно найти по пути `/var/lib/pgbackrest/main`.

## Восстановление бэкапа

1. На сервере БД останавливаем работающий кластер командой `sudo pg_ctlcluster 14 main stop`.
2. Восстанавливаем бэкап командой: `sudo -u postgres pgbackrest --stanza=main --log-level-console=info --delta --recovery-option=recovery_target=immediate restore` (для восстановления полного бэкапа убираем ключ `--recovery-option=recovery_target=immediate`).
3. После восстановления может оказаться так, что база зависнет в режиме восстановления (будут ошибки в духе `ERROR: cannot execute DROP DATABASE in a read-only transaction`). Решается командой: `sudo -u postgres psql -c "select pg_wal_replay_resume()"`.
4. Запускаем кластер: `sudo pg_ctlcluster 14 main start`.
5. На всякий случай делаем еще один бэкап (**уже переходим на сервер 10.177.132.148**) командой: `sudo pgbackrest --stanza=main backup`.