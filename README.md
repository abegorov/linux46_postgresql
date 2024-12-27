# Репликация PostgeSQL

## Задание

1. Настроить **hot_standby** репликацию с использованием слотов.
2. Настроить правильное резервное копирование

## Реализация

Задание сделано на **rockylinux/9** версии **v9.5-20241118.0**. Для автоматизации процесса написан **Ansible Playbook** [playbook.yml](playbook.yml) который последовательно запускает следующие роли:

- **postgresql_install** - устанавливает **postgresql-server** и разрешает сервис **postgrsql** в **firewalld** на всех узлах.
- **epel_release** - добавляет репозиторий **Extra Packages for Enterprise Linux (EPEL)**.
- **barman_cli** - устанавливает **barman-cli** на всех узлах.
- **barman_install** - устанавливает **barman** на узле **barman**.
- **ssh_key** - генерит **ssh** ключ для пользователей заданных в перменной **ssh_key_users** (для пользователя **barman** на узле **barman**) и сохраняет его в файл директории `passwords` (`passwords/barman.pub`).
- **ssh_known_hosts** - добавляет узлы из переменной **ssh_known_hosts_list** в файл `~/.ssh/known_hosts` для пользователей из переменной **ssh_known_hosts_user** (**leader** для пользователя **barman** на узле **barman**).
- **pgpass** - создаёт файлы `.pgpass` со значениями из переменной **pgpass_files** (указывает имя пользователя и пароль для подключения к **leader** на узле **barman** для пользователя **barman**).
- **postgresql_init** - вызывает `postgresql-setup --initdb` на узле **leader**.
- **postgresql_config** - настраивает **PostgreSQL** на узлах **leader** и **replica** (`postgresql.conf` и `pg_hba.conf`), сама конфигурация хранится для каждого узла в переменных, начинающихся с **postgresql_config**.
- **postgresql_users** - создаёт на сервере **PostreSQL** (на **leader**) пользователей, определённых в переменной **postgresql_users_list**.
- **ssh_authorized_keys** - добавляет ключи пользователей из переменной **ssh_authorized_keys_users_files** в файл `~/.ssh/authorized_keys` (ключ `passwords/barman.pub` для пользователя `postgres` на узле **leader**).
- **postgresql_replica** - настраивает узел **replica**, как реплику узла **leader**.
- **barman_config** - настраивает **barman** на узле **barman** и делает резервную копию узла **leader**.

Данные роли настраиваются с помощью переменных, определённых в следующих файлах:

- [group_vars/all.yml](group_vars/all.yml] - общие переменные для всех узлов;
- [host_vars/leader.yml](host_vars/leader.yml) - переменные для узла **leader**;
- [host_vars/replica.yml](host_vars/replica.yml) - переменные для узла **replica**;
- [host_vars/barman.yml](host_vars/barman.yml) - переменные для узла **barman**.

Конфигурация **PostgreSQL** находится в файлах [defaults/main.yml](roles/postgresql_config/defaults/main.yml) и переопределяется в [group_vars/all.yml](group_vars/all.yml], переменные из этих файлов используются в шаблонах [pg_hba.conf](roles/postgresql_config/templates/pg_hba.conf) и [postgresql.conf](roles/postgresql_config/templates/postgresql.conf). Начиная с версии 12, **PostgreSQL** не использует файл `recovery.conf`.

Конфигурация **barman** находится в файлах [defaults/main.yml](roles/barman_config/defaults/main.yml) и переопределяется в файле [host_vars/barman.yml](host_vars/barman.yml). Для её генерации используются шаблоны [barman.conf](roles/barman_config/templates/barman.conf) и [server.conf](roles/barman_config/templates/server.conf).

## Запуск

Необходимо скачать **VagrantBox** для **rockylinux/9** версии **v9.5-20241118.0** и добавить его в **Vagrant** под именем **rockylinux/9/v9.5-20241118.0**. Сделать это можно командами:

```shell
curl -OL https://dl.rockylinux.org/pub/rocky/9.5/images/x86_64/Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box
vagrant box add Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box --name "rockylinux/9/v9.5-20241118.0"
rm Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box
```

Для того, чтобы **vagrant 2.3.7** работал с **VirtualBox 7.1.0** необходимо добавить эту версию в **driver_map** в файле **/usr/share/vagrant/gems/gems/vagrant-2.3.7/plugins/providers/virtualbox/driver/meta.rb**:

