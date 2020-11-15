# ДЗ №37 PostgreSQL
--------------------------------------------------------------------------------------------
```
Администрирование postgres
1. Установить postgres, сделать базовые настройки доступов
2. С помощью mamonsu подогнать конфиг сервера под ресурсы машины
3. Развернуть Barman и настроить резервное копирование postgres
```
# Практическая часть
## Установка zabbix сервера с POSTGRES SQL на Centos 7
- Для начала установим postgresql на наш сервер
```
[root@server vagrant]# rpm -Uvh https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-redhat-repo-42.0-11.noarch.rpm
Retrieving https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-redhat-repo-42.0-11.noarch.rpm
warning: /var/tmp/rpm-tmp.uvQNld: Header V4 DSA/SHA1 Signature, key ID 442df0f8: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:pgdg-redhat-repo-42.0-11         ################################# [100%]
[root@server vagrant]# yum install postgresql96-server postgresql96
```
- Инициализируем базу данных и запустим службу postgresql
```
[root@server vagrant]# /usr/pgsql-9.6/bin/postgresql96-setup initdb
Initializing database ... OK

[root@server vagrant]# systemctl start postgresql-9.6
[root@server vagrant]# systemctl enable postgresql-9.6
Created symlink from /etc/systemd/system/multi-user.target.wants/postgresql-9.6.service to /usr/lib/systemd/system/postgresql-9.6.service.
[root@server vagrant]# 
```
## НАСТРОЙКА POSTGRESQL CENTOS 7
```
[root@server vagrant]# su postgres
bash-4.2$ psql
could not change directory to "/home/vagrant": Permission denied
psql (9.6.20)
Type "help" for help.

postgres=# 
```
- Меняем пароль
```
postgres=# \password postgres
Enter new password: 
Enter it again: 
postgres=# \quit
bash-4.2$ 
```
- Создаем базу данных zabbix
```
postgres=# createdb zabbix
```
- Далее ставим сам zabbix
```
[root@server vagrant]# rpm -ivh http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm
Retrieving http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm
warning: /var/tmp/rpm-tmp.YOYzGy: Header V4 RSA/SHA512 Signature, key ID a14fe591: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:zabbix-release-3.4-2.el7         ################################# [100%]
[root@server vagrant]# yum install zabbix-server-pgsql zabbix-web-pgsql -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.sale-dedic.com
 * extras: mirrors.datahouse.ru
 * updates: mirror.reconn.ru
zabbix
...
[root@server vagrant]# zcat /usr/share/doc/zabbix-server-pgsql-3.4.15/create.sql.gz | psql -U postgres zabbix
```
## Настройка базы данных для Zabbix сервера
- Меняем zabbix_server.conf для использования созданной базы данных и запускаем службу.
```
[root@server vagrant]# vi /etc/zabbix/zabbix_server.conf
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=<пароль>
[root@server vagrant]# systemctl start zabbix-server
[root@server vagrant]# systemctl enable zabbix-server
Created symlink from /etc/systemd/system/multi-user.target.wants/zabbix-server.service to /usr/lib/systemd/system/zabbix-server.service.
```
## Настройка PHP для Zabbix веб-интерфейса
- Файл конфигурации Apache для Zabbix веб-интерфейса располагается в /etc/httpd/conf.d/zabbix.conf. Некоторые настройки PHP уже выполнены. Однако, необходимо раскомментировать “date.timezone” настройку и указать корректный для вас часовой пояс.
```
[root@server vagrant]# vi /etc/httpd/conf.d/zabbix.conf
    <IfModule mod_php5.c>
        php_value max_execution_time 300
        php_value memory_limit 128M
        php_value post_max_size 16M
        php_value upload_max_filesize 2M
        php_value max_input_time 300
        php_value max_input_vars 10000
        php_value always_populate_raw_post_data -1
        php_value date.timezone Europe/Kaliningrad
    </IfModule>
```
- Запускаем web сервер
```
[root@server vagrant]# systemctl start httpd
[root@server vagrant]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
```
## Настройка SELinux
```
[root@server vagrant]# setsebool -P httpd_can_connect_zabbix on
[root@server vagrant]# setsebool -P httpd_can_network_connect_db on
[root@server vagrant]# systemctl restart httpd
```
## Настройка Mamonsu
- Скачиваем mamonsu и настроиваем 
```
[root@server vagrant]# rpm -i https://repo.postgrespro.ru/mamonsu/keys/centos.rpm
[root@server vagrant]# yum install mamonsu
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.axelname.ru
 * epel: mirror.cherryservers.com
 * extras: mirror.logol.ru
 * updates: centos-mirror.rbc.ru
mamonsu                                                                                                                                                              | 2.9 kB  00:00:00     
mamonsu/7/x86_64/primary_db                                                                                                                                          | 2.1 kB  00:00:00     
Resolving Dependencies
--> Running transaction check
---> Package mamonsu.noarch 0:2.5.1-1.el7 will be installed
...
Installed:
  mamonsu.noarch 0:2.5.1-1.el7                                                                                                                                                              

Dependency Installed:
  python3.x86_64 0:3.6.8-17.el7            python3-libs.x86_64 0:3.6.8-17.el7            python3-pip.noarch 0:9.0.3-8.el7            python3-setuptools.noarch 0:39.2.0-10.el7  

Complete!
[root@server vagrant]# mamonsu export template template.xml --add-plugins /etc/mamonsu/plugins
Template for mamonsu has been saved as template.xml
[root@server vagrant]# ll
total 184
-rw-r--r-- 1 root root 186828 ноя 14 21:06 template.xml
```
- Создаем пользователя и базу mamonsu, далее экспортируем agent.conf со след параметрами
```
[postgres]
enabled = True
user = mamonsu
password = mamonsu
database = mamonsu
host = localhost
port = 5432
application_name = mamonsu
query_timeout = 10

[zabbix]
enabled = True
client = localhost
address = 127.0.0.1
port = 10051
```
- После данного шага добавляем наш postgresql хост. И затем экспортируя шаблон импортируем его в zabbix, через веб управление мониторингом

