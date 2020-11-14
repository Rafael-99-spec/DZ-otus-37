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
Скачиваем mamonsu и настроиваем 
```
[root@server vagrant]# git clone https://github.com/postgrespro/mamonsu
Cloning into 'mamonsu'...
remote: Enumerating objects: 377, done.
remote: Counting objects: 100% (377/377), done.
remote: Compressing objects: 100% (215/215), done.
remote: Total 6431 (delta 183), reused 302 (delta 159), pack-reused 6054
Receiving objects: 100% (6431/6431), 30.33 MiB | 3.41 MiB/s, done.
Resolving deltas: 100% (3929/3929), done.
[root@server mamonsu]# python setup.py build
running build
running build_py
creating build
creating build/lib
creating build/lib/mamonsu
copying mamonsu/__init__.py -> build/lib/mamonsu
creating build/lib/mamonsu/lib
copying mamonsu/lib/__init__.py -> build/lib/mamonsu/lib
copying mamonsu/lib/config.py -> build/lib/mamonsu/lib
copying mamonsu/lib/const.py -> build/lib/mamonsu/lib
copying mamonsu/lib/default_config.py -> build/lib/mamonsu/lib
copying mamonsu/lib/get_keys.py -> build/lib/mamonsu/lib
copying mamonsu/lib/parser.py -> build/lib/mamonsu/lib
copying mamonsu/lib/platform.py -> build/lib/mamonsu/lib
copying mamonsu/lib/plugin.py -> build/lib/mamonsu/lib
copying mamonsu/lib/queue.py -> build/lib/mamonsu/lib
copying mamonsu/lib/runner.py -> build/lib/mamonsu/lib
copying mamonsu/lib/sender.py -> build/lib/mamonsu/lib
copying mamonsu/lib/supervisor.py -> build/lib/mamonsu/lib
copying mamonsu/lib/zbx_template.py -> build/lib/mamonsu/lib
creating build/lib/mamonsu/plugins
copying mamonsu/plugins/__init__.py -> build/lib/mamonsu/plugins
creating build/lib/mamonsu/tools
copying mamonsu/tools/__init__.py -> build/lib/mamonsu/tools
creating build/lib/mamonsu/lib/senders
copying mamonsu/lib/senders/__init__.py -> build/lib/mamonsu/lib/senders
copying mamonsu/lib/senders/log.py -> build/lib/mamonsu/lib/senders
copying mamonsu/lib/senders/zbx.py -> build/lib/mamonsu/lib/senders
creating build/lib/mamonsu/plugins/common
copying mamonsu/plugins/common/__init__.py -> build/lib/mamonsu/plugins/common
copying mamonsu/plugins/common/health.py -> build/lib/mamonsu/plugins/common
creating build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/__init__.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/archive_command.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/bgwriter.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/cfs.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/checkpoint.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/connections.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/databases.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/health.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/instance.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/oldest.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/pg_buffercache.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/pg_locks.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/pg_stat_statement.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/pg_wait_sampling.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/plugin.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/pool.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/prepared_transaction.py -> build/lib/mamonsu/plugins/pgsql
copying mamonsu/plugins/pgsql/xlog.py -> build/lib/mamonsu/plugins/pgsql
creating build/lib/mamonsu/plugins/system
copying mamonsu/plugins/system/__init__.py -> build/lib/mamonsu/plugins/system
copying mamonsu/plugins/system/plugin.py -> build/lib/mamonsu/plugins/system
creating build/lib/mamonsu/tools/agent
copying mamonsu/tools/agent/__init__.py -> build/lib/mamonsu/tools/agent
copying mamonsu/tools/agent/agent.py -> build/lib/mamonsu/tools/agent
copying mamonsu/tools/agent/start.py -> build/lib/mamonsu/tools/agent
creating build/lib/mamonsu/tools/bootstrap
copying mamonsu/tools/bootstrap/__init__.py -> build/lib/mamonsu/tools/bootstrap
copying mamonsu/tools/bootstrap/sql.py -> build/lib/mamonsu/tools/bootstrap
copying mamonsu/tools/bootstrap/start.py -> build/lib/mamonsu/tools/bootstrap
creating build/lib/mamonsu/tools/report
copying mamonsu/tools/report/__init__.py -> build/lib/mamonsu/tools/report
copying mamonsu/tools/report/format.py -> build/lib/mamonsu/tools/report
copying mamonsu/tools/report/os_linux.py -> build/lib/mamonsu/tools/report
copying mamonsu/tools/report/os_win.py -> build/lib/mamonsu/tools/report
copying mamonsu/tools/report/pgsql.py -> build/lib/mamonsu/tools/report
copying mamonsu/tools/report/start.py -> build/lib/mamonsu/tools/report
creating build/lib/mamonsu/tools/sysinfo
copying mamonsu/tools/sysinfo/__init__.py -> build/lib/mamonsu/tools/sysinfo
copying mamonsu/tools/sysinfo/linux.py -> build/lib/mamonsu/tools/sysinfo
copying mamonsu/tools/sysinfo/linux_shell.py -> build/lib/mamonsu/tools/sysinfo
creating build/lib/mamonsu/tools/tune
copying mamonsu/tools/tune/__init__.py -> build/lib/mamonsu/tools/tune
copying mamonsu/tools/tune/pgsql.py -> build/lib/mamonsu/tools/tune
copying mamonsu/tools/tune/start.py -> build/lib/mamonsu/tools/tune
copying mamonsu/tools/tune/system.py -> build/lib/mamonsu/tools/tune
creating build/lib/mamonsu/tools/zabbix_cli
copying mamonsu/tools/zabbix_cli/__init__.py -> build/lib/mamonsu/tools/zabbix_cli
copying mamonsu/tools/zabbix_cli/operations.py -> build/lib/mamonsu/tools/zabbix_cli
copying mamonsu/tools/zabbix_cli/request.py -> build/lib/mamonsu/tools/zabbix_cli
copying mamonsu/tools/zabbix_cli/start.py -> build/lib/mamonsu/tools/zabbix_cli
creating build/lib/mamonsu/plugins/pgsql/driver
copying mamonsu/plugins/pgsql/driver/__init__.py -> build/lib/mamonsu/plugins/pgsql/driver
copying mamonsu/plugins/pgsql/driver/checks.py -> build/lib/mamonsu/plugins/pgsql/driver
copying mamonsu/plugins/pgsql/driver/connection.py -> build/lib/mamonsu/plugins/pgsql/driver
copying mamonsu/plugins/pgsql/driver/pool.py -> build/lib/mamonsu/plugins/pgsql/driver
creating build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/__init__.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/disk_sizes.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/disk_stats.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/la.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/memory.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/net.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/open_files.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/pg_probackup.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/proc_stat.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/scripts.py -> build/lib/mamonsu/plugins/system/linux
copying mamonsu/plugins/system/linux/uptime.py -> build/lib/mamonsu/plugins/system/linux
creating build/lib/mamonsu/plugins/system/windows
copying mamonsu/plugins/system/windows/__init__.py -> build/lib/mamonsu/plugins/system/windows
copying mamonsu/plugins/system/windows/cpu.py -> build/lib/mamonsu/plugins/system/windows
copying mamonsu/plugins/system/windows/disk_stats.py -> build/lib/mamonsu/plugins/system/windows
copying mamonsu/plugins/system/windows/helpers.py -> build/lib/mamonsu/plugins/system/windows
copying mamonsu/plugins/system/windows/memory.py -> build/lib/mamonsu/plugins/system/windows
copying mamonsu/plugins/system/windows/network.py -> build/lib/mamonsu/plugins/system/windows
creating build/lib/mamonsu/plugins/pgsql/driver/pg8000
copying mamonsu/plugins/pgsql/driver/pg8000/__init__.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000
copying mamonsu/plugins/pgsql/driver/pg8000/_version.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000
copying mamonsu/plugins/pgsql/driver/pg8000/converters.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000
copying mamonsu/plugins/pgsql/driver/pg8000/core.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000
copying mamonsu/plugins/pgsql/driver/pg8000/exceptions.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000
creating build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying mamonsu/plugins/pgsql/driver/pg8000/scramp/__init__.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying mamonsu/plugins/pgsql/driver/pg8000/scramp/_version.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying mamonsu/plugins/pgsql/driver/pg8000/scramp/core.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying mamonsu/plugins/pgsql/driver/pg8000/scramp/utils.py -> build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp


[root@server mamonsu]# python setup.py install
running install
running bdist_egg
running egg_info
creating mamonsu.egg-info
writing mamonsu.egg-info/PKG-INFO
writing top-level names to mamonsu.egg-info/top_level.txt
writing dependency_links to mamonsu.egg-info/dependency_links.txt
writing entry points to mamonsu.egg-info/entry_points.txt
writing manifest file 'mamonsu.egg-info/SOURCES.txt'
reading manifest file 'mamonsu.egg-info/SOURCES.txt'
reading manifest template 'MANIFEST.in'
writing manifest file 'mamonsu.egg-info/SOURCES.txt'
installing library code to build/bdist.linux-x86_64/egg
running install_lib
running build_py
creating build/bdist.linux-x86_64
creating build/bdist.linux-x86_64/egg
creating build/bdist.linux-x86_64/egg/mamonsu
copying build/lib/mamonsu/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu
creating build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/config.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/const.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/default_config.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/get_keys.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/parser.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/platform.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/plugin.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/queue.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/runner.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/sender.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/supervisor.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
copying build/lib/mamonsu/lib/zbx_template.py -> build/bdist.linux-x86_64/egg/mamonsu/lib
creating build/bdist.linux-x86_64/egg/mamonsu/lib/senders
copying build/lib/mamonsu/lib/senders/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/lib/senders
copying build/lib/mamonsu/lib/senders/log.py -> build/bdist.linux-x86_64/egg/mamonsu/lib/senders
copying build/lib/mamonsu/lib/senders/zbx.py -> build/bdist.linux-x86_64/egg/mamonsu/lib/senders
creating build/bdist.linux-x86_64/egg/mamonsu/plugins
copying build/lib/mamonsu/plugins/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/common
copying build/lib/mamonsu/plugins/common/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/common
copying build/lib/mamonsu/plugins/common/health.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/common
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/archive_command.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/bgwriter.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/cfs.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/checkpoint.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/connections.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/databases.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/health.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/instance.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/oldest.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/pg_buffercache.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/pg_locks.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/pg_stat_statement.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/pg_wait_sampling.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/plugin.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/pool.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/prepared_transaction.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
copying build/lib/mamonsu/plugins/pgsql/xlog.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver
copying build/lib/mamonsu/plugins/pgsql/driver/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver
copying build/lib/mamonsu/plugins/pgsql/driver/checks.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver
copying build/lib/mamonsu/plugins/pgsql/driver/connection.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver
copying build/lib/mamonsu/plugins/pgsql/driver/pool.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/_version.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/converters.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/core.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/exceptions.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp/_version.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp/core.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp
copying build/lib/mamonsu/plugins/pgsql/driver/pg8000/scramp/utils.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/system
copying build/lib/mamonsu/plugins/system/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system
copying build/lib/mamonsu/plugins/system/plugin.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/disk_sizes.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/disk_stats.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/la.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/memory.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/net.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/open_files.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/pg_probackup.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/proc_stat.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/scripts.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
copying build/lib/mamonsu/plugins/system/linux/uptime.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux
creating build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows
copying build/lib/mamonsu/plugins/system/windows/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows
copying build/lib/mamonsu/plugins/system/windows/cpu.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows
copying build/lib/mamonsu/plugins/system/windows/disk_stats.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows
copying build/lib/mamonsu/plugins/system/windows/helpers.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows
copying build/lib/mamonsu/plugins/system/windows/memory.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows
copying build/lib/mamonsu/plugins/system/windows/network.py -> build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows
creating build/bdist.linux-x86_64/egg/mamonsu/tools
copying build/lib/mamonsu/tools/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/tools
creating build/bdist.linux-x86_64/egg/mamonsu/tools/agent
copying build/lib/mamonsu/tools/agent/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/agent
copying build/lib/mamonsu/tools/agent/agent.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/agent
copying build/lib/mamonsu/tools/agent/start.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/agent
creating build/bdist.linux-x86_64/egg/mamonsu/tools/bootstrap
copying build/lib/mamonsu/tools/bootstrap/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/bootstrap
copying build/lib/mamonsu/tools/bootstrap/sql.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/bootstrap
copying build/lib/mamonsu/tools/bootstrap/start.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/bootstrap
creating build/bdist.linux-x86_64/egg/mamonsu/tools/report
copying build/lib/mamonsu/tools/report/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/report
copying build/lib/mamonsu/tools/report/format.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/report
copying build/lib/mamonsu/tools/report/os_linux.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/report
copying build/lib/mamonsu/tools/report/os_win.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/report
copying build/lib/mamonsu/tools/report/pgsql.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/report
copying build/lib/mamonsu/tools/report/start.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/report
creating build/bdist.linux-x86_64/egg/mamonsu/tools/sysinfo
copying build/lib/mamonsu/tools/sysinfo/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/sysinfo
copying build/lib/mamonsu/tools/sysinfo/linux.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/sysinfo
copying build/lib/mamonsu/tools/sysinfo/linux_shell.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/sysinfo
creating build/bdist.linux-x86_64/egg/mamonsu/tools/tune
copying build/lib/mamonsu/tools/tune/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/tune
copying build/lib/mamonsu/tools/tune/pgsql.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/tune
copying build/lib/mamonsu/tools/tune/start.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/tune
copying build/lib/mamonsu/tools/tune/system.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/tune
creating build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli
copying build/lib/mamonsu/tools/zabbix_cli/__init__.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli
copying build/lib/mamonsu/tools/zabbix_cli/operations.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli
copying build/lib/mamonsu/tools/zabbix_cli/request.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli
copying build/lib/mamonsu/tools/zabbix_cli/start.py -> build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/config.py to config.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/const.py to const.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/default_config.py to default_config.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/get_keys.py to get_keys.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/parser.py to parser.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/platform.py to platform.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/plugin.py to plugin.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/queue.py to queue.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/runner.py to runner.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/sender.py to sender.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/supervisor.py to supervisor.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/zbx_template.py to zbx_template.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/senders/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/senders/log.py to log.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/lib/senders/zbx.py to zbx.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/common/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/common/health.py to health.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/archive_command.py to archive_command.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/bgwriter.py to bgwriter.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/cfs.py to cfs.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/checkpoint.py to checkpoint.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/connections.py to connections.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/databases.py to databases.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/health.py to health.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/instance.py to instance.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/oldest.py to oldest.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/pg_buffercache.py to pg_buffercache.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/pg_locks.py to pg_locks.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/pg_stat_statement.py to pg_stat_statement.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/pg_wait_sampling.py to pg_wait_sampling.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/plugin.py to plugin.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/pool.py to pool.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/prepared_transaction.py to prepared_transaction.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/xlog.py to xlog.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/checks.py to checks.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/connection.py to connection.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pool.py to pool.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/_version.py to _version.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/converters.py to converters.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/core.py to core.pyc
  File "build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/core.py", line 637
    str(source_address) + ").") from e
                                   ^
SyntaxError: invalid syntax

byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/exceptions.py to exceptions.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp/_version.py to _version.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp/core.py to core.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/pgsql/driver/pg8000/scramp/utils.py to utils.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/plugin.py to plugin.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/disk_sizes.py to disk_sizes.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/disk_stats.py to disk_stats.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/la.py to la.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/memory.py to memory.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/net.py to net.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/open_files.py to open_files.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/pg_probackup.py to pg_probackup.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/proc_stat.py to proc_stat.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/scripts.py to scripts.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/linux/uptime.py to uptime.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows/cpu.py to cpu.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows/disk_stats.py to disk_stats.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows/helpers.py to helpers.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows/memory.py to memory.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/plugins/system/windows/network.py to network.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/agent/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/agent/agent.py to agent.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/agent/start.py to start.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/bootstrap/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/bootstrap/sql.py to sql.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/bootstrap/start.py to start.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/report/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/report/format.py to format.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/report/os_linux.py to os_linux.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/report/os_win.py to os_win.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/report/pgsql.py to pgsql.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/report/start.py to start.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/sysinfo/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/sysinfo/linux.py to linux.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/sysinfo/linux_shell.py to linux_shell.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/tune/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/tune/pgsql.py to pgsql.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/tune/start.py to start.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/tune/system.py to system.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli/__init__.py to __init__.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli/operations.py to operations.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli/request.py to request.pyc
byte-compiling build/bdist.linux-x86_64/egg/mamonsu/tools/zabbix_cli/start.py to start.pyc
creating build/bdist.linux-x86_64/egg/EGG-INFO
copying mamonsu.egg-info/PKG-INFO -> build/bdist.linux-x86_64/egg/EGG-INFO
copying mamonsu.egg-info/SOURCES.txt -> build/bdist.linux-x86_64/egg/EGG-INFO
copying mamonsu.egg-info/dependency_links.txt -> build/bdist.linux-x86_64/egg/EGG-INFO
copying mamonsu.egg-info/entry_points.txt -> build/bdist.linux-x86_64/egg/EGG-INFO
copying mamonsu.egg-info/top_level.txt -> build/bdist.linux-x86_64/egg/EGG-INFO
copying mamonsu.egg-info/zip-safe -> build/bdist.linux-x86_64/egg/EGG-INFO
creating dist
creating 'dist/mamonsu-2.5.1-py2.7.egg' and adding 'build/bdist.linux-x86_64/egg' to it
removing 'build/bdist.linux-x86_64/egg' (and everything under it)
Processing mamonsu-2.5.1-py2.7.egg
Copying mamonsu-2.5.1-py2.7.egg to /usr/lib/python2.7/site-packages
Adding mamonsu 2.5.1 to easy-install.pth file
Installing mamonsu script to /usr/bin

Installed /usr/lib/python2.7/site-packages/mamonsu-2.5.1-py2.7.egg
Processing dependencies for mamonsu==2.5.1
Finished processing dependencies for mamonsu==2.5.1
```