```ruby
          driver_map   = {
            "4.0" => Version_4_0,
            "4.1" => Version_4_1,
            "4.2" => Version_4_2,
            "4.3" => Version_4_3,
            "5.0" => Version_5_0,
            "5.1" => Version_5_1,
            "5.2" => Version_5_2,
            "6.0" => Version_6_0,
            "6.1" => Version_6_1,
            "7.0" => Version_7_0,
            "7.1" => Version_7_0,
          }
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.1.4_SUSE r165100**
- **Ansible 2.18.1**
- **Python 3.11.11**
- **Jinja2 3.1.4**

## Проверка

Проверим работу репликации:

```text
❯ vagrant ssh leader -c "sudo -u postgres psql -xc 'SELECT * FROM pg_stat_replication;'"
could not change directory to "/home/vagrant": Permission denied
-[ RECORD 1 ]----+------------------------------
pid              | 29808
usesysid         | 16384
usename          | replication
application_name | walreceiver
client_addr      | 192.168.57.12
client_hostname  |
client_port      | 48184
backend_start    | 2024-12-27 17:35:50.743728+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/60001B0
write_lsn        | 0/60001B0
flush_lsn        | 0/60001B0
replay_lsn       | 0/60001B0
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-12-27 19:36:22.552293+03
-[ RECORD 2 ]----+------------------------------
pid              | 29812
usesysid         | 16385
usename          | barman
application_name | barman_receive_wal
client_addr      | 192.168.57.13
client_hostname  |
client_port      | 46928
backend_start    | 2024-12-27 17:36:02.349131+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/60001B0
write_lsn        | 0/60001B0
flush_lsn        | 0/6000000
replay_lsn       |
write_lag        | 00:00:05.650622
flush_lag        | 01:59:50.434485
replay_lag       | 02:00:12.123701
sync_priority    | 0
sync_state       | async
reply_time       | 2024-12-27 19:36:14.479494+03

❯ vagrant ssh replica -c "sudo -u postgres psql -xc 'SELECT * FROM pg_stat_wal_receiver;'"
could not change directory to "/home/vagrant": Permission denied
-[ RECORD 1 ]---------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 29146
status                | streaming
receive_start_lsn     | 0/3000000
receive_start_tli     | 1
written_lsn           | 0/60001B0
flushed_lsn           | 0/60001B0
received_tli          | 1
last_msg_send_time    | 2024-12-27 19:38:12.648892+03
last_msg_receipt_time | 2024-12-27 19:38:12.648943+03
latest_end_lsn        | 0/60001B0
latest_end_time       | 2024-12-27 17:41:27.953987+03
slot_name             | replica
sender_host           | 192.168.57.11
sender_port           | 5432
conninfo              | user=replication password=******** channel_binding=prefer dbname=replication host=192.168.57.11 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
```

Создадим базу данных и проверим, что она реплицировалась на реплику:

```text
❯ vagrant ssh leader -c "sudo -u postgres psql -xc 'CREATE DATABASE otus;'"
could not change directory to "/home/vagrant": Permission denied
CREATE DATABASE

❯ vagrant ssh replica -c 'sudo -u postgres psql -xc \\l'
could not change directory to "/home/vagrant": Permission denied
List of databases
-[ RECORD 1 ]-----+----------------------
Name              | otus
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges |
-[ RECORD 2 ]-----+----------------------
Name              | postgres
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges |
-[ RECORD 3 ]-----+----------------------
Name              | template0
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges | =c/postgres          +
                  | postgres=CTc/postgres
-[ RECORD 4 ]-----+----------------------
Name              | template1
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges | =c/postgres          +
                  | postgres=CTc/postgres
```

Проверим работу резервного копирования:

```text
❯ vagrant ssh barman -c 'sudo -u barman barman check leader'
Server leader:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: OK (interval provided: 4 days, latest backup age: 2 hours, 6 minutes, 46 seconds)
        backup minimum size: OK (23.5 MiB)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: OK (have 1 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK
```

Сделаем резервную копию:

```text
❯ vagrant ssh barman -c 'sudo -u barman barman backup leader'
Starting backup using postgres method for server leader in /var/lib/barman/leader/base/20241227T164449
Backup start at LSN: 0/6000A10 (000000010000000000000006, 00000A10)
Starting backup copy via pg_basebackup for 20241227T164449
Copy done (time: less than one second)
Finalising the backup.
Backup size: 31.1 MiB
Backup end at LSN: 0/8000000 (000000010000000000000007, 00000000)
Backup completed (start time: 2024-12-27 16:44:49.553373, elapsed time: less than one second)
Processing xlog segments from streaming for leader
        000000010000000000000006
        000000010000000000000007
```

Удалим базу данных **otus** и восстановим её из резервной копии:

```text
❯ vagrant ssh leader -c "sudo -u postgres psql -xc 'DROP DATABASE otus;'"
could not change directory to "/home/vagrant": Permission denied
DROP DATABASE

❯ vagrant ssh barman -c 'sudo -u barman barman list-backup leader'
leader 20241227T164449 - F - Fri Dec 27 19:44:50 2024 - Size: 31.1 MiB - WAL Size: 0 B
leader 20241227T143623 - F - Fri Dec 27 17:36:24 2024 - Size: 23.5 MiB - WAL Size: 32.8 KiB

❯ vagrant ssh leader -c "sudo systemctl stop postgresql.service"

❯ vagrant ssh leader -c "sudo -u postgres rm /var/lib/pgsql/data/ -r"

❯ vagrant ssh barman -c 'sudo -u barman barman recover leader 20241227T164449 /var/lib/pgsql/data/ --remote-ssh-comman "ssh postgres@192.168.57.11"'
Starting remote restore for server leader using backup 20241227T164449
Destination directory: /var/lib/pgsql/data/
Remote command: ssh postgres@192.168.57.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2024-12-27 16:54:17.037278+00:00, elapsed time: 3 seconds)
Your PostgreSQL server has been successfully prepared for recovery!