![image](https://github.com/Rafael-99-spec/DZ-otus-37//blob/master/import_templatexmp.PNG)

- После добавления zabbix сервера в hosts, добавим наш template который мы ранее импортировали в качестве ```*.xml``` файла.

![image](https://github.com/Rafael-99-spec/DZ-otus-37//blob/master/123456789.PNG)

## Настройка резервного копирования с помощью - 
- Установим barman на второй вм(backup)
```
[root@backup vagrant]# sudo yum -y install barman
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.axelname.ru
 * epel: ftp.lysator.liu.se
 * extras: mirror.logol.ru
 * updates: centos-mirror.rbc.ru
```
- Далее отредактируем /etc/barman.conf
```
; Barman, Backup and Recovery Manager for PostgreSQL
;
; Main configuration file

[barman]
barman_user = barman
configuration_files_directory = /etc/barman.d
barman_home = /var/lib/barman
log_file = /var/log/barman/barman.log
log_level = DEBUG
path_prefix = /usr/pgsql-11/bin
compression = gzip
```
/etc/barman.d/master.conf
```
description =  "PostgreSQL Backup"
conninfo = host=master user=barman dbname=postgres
streaming_conninfo = host=master user=barman_streaming_user dbname=postgres
backup_method = postgres
streaming_archiver = on
slot_name = barman
archiver = on
```
/var/lib/barman/.pgpass
```
master:5432:*:barman:barman
master:5432:*:barman_streaming_user:barman
```
/var/lib/barman/.ssh/authorized_key
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDSYrEXSrxtdmK8BB2YnPiK9LKEGhmDqo0LrDfyCb2hBlp/q9noEcwPFxTqrn3mRZiYA9Guyv9plSwLDa3t8C2491HpQw1RnNvXnESGsTz6ubg+OUSfB4w19RZmGIEwG93HNwzlvasJBIwoDZCJWnoUjP4DunAUWOyH1THtb6G1IuR/aco/Og7/Algubp0+mqr3jK6B/Fz8xnPyErvb5I/4eTX75Ki0vFgHkv2JFXjdPW3SvEvlV9rNx90lc5jdOlt9uc/Lh3dxkCFO2EwZGBA5ENWQERWmW75UGldAckqpy82VbjcGPJSUiQc21M1pUQ9ZakVmhaWbFixaDawl85u7 postgres@server
```
/var/lib/barman/.ssh/id_rsa
```
-----BEGIN RSA PRIVATE KEY-----
qouiYYvJcK+oUTXWKp1X/U+IiFuN9LLmwD6xvaAS7M+4iVTUWCuPyIjUhal5YqjH
8S9pO/TRT5odbT+t46UlkL3vBCaoERzVCLNNMk0L2EPlHF+tVYTJfIwCuq9agGeO
e8dKH4XzXiEPplOwpslyxtWC2XrtZpVhGuXQHz3MI8bIGZqqW0gkeHykNxeAnhGi
RVWHo2mwE4qgs387pfriow2MVyzZOlFDjK2NBvDOympYfvGA6id1O9ffSLN/Q/f2
i/fDX0cnQTgHYCdT63CvuLd0wOSu/8qdXEaxp0zBz24++wR8klblLOR1V8IzNp+g
Ll/vNpaiTL9G5cCvpV5t/1sycHEc+xFtvQI26QIDAQABAoIBADCVWbFQX8xl6FSM
J1nCYDx/neNrhMOUmkjen7OdKSXgkL3/Ojk3tnLPB5xJCazET5XmSwItIOuoLdrq
dOe9bKFfFtFDiMHWsdxQj7JGe8o562BXL7l8MjmXnJRt0f8yPhsCnSGyajDJ8wjE
UtA0ar7zMIb9oNtPCYW71DNXTYPUBoCQOo6ZV6qIePkUNSfkhY4eifllBnug7kcD
YrbO7po+Gp47IiGSpP/ehbiFTEi3GfcvwPOLaZz5vTs3913dxXIqAiOKZ8txHjDY
qouiYYvJcK+oUTXWKp1X/U+IiFuN9LLmwD6xvaAS7M+4iVTUWCuPyIjUhal5YqjH
v8YUoMECgYEA2D59zQbbcg12x17ZB9BkwSZdT8cRK7iFk31j9m8cEoeGKxNGmAzK
PQX6tdzYYME7cWVgxUzqrdajtcLI0XH9wpc7mPANa05oHB1kJQDN6uSR430gQtVe
aBSat7LwOpOjEVYmysiPNWMuykEo5AzeuvBHuHzXngXw9QUhEvnpfjUCgYEA0IeX
ExDDEt5J35Ax0guVjsxf4X1zoHRmqi2/5n3n9m4938JjXT4GULcWobNAAcINr0nq
3K2gTOAAX848pHwNwy58hR2TC+nkKbb3VQDutFGdZ2Xyp0Fb+dDqpkKjR1dSSI5z
dsWx9efKr2PUHpChau/HSOdV4oH3kl/z+ObzPGUCgYBCmhSzBi6mlSEFTOA5eOTf
XIqW3LAcMCvr/k3Ag/44csdPExPGFwJfAy1xwABg5IMDbP7+Ja+ONTKc885YO+y1
d1DizOTFLRQBvMewYewKMbYBQ/Ogwgjes6HnfFRjJj+uQkOWZ2k8Pz0VDDak7pXX
K9RbLRBX2mqZfKfwKUrSFQKBgCGaiAThgZ4LxjnJoc2oYjx1wMm0jqp/t3+bCb6Z
8YRrtXrWd26yLRBawMHkAd+Gpu/laHyRWjCpNEY8FNeoygr29cf5wRV9ZnA2dNr0
4IKcWFIuQpEjXi/+s6GBQZCgiLj6g67TIt9ur+Hdo3QdeHWkGCguZ0+uA/hJkCY/
CVllAoGAGGMQSa537do/UDajKNglyxEBwIpPZ0HhF5aehaPQLVU6eymXKAGSVlfk
vS/nzYGBVcGOzAXiETeol+OHPYNr1xlEuxIw0jhO/QrlvSr0FRGfomMQ472PByoM
Qe4zaRYfTKkr3vuWlILZyBF77I8uxL8uuTFt4UBZm6bQialnxv8=
-----END RSA PRIVATE KEY-----
```
/var/lib/barman/.ssh/id_rsa.pub
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCwJU7H47DfDdU1JO8lPklJ7P/fgMRyyqmSytBjqLaugV5NKA3xL2k79NFPmh1tP63jpSWQve8EJqgRHNUIs00yTQvYQ+UcX61VhMl8jAK6r1qAZ457x0ofhfNeIQ+mU7CmyXLG1YLZeu1mlWEa5dAfPcwjxsgZmqpbSCR4fKQ3F4CeEaJFVYejabATiqCzfzul+uKjDYxXLNk6UUOMrY0G8M7Kalh+8YDqJ3U7199Is39D9/aL98NfRydBOAdgJ1PrcK+4t3TA5K7/yp1cRrGnTMHPbj77BHySVuUs5HVXwjM2n6AuX+82lqJMv0blwK+lXm3/WzJwcRz7EW29Ajbp barman@backup
```
- На ВМ server(Сервер zabbix+postgresql) отредактируем след файлы
/var/lib/pgsql/9.6/data/pg_hba.conf
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
host    all            barman            192.168.11.0/24          md5
host    replication    barman_streaming_user            192.168.11.0/24          md5
host    all             all             192.168.11.0/24         trust
host    replication     all             192.168.11.0/24         trust
```
/var/lib/pgsql/9.6/data/postgresql.conf
```
# -----------------------------
# PostgreSQL configuration file
# -----------------------------
#

listen_addresses = '*'
max_connections = 100
shared_buffers = 128MB
dynamic_shared_memory_type = posix

#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

wal_level = 'replica'    	# minimal, replica, or logical

max_wal_size = 1GB
min_wal_size = 80MB

# - Archiving -

archive_mode = on		# enables archiving; off, on, or always
archive_command = 'barman-wal-archive backup master %p'


#------------------------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------------------------

# - Sending Servers -

# Set these on the master and on any standby that will send replication data.

max_wal_senders = 10		# max number of walsender processes
wal_keep_segments = 32		# in logfile segments; 0 disables
max_replication_slots = 10	# max number of replication slots

# - Standby Servers -

# These settings are ignored on a master server.

hot_standby = on			# "off" disallows queries during recovery

# REPORTING AND LOGGING

log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_line_prefix = '%m [%p] '
log_timezone = 'UTC'

# Locale

datestyle = 'iso, mdy'
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
```
/var/lib/pgsql/.ssh/config
```
Host backup
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```
/var/lib/pgsql/.ssh/ida_rsa
```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA0mKxF0q8bXZivAQdmJz4ivSyhBoZg6qNC6w38gm9oQZaf6vZ
6BHMDxcU6q595kWYmAPRrsr/aZUsCw2t7fAtuPdR6UMNUZzb15xEhrE8+rm4PjlE
nweMNfUWZhiBMBvdxzcM5b2rCQSMKA2QiVp6FIz+A7pwFFjsh9Ux7W+htSLkf2nK
PzoO/wJYLm6dPpqq94yugfxc/MZz8hK72+SP+Hk1++SotLxYB5L9iRV43T1t0rxL
5VfazcfdJXOY3TpbfbnPy4d3cZAhTthMGRgQORDVkBEVplu+VBpXQHJKqcvNlW43
BjyUlIkHNtTNaVEPWWpFZoWlmxYsWg2sJfObuwIDAQABAoIBAHrVlH/86rcef9c2
r/EC9TpsVC487tipI2DFVITEmysBAqW4OKn+eh31ZAkBiBCCYe2fjTV44FdM+UIa
4oohyRBNlk2TEJut8c2ZN4lMwkXBWYk69o3DYmG+jy1c8VCddIdz5NveOZYySYK5
KMKJSO3mxAh5OicnJDLKjzQKEWgnwAj0fZ3DHlrzU/WzhO/N4Mn1+vWloRug+oGU
QsXRMim+7RLdURtGUr+UO4Yq2ri5/Ql57Sg3ie60QBVjQuLRhDsFhdkijaWjMJpP
uP5cFmhrmdu/LyxW9L8ukpvof79FLdIi+Bih84WF02QMsKbJuNWFo4u5UmagqDfg
SWonDEECgYEA8Cm43sj1S4cMe7yM8+IHJ9PgBfwN7SwtkrG2Zv6Bi252iUJ9AzUn
k+bF6cR0nD9bcJR9tGKP9pMySfiVRkXncmVYOVIPF6znbEJ1LgpHEw8MiE6Ve+mj
Pd4UmYPSJoXSEEdc5SlzojfI85QYrkrDSwoly1NFqEZ419MvAfC08wUCgYEA4EJJ
GEyRwWTlHaD8MPdmLKW+SbW5MJiLdUdTr2lgSccuCUDa9jT5WU962GUnhPnNuBGB
nJTJYDMabUUlPRZ2/cc6kFQOFfV3XvJfdKu2JF3Cj3fNNBLZ//cLPfYI6h+Nv+6h
jI2Or59uSaCzT/ZlTlmpQNmgc0uaLhgFPwnxD78CgYBtp28ccX7mPEQr3vwwgnwn
6Cp6MQqexrQMLY4N2piFdCs1IqF3rHZkplKpGKTxjlAOyA3ZJcN7ntuwQIrPqi0x
4yn0Cg6QDccge/uKyPCIuC9NsSu5hwScw+B9810pb6JpAlxc2Z9NatEavfzC36np
gjmda2j7mymjyW3GIgRMjQKBgDG8oceA2+a/gM0UcjpN9Fw8mjpw0lTD0FI/coD5
5wAV69DjkGyAjTjQltc9gAlO+eA0CcH3gb4TN246oqqsu9FHCWcPLVyTZ1koeiE/
IBNqtAbrtBgzgiPx341rbsi2HNMPksbAcn/i5SvxNzOp2wgIfLBEVACeKODGNQup
IcyzAoGBAIlMHi8ISnkwE75vvI+B3ZuGAZR9OpCt4Tbc7AZs2a8z7YVu7SrZrkrB
tQjqyIxXdWxu0cfrTuL/zfEyvsfr26joNKu1h+1oX1kbvhu253/OYjzYWGgihpBK
FPWaAJa01W38xJoQPLJC7kBE/fo3VLOWDgjH34oahnK4gWfx3HW1
-----END RSA PRIVATE KEY-----
```
/var/lib/pgsql/.ssh/id_rsa.pub
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDSYrEXSrxtdmK8BB2YnPiK9LKEGhmDqo0LrDfyCb2hBlp/q9noEcwPFxTqrn3mRZiYA9Guyv9plSwLDa3t8C2491HpQw1RnNvXnESGsTz6ubg+OUSfB4w19RZmGIEwG93HNwzlvasJBIwoDZCJWnoUjP4DunAUWOyH1THtb6G1IuR/aco/Og7/Algubp0+mqr3jK6B/Fz8xnPyErvb5I/4eTX75Ki0vFgHkv2JFXjdPW3SvEvlV9rNx90lc5jdOlt9uc/Lh3dxkCFO2EwZGBA5ENWQERWmW75UGldAckqpy82VbjcGPJSUiQc21M1pUQ9ZakVmhaWbFixaDawl85u7 postgres@server
```
