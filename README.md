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
- Установим barman
```
[root@server vagrant]# sudo yum -y install barman
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.axelname.ru
 * epel: ftp.lysator.liu.se
 * extras: mirror.logol.ru
 * updates: centos-mirror.rbc.ru
```
