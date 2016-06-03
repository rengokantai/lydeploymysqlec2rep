#### lydeploymysqlec2rep
#####Building The VPC With Private And Public Subnets
configure:
create vpc. lymaria, 10.0.0.0/16  
create subnet,private,lymaria-pri1, us-east-1a,10.0.0.0/24  
create subnet,private,lymaria-pri2,us-east-1b,10.0.1.0/24  
create subnet,public,lymaria-pub1,us-east-1a,10.0.2.0/24  
create subnet,public,lymaria-pub2,us-east-1c,10.0.3.0/24  
create a igw,deselect vpc filter,attach to lymaria  

by default, route table only has private subnet.create a new RT.
add new RT, add route 0.0.0.0/0,target=igw.  

change two public subnets route table to public RT.
######Adding A Bastion Host And Configuring Security Groups
create a ec2, centos7, subnet=pub1,connect. name=bastion
```
ssh centos@ip
yum install -y firewalld
systemctl start firewalld
firewall-cmd --get-default-zone
firewall-cmd --list-all-zone
firewall-cmd --zone=trusted --add-source=pubip/32 --permanent
firewall-cmd --zone=trusted --add-port=22/tcp
firewall-cmd --reload
```
logout and log in to test, then
```
firewall-cmd --zone=public --list-all
firewall-cmd --zone=public --remove-service=ssh  --permanent
```
######Deploying The EC2 Instances And Testing
create a ec2, centos7, subnet=pri1,connect. name=backup, autoip=disabled, security group: add ssh/bastionip/32    
create s3, lymariaback,  
create a new policy, arn:aws:s3:::lymariaback  

create a ec2 centos7, subnet=pri2,connect. name=server, autoip=random, security group: add ssh/bastionip/32, mysql/all public subnet CIDR (copy 2) that is 10.0.2.0/24,10.0.3.0/24
create another ec2 centos7, subnet=pri1,connect. name=slaveserver, autoip=random, security group: add ssh/bastionip/32, mysql/all public subnet CIDR (copy 2)  that is 10.0.2.0/24,10.0.3.0/24
Add a rule: all tcp, source=sg name
```
until now:
bastion: 10.0.2.0/24  //pub1
backup: 10.0.1.0/24 //pri2
master db server:10.0.0.0/24 // pri1
slave db server : 10.0.1.0/24// pri2
```
connetion test
```
connect from bastion to backup.
connect from bastion to mster.
connect from bastion to slave.
```
######Adding A NAT Instance For The Private Subnets
create a ec2,communityAMI->nat, subnet=pub2,connect. autoip=enable,name=nat security group: ssh/=bastionip,and private subnet cidr that is: http/10.0.0.0/24 https/10.0.0.0/24 http/10.0.1.0/24 https/10.0.1.0/24 / then disable source check.  

configure vpc to allow pri subnet connect to internet. GO to RT, add rule 0.0.0.0 target=nat
```
```
connection test,from bastion to nat. in nat, test ping google.com
######Configuring Internal Route53 DNS
route53->create hosted zone->name:lymaria.internal,type=vpc internal. go to vpc enable dns,dns hostname  
then create record set.
```
backup.lymaria.internal, type=A, value = 10.0.0.1 (backup ip)
dbmaster.lymaria.internal, type=A, value = 10.0.0.1 master ip)
dbslave.lymaria.internal, type=A, value = 10.0.0.1 (slave ip)
prod.lymaria.internal, type=A, value = 10.0.0.1 master ip)
```
login to all three server,(except nat)
```
sudo yum install vim -y
```
then
```
vim /etc/resolv.conf
```
edit
```
search ... lymaria.internal
``` 
next time we can login by using
```
ssh -i my.pem backup.lymaria.internal
```
in bastion,
```
sudo hostnamectl set-hostname bastion
```
in other three, set
```
sudo hostnamectl set-hostname backup.lymaria.internal
sudo hostnamectl set-hostname dbmaster.lymaria.internal
sudo hostnamectl set-hostname dbslave.lymaria.internal
```
#####Configuring MariaDB/MySQL
######Configuring The Master Server For Replication
in master db server
```
sudo yum install -y mariadb-server
sudo systemctl start mariadb
vim /etc/my.cnf
```
config
```
# instructions in http://fedoraproject.org/wiki/Systemd
log-basename=master
log-bin
binlog-format=row
server_id=1

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid
```
then
```
sudo systemctl restart mariadb
```
then
```
sudo mysql
create user 'test'@'%' identified by 'pass';
grant REPLICATION SLAVE ON *.* TO test;
```
######Configuring The Slave Server For Replication
```
vim /etc/my.cnf
```
only change one line
```
server_id=2

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid
```
from slave, connect to master
```
mysql -h dbmaster.lymaria.internal -u test -p
exit;
```
then local connection
```
sudo mysql
change master to MASTER_HOST='dbmaster.lymaria.internal',MASTER_USER='test',MASTER_PASSWORD='pass',MASTER_PORT=3306,MASTER_CONNECT_RETRY=10;
```
then
```
start slave;
show slave status;
```
in master server:
```
sudo mysql
select user from mysql.user;
create database ke;  //should also see in slave server
```
#####Backing Up The Database Servers
######Overview Of Different Backup Methods
schedule backup on non peek hours, backup on slave lock the table before backup, unlock after backup
###### Mysqldump to S3
install pip for centos. in backup server
```
sudo yum-config-manager --add-repo http://dl.fedoraproject.org/pub/epel/7/x86_64/  (apple repo)
sudo yum repolist
cd /etc/yum.repos.d
vim dl.fedo...
```
edit
```
gpgcheck=0
```
then
```
sudo yum install -y python-pip
sudo pip install awscli
```
copy ip addr of backup server, then add a security rule to dbmaster, MYSQL->source=backup ip,then insall mariasb client
```
sudo yum install mariadb
```
in masterdb, create a user for backup
```
sudo mysql
create user 'backup'@'%' identified by 'pass';
grant select,lock tables, show view,event,trigger,reload,super on *.* to 'backup'@'%';
```
then, on backup server, we connect to masterdb
```
mysql -u backup -h dbmaster.lymaria.internal -p
exit;
```
then try dumping
```
mysqldump -h  dbmaster.lymaria.internal -ubackup -ppass --all-databases >backup.sql
```
create a sync script
```
sudo mysqldump -h  dbmaster.lymaria.internal -ubackup -ppass --all-databases >backup.sql
sudo aws s3 sync ~ s3://lymariaback
```

