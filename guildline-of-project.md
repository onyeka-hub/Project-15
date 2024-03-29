Implementation guide 


1. Create a VPC
2. Create the subnets 
3. Create the Internet Gateway
4. Create route table for the public subnet to use  (Public Route Table)
5. Create a route in the public route table and point to the Internet gateway
6. Associate the public subnets to the created route table
7. Create a NAT Gateway so that servers in the private subnet can reach the internet to for example download stuff (Outbound)
8. Create route table for the private subnet to use (Private Route Table)
9. Create a route in the created route table and point to the NAT Gateway
10. Associate the private subnets (for compute only) to the private route table
11. Create security group for Bastion. Allow all DevOps engineers to connect over SSH to the Bastion server
12. Create security group and allow the entire world to talk to the ALB (HTTP/HTTS)
13. Create security group and allow the ALB to talk to the Nginx proxy server. (Nginx). Also allow Bastion to talk to nginx
14. Create an External facing Application Load Balancer (ALB)
15. Create a Listener (port 80) and target group
16. Update VPC settings to enable DNS Hostnames
17. Update public subnet settings to auto assign public IP address
18. Create Key Pair for SSH
19. Create a Launch Template for Bastion (Use a redhat based AMI) (include subnet settings for auto scaling in templates) 
20. Create ASG for Bastion
21. Connect to Bastion server launched in the Public Subnet over SSH
22. Create a Launch Template for nginx (Use a Ubuntu based AMI) (Private network)
22. Create ASG for nginx
23. Connect to the nginx server launched in the Private Subnet (Use SSH Agent to forward the public IP)

## SSH Config Sample
Host *
     StrictHostKeyChecking no

Host bastion
    ForwardAgent yes
    HostName 18.134.146.217
    Port 22
    User ec2-user
    IdentityFile ~/Downloads/temp-delete.cer

Host proxy
    ForwardAgent yes
    HostName 18.134.146.217
    Port 22
    User ec2-user
    IdentityFile ~/Downloads/temp-delete.cer
    ProxyCommand ssh bastion -W %h:%p

Host tooling
    ForwardAgent yes
    HostName 10.0.5.183
    Port 22
    User ec2-user
    IdentityFile ~/Downloads/temp-delete.cer
    ProxyCommand ssh bastion -W %h:%p


24. Ensure that the nginx Target group is healthy
    1.  Check the security of the instance and ensure it allows port 80
25. Connect to server and Install nginx
26. Create Route53 entry. Point the Domain name to the public ALB (CNAME Record)
27. Create security group for internal ALB. (Alow traffic from Nginx proxy)
28. Create Security group for Wordpress site (Allow traffic from internal ALB)
29. Create Security group for Tooling site (Allow traffic from internal ALB)
30. Create target group for Wordpress site
31. Create target group for tooling site
32. Create internal ALB and configure listeners with Host header rules for default web page, wordpress and tooling sites

33. Configure instance profile and give the tooling and wordpress instances relevant permissions to access AWS resources (for example, S3, EFS)
34. Create Launch Template for Tooling ASG (Ensure the IAM for instance profile is configured) (RedHat Linux)
35. Create Launch Template for Wordpress ASG (Ensure the IAM for instance profile is configured)  (RedHat Linux)
36. Create ASG for Tooling instances
37. Create ASG for Wordpress instances
38. Configure nginx to upstream to internal ALB

    # Configure Proxy
```
sudo unlink /etc/nginx/sites-enabled/default

sudo vi /etc/nginx/sites-available/tooling-reverse-proxy.conf
```

    ## Configure Tooling Reverse Proxy

```
server {
    listen 80;
    server_name tooling.dev.darey.io;
    location / {
        proxy_pass http://internal-test-288727612.eu-west-2.elb.amazonaws.com/;
        proxy_set_header Host $host;
    }
  }
```

```
sudo ln -s /etc/nginx/sites-available/tooling-reverse-proxy.conf /etc/nginx/sites-enabled/tooling-reverse-proxy.conf
```

    ## Configure Wordpress Reverse Proxy

```
sudo vi /etc/nginx/sites-available/wordpress-reverse-proxy.conf

server {
    listen 80;
    server_name wordpress.dev.darey.io;
    location / {
        proxy_pass http://internal-test-288727612.eu-west-2.elb.amazonaws.com/;
        proxy_set_header Host $host;
    }
  }
```

```
sudo ln -s /etc/nginx/sites-available/wordpress-reverse-proxy.conf /etc/nginx/sites-enabled/wordpress-reverse-proxy.conf 

sudo nginx -t

systemctl reload nginx
```

39. Configure Tooling and Wordpress (A simulation)

## Userdata for nginx
```
sudo yum update -y
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum install -y nginx git
sudo systemctl restart nginx
### Update the web page  
sudo vi /usr/share/nginx/html/index.html
```

40. Create Security Group for EFS - Allow access from Tooling and Wordpress on NFS port
41. Install **botocore** - https://docs.aws.amazon.com/efs/latest/ug/install-botocore.html

```
sudo yum update -y
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum -y install wget python3 git
```

```
if [[ "$(python3 -V 2>&1)" =~ ^(Python 3.6.*) ]]; then
    sudo wget https://bootstrap.pypa.io/pip/3.6/get-pip.py -O /tmp/get-pip.py
elif [[ "$(python3 -V 2>&1)" =~ ^(Python 3.5.*) ]]; then
    sudo wget https://bootstrap.pypa.io/pip/3.5/get-pip.py -O /tmp/get-pip.py
elif [[ "$(python3 -V 2>&1)" =~ ^(Python 3.4.*) ]]; then
    sudo wget https://bootstrap.pypa.io/pip/3.4/get-pip.py -O /tmp/get-pip.py
else
    sudo wget https://bootstrap.pypa.io/get-pip.py -O /tmp/get-pip.py
fi
```

```
sudo python3 /tmp/get-pip.py
pip install botocore


setsebool -P httpd_can_network_connect=1
setsebool -P httpd_can_network_connect_db=1
setsebool -P httpd_execmem=1
setsebool -P httpd_use_nfs 1


git clone https://github.com/aws/efs-utils
cd efs-utils
sudo yum install -y make
sudo yum install -y rpm-build
sudo make rpm 
sudo yum install -y  ./build/amazon-efs-utils*rpm

sudo mkdir /var/www
sudo mount -t efs -o tls fs-0dd171cec888d1c4a:/ /var/www
```



42.  Create EFS File system (SG)
    
43. Create Security Group for RDS
44. Create the KMS key for RDS data encryption
45. Create DB subnet group
46. Create RDS/Aurora Database


### Home Work

Relying on your recent experience working with Tooling website, NFS, MySQL Database and cloud concepts. Implement the entire project end to end deploying the real tooling application. Ensure that its configuration points to the RDS/Aurora Database you created.

-- Use any performance testing tool to stress the servers so that autoscaling can be triggered.
-- Implement more sub domain websites
-- Using your knowledge and experience working with lets encrypt to configure TLS/SSL. Learn how ACM works, and terminate SSL on the public load balancer
-- Ensure to configure /etc/fstab after mounting EFS