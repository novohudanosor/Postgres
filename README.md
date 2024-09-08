# Postgres
Postgres: Backup + Репликация 
## Что нужно сделать?
* настроить hot_standby репликацию с использованием слотов
* настроить правильное резервное копирование
# Описание, выполнения домашнего задания:
1. Запустим наши ВМ командой ```vagrant up```. Будет создано три виртуальных машины. Для удобства на все хосты можно установить текстовый редактор vim и утилиту telnet:
``` 
apt install -y vim telnet
```
* Команды должны выполняться от root-пользователя Для перехода в root-пользователя вводим ```sudo -i```
2. **Настройка hot-standby репликации с использованием слотов**
* Перед настройкой репликации необходимо установить postgres-server на хосты node1 и node2:
```
apt install postgresql postgresql-contrib
systemctl start postgresql
systemctl enable postgresql
```
* Создание пользователя для репликации на хосте node1:
```
root@node1:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE USER replicator WITH REPLICATION Encrypted PASSWORD 'Otus2024!';
...
```
* В файле /etc/postgresql/14/main/postgresql.conf указываем следующие параметры:   (конфигурационный файл на обоих хостах)
```
data_directory = '/var/lib/postgresql/14/main'          # use data in another directory
hba_file = '/etc/postgresql/14/main/pg_hba.conf'        # host-based authentication file
ident_file = '/etc/postgresql/14/main/pg_ident.conf'    # ident configuration file
external_pid_file = '/var/run/postgresql/14-main.pid'   # write an extra PID file
#Указываем ip-адреса, на которых postgres будет слушать трафик на порту 5432 (параметр port)
listen_addresses = 'localhost, 192.168.56.11'
#Указываем порт порт postgres
port = 5432
#Устанавливаем максимально 100 одновременных подключений
max_connections = 100
log_directory = 'log'
log_filename = 'postgresql-%a.log'
log_rotation_age = 1d
log_rotation_size = 0
log_truncate_on_rotation = on
max_wal_size = 1GB
min_wal_size = 80MB
log_line_prefix = '%m [%p] '
#Указываем часовой пояс для Москвы
log_timezone = 'UTC+3'
timezone = 'UTC+3'
datestyle = 'iso, mdy'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
#можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления;
hot_standby = on
#Включаем репликацию
wal_level = replica
#Количество планируемых слейвов
max_wal_senders = 3
#Максимальное количество слотов репликации
max_replication_slots = 3
#будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.
hot_standby_feedback = on
#Включаем использование зашифрованных паролей
password_encryption = scram-sha-256
include_dir = 'conf.d'
```
 * на обоих хостах настраиваем параметры подключения в файле /etc/postgresql/14/main/pg_hba.conf:
```
# Database administrative login by Unix domain socket
local   all             postgres                                peer
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
host    replication     replicator      192.168.56.11/32        scram-sha-256
host    replication     replicator      192.168.56.12/32        scram-sha-256
host    all             barman          192.168.56.13/32        scram-sha-256
host    replication     barman          192.168.56.13/32        scram-sha-256
```
*Две последние строки в файле разрешают репликацию пользователю replication.*
* Перезапустим postgresql на обоих хостах:
```
systemctl restart postgresql
```
## На хосте node2: 
1) Останавливаем postgresql-server:``` systemctl stop postgresql ```
2) С помощью утилиты pg_basebackup копируем данные с node1:
``` pg_basebackup -h 192.168.57.11 -U    /var/lib/postgresql/14/main/ -R -P ```
3) В файле  /etc/postgresql/14/main/postgresql.conf меняем параметр:
``` listen_addresses = 'localhost, 192.168.57.12'  ```
4) Запускаем службу postgresql-server: ``` systemctl start postgresql ```


