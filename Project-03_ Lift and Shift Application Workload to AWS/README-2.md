# 3. AWS Cloud for Web App Setup [Lift & Shift]
## This project 
- Multi Tier web application stack
- Host and run on aws cloud
## Scenario
- Application Services running on Physical/ Virtual machines
- Work load in your Datacenter
- Have a lots of team
##Problem
- Complex Management
- Scale UP/down complexity
- UpFront CapEx & Regular OpEx
- Manual Process
- Difficult to automate
- Time consuming
## Solution 
- Automation
- IAAS
- PayAsUgo
- Flexiblity
- Ease of Infra Management
## AWS Services
- EC2 Instances : VM for Tomcat , RabbitMQ, Memcache, MySQL
- ELB : Nginx load balancer replacement
- Autoscaling : Automation for VM scaling
- S3/EFS Storage : Shared storage
- Route53 : private DNS service
## Qbjective
- Flexible infra
- No UpFront cost
- Modernize Effectively
- IAAC 

##  Step by step 
1. Security Group & Key Pairs
- Need a Domain 
- Create SG  
 + vprofile-ELB-SG : 80|0.0.0.0/0 , 80|::/0, 443|::/0, 443|0.0.0.0/0
 + vprofile-app-SG : 8080|vprofile-ELB-SG , 
 + vprofile-backen-SG : 3306|vprofile-app-SG, 11211|vprofile-app-SG , 5672|vprofile-app-SG, all trafic|vprofile-backen-SG 
- Create keypair : vprofile-key 
2. Ec2Instaces 
- Clone repository (have bash script to use data user)
- Create EC2 
- Type : Centos 9 stream / t2.micro
	+ Name : vprofile-db01, Project : vprofile , SG : vprofile-backen-SG, user data : mysql.sh
	+ Name : vprofile-mc01, Project : vprofile , SG : vprofile-backen-SG, user data : memcache.sh
	+ Name : vprofile-mq01, Project : vprofile , SG : vprofile-backen-SG, user data : rabbitmq.sh
- Type : Ubuntu 22.03 / t2.micro
	+ Name : vprofile-app01, Project : vprofile , SG : vprofile-app-SG, user data : tomcat_ubuntu.sh
- Check everything
```
# vprofile-db01
mysql -u admin -padmin123 accounts
show tables
quit

# vprofile-mc01
sudo -i 
ss -tunlp | grep 11211
# vprofile-mq01
systemctl status rabbitmq-server
# vprofile-app01
sudo -i
systemctl status tomcat9
ls /var/lib/tomcat9/
ls /var/lib/tomcat9/webapps/
```
- Route53
	+ Create hosted zone
	+ vprofile.in
	+ Private hosted zone
	+ Region : US East - 1 
		+ Simple routing 
		+ Define simple record : db01
		+ copy ip vprofile-db01 -> endpoint 
		
		+ Simple routing 
		+ Define simple record : mc01
		+ copy ip vprofile-mc01 -> endpoint 
		
		+ Simple routing 
		+ Define simple record : mq01
		+ copy ip vprofile-mc01 -> endpoint 

3. Build And Deploy .Artifacts
- Change code and build
- application.properties
`//db01.vprofile.in`
`//mc01.vprofile.in`
`//mq01.vprofile.in`
- Test and build
```
# Change visual code to bash 
# View : select default profile -> git bash {choose bash}
cd vprofile-project
mvn -version
mvn install 
# Then have target file 
```
- Create user : s3admin : S3Full...
- Create access key 
- Next we will configure our `aws cli` to use iam user credentials.
```sh
aws configure
AccessKeyID: 
SecretAccessKey:
region: us-east-1
format: json
```

- Create bucket. Note: S3 buckets are global so the naming must be UNIQUE!
```
aws s3 mb s3://vprofile-artifact-storage-rd 
```
- Go to target directory and copy the artifact to bucket with below command. Then verify by listing objects in the bucket.
```
aws s3 cp vprofile-v2.war s3://vprofile-artifact-storage-rd
aws s3 ls vprofile-artifact-storage-rd
```
- We can verify the same from AWS Console.
- In order to download our artifact onto Tomcat server, we need to create IAM role for Tomcat. Once role is created we will attach it to our `app01` server.
```sh
Type: EC2
Name: vprofile-artifact-storage-role
Policy: s3FullAccess
```
- Before we login to our server, we need to add SSH access on port 22 to our `vprofile-app-SG`.

- Then connect to `app011` Ubuntu server.
```sh
ssh -i "vprofile-prod-key.pem" ubuntu@<public_ip_of_server>
sudo su -
systemctl status tomcat9
```

- We will delete `ROOT` (where default tomcat app files stored) directory under `/var/lib/tomcat8/webapps/`. Before deleting it we need to stop Tomcat server. 
```sh
cd /var/lib/tomcat9/webapps/
systemctl stop tomcat9
rm -rf ROOT
```
- Next we will download our artifact from s3 using aws cli commands. First we need to install `aws cli`. We will initially download our artifact to `/tmp` directory, then we will copy it under `/var/lib/tomcat8/webapps/` directory as `ROOT.war`. Since this is the default app directory, Tomcat will extract the compressed file.
```sh
apt install awscli -y
aws s3 ls s3://vprofile-artifact-storage-rd
aws s3 cp s3://vprofile-artifact-storage-rd/vprofile-v2.war /tmp/vprofile-v2.war
cd /tmp
cp vprofile-v2.war /var/lib/tomcat8/webapps/ROOT.war
systemctl start tomcat9
```

- We can also verify `application.properties` file has the latest changes.
```sh
cat /var/lib/tomcat9/webapps/ROOT/WEB-INF/classes/application.properties
```

- We can validate network connectivity from server using `telnet`.
```sh
apt install telnet
telnet db01.vprofile.in 3306
```
4. Load Balancer & DNS
- Create target group 
	+ Port :8080
	+ Heath check : /login
	
	+ Override : 8080 , threshold : 3 , ...
	+ Tick app 
	+ Include as pending below
- Create Application Load Balancer 

```sh
vprofile-prod-elb
Internet Facing
Select all AZs
SecGrp: vprofile-elb-secGrp
Listeners: HTTP, HTTPS
Select the certificate for HTTPS : goophy.in 
Go to domain.web to create CNAME : with url have just created
-> Then check vprofile.goophy.in on browser

```
5. Autoscaling Group

- Create image of vprofile-app01 
- Launch Autoscaling configure
	+ vprofile-app-LC
	+ AMI : 
	+ choose instance type : t2.micro
	+ IAM
	+ Enable monitoring 
	+ SG 
	+ Keypair 
- Creat Autoscaling Group
	+ Enable Application Load Balancer
	+ Tick health check
6. Validate & Summarize
