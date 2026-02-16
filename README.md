# aws-3tier-flask-rds-project
3tier aws project 

Welcome to my guide on 3-tier architecture using AWS. In this document, I’ll walk you through the detailed process I followed to design this architectural pattern. Before we get started, I want to thank you for visiting my page.

The 3-tier architecture is a widely adopted approach in software development, especially for web applications. It divides an application into three separate layers, each handling specific responsibilities

<img width="1536" height="1024" alt="ChatGPT Image Feb 16, 2026, 11_51_16 AM" src="https://github.com/user-attachments/assets/1c003e6b-1342-4d50-bece-3c124b8af35a" />

**1- VPC**

as you can see in this setup , there is a public subnet and a private subnet .. the public subnet will host the EC2 instanse responsible for communicating with the public internet . the private subnet will host the private EC2 instance which will host the App itself and it is private because we dont want the application tier to exposed to outer risks or traffic to be more secure .

finally there is a backup subnet which was needed as the RDS DB needed another subnet in differenet AZ for backup puposes .
the routing table was configured for each subnet . also at the end we can see internet gateway configured which is responsible for exposing the VPC to the internet . the vpc has CIDR 10.0.0.0/16

<img width="1911" height="784" alt="image - 2026-02-16T135343 575" src="https://github.com/user-attachments/assets/c56eac79-3ff1-449c-9074-c6b3c9080275" />



**2: Web Tier**

To begin building the internet-facing web tier, start by creating a security group for the EC2 instances you’ll launch. Allow SSH and HTTP  access. While this security group is somewhat permissive, it’s acceptable for the purposes of this lab. You can give it a name like web-sg


<img width="1745" height="640" alt="image" src="https://github.com/user-attachments/assets/6647abd1-bf7a-44e7-8483-6845882caaf0" />
<img width="1234" height="356" alt="image" src="https://github.com/user-attachments/assets/b32d8e6b-8a23-4aa7-a936-2a1ef8fdd8d4" />

now we go to the EC2 dashboard to create the EC2 for web tier

AMI: Amazon Linux 2023

Instance Type: t2.micro

Key Pair: Create New

Subnet: "" the public subnet created ""

Security Group: web-sg

VPC: The VPC from Step 1

and now , we need to follow the following command to build the nginx server

sudo dnf update -y
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx

Installing Nginx here means your EC2 instance can actually show web pages when someone visits your site.

Think of Nginx as the “front door” to your application—it receives visitors’ requests.

now once we access the public ip of the EC2 we can see the welcome nginx page .

<img width="1058" height="431" alt="image" src="https://github.com/user-attachments/assets/52b1e004-124c-4801-9830-5a3806f54971" />


**3- app tier**

Configuring the application tier is quite similar to the web tier, but there are some important differences. You’ll need to create a new security group, app-sg, which allows SSH access only from the web tier and opens port 5000 for the Flask application.
