#### lydeploymysqlec2rep
#####1
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
######2
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
######3
create a ec2, centos7, subnet=pri1,connect. name=backup, autoip=disabled, security group: add ssh/bastionip/32    
create s3, lymariaback,  
create a new policy, arn:aws:s3:::lymariaback  

create a ec2 centos7, subnet=pri2,connect. name=server, autoip=random, security group: add ssh/bastionip/32, mysql/all public subnet CIDR (copy 2) 
create another ec2 centos7, subnet=pri1,connect. name=slaveserver, autoip=random, security group: add ssh/bastionip/32, mysql/all public subnet CIDR (copy 2)  
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
