# Large MySQL to AWS RDS
Basic directions and example scripts to help you migrate a large (>1TB) MySQL Database to AWS RDS whilst minimising production impacts. I hope this helps someone one day.

## Assumptions & Notes
* You have setup your current production server as a Master
* This documents the experience going from MySQL 5.6(local on Windows) to 5.7(RDS)
* The fastest and safest way to get data out is to copy the nightly disk backup and then operate on that
* STATEMENT based replication kept getting failures and could not catch up, you need to switch to ROW
* mydumper and mysqlpump both crashed with memory issues on both Windows and Linux attempting an all databases export. We needed to move to a database by database model, and we reverted to mysqldump for stability with the cost of performance
* We successully created a database by database mydumper export and import and it was quickest. We moved back to mysqldump to simplify and get routines/triggers
* Its best to avoid database busy times such as billing periods as the data is changing so rapidly as this can cause sync and load issues
* In our case the current system was in a large single INNODB file, and this also needs to be migrated to file per table, therefore a logical backup is a necessity
* Some of this is from notes and memory so may not be 100% accurate
* Switching the slave to master is an exercise for the reader

## Process Outline
1. Switch master (and slaves) to binlog_format ROW from default STATEMENT to setup reliable replication
2. Create an EC2 with MySQL 5.6 and some fast EBS 
3. Copy the nightly data folder backup to the EBS volume
4. Configure MySQL 5.6 for good batch performance and point at nightly backup data folder
5. Start up MySQL 5.6, then mysql_upgrade, then restart the database
6. Export each database excluding mysql, performance_schema & information_schema to the EBS volume. n.b. Excluding the mysql database may cause us replication issues later
7. Create a large RDS MySQL 5.7 with a lot of resource, you can downsize later, and configure accordingly
8. Load data into RDS
9. Configure and start slave replication
10. If all goes well clean up, remove EC2 & EBS and downsize RDS

## Steps

### 1. Set binlog format
Ensure you have replication set up and that it is running **ROW** and not the default **STATEMENT** in */etc/my.cnf* on master and all slaves
```
binlog_format=ROW
log-bin=mysql-bin
```
### 2. Create MySQL EC2
Create an EC2 and install MySQL 5.6
* We used an m5.large
* We created a 4TB gp2 EBS volume with 12000 IOPS
* Install MySQL
```bash
sudo yum-config-manager --disable mysql57-community
sudo yum-config-manager --enable mysql56-community
sudo yum install mysql-community-server
```
### 3. Copy Nightly File Backup to EBS volme
We were on Windows (pukes)
```bat
net stop MySQL
xcopy d:\bin\MySQL\*.* e:\DB_Backup\bin\MySQL /s /c /y
net start MySQL
```
We then used cygwin to scp the data directory with the AWS key file, but could have used [PuttySCP](https://www.chiark.greenend.org.uk/~sgtatham/putty/) instead but we are lazy
```
scp -i db_ec2.pem -r '/cygdrive/e/DB_Backup/bin/MYSQL/MySQL Server/data' ec2-user@EC2_IP_Address:/EBS_Volume_Name
```

### 4. Setup MySQL 5.6
Make sure the mysql user owns the directory and everything beneath it
```bash
sudo chown mysql:mysql /EBS_Volume_Name/data
```
Configure MySQL 5.6 */etc/my.cnf*
```
datadir=/EBS_Volume_Name/data
innodb_file_per_table=1
lower_case_table_names=1
binlog_format=ROW
log-bin=mysql-bin
skip-log-bin
server-id=5
innodb_change_buffering=all
innodb_open_files=32768
max_allowed_packet=1G
innodb_change_buffer_max_size=25
innodb_buffer_pool_size=4G
innodb_log_file_size=1G
innodb_log_buffer_size=128M
innodb_flush_log_at_trx_commit=2
innodb_autoextend_increment=256M
innodb_doublewrite=0
innodb_autoinc_lock_mode=2
innodb_io_capacity_max=1500
innodb_io_capacity=700
```
Take note of the last binlog in the data directory as we will use that down below
```
⚠️ We will assume mysql-bin.666666, look down below where that will be used
```

### 5. Start MySQL 5.6

Startup mysql, look in the log for issues
```bash
sudo /sbin/service mysqld start
```
If all goes well, upgrade, check the log for any issues
```bash
mysql_upgrade -u root -p > mysql_upgrade_56.log 2>&1
```
Check the log for any issues, and if all good restart
```bash
sudo /sbin/service mysqld stop
sudo /sbin/service mysqld start
```

### 6. Export Database
Write basic bash script(s) to export database by database. We had one huge database and lots of smaller ones so we create two scripts and ran them in parallel
```bash
#!/bin/bash
mkdir /EBS_Volume_Name/dump
DBPASS='mydbpass'
for DATABASE in db1 db2 db3
do
	mysqldump -u root -p$DBPASS --routines --triggers --max-allowed-packet=512MB --opt $DATABASE | gzip > /EBS_Volume_Name/dump/$DATABASE.sql.gz
done
```
🛑 Don't do too many in parallel as you may overwhelm your database and get failures

You can probably shut down your database at this stage
```bash
sudo /sbin/service mysqld stop
```

### 7. Create RDS MySQL 5.7
Create an RDS with MySQL 5.7
* We used an db.r5.xlarge with the latest 5.7
* We created a 1.5TB with 14000 IOPS
* Create a parameter group to suit faster loading
```
innodb_file_per_table=1
binlog_format=ROW
innodb_change_buffering=all
innodb_open_files=32768
max_allowed_packet=1073741824
innodb_change_buffer_max_size=25
innodb_buffer_pool_size=25600000000
innodb_log_file_size=134217728
innodb_log_buffer_size=8388608
```
* startup instance

### 8. Load data into RDS MySQL 5.7
Write basic bash script(s) to import database by database. We had one huge database and lots of smaller ones so we create two scripts and ran them in parallel
```bash
#!/bin/bash
DBSRVR='RDS_Name'
DBUSER='mydbuser'
DBPASS='mydbpass'
for DATABASE in db1 db2 db3
do
	gunzip -c /EBS_Volume_Name/dump/$DATABASE.sql.gz | mysql -h $DBSRVR -u $DBUSER -p$DBPASS $DATABASE
done
```
🛑 Don't do too many in parallel as you may overwhelm your database and get failures

### 9. Make RDS Slave
Tell RDS to be a slave to your master, and cross your fingers 🤞
```SQL
CALL mysql.rds_set_external_master('master_server', 3306, 'master_replication_user', 'master_replication_pass', 'mysql-bin.666666', 0, 0);
CALL mysql.rds_start_replication;
SHOW SLAVE STATUS;
```
📒 Running mysql from the commndline and running *SHOW SLAVE STATUS \G* gives a nice rowwise output

### 10. Cleanup
Never forget to clean up your mess, this stuff costs money
* Turn off EC2
* Delete EBS volume
* Reconfigure RDS to defaults, and then downsize

Now you can do the next migration steps of taking that to a Master which should be specifc to your environment. Good luck!