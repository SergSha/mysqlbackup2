<h3>### MySQL ###</h3>

<p>репликация mysql</p>

<h4>Описание домашнего задания</h4>

<p>В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp<br />
Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:<br />
| bookmaker   |<br />
| competition |<br />
| market      |<br />
| odds        |<br />
| outcome     |</p>

<ul style="disc">
<li>Настроить GTID репликацию<br />
x<br />
варианты которые принимаются к сдаче</li>
<li>рабочий вагрантафайл</li>
<li>скрины или логи SHOW TABLES</li>
<li>конфиги</li>
<li>пример в логе изменения строки и появления строки на реплике</li>
</ul>


<h4>Создание стенда MySQL</h4>

<p>Содержимое Vagrantfile:</p>

<pre>[user@localhost mysqlbackup2]$ vi ./Vagrantfile</pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :master => {
    :box_name => "centos/7",
    :vm_name => "master",
    :ip => '192.168.50.10',
    :mem => '1048'
  },
  :replica => {
    :box_name => "centos/7",
    :vm_name => "replica",
    :ip => '192.168.50.11',
    :mem => '1048'
  }
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      box.vm.network "private_network", ip: boxconfig[:ip]
      box.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", boxconfig[:mem]]
      end
#      if boxconfig[:vm_name] == "replica"
#        box.vm.provision "ansible" do |ansible|
#          ansible.playbook = "ansible/playbook.yml"
#          ansible.inventory_path = "ansible/hosts"
#          ansible.become = true
#          ansible.verbose = "vvv"
#          ansible.host_key_checking = "false"
#          ansible.limit = "all"
#        end
#      end
    end
  end
end</pre>

<p>Запустим виртуальные машины:</p>

<pre>[user@localhost mysqlbackup2]$ vagrant up</pre>

<pre>[user@localhost mysqlbackup2]$ vagrant status
Current machine states:

master                    running (virtualbox)
replica                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost mysqlbackup2]$</pre>

<p>Подключаемся по ssh к серверу master и зайдём с правами root:</p>

<pre>[user@localhost mysqlbackup2]$ vagrant ssh master
[vagrant@master ~]$ sudo -i
[root@master ~]#</pre>

<p>Подключаем Percona репозиторий последней версии:</p>

<pre>[root@master ~]# [root@master ~]# yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y
...
Installed:
  percona-release.noarch 0:1.0-27

Complete!
[root@master ~]#</pre>

<pre>[root@master ~]# ls -l /etc/yum.repos.d/
total 44
-rw-r--r--. 1 root root 1664 Apr  7  2020 CentOS-Base.repo
-rw-r--r--. 1 root root 1309 Apr  7  2020 CentOS-CR.repo
-rw-r--r--. 1 root root  649 Apr  7  2020 CentOS-Debuginfo.repo
-rw-r--r--. 1 root root  630 Apr  7  2020 CentOS-Media.repo
-rw-r--r--. 1 root root 1331 Apr  7  2020 CentOS-Sources.repo
-rw-r--r--. 1 root root 7577 Apr  7  2020 CentOS-Vault.repo
-rw-r--r--. 1 root root  314 Apr  7  2020 CentOS-fasttrack.repo
-rw-r--r--. 1 root root  616 Apr  7  2020 CentOS-x86_64-kernel.repo
<b>-rw-r--r--. 1 root root  780 Nov  4 09:29 percona-original-release.repo
-rw-r--r--. 1 root root  301 Nov  4 09:29 percona-prel-release.repo</b>
[root@master ~]#</pre>

<p>Список доступных для установки пакетов Percona-Server:</p>

<pre>[root@master ~]# yum list | grep -i percona-server-server
Failed to set locale, defaulting to C
Percona-Server-server-55.x86_64             5.5.62-rel38.14.el7        percona-release-x86_64
Percona-Server-server-56.x86_64             5.6.51-rel91.0.1.el7       percona-release-x86_64
Percona-Server-server-57.x86_64             5.7.39-42.1.el7            percona-release-x86_64
[root@master ~]#</pre>

<p>Установим пакет Percona-Server-server-57:</p>

<pre>[root@master ~]# yum install Percona-Server-server-57 -y</pre>

<p>Закидываем конфиги в директорий /etc/my.cnf.d/:</p>

<pre>[root@master ~]# ls -l /etc/my.cnf.d/
total 20
-rw-r--r--. 1 root root 207 Nov  4 10:03 01-base.cnf
-rw-r--r--. 1 root root  48 Nov  4 10:03 02-max-connections.cnf
-rw-r--r--. 1 root root 487 Nov  4 10:03 03-performance.cnf
-rw-r--r--. 1 root root  66 Nov  4 10:03 04-slow-query.cnf
-rw-r--r--. 1 root root 385 Nov  4 10:03 05-binlog.cnf
[root@master ~]#</pre>

<pre>[root@master ~]# vi /etc/my.cnf.d/01-base.cnf
[mysqld]
pid-file=/var/run/mysqld/mysqld.pid
log-error=/var/log/mysqld.log
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
symbolic-links=0

server-id = 1
innodb_file_per_table = 1
skip-name-resolve</pre>

<pre>[root@master ~]# vi /etc/my.cnf.d/02-max-connections.cnf
[mysqld]
wait-timeout = 60
max-connections = 500</pre>

