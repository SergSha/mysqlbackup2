<h3>### MySQL ###</h3>

<p>репликация mysql</p>

<h4>Описание домашнего задания</h4>

<p>В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp<br />
Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:<br />
| bookmaker&nbsp;&nbsp; |<br />
| competition&nbsp; |<br />
| market&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |<br />
| odds&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |<br />
| outcome&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |</p>

<ul style="disc">
<li>Настроить GTID репликацию<br />
x<br />
варианты которые принимаются к сдаче</li>
<li>рабочий вагрантафайл</li>
<li>скрины или логи SHOW TABLES</li>
<li>конфиги</li>
<li>пример в логе изменения строки и появления строки на реплике</li>
</ul>

<h4>Создание стенда MySQL replication</h4>

<p>Содержимое Vagrantfile:</p>

<pre>[user@localhost mysqlbackup2]$ <b>vi ./Vagrantfile</b></pre>

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

<pre>[user@localhost mysqlbackup2]$ <b>vagrant up</b></pre>

<pre>[user@localhost mysqlbackup2]$ <b>vagrant status</b>
Current machine states:

master                    running (virtualbox)
replica                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost mysqlbackup2]$</pre>

<h4>Сервер master</h4>

<p>Подключаемся по ssh к серверу master и зайдём с правами root:</p>

<pre>[user@localhost mysqlbackup2]$ <b>vagrant ssh master</b>
[vagrant@master ~]$ <b>sudo -i</b>
[root@master ~]#</pre>

<p>Подключаем Percona репозиторий последней версии:</p>

<pre>[root@master ~]# <b>yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y</b>
...
Installed:
  percona-release.noarch 0:1.0-27

Complete!
[root@master ~]#</pre>

<pre>[root@master ~]# <b>ls -l /etc/yum.repos.d/</b>
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

<pre>[root@master ~]# <b>yum list | grep -i percona-server-server</b>
Failed to set locale, defaulting to C
Percona-Server-server-55.x86_64             5.5.62-rel38.14.el7        percona-release-x86_64
Percona-Server-server-56.x86_64             5.6.51-rel91.0.1.el7       percona-release-x86_64
Percona-Server-server-57.x86_64             5.7.39-42.1.el7            percona-release-x86_64
[root@master ~]#</pre>

<p>Установим пакет Percona-Server-server-57:</p>

<pre>[root@master ~]# <b>yum install Percona-Server-server-57 -y</b></pre>

<p>Закидываем конфиги в директорий /etc/my.cnf.d/:</p>

<pre>[root@master ~]# <b>ls -l /etc/my.cnf.d/</b>
total 20
-rw-r--r--. 1 root root 207 Nov  4 10:03 01-base.cnf
-rw-r--r--. 1 root root  48 Nov  4 10:03 02-max-connections.cnf
-rw-r--r--. 1 root root 487 Nov  4 10:03 03-performance.cnf
-rw-r--r--. 1 root root  66 Nov  4 10:03 04-slow-query.cnf
-rw-r--r--. 1 root root 385 Nov  4 10:03 05-binlog.cnf
[root@master ~]#</pre>

<pre>[root@master ~]# <b>vi /etc/my.cnf.d/01-base.cnf</b>
[mysqld]
pid-file=/var/run/mysqld/mysqld.pid
log-error=/var/log/mysqld.log
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
symbolic-links=0

<b>server-id = 1</b>
innodb_file_per_table = 1
skip-name-resolve</pre>

<pre>[root@master ~]# <b>vi /etc/my.cnf.d/02-max-connections.cnf</b>
[mysqld]
wait-timeout = 60
max-connections = 500</pre>

<pre>[root@master ~]# <b>vi /etc/my.cnf.d/03-performance.cnf</b>
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

<pre>[root@master ~]# <b>vi /etc/my.cnf.d/04-slow-query.cnf</b>
[mysqld]
slow-query-log = 1
log-output = TABLE
long-query-time = 2</pre>

<pre>[root@master ~]# <b>vi /etc/my.cnf.d/05-binlog.cnf</b>
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

<pre>[root@master ~]# <b>systemctl start mysql</b>
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

