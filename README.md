# AWS Three-Tier Architecture Project

## Overview

This project demonstrates a three-tier architecture on AWS, designed to ensure high availability, scalability, and reliability. The architecture consists of the following tiers:

1. **Web Tier**: Handles client requests and serves the front-end website.
2. **Application Tier**: Manages business logic and processes API requests.
3. **Database Tier**: Manages data storage and retrieval.

### Components

1. **External Load Balancer (Public-Facing ALB)**
   - **Role**: Entry point for all client traffic.
   - **Functionality**:
     - Distributes incoming client requests to web tier EC2 instances.
     - Ensures even distribution of traffic for better performance and reliability.
     - Performs health checks to ensure only healthy instances receive traffic.

2. **Web Tier**
   - **Role**: Serves the front-end of the application and redirects API calls.
   - **Components**:
     - **Nginx Web Servers**: Running on EC2 instances.
     - **React.js Website**: The front-end application served by Nginx.
   - **Functionality**:
     - Serving the Website: Nginx serves the static files for the React.js application to the clients.
     - Redirecting API Calls: Nginx is configured to route API requests to the internal-facing load balancer of the application tier.

3. **Internal Load Balancer (Application Tier ALB)**
   - **Role**: Manages traffic between the web tier and the application tier.
   - **Functionality**:
     - Receives API requests from the web tier.
     - Distributes these requests to the appropriate EC2 instances in the application tier.
     - Ensures high availability and load balancing within the application tier.

4. **Application Tier**
   - **Role**: Handles the application logic and processes API requests.
   - **Components**:
     - **Node.js Application**: Running on EC2 instances.
   - **Functionality**:
     - Processing Requests: The Node.js application receives API requests, performs necessary computations or data manipulations.
     - Database Interaction: Interacts with the Aurora MySQL database to fetch or update data.
     - Returning Responses: Sends the processed data back to the web tier via the internal load balancer.

5. **Database Tier (Aurora MySQL Multi-AZ Database)**
   - **Role**: Provides reliable and scalable data storage.
   - **Functionality**:
     - Data Storage: Stores all the application data in a structured format.
     - Multi-AZ Setup: Ensures high availability and fault tolerance by replicating data across multiple availability zones.
     - Data Retrieval and Manipulation: Handles queries and transactions from the application tier to manage the data.

### Additional Components

- **Load Balancing**
  - **Purpose**: Distributes incoming traffic evenly across multiple instances to prevent bottlenecks.
  - **Implementation**:
    - **Web Tier**: The external load balancer distributes traffic to web servers.
    - **Application Tier**: The internal load balancer distributes API requests to application servers.

- **Health Checks**
  - **Purpose**: Continuously monitors the health of instances to ensure only healthy instances receive traffic.
  - **Implementation**:
    - **Web Tier**: Health checks by the external load balancer to ensure web servers are responsive.
    - **Application Tier**: Health checks by the internal load balancer to ensure application servers are operational.

- **Auto Scaling Groups**
  - **Purpose**: Automatically adjusts the number of running instances based on traffic load.
  - **Implementation**:
    - **Web Tier**: Auto-scaling based on metrics like CPU usage or request count to add or remove web server instances.
    - **Application Tier**: Auto-scaling based on similar metrics to adjust the number of application server instances.

- **AWS Certificate Manager (ACM)**
  - **Purpose**: Manages SSL/TLS certificates to secure data in transit.
  - **Implementation**:
    - **Certificate Provisioning**: ACM provides and manages SSL/TLS certificates.
    - **Certificate Deployment**: ACM certificates are associated with the public-facing Application Load Balancer (ALB) to enable HTTPS traffic.
    - **Automatic Renewal**: ACM automatically renews certificates before they expire.

- **Amazon Route 53**
  - **Purpose**: Manages DNS records and directs user traffic to AWS resources.
  - **Implementation**:
    - **DNS Management**: Route 53 handles DNS queries for your domain, translating them into IP addresses for your ALB.
    - **Traffic Routing**: Route 53 directs client requests to the public-facing Application Load Balancer based on DNS records.
    - **Health Checks and Failover**: Optionally performs health checks on your endpoints and can reroute traffic to healthy resources if needed.

## Project Setup Instructions

### Clone the Git Repository

```bash
git clone https://github.com/vaibhav/aws_3tier_architecture.git
```

### Application Server Setup

1. **Launch an EC2 Instance** in the APP subnet of the Custom VPC.

2. **Install MySQL** on the app server:

   ```bash
   sudo yum install mysql -y
   ```

3. **Configure MySQL Database**:

   Connect to the database:

   ```bash
   mysql -h mydb.cfpgnjehw330.ap-south-1.rds.amazonaws.com -u admin -p
   ```

   Execute the following commands:

   ```sql
   CREATE DATABASE webappdb;
   SHOW DATABASES;
   USE webappdb;

   CREATE TABLE IF NOT EXISTS transactions (
     id INT NOT NULL AUTO_INCREMENT, 
     amount DECIMAL(10,2), 
     description VARCHAR(100), 
     PRIMARY KEY(id)
   );

   SHOW TABLES;
   INSERT INTO transactions (amount, description) VALUES ('400', 'groceries');
   SELECT * FROM transactions;
   ```

4. **Update Application Configuration** with DB information:

   Update `application-code/app-tier/DbConfig.js` with your database credentials.

5. **Install and Configure Node.js and PM2**:

   ```bash
   curl -o- https://raw.githubusercontent.com/vaibhav/aws_3tier_architecture/main/install.sh | bash
   source ~/.bashrc

   nvm install 16
   nvm use 16

   npm install -g pm2
   ```

   Download application code from S3 and start the application:

   ```bash
   cd ~/
   aws s3 cp s3://3tierproject-vaibhav/application-code/app-tier/ app-tier --recursive

   cd ~/app-tier
   npm install
   pm2 start index.js

   pm2 list
   pm2 logs
   pm2 startup
   pm2 save
   ```

   Verify that the application is running:

   ```bash
   curl http://localhost:4000/health
   ```

   It should return: `This is the health check`.

### Internal Load Balancer

1. **Create an internal load balancer** and update the Nginx configuration with the internal load balancer IP:

   ```text
   internal-app-alb-574972862.ap-south-1.elb.amazonaws.com
   ```

### Web Tier Setup

1. **Launch EC2 Instance** in Web Subnets of the Custom VPC.

2. **Web Tier Installation**:

   Install Node.js and Nginx on the web tier:

   ```bash
   curl -o- https://raw.githubusercontent.com/vaibhav/aws_3tier_architecture/main/install.sh | bash
   source ~/.bashrc
   nvm install 16
   nvm use 16

   cd ~/
   aws s3 cp s3://3tierproject-vaibhav/application-code/web-tier/ web-tier --recursive

   cd ~/web-tier
   npm install
   npm run build

   sudo amazon-linux-extras install nginx1 -y
   ```

   Update Nginx configuration:

   ```bash
   cd /etc/nginx
   ls

   sudo rm nginx.conf
   sudo aws s3 cp s3://3tierproject-vaibhav/application-code/nginx.conf .

   sudo service nginx restart

   chmod -R 755 /home/ec2-user

   sudo chkconfig nginx on
   ```

### SSL/TLS Certificate and Domain Configuration

1. **Generate an ACM Certificate** and add an A record with a CNAME record to access the application securely with "https" protocol.

For further guidance on AWS and DevOps, you can contact me on Linkedin: https://www.linkedin.com/in/vaibhav-cloud/

Feel free to reach out with any questions or issues regarding this project.