<pre>[root@master ~]# vi /etc/my.cnf.d/03-performance.cnf
[mysqld]
skip-external-locking
key-buffer-size = 384M
max-allowed-packet = 16M
table-open-cache = 5000
sort-buffer-size = 64M
join-buffer-size = 64M
read-buffer-size = 2M
read-rnd-buffer-size = 8M
myisam-sort-buffer-size = 64M
thread-cache-size = 8
query-cache-limit = 64M
query-cache-size = 1024M
tmp-table-size = 1024M
max-heap-table-size = 1024M
#thread-concurrency = 8 # Из за этого параметра на Vagrant-овской виртуалке mysql не взлетает</pre>

<pre>[root@master ~]# vi /etc/my.cnf.d/04-slow-query.cnf
[mysqld]
slow-query-log = 1
log-output = TABLE
long-query-time = 2</pre>

<pre>[root@master ~]# vi /etc/my.cnf.d/05-binlog.cnf
[mysqld]
log-bin = mysql-bin
expire-logs-days = 7
max-binlog-size = 16M
binlog-format = "MIXED"

# GTID replication config
log-slave-updates = On
gtid-mode = On
enforce-gtid-consistency = On

# Эта часть только для слэйва - исключаем репликацию таблиц
#replicate-ignore-table=bet.events_on_demand
#replicate-ignore-table=bet.v_same_event</pre>

<p>Запускаем сервис mysql:</p>

<pre>[root@master ~]# systemctl start mysql
[root@master ~]# systemctl status mysql
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2022-11-04 11:30:39 UTC; 8s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 24655 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 24597 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 24657 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─24657 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/my...

Nov 04 11:30:35 master systemd[1]: Starting MySQL Server...
Nov 04 11:30:39 master systemd[1]: Started MySQL Server.
[root@master ~]#</pre>

<p>Находим пароль для пользователя root:</p>

<pre>[root@master ~]# cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
<b>A+0!Id>(VS#M</b>
[root@master ~]#</pre>

<p>Подключаемся к mysql:</p>

<pre>[root@master ~]# mysql -uroot -p'A+0!Id>(VS#M'
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.39-42-log

Copyright (c) 2009-2022 Percona LLC and/or its affiliates
Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql></pre>

<p> и меняем пароль для доступа к полному функционалу:</p>

<pre>mysql> ALTER USER USER() IDENTIFIED BY 'root@Otus1234';
Query OK, 0 rows affected (0.00 sec)

mysql></pre>

<p>Репликацию будем настраивать с использованием GTID.</p>

<p>Смотрим атрибут server_id:</p>

<pre>mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)

mysql></pre>

<p>Убеждаемся что GTID включен:</p>

<pre>mysql> SHOW VARIABLES LIKE 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0.00 sec)

mysql></pre>

<p>Создадим тестовую базу bet:</p>

<pre>mysql> CREATE DATABASE bet;
Query OK, 1 row affected (0.00 sec)

mysql> exit;
Bye
[root@master ~]#</pre>

<p>Загрузим в нее дамп:</p>

<pre>[root@master ~]# mysql -uroot -p -D bet < /vagrant/bet.dmp 
Enter password: 
[root@master ~]#</pre>

<p>Проверим, что загрузилась база данных bet:</p>

<pre>[root@master ~]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.39-42-log Percona Server (GPL), Release 42, Revision b0a7dc2da2e

Copyright (c) 2009-2022 Percona LLC and/or its affiliates
Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> <b>USE bet;</b>
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> <b>SHOW TABLES;</b>
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
7 rows in set (0.00 sec)

mysql></pre>

<p>Создадим пользователя repl для репликации:</p>

<pre>mysql> CREATE USER 'repl'@'%' IDENTIFIED BY 'repl@Otus1234';
Query OK, 0 rows affected (0.01 sec)

mysql> SELECT user,host FROM mysql.user where user='repl';
+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0.00 sec)

mysql></pre>

<p>Даём ему права на эту самую репликацию:</p>

<pre>mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY 'repl@Otus1234';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> exit;
Bye
[root@master ~]#</pre>

<p>Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию:</p>

<pre>[root@master ~]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > /vagrant/master.sql
Enter password: 
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events. 
[root@master ~]# logout
[vagrant@master ~]$ logout
Connection to 127.0.0.1 closed.
[user@localhost mysqlbackup2]$</pre>

<p>Настройка сервера master завершена. Файл дампа будем заливать на сервер replica.</p>

<p>Извлечём файл дампа master.sql с сервера master:</p>

<pre>[user@localhost mysqlbackup2]$ scp -i ./.vagrant/machines/master/virtualbox/private_key vagrant@192.168.50.10:/vagrant/master.sql .
master.sql                                    100%  974KB  67.5MB/s   00:00
[user@localhost mysqlbackup2]$</pre>

<p>Отправим файл дампа master.sql на сервер replica:</p>

<pre>[user@localhost mysqlbackup2]$ scp -i ./.vagrant/machines/replica/virtualbox/private_key ./master.sql vagrant@192.168.50.11:/vagrant/
master.sql                                    100%  974KB  58.3MB/s   00:00
[user@localhost mysqlbackup2]$</pre>

<p>В отдельном окне теримнала подключаемся по ssh к серверу replica и зайдём с правами root:</p>

<pre>[user@localhost mysqlbackup2]$ vagrant ssh replica
[vagrant@replica ~]$ sudo -i
[root@replica ~]#</pre>