<pre>[root@master ~]# <b>cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'</b>
A+0!Id>(VS#M
[root@master ~]#</pre>

<p>Подключаемся к mysql:</p>

<pre>[root@master ~]# <b>mysql -uroot -p'A+0!Id>(VS#M'</b>
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

<pre>mysql> <b>ALTER USER USER() IDENTIFIED BY 'root@Otus1234';</b>
Query OK, 0 rows affected (0.00 sec)

mysql></pre>

<p>Репликацию будем настраивать с использованием GTID.</p>

<p>Смотрим атрибут server_id:</p>

<pre>mysql> <b>SELECT @@server_id;</b>
+-------------+
| @@server_id |
+-------------+
|           <b>1</b> |
+-------------+
1 row in set (0.00 sec)

mysql></pre>

<p>Убеждаемся что GTID включен:</p>

<pre>mysql> <b>SHOW VARIABLES LIKE 'gtid_mode';</b>
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | <b>ON</b>    |
+---------------+-------+
1 row in set (0.00 sec)

mysql></pre>

<p>Создадим тестовую базу bet:</p>

<pre>mysql> <b>CREATE DATABASE bet;</b>
Query OK, 1 row affected (0.00 sec)

mysql> exit;
Bye
[root@master ~]#</pre>

<p>Загрузим в нее дамп:</p>

<pre>[root@master ~]# <b>mysql -uroot -p -D bet < /vagrant/bet.dmp </b>
Enter password: 
[root@master ~]#</pre>

<p>Проверим, что загрузилась база данных bet:</p>

<pre>[root@master ~]# <b>mysql -uroot -p</b>
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

<pre>mysql> <b>CREATE USER 'repl'@'%' IDENTIFIED BY 'repl@Otus1234';</b>
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

<pre>mysql> <b>GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY 'repl@Otus1234';</b>
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> exit;
Bye
[root@master ~]#</pre>

<p>Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию:</p>

<pre>[root@master ~]# <b>mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > /vagrant/master.sql</b>
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

<h4>Сервер replica</h4>

<p>В отдельном окне теримнала подключаемся по ssh к серверу replica и зайдём с правами root:</p>

<pre>[user@localhost mysqlbackup2]$ <b>vagrant ssh replica</b>
[vagrant@replica ~]$ <b>sudo -i</b>
[root@replica ~]#</pre>

<p>Аналогично серверу master устанавливаем и настраиваем необходимые пакеты и конфиги.</p>

<p>Подключаем Percona репозиторий последней версии:</p>

<pre>[root@replica ~]# <b>yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y</b>
...
Installed:
  percona-release.noarch 0:1.0-27

Complete!
[root@replica ~]#</pre>

<p>Установим пакет Percona-Server-server-57:</p>

<pre>[root@replica ~]# <b>yum install Percona-Server-server-57 -y</b></pre>

<p>Закидываем конфиги в директорий /etc/my.cnf.d/:</p>

<pre>[root@replica ~]# <b>ls -l /etc/my.cnf.d/</b>
total 20
-rw-r--r--. 1 root root 207 Nov  4 10:03 01-base.cnf
-rw-r--r--. 1 root root  48 Nov  4 10:03 02-max-connections.cnf
-rw-r--r--. 1 root root 487 Nov  4 10:03 03-performance.cnf
-rw-r--r--. 1 root root  66 Nov  4 10:03 04-slow-query.cnf
-rw-r--r--. 1 root root 385 Nov  4 10:03 05-binlog.cnf
[root@replica ~]#</pre>

<p>На сервере replica атрибут server_id должен отличаться от server_id на сервере master, в данном случае, присвоим значение 2:</p>

<pre>[root@replica ~]# <b>vi /etc/my.cnf.d/01-base.cnf</b>
[mysqld]
pid-file=/var/run/mysqld/mysqld.pid
log-error=/var/log/mysqld.log
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
symbolic-links=0

<b>server-id = 2</b>
innodb_file_per_table = 1
skip-name-resolve</pre>

<pre>[root@replica ~]# <b>vi /etc/my.cnf.d/02-max-connections.cnf</b>
[mysqld]
wait-timeout = 60
max-connections = 500</pre>

<pre>[root@replica ~]# <b>vi /etc/my.cnf.d/03-performance.cnf</b>
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

<pre>[root@replica ~]# <b>vi /etc/my.cnf.d/04-slow-query.cnf</b>
[mysqld]
slow-query-log = 1
log-output = TABLE
long-query-time = 2</pre>

<p>Раскомментируем в /etc/my.cnf.d/05-binlog.cnf последние две строки, таким образом указываем таблицы, которые будут игнорироваться при репликации:</p>

<pre>[root@replica ~]# <b>vi /etc/my.cnf.d/05-binlog.cnf</b>
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
<b>replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event</b></pre>

<p>Запускаем сервис mysql:</p>

<pre>[root@replica ~]# <b>systemctl start mysql</b>
[root@replica ~]#</pre>

<p>Находим пароль для пользователя root:</p>

<pre>[root@replica ~]# <b>cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'</b>
-tg6%oDu;-O(
[root@replica ~]#</pre>

<p>Подключаемся к mysql:</p>

<pre>[root@replica ~]# <b>mysql -uroot -p'-tg6%oDu;-O('</b>
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

<pre>mysql> <b>ALTER USER USER() IDENTIFIED BY 'root@Otus1234';</b>
Query OK, 0 rows affected (0.00 sec)

mysql></pre>

<p>Смотрим атрибут server_id:</p>

<pre>mysql> <b>SELECT @@server_id;</b>
+-------------+
| @@server_id |
+-------------+
|           <b>2</b> |
+-------------+
1 row in set (0.00 sec)

mysql></pre>

<p>Заливаем дамп master.sql:</p>

<pre>mysql> <b>SOURCE /vagrant/master.sql;</b></pre>

<p>Убеждаемся что база есть:</p>

<pre>mysql> <b>SHOW DATABASES LIKE 'bet';</b>
+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
1 row in set (0.00 sec)

mysql></pre>

<p>и она без лишних таблиц:</p>

<pre>mysql> <b>SHOW TABLES;</b>
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0.00 sec)

mysql></pre>

<p>Как мы видим, что таблицы v_same_event и events_on_demand отсутствуют.</p>

<p>Теперь подключаем к master:</p>

<pre>mysql> <b>CHANGE MASTER TO MASTER_HOST="192.168.50.10",MASTER_PORT=3306,MASTER_USER="repl",MASTER_PASSWORD="repl@Otus1234",MASTER_AUTO_POSITION=1;</b>
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql></pre>

<p>и запускаем slave:</p>

<pre>mysql> <b>START SLAVE;</b>
Query OK, 0 rows affected (0.00 sec)

mysql></pre>

<pre>mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.50.10
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 119192
               Relay_Log_File: replica-relay-bin.000002
                Relay_Log_Pos: 414
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 119192
              Relay_Log_Space: 623
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 19b19715-5c34-11ed-af28-5254004d77d3
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 19b19715-5c34-11ed-af28-5254004d77d3:1-39
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

mysql></pre>

<h4>Проверка работы репликации</h4>

<p>Проверим репликацию в действии. <br />
На сервере master:</p>

<pre>mysql> <b>USE bet;</b>
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql></pre>

<pre>mysql> <b>SELECT * FROM bookmaker;</b>
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
4 rows in set (0.00 sec)

mysql></pre>

<p>В таблицу bookmaker внесём запись:</p>

<pre>mysql> <b>INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');</b>
Query OK, 1 row affected (0.00 sec)

mysql></pre>

<p>Убедимся, что в таблице bookmaker появилась новая запись:</p>

<pre>mysql> <b>SELECT * FROM bookmaker;</b>
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  <b>1</b> | <b>1xbet</b>          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.00 sec)