❯ vagrant ssh leader -c "sudo systemctl start postgresql.service"
```

Проверим, что база восстановилась:

```text
❯ vagrant ssh leader -c 'sudo -u postgres psql -xc \\l'
could not change directory to "/home/vagrant": Permission denied
List of databases
-[ RECORD 1 ]-----+----------------------
Name              | otus
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges |
-[ RECORD 2 ]-----+----------------------
Name              | postgres
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges |
-[ RECORD 3 ]-----+----------------------
Name              | template0
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges | =c/postgres          +
                  | postgres=CTc/postgres
-[ RECORD 4 ]-----+----------------------
Name              | template1
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges | =c/postgres          +
                  | postgres=CTc/postgres
```

Перезапустим реплику:

```text
❯ vagrant ssh replica -c "sudo systemctl stop postgresql.service"

❯ vagrant ssh replica -c "sudo -u postgres rm /var/lib/pgsql/data/ -r"

❯ vagrant provision
```

```text
PLAY [Replica provision] *******************************************************

TASK [postgresql_replica : Init PostgreSQL database cluster] *******************
changed: [replica]

TASK [postgresql_config : Configure PostgreSQL server] *************************
--- before: /var/lib/pgsql/data/postgresql.conf
+++ after: /home/abegorov/.ansible/tmp/ansible-local-90534yl9sj3eh/tmpifsi85wc/postgresql.conf
@@ -21,7 +21,7 @@
 lc_numeric = 'en_US.UTF-8'
 lc_time = 'en_US.UTF-8'
 default_text_search_config = 'pg_catalog.english'
-listen_addresses = '192.168.57.11'
+listen_addresses = '192.168.57.12'
 hot_standby = on
 wal_level = 'replica'
 max_wal_senders = 3

changed: [replica]

TASK [postgresql_config : Configure PostgreSQL host-based authentication] ******
ok: [replica]

TASK [postgresql_config : Enable and start service postgresql] *****************
changed: [replica]

RUNNING HANDLER [postgresql_config : Reload service postgresql] ****************
changed: [replica]
```

Проверим реплику после восстановления:

```text
❯ vagrant ssh replica -c 'sudo -u postgres psql -xc \\l'
could not change directory to "/home/vagrant": Permission denied
List of databases
-[ RECORD 1 ]-----+----------------------
Name              | otus
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges |
-[ RECORD 2 ]-----+----------------------
Name              | postgres
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges |
-[ RECORD 3 ]-----+----------------------
Name              | template0
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges | =c/postgres          +
                  | postgres=CTc/postgres
-[ RECORD 4 ]-----+----------------------
Name              | template1
Owner             | postgres
Encoding          | UTF8
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
Access privileges | =c/postgres          +
                  | postgres=CTc/postgres

❯ vagrant ssh leader -c "sudo -u postgres psql -xc 'SELECT * FROM pg_stat_replication;'"
could not change directory to "/home/vagrant": Permission denied
-[ RECORD 1 ]----+------------------------------
pid              | 30768
usesysid         | 16385
usename          | barman
application_name | barman_receive_wal
client_addr      | 192.168.57.13
client_hostname  |
client_port      | 36842
backend_start    | 2024-12-27 19:56:01.711902+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/A000060
write_lsn        | 0/A000060
flush_lsn        | 0/A000000
replay_lsn       |
write_lag        | 00:00:02.344664
flush_lag        | 00:02:33.701881
replay_lag       | 00:05:51.511417
sync_priority    | 0
sync_state       | async
reply_time       | 2024-12-27 20:01:53.227507+03
-[ RECORD 2 ]----+------------------------------
pid              | 32265
usesysid         | 16384
usename          | replication
application_name | walreceiver
client_addr      | 192.168.57.12
client_hostname  |
client_port      | 49226
backend_start    | 2024-12-27 19:59:22.086496+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/A000060
write_lsn        | 0/A000060
flush_lsn        | 0/A000060
replay_lsn       | 0/A000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2024-12-27 20:01:53.64446+03

❯ vagrant ssh replica -c "sudo -u postgres psql -xc 'SELECT * FROM pg_stat_wal_receiver;'"
could not change directory to "/home/vagrant": Permission denied
-[ RECORD 1 ]---------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 31026
status                | streaming
receive_start_lsn     | 0/A000000
receive_start_tli     | 1
written_lsn           | 0/A000060
flushed_lsn           | 0/A000060
received_tli          | 1
last_msg_send_time    | 2024-12-27 20:00:52.289139+03
last_msg_receipt_time | 2024-12-27 20:00:52.288562+03
latest_end_lsn        | 0/A000060
latest_end_time       | 2024-12-27 19:59:22.089618+03
slot_name             | replica
sender_host           | 192.168.57.11
sender_port           | 5432
conninfo              | user=replication password=******** channel_binding=prefer dbname=replication host=192.168.57.11 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
```
