# Build AWS Compute and install Oracle on Docker, create an AMI and use it then with terraform for an aws compute service

The big advantage of doing this you can start an Oracle DB 19c EE very fast if you run it with terraform with pre-configured aws image.

Before doing this please check your license with Oracle. For Demo around this you can hopefully use the [OTN developer license](https://www.oracle.com/downloads/licenses/standard-license.html).
 Here is the step-by-step guide. Please enter your own values:

Create AWS EC2 Compute:
* Region: FRA
* Name: cmoradb19c
* AMI: Amazon Linux 2 Kernel 5.10 AMI 2.0.20240529.0 x86_64 HVM gp2 - ami-06801a226628c00ce
* Architecture: 64-bit (x86)
* Instance-TYpe: m5.4xlarge (16 vCPU, 64 GiB memory, 1000 GiB storage (gp3 volume, 3000 IOPS))
* Key pair: cmawskeyoraclegg
* Default VPC
* Subnet: Default-Public-Subnet
* Autoassign Public IP : enable
* Create Security group: cmoradb19c_sg
* Port 22 for ssh Source Type: custom Source: MyIP
* Port 1521 DB Source Type: custom Source: MyIP and the IP for other service need to contact the DB
* Port 5500 EM Source Type: custom Source: MyIP
* Storage 1 x 1000 GiB GP3 Root volume, 3000 IOPS, (Not encrypted)
*
*  Launch Instance: After Luanch add TAG owner=cmutzlitz@confluent.io

 Now, we need to install everything into the DB:

```bash
# set permission to key
chmod 400 ~/keys/cmawskeyoraclegg.pem
#ssh into instance:
ssh -i ~/keys/cmawskeyoraclegg.pem ec2-user@XXXX.eu-central-1.compute.amazonaws.com
# Update the compute service with software
sudo su -
yum update -y
# install git
yum install git -y
# install java
yum install java-1.8.0-openjdk-devel.x86_64 -y
# glib
sudo yum install glibc-devel
# install docker
yum install -y docker
# set environment
echo vm.max_map_count=262144 >> /etc/sysctl.conf
sysctl -w vm.max_map_count=262144
echo "    *       soft  nofile  65535
    *       hard  nofile  65535" >> /etc/security/limits.conf
sed -i -e 's/1024:4096/65536:65536/g' /etc/sysconfig/docker
# enable docker    
usermod -a -G docker ec2-user
service docker start
chkconfig docker on
curl -L https://github.com/docker/compose/releases/download/1.21.0/docker-compose-`uname -s`-`uname -m` | sudo tee /usr/local/bin/docker-compose > /dev/null
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# install Oracle container
mkdir -p /home/ec2-user/software
chown ec2-user:ec2-user -R /home/ec2-user/software
cd /home/ec2-user/software
# Get oracle Docker images, we need to build own image
git clone https://github.com/oracle/docker-images.git
cd docker-images/OracleDatabase/SingleInstance/dockerfiles/19.3.0
# Download the Binaries in docker-images/OracleDatabase/SingleInstance/dockerfiles/19.3.0
# get download zip from here http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html
# https://download.oracle.com/otn/linux/oracle19c/190000/LINUX.X64_193000_db_home.zip
# the file name is LINUX.X64_193000_db_home.zip
# scp to aws compute: 
# on local desktop: go into dir of binararies ~/Downloads, with scp it takes a while to copy 3.11 GB 
scp -i ~/keys/cmawskeyoraclegg.pem LINUX.X64_193000_db_home.zip ec2-user@XXXX.eu-central-1.compute.amazonaws.com:~/.
# in aws ssh session, move Oracle 19c into docker-images/OracleDatabase/SingleInstance/dockerfiles/19.3.0 directory
mv /home/ec2-user/LINUX.X64_193000_db_home.zip .
mkdir -p /home/ec2-user/software/oracle
cd /home/ec2-user/software/oracle      
wget https://download.oracle.com/otn_software/linux/instantclient/1925000/instantclient-sqlplus-linux.x64-19.25.0.0.0dbru.zip
unzip instantclient-sqlplus-linux.x64-19.25.0.0.0dbru.zip
rm instantclient-sqlplus-linux.x64-19.25.0.0.0dbru.zip
cd ..
wget https://download.oracle.com/otn_software/linux/instantclient/1919000/instantclient-basic-linux.x64-19.19.0.0.0dbru.el9.zip
unzip instantclient-basic-linux.x64-19.19.0.0.0dbru.el9.zip
rm instantclient-basic-linux.x64-19.19.0.0.0dbru.el9.zip
cd instantclient_19_19
cp -r * ../oracle/instantclient_19_25/
cd ..
rm -rf instantclient_19_25/
cd oracle/instantclient_19_25
export LD_LIBRARY_PATH=/home/ec2-user/software/oracle/instantclient_19_25
#wget https://download.oracle.com/otn_software/linux/instantclient/214000/instantclient-sqlplus-linux.x64-21.4.0.0.0dbru.zip 
#unzip -d /home/ec2-user/software/oracle instantclient-sqlplus-linux.x64-21.4.0.0.0dbru.zip
export PATH=/home/ec2-user/software/oracle/instantclient_19_25:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
export ORACLE_HOME=/home/ec2-user/software/oracle/instantclient_19_25
export SQL_PATH=$ORACLE_HOME
export TNS_ADMIN=/home/ec2-user/software/oracle
#try
sqlplus
# add in /home/ec2-user/.bash_profile
# export LD_LIBRARY_PATH=/home/ec2-user/software/oracle/instantclient_21_14:$LD_LIBRARY_PATH
# export PATH=.....:$LD_LIBRARY_PATH
# export ORACLE_HOME=/home/ec2-user/software/oracle/instantclient_19_25
# export SQL_PATH=$ORACLE_HOME
# export TNS_ADMIN=/home/ec2-user/software/oracle
# export ORACLE_SID=ORCLCDB
cd ../..
chown -R ec2-user:ec2-user docker-images/
chown -R ec2-user:ec2-user oracle/
# Follow the steps from here: https://github.com/oracle/docker-images/tree/main/OracleDatabase/SingleInstance
cd /home/ec2-user/software/docker-images/OracleDatabase/SingleInstance/dockerfiles/
# Build image for Enterprise Edition
./buildContainerImage.sh -v 19.3.0 -e –I
# Check images
docker images
# Run the database as ec2-user, exit from root first
exit 
sudo docker run \
--name oracle19c \
-p 1521:1521 \
-p 5500:5500 \
-e ORACLE_PDB=ORCLPDB1 \
-e ORACLE_PWD=confluent123 \
-e ORACLE_MEM=4000 \
-v /opt/oracle/oradata \
-d oracle/database:19.3.0-ee
# The database need s to be 15 minutes to be up and running
sudo docker ps

#Connect to DB
Hostname: localhost
Port: 1521
Service Name: ORCLPDB1
Username: sys
Password: confluent123
Role: AS SYSDBA

# getting shell into oracleDB container, and check if DB is running
sudo docker exec -it oracle19c /bin/bash
ps -ef | grep ora
export TNS_ADMIN=/opt/oracle/product/19c/dbhome_1/network/admin
cat $TNS_ADMIN/tnsnames.ora

# It takes a while till services are up and running
sqlplus /nolog
SQL> connect sys/confluent123@ORCLCDB as sysdba
SQL> show pdbs
    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 ORCLPDB1                       READ WRITE NO
# If you see two PDBS the Oralce is ready configured      
SQL> exit;   
# check services
lsnrctl status
Services Summary...
Service "1a0e330b9eaa0b62e063020011ac10bd" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
Service "ORCLCDB" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
Service "ORCLCDBXDB" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
Service "orclpdb1" has 1 instance(s).
  Instance "ORCLCDB", status READY, has 1 handler(s) for this service...
The command completed successfully
# ORCLDBC is running and ORCLPDB1 is up
exit

# setup env
cd /home/ec2-user/software/oracle
export TNS_ADMIN=/home/ec2-user/software/oracle
vi tnsnames.ora
ORCLCDB= (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))(CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = ORCLCDB)))
ORCLPDB1= (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))(CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = ORCLPDB1)))
cat $TNS_ADMIN/tnsnames.ora

# logging into container
sqlplus sys/confluent123@ORCLCDB as sysdba
SQL> select instance_name, con_id, version from v$instance;
# better way to where I AM
SQL> select decode(sys_context('USERENV', 'CON_NAME'),'CDB$ROOT',sys_context('USERENV', 'DB_NAME'),sys_context('USERENV', 'CON_NAME')) DB_NAME, 
            decode(sys_context('USERENV','CON_ID'),1,'CDB','PDB') TYPE 
       from DUAL;
SQL> connect sys/confluent123@ORCLPDB1 as sysdba
SQL> select instance_name, con_id, version from v$instance;
SQL> select dbid, con_id, name from v$pdbs;
SQL> alter session set container = ORCLPDB1;
# better way to check
SQL> select decode(sys_context('USERENV', 'CON_NAME'),'CDB$ROOT',sys_context('USERENV', 'DB_NAME'),sys_context('USERENV', 'CON_NAME')) DB_NAME, 
            decode(sys_context('USERENV','CON_ID'),1,'CDB','PDB') TYPE 
       from DUAL;
SQL> exit

# try to connect without errors
sqlplus sys/confluent123@ORCLCDB as sysdba

# copy in a different terminal the installation scripts from https://github.com/ora0600/confluent-cdc-workshop/tree/main/terraform/aws/oraclexe21c/docker/scripts
scp -i ~/keys/cmawskeyoraclegg.pem 02_create_user.sql ec2-user@XXXX.eu-central-1.compute.amazonaws.com:~/.
scp -i ~/keys/cmawskeyoraclegg.pem 03_create_schema_datamodel.sql ec2-user@XXXX.eu-central-1.compute.amazonaws.com:~/.
scp -i ~/keys/cmawskeyoraclegg.pem 04_load_data.sql ec2-user@XXXX.eu-central-1.compute.amazonaws.com:~/.
scp -i ~/keys/cmawskeyoraclegg.pem 06_data_generator.sql ec2-user@XXXXX.eu-central-1.compute.amazonaws.com:~/.
scp -i ~/keys/cmawskeyoraclegg.pem 05_21c_privs.sql ec2-user@XXXX.eu-central-1.compute.amazonaws.com:~/.

# Back to AWS terminal, create user ordermgmt
cd ~/
mv *.sql software/oracle/
cd /home/ec2-user/software/oracle
sqlplus sys/confluent123@ORCLPDB1 as sysdba
SQL> @02_create_user.sql
sqlplus ordermgmt/kafka@ORCLPDB1
sql> @03_create_schema_datamodel.sql
sqlplus ordermgmt/kafka@ORCLPDB1
sql> @04_load_data.sql
sqlplus ordermgmt/kafka@ORCLPDB1
sql> @06_data_generator.sql
SQL> exit

# Create CDC User with Roles and privileges for PDB
# you need to change 05_21c_privs.sql before (this is for the Confluent CDC Connector based on Logminer, if you do not need this, leave it out)
# change XEPDB1 to ORCLPDB1
sqlplus sys/confluent123@ORCLPDB1 as sysdba
sql> @05_21c_privs.sql

# Enable archive log into DB
sudo su -
docker exec -it oracle19c /bin/bash
export ORACLE_SID=ORCLCDB
sqlplus /nolog
sql> CONNECT sys/confluent123 AS SYSDBA
# REDO LOG Structure
# Group 3 /opt/oracle/oradata/ORCLCDB/redo03.log
# Group 2 /opt/oracle/oradata/ORCLCDB/redo02.log
# Group 1 /opt/oracle/oradata/ORCLCDB/redo01.log
# We need to add 3 new groups with 2GB and afterwards drop the old
# Update redo log file size and increase groups using the steps captured below
SQL>  alter database add logfile group 4  '/opt/oracle/oradata/ORCLCDB/redo04.log' size 2G;
SQL>  alter database add logfile group 5  '/opt/oracle/oradata/ORCLCDB/redo05.log' size 2G;
SQL>  alter database add logfile group 6  '/opt/oracle/oradata/ORCLCDB/redo06.log' size 2G;
# check structure
SQL> SELECT a.group#,b.member,a.members, a.bytes/1024/1024 as MB, a.status FROM v$log a,v$logfile b WHERE a.group# = b.group#;
# Switch the log file to the newly added redo log groups and delete the old inactive groups. Switch the baseline/default 3 redo log groups to inactive state so that they can be deleted.
SQL>  alter system switch logfile;
# Based on the current redo log state and need make the old groups inactive 
SQL> alter system archive log group 1;
SQL> alter system archive log group 2;
SQL> alter system archive log group 3;
# Drop the old groups once they are inactive, check with SELECT a.group#,b.member,a.members, a.bytes/1024/1024 as MB, a.status FROM v$log a,v$logfile b WHERE a.group# = b.group#;
# It takes while till log group 3 is inactive
SQL> ALTER DATABASE DROP LOGFILE GROUP 1; 
SQL> ALTER DATABASE DROP LOGFILE GROUP 2;
SQL> ALTER DATABASE DROP LOGFILE GROUP 3;
# now, you should have 3 group a 2GB memebers
SQL> SELECT a.group#,b.member,a.members, a.bytes/1024/1024 as MB, a.status FROM v$log a,v$logfile b WHERE a.group# = b.group#;
#    GROUP# MEMBER                                 MEMBERS   MB STATUS
#---------- -------------------------------------- ------- ---- -----
#         4 /opt/oracle/oradata/ORCLCDB/redo04.log       1 2048 CURRENT
#         5 /opt/oracle/oradata/ORCLCDB/redo05.log       1 2048 UNUSED
#         6 /opt/oracle/oradata/ORCLCDB/redo06.log       1 2048 UNUSED
# Turn on Archivelog Mode
sql> shutdown immediate
sql> startup mount
sql> alter database archivelog;
sql> alter database open;
# Should show "Database log mode: Archive Mode"
sql> archive log list
sql> ALTER SESSION SET CONTAINER=cdb$root;
sql> ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
sql> ALTER SESSION SET CONTAINER=ORCLPDB1;
sql> ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
sql> ALTER SESSION SET CONTAINER=cdb$root;
sql> ALTER SYSTEM SET commit_logging = 'BATCH' CONTAINER=ALL;
sql> ALTER SYSTEM SET commit_wait = 'NOWAIT' CONTAINER=ALL;
sql> exit;
exit;
# back as ec2-user
exit

# Database is prepared
# check external access with sql developer I am running version 23.1
# I do create two connections in sqldeveloper
#1: oracle21c aws PDB1 as sysdba CDC
User: sys
PW: confluent123
Connection type: BASIC
HOST: 3.76.212.17
Port: 1521
service name: ORCLPDB1
#2: oracle21c aws CDB1 as sysdba CDC
User: sys
PW: confluent123
Connection type: BASIC
HOST: 3.76.212.17
Port: 1521
service name: ORCLCDB

# Setup user and priv for GG, I will use DBA to make it easier, if you do not need this let it out
sqlplus sys/confluent123@orclcdb as sysdba
# CGGNORTH DATABASE SETUP AT CDB LEVEL
sql> ALTER SESSION SET CONTAINER=cdb$root;
sql> ALTER SYSTEM SET ENABLE_GOLDENGATE_REPLICATION=TRUE;
# sql> ALTER SYSTEM SET STREAMS_POOL_SIZE=2G;
sql> ALTER DATABASE FORCE LOGGING;
sql> ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
sql> ARCHIVE LOG LIST;
# sql> CREATE TABLESPACE GG_DATA DATAFILE '/opt/oracle/oradata/ORCLCDB/gg_data.dbf' SIZE 100M AUTOEXTEND ON NEXT 100M;
sql> CREATE USER c##ggadmin IDENTIFIED BY confluent123 CONTAINER=ALL DEFAULT TABLESPACE USERS TEMPORARY TABLESPACE TEMP;
# Strong Password Policy is ENABLED. Password should contain at least: one lowercase [a..z] character, one uppercase [A..Z] character, one digit [0..9], and one special character [- ! @ % & * . # _]. Length should be between 8..30 characters.
sql> ALTER USER c##ggadmin IDENTIFIED BY "Confluent12!" CONTAINER=ALL;
sql> GRANT ALTER SYSTEM TO c##ggadmin CONTAINER=ALL;
sql> GRANT DBA TO c##ggadmin CONTAINER=ALL;
sql> GRANT CREATE SESSION TO c##ggadmin CONTAINER=ALL;
sql> GRANT ALTER ANY TABLE TO c##ggadmin CONTAINER=ALL;
sql> GRANT RESOURCE TO c##ggadmin CONTAINER=ALL;
sql> EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('c##ggadmin ',CONTAINER=>'ALL');
sql> ALTER SESSION SET CONTAINER=orclpdb1;
sql> CREATE USER ggadmin IDENTIFIED BY PASSWORD DEFAULT TABLESPACE USERS TEMPORARY TABLESPACE TEMP CONTAINER=CURRENT; 
# Strong Password Policy is ENABLED. Password should contain at least: one lowercase [a..z] character, one uppercase [A..Z] character, one digit [0..9], and one special character [- ! @ % & * . # _]. Length should be between 8..30 characters.
sql> ALTER USER ggadmin IDENTIFIED BY "Confluent12!" CONTAINER=CURRENT;
sql> GRANT CREATE SESSION TO ggadmin CONTAINER=CURRENT;
sql> GRANT ALTER ANY TABLE TO ggadmin CONTAINER=CURRENT;
sql> GRANT RESOURCE TO ggadmin CONTAINER=CURRENT;
sql> GRANT DBA TO ggadmin CONTAINER=CURRENT;
sql> EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('ggadmin');
sql> exit
```

Database is now running in AWS Compute. Noew, prepare for image copying.

```bash
ssh -i ~/keys/cmawskeyoraclegg.pem ec2-user@XXXX.eu-central-1.compute.amazonaws.com
# Stop container
docker container ls -a
docker ps
docker stop oracle19c
# image is still there
docker image ls
docker ps
```

Now save AMI image from compute service so that we are able to create a new compute from this image and everything is implemented.
* click on running compute ec2 instance
* Go to the AWS console
* go to your EC2 dashboard : in the instances list
* select your instance, right click on your instance
* select image and template / create image
* AMI ID: ami-XXXX
* AMI Name: oracleDB19cEE
* AMI description : Oracle 19.3c EE with all precononfigurtion setup for Confluent CDC
* Region: Frankfurt
* owner=cmuetzlitz@confluent.io
* divvy_owner=cmuetzlitz@confluent.io
* divvy_last_modified_by=cmuetzlitz@confluent.io
* content=OracleDB19.3cEE

See [guide for storing Compute as AMI image](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/tkv-create-ami-from-instance.html)

It takes a while still image is stored.

Now, you can shutdown your AWS Compute service, and can create compute services based on this image. 
I use Terraform for this, see [my Oracle19c DB EE sample](https://github.com/ora0600/confluent-new-cdc-connector/tree/main/oracle19c)