mysql></pre>

<p>Проверяем, внеслись ли изменения на сервере replica:</p>

<pre>mysql> <b>USE bet;</b>
Database changed
mysql> <b>SELECT * FROM bookmaker;</b>
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  <b>1</b> | <b>1xbet</b>          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.00 sec)

mysql></pre>

<p>Как мы наблюдаем, что изменения внеслись на сервере replica. <br />
В бинлогах мы также можем увидеть записи об изменениях:</p>

<pre>[root@replica ~]# <b>mysqlbinlog /var/lib/mysql/mysql-bin.000001</b>
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#221104 17:08:29 server id 2  end_log_pos 123 CRC32 0x5f8a99a8 	Start: binlog v 4, server v 5.7.39-42-log created 221104 17:08:29 at startup
ROLLBACK/*!*/;
BINLOG '
DUdlYw8CAAAAdwAAAHsAAAAAAAQANS43LjM5LTQyLWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAANR2VjEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AaiZil8=
'/*!*/;
# at 123
#221104 17:08:29 server id 2  end_log_pos 154 CRC32 0x4d724e4e 	Previous-GTIDs
# [empty]
# at 154
#221104 17:08:30 server id 2  end_log_pos 177 CRC32 0x66e3d39f 	Stop
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;</pre>

<pre>[root@replica ~]# <b>mysqlbinlog /var/lib/mysql/mysql-bin.000002</b>
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#221104 17:08:32 server id 2  end_log_pos 123 CRC32 0x936fd1a9 	Start: binlog v 4, server v 5.7.39-42-log created 221104 17:08:32 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
EEdlYw8CAAAAdwAAAHsAAAABAAQANS43LjM5LTQyLWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAQR2VjEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AanRb5M=
'/*!*/;
# at 123
#221104 17:08:32 server id 2  end_log_pos 154 CRC32 0xde494395 	Previous-GTIDs
# [empty]
# at 154
#221104 17:55:14 server id 1  end_log_pos 219 CRC32 0xda770b8a 	GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= '19b19715-5c34-11ed-af28-5254004d77d3:40'/*!*/;
# at 219
#221104 17:55:14 server id 1  end_log_pos 292 CRC32 0x400c7127 	Query	thread_id=7	exec_time=0	error_code=0
SET TIMESTAMP=1667584514/*!*/;
SET @@session.pseudo_thread_id=7/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 292
#221104 17:55:14 server id 1  end_log_pos 419 CRC32 0xa1370e10 	Query	thread_id=7	exec_time=0	error_code=0
use `bet`/*!*/;
SET TIMESTAMP=1667584514/*!*/;
<b>INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet')</b>
/*!*/;
# at 419
#221104 17:55:14 server id 1  end_log_pos 450 CRC32 0x059a94d9 	Xid = 741
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
[root@replica ~]#</pre>