lock before dump. Use slave server!
```
sudo mysqldump -h  dbslave.lymaria.internal -ubackup -ppass --lock-tables --all-databases >backup.sql
```
######Creating The EBS Snapshots
backupserver:
```
sudo yum install -y jq
```
create a backup script
```
vim snapshot
```

original snap script from nugget
```
#! /bin/bash
ACTION=$1
AGE=$2

if [ -z $ACTION ];
Running transaction test
then
        echo "Usage $1: define action backup or delete"
        exit 1
fi

if [ "$ACTION" = "delete" ] && [ -z $AGE ];
then
        echo "enter the age of backup to delete"
        exit 1
fi
                     {
function backup_ebs(){
        for volume in $(sudo aws ec2 describe-volumes | sudo jq .Volumes[].Volum
eId | sed 's/\"//g')
        do
                echo Creating snapshot for $volume $(sudo aws ec2 create-snapsho
t --volume-id $volume --description "backscript")
}       done
}
function delete_snapshots(){
        for snapshot in $(sudo aws ec2 describe-snapshots --filters Name=descrip
tion,Values=backscript |jq .Snapshots[].SnapshotId|sed 's/\"//g')
        do
                SNAPSHOTDATE=$(sudo aws ec2 describe-snapshots --filters Name=sn
apshot-id,Values=$snapshot | jq .Snapshots[].StartTime |cut -d T -f1 |sed 's/\"/
/g')
                STARTDATE=$(date +%s)
                ENDDATE=$(date -d $SNAPSHOTDATE +%s)
                INTERVAL=$[ (STARTDATE - ENDDATE) / (60*60*24) ]
                if (( $INTERVAL >= $AGE ));
                then
                        sudo aws ec2 delete-snapshot --snapshot-id $snapshot
                fi
        done
}

case $ACTION in
        "backup")
                backup_ebs
        ;;
        "delete")
                delete_snapshots
        ;;
esac
```
select slavedb, assign a tag, key=slave, value is empty.check
```
sudo aws ec2 describe-instances --filters "Name=tag-key ,Values=slave"
sudo aws ec2 describe-instances --filters "Name=tag-key ,Values=slave" |jq .Reservations[].Instances[].InstanceId | sed 's/\"//g'
```
create a backup script in backup server
```
#! /bin/bash
coproc mysql {
sudo mysql -hdbslave.lymaria.internal -ubackup -ppass
}

echo 'flush tables with read lock;' >&"${mysql[1]}"
echo 'set global read_only = ON;' >&"${mysql[1]}"
ssh -i mykey.pem centos@dbslave.lymaria.internal xfs_freeze -f /
./snapshot backup
ssh -i mykey.pem centos@dbslave.lymaria.internal xfs_freeze -u /
echo 'set global read_only = OFF;' >&"${mysql[1]}"
echo 'unlock tables;' >&"${mysql[1]}"
```
create .my.cnf in backup server (current dict)
```
[client]
password="pass"
```
generate key on backupserver
```
ssh-keygen
```
then
```
cd .ssh
cat id_rsa.pub (copy this result to slave server's authorized_keys)
```
script snapshot:
```
#! /bin/bash
ACTION=$1
AGE=$2

if [ -z $ACTION ];
then
        echo "Usage $1: define action backup or delete"
        exit 1
fi

if [ "$ACTION" = "delete" ] && [ -z $AGE ];
then
        echo "enter the age of backup to delete"
        exit 1
fi

function backup_ebs(){
        for instance in $(sudo aws ec2 describe-instances --filters "Name=tag-ke
y ,Values=slave" |jq .Reservations[].Instances[].InstanceId | sed 's/\"//g')
        do


        for volume in $(sudo aws ec2 describe-volumes --filters "Name=attachment
.instance-id,Values=$instance"| jq .Volumes[].VolumeId | sed 's/\"//g')
        do
                echo Creating snapshot for $volume $(sudo aws ec2 create-snapsho
t --volume-id $volume --description "backscript")
        done
done
}

function delete_snapshots(){
        for snapshot in $(sudo aws ec2 describe-snapshots --filters Name=descrip
tion,Values=backscript |jq .Snapshots[].SnapshotId|sed 's/\"//g')
        do
                SNAPSHOTDATE=$(sudo aws ec2 describe-snapshots --filters Name=sn
apshot-id,Values=$snapshot | jq .Snapshots[].StartTime |cut -d T -f1 |sed 's/\"/
/g')
                STARTDATE=$(date +%s)
                ENDDATE=$(date -d $SNAPSHOTDATE +%s)
                INTERVAL=$[ (STARTDATE - ENDDATE) / (60*60*24) ]
                if (( $INTERVAL >= $AGE ));
                then
                        sudo aws ec2 delete-snapshot --snapshot-id $snapshot
                fi
        done
}


case $ACTION in
        "backup")
                backup_ebs
        ;;
        "delete")
                delete_snapshots
        ;;
esac
```
