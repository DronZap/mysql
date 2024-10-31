# mysql
Устанавливаем mysql
```
wget https://repo.mysql.com/mysql-community-release-el7.rpm
sudo rpm -ivh mysql-community-release-el7.rpm
sudo yum update
sudo yum install mysql-server
```
Устанавливаем percona-server-57
```
sudo yum module disable mysql
sudo yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
sudo percona-release setup ps57
sudo yum install Percona-Server-server-57
```
Копируем конфиги
```
cp /vagrant/conf/conf.d/* /etc/my.cnf.d/
```
Запускаем службу
```
systemctl start mysql
```
Смотрим пароль
```
cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
3-K<pTKwu,Mc
```
Подключаемся к mysql и меняем пароль
```
mysql -uroot -p'3-K<pTKwu,Mc'
ALTER USER USER() IDENTIFIED BY '3-K<pTKwu,Mc';
```
Проверяем server-id на мастере
```
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0,00 sec)
```
Проверяем что GTID включен
```
mysql> SHOW VARIABLES LIKE 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0,02 sec)
```
Создаем тестовую базу и загружаем в нее дамп
```
mysql> CREATE DATABASE bet;
Query OK, 1 row affected (0,01 sec)

mysql -uroot -p -D bet < /vagrant/bet.dmp
mysql> USE bet;
mysql> SHOW TABLES;
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
7 rows in set (0,00 sec)
```
Создаем пользователя для репликации и даем права на репликацию
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
Query OK, 0 rows affected (0,00 sec)

mysql> SELECT user,host FROM mysql.user where user='repl';
+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0,00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
Query OK, 0 rows affected, 1 warning (0,01 sec)
```
Дампим базу, исключая часть таблиц
```
 mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```
На сервере slave также копируем конфиги
```
cp /vagrant/conf/conf.d/* /etc/my.cnf.d/
```
Через директиву не менялся server_id, поэтому выполнил следующую команду
```
mysql> SET GLOBAL server_id=2;
Query OK, 0 rows affected (0,00 sec)

mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
1 row in set (0,00 sec)
```
Раскомментируем в /etc/my.cnf.d/05-binlog.cnf строки:
```
replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event
```
Заливаем дамп мастера и убеждаемсā что база естþ и она без лишних таблиц:
```
mysql> SOURCE /vagrant/master.sql;
mysql> SHOW DATABASES LIKE 'bet';
+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
1 row in set (0,00 sec)

mysql> USE bet;
Database changed
mysql> SHOW TABLES;
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0,00 sec)
```
Подключаем и запускаем слэйв
```
mysql> CHANGE MASTER TO MASTER_HOST = "192.168.11.150", MASTER_PORT = 3306, MASTER_USER="repl",MASTER_PASSWORD="!OtusLinux2018", MASTER_AUTO_POSITION =1;
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.11.150
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 451
               Relay_Log_File: slave-relay-bin.000004
                Relay_Log_Pos: 664
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      Replicate_Wild_Do_Table: 0ab5a0f5-97b5-11ef-bc9e-5254004d77d3:1
  Replicate_Wild_Ignore_Table: 0ab5a0f5-97b5-11ef-bc9e-5254004d77d3:1
```
Проверяем репликацию
```
mysql> USE bet;
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
mysql> SELECT * FROM bookmaker;
+----+---------------------------+
| id | bookmaker_name |
+----+---------------------------+
| 1 | 1xbet |
| 4 | betway |
| 5 | bwin |
| 6 | ladbrokes |
| 3 | unibet |
+----+--------------------------+
5 rows in set (0,00 sec)
```
На слейве
```
mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
| 1 | 1xbet |
| 4 | betway |
| 5 | bwin |
| 6 | ladbrokes |
| 3 | unibet |
+----+----------------+
```
