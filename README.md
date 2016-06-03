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
