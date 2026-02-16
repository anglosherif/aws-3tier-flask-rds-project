# aws-3tier-flask-rds-project
3tier aws project 

Welcome to my guide on 3-tier architecture using AWS. In this document, I’ll walk you through the detailed process I followed to design this architectural pattern. Before we get started, I want to thank you for visiting my page.

The 3-tier architecture is a widely adopted approach in software development, especially for web applications. It divides an application into three separate layers, each handling specific responsibilities

<img width="1536" height="1024" alt="ChatGPT Image Feb 16, 2026, 11_51_16 AM" src="https://github.com/user-attachments/assets/1c003e6b-1342-4d50-bece-3c124b8af35a" />

**1- VPC**

as you can see in this setup , there is a public subnet and a private subnet .. the public subnet will host the EC2 instanse responsible for communicating with the public internet . the private subnet will host the private EC2 instance which will host the App itself and it is private because we dont want the application tier to exposed to outer risks or traffic to be more secure .

finally there is a backup subnet which was needed as the RDS DB needed another subnet in differenet AZ for backup puposes .
the routing table was configured for each subnet . also at the end we can see internet gateway configured which is responsible for exposing the VPC to the internet . the vpc has CIDR 10.0.0.0/16

security groups is configured on each ec2 and on the DB layer 

Internet
   ↓
Public EC2 (app entry)
   ↓ (SSH / App traffic)
Private EC2 (Flask)
   ↓ (3306 MySQL)
RDS MySQL


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

Configuring the application tier is quite similar to the web tier, but there are some important differences. You’ll need to create a new security group, app-sg, which allows SSH access only from the web tier and opens TCP port 5000 for the Flask application.
and also allow writing to the RDS DB 
<img width="1573" height="492" alt="image" src="https://github.com/user-attachments/assets/1db7e90c-fae6-4bd9-9e36-4fee5c54a482" />
<img width="1508" height="402" alt="image" src="https://github.com/user-attachments/assets/4327c123-dfdc-4d49-be26-52f725c7eae5" />

but the issue is that how to configure all needed packages inside an ec2 instance that is not connected to internet ! , the solution was that i hade to install all needed packages inside the public ec2 and then transfer them to the private ec2 and finally i can do the install for the packages .

for example , here is s the steps to install mysql-connector-python on the private ec2 :-
1) on plublic ec2
   pip3 download mysql-connector-python -d mysql-offline
   scp -i ~/ec2-3tier.pem -r mysql-offline ec2-user@10.0.2.230:~
2) on private ec2
   mkdir mysql-offline
   cd mysql-offline
   pip3 install --no-index --find-links=. mysql-connector-python

this way is used to install any package needed in the private EC2 instance and this method is called Air-gapped installation

now lets see the application code 



from flask import Flask, jsonify
import mysql.connector
from mysql.connector import Error
app = Flask(__name__)
# ------------------------
# RDS Database Configuration
# ------------------------
DB_HOST = "database-3tier.c4hossiaq227.us-east-1.rds.amazonaws.com"
DB_USER = "admin"
DB_PASSWORD = "Password :D"
DB_NAME = "three_tier_app"
DB_PORT = 3306
# ------------------------
def get_connection(database=None):
    return mysql.connector.connect(
        host=DB_HOST,
        user=DB_USER,
        password=DB_PASSWORD,
        database=database,
        port=DB_PORT
    )

# ------------------------
# Initialize Database + Table
# ------------------------
@app.route("/init")
def initialize_database():
    try:
        connection = get_connection()
        cursor = connection.cursor()

        cursor.execute("CREATE DATABASE IF NOT EXISTS three_tier_app;")
        cursor.execute("USE three_tier_app;")

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(100),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
        """)

        return "✅ Database and table created successfully!"

    except Error as e:
        return f"❌ Error: {e}"
    finally:
        if 'connection' in locals() and connection.is_connected():
            cursor.close()
            connection.close()

# ------------------------
# Home Route
# ------------------------
@app.route("/")
def home():
    return "🚀 3-Tier App is Working from Private EC2!"

# ------------------------
# Add User
# ------------------------
@app.route("/add/<name>")
def add_user(name):
    try:
        connection = get_connection(DB_NAME)
        cursor = connection.cursor()

        cursor.execute("INSERT INTO users (name) VALUES (%s)", (name,))
        connection.commit()

        return f"✅ User '{name}' added successfully!"

    except Error as e:
        return f"❌ Error: {e}"
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()

# ------------------------
# List Users
# ------------------------
@app.route("/users")
def list_users():
    try:
        connection = get_connection(DB_NAME)
        cursor = connection.cursor()

        cursor.execute("SELECT * FROM users;")
        records = cursor.fetchall()

        users = []
        for row in records:
            users.append({
                "id": row[0],
                "name": row[1],
                "created_at": str(row[2])
            })

        return jsonify(users)

    except Error as e:
        return f"❌ Error: {e}"
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()

# ------------------------
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)



app.py file is the backend application of the 3-tier architecture. It uses Flask to create a small web server that runs on the private EC2 instance and listens on port 5000. When someone accesses the root URL (/), it simply returns a message confirming that the application server is running. When someone accesses the /db route, the application attempts to connect to the Amazon RDS MySQL database using the configured host, username, and password. If the connection is successful, it confirms that the app can communicate with the database; if not, it returns an error message. In short, app.py acts as the application layer between the user and the database, proving that networking, security groups, and database integration in the 3-tier architecture are working correctly.

the following pic shows how to run the app :-
<img width="1096" height="532" alt="image" src="https://github.com/user-attachments/assets/f5c2878e-5d89-4c38-87b4-d82bb6b4e6be" />


**4- DB tier**

while creating the DB , i faced an error , the rds DB refused to be created as the i have only 2 subnets and both are created in one az .. rds has to have a backup az in the setup .
so , i created another subnet and configured it in another az .
only then the RDS DB is now created .
the inbound security group is allowed only from the private ec2

<img width="1868" height="1011" alt="image - 2026-02-16T174013 171" src="https://github.com/user-attachments/assets/94f6b9f9-6c86-4036-a119-8b68e2b8af5e" />


