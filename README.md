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

<pre></pre>









