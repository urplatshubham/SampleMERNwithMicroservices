![Screenshot (803)](https://github.com/user-attachments/assets/bf118f48-80e9-4bde-a896-40a252005e39)# Deploying a Dockerized MERN Backend on AWS EC2

This guide outlines the steps to deploy a Dockerized backend service from a MERN stack application using Amazon ECR and EC2, along with all steps from setting up the environment to deployment. Visual examples are provided to ensure clarity.

---

## **Prerequisites**

- AWS CLI installed and configured with access keys and the correct region.
- Docker installed locally.
- MERN backend application Dockerized and pushed to Amazon ECR.
- Key pair for EC2 instance access.

---

## **Step 1: Set Up the AWS Environment**

1. **Install AWS CLI and Configure Credentials**:

   - Install the AWS CLI by following the instructions at [AWS CLI Installation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).
   - Configure it with your access and secret keys:
     ```bash
     aws configure
     ```

2. **Install Boto3 for Python**:

   - Install Boto3 using pip:
     ```bash
     pip install boto3
     ```

---

## **Step 2: Prepare the MERN Application**

### **2.1 Dockerize the Backend Application**

- Create a `Dockerfile` in the backend directory:

  ```dockerfile
  FROM node:16
  WORKDIR /usr/src/app
  COPY package*.json ./
  RUN npm install
  COPY . .
  EXPOSE 5000
  CMD ["npm", "start"]
  ```

- Build the Docker image:

  ```bash
  docker build -t mern-backend .
  ```

![Screenshot (803)](https://github.com/user-attachments/assets/585620e2-a55b-4304-bbba-2aaa64cf4298)

### **2.2 Dockerize the Frontend Application**

- Create a `Dockerfile` in the frontend directory:

  ```dockerfile
  FROM node:16
  WORKDIR /usr/src/app
  COPY package*.json ./
  RUN npm install
  COPY . .
  RUN npm run build
  FROM nginx:alpine
  COPY --from=0 /usr/src/app/build /usr/share/nginx/html
  EXPOSE 80
  CMD ["nginx", "-g", "daemon off;"]
  ```

- Build the Docker image:

  ```bash
  docker build -t mern-frontend .
  ```

### **2.3 Push Docker Images to Amazon ECR**

1. **Authenticate Docker with ECR**:

   ```bash
   aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
   ```

2. **Create repositories for the images**:

   ```bash
   aws ecr create-repository --repository-name mern-backend
   aws ecr create-repository --repository-name mern-frontend
   ```

3. **Tag and push the images**:

   ```bash
   docker tag mern-backend:latest <account-id>.dkr.ecr.<region>.amazonaws.com/mern-backend:latest
   docker push <account-id>.dkr.ecr.<region>.amazonaws.com/mern-backend:latest
   ```

   ![Screenshot (803)](https://github.com/user-attachments/assets/6d4a9a9f-16ba-462d-a73b-a83e27ea0f38)


---

## **Step 3: Deploy Backend Using Docker on EC2**

1. **Launch an EC2 Instance**:

   - Use the AWS Management Console to launch an instance with Ubuntu 20.04 LTS.
   - Configure security groups to allow inbound traffic on ports `5000` (backend) and `80` (frontend).

2. **Connect to the Instance**:

   - SSH into the instance:
     ```bash
     ssh -i <key-file.pem> ubuntu@<public-ip>
     ```

3. **Install Docker on EC2**:

   - Update the system and install Docker:
     ```bash
     sudo apt update -y && sudo apt upgrade -y
     sudo apt install -y docker.io
     sudo systemctl start docker
     sudo systemctl enable docker
     ```

4. **Pull and Run the Backend Container**:

   - Authenticate Docker with ECR:
     ```bash
     aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
     ```
   - Pull and run the container:
     ```bash
     docker pull <account-id>.dkr.ecr.<region>.amazonaws.com/mern-backend:latest
     docker run -d -p 5000:5000 <account-id>.dkr.ecr.<region>.amazonaws.com/mern-backend:latest
     ```

---

## **Step 4: Continuous Integration with Jenkins**

1. **Set Up Jenkins**:

   - Install Jenkins on an EC2 instance.
   - Configure Jenkins with necessary plugins.

2. **Create Jenkins Jobs**:

   - Create Jenkins jobs for building and pushing Docker images to ECR.
   - Trigger the Jenkins jobs whenever there's a new commit in the GitHub repository.

---

## **Step 5: Infrastructure as Code (IaC) with Boto3**

1. **Define Infrastructure with Boto3 (Python Script)**:

   ```python
   import boto3

   ec2 = boto3.resource('ec2')

   # Create VPC
   vpc = ec2.create_vpc(CidrBlock='10.0.0.0/16')
   vpc.create_tags(Tags=[{"Key": "Name", "Value": "my-vpc"}])
   vpc.wait_until_available()

   # Create Subnet
   subnet = vpc.create_subnet(CidrBlock='10.0.1.0/24')

   # Create Internet Gateway
   igw = ec2.create_internet_gateway()
   vpc.attach_internet_gateway(InternetGatewayId=igw.id)

   # Create Route Table
   route_table = vpc.create_route_table()
   route_table.create_route(
       DestinationCidrBlock='0.0.0.0/0',
       GatewayId=igw.id
   )
   route_table.associate_with_subnet(SubnetId=subnet.id)

   # Create Security Group
   security_group = ec2.create_security_group(
       GroupName='my-security-group',
       Description='Allow SSH and HTTP',
       VpcId=vpc.id
   )
   security_group.authorize_ingress(
       IpPermissions=[
           {
               'IpProtocol': 'tcp',
               'FromPort': 22,
               'ToPort': 22,
               'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
           },
           {
               'IpProtocol': 'tcp',
               'FromPort': 80,
               'ToPort': 80,
               'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
           }
       ]
   )

   # Launch EC2 Instance
   instances = ec2.create_instances(
       ImageId='ami-0abcdef1234567890',  # Replace with a valid AMI ID
       MinCount=1,
       MaxCount=1,
       InstanceType='t2.micro',
       KeyName='your-key-name',
       SecurityGroupIds=[security_group.group_id],
       SubnetId=subnet.id
   )

   print("EC2 Instance created with ID:", instances[0].id)
   ```

---

## **Step 6: Deploying Backend Services**

1. **Deploy Backend on EC2 with ASG**:

   - Use Boto3 to deploy EC2 instances with the Dockerized backend application in the ASG.

---

## **Step 7: Set Up Networking**

1. **Create Load Balancer**:

   - Set up an Elastic Load Balancer (ELB) for the backend ASG.

2. **Configure DNS**:

   - Set up DNS using Route 53 or any other DNS service.

---

## **Step 8: Deploying Frontend Services**

1. **Deploy Frontend on EC2**:

   - Use Boto3 to deploy EC2 instances with the Dockerized frontend application.

---

## **Step 9: AWS Lambda Deployment**

1. **Create Lambda Functions**:

   ```python
   import boto3
   import datetime

   # Initialize the boto3 client
   lambda_client = boto3.client('lambda')
   s3_client = boto3.client('s3')

   # Define Lambda Function to Backup Database
   def create_lambda_function():
       response = lambda_client.create_function(
           FunctionName='db-backup-lambda',
           Runtime='python3.8',
           Role='arn:aws:iam::your-account-id:role/your-lambda-role',
           Handler='lambda_function.lambda_handler',
           Code={
               'ZipFile': open('lambda_function.zip', 'rb').read()
           },
           Description='Backup database to S3',
           Timeout=15,
           MemorySize=128
       )
       print("Lambda Function created:", response['FunctionArn'])

   def lambda_handler(event, context):
       bucket_name = 'my-db-backup-bucket'
       timestamp = datetime.datetime.now().strftime("%Y-%m-%d-%H-%M-%S")
       backup_file_name = f"db-backup-{timestamp}.sql"

       # Simulate backup logic
       s3_client.put_object(Bucket=bucket_name, Key=backup_file_name, Body="Database backup content")
       print(f"Backup {backup_file_name} saved to S3 bucket {bucket_name}")

   create_lambda_function()
   ```

   - The Lambda function automatically backs up the database to S3 with a timestamp.

---

## **Step 10: Kubernetes (EKS) Deployment**

1. **Create EKS Cluster**:

   - Use `eksctl` or other tools to create an Amazon EKS cluster.

2. **Deploy Application with Helm**:

   - Use Helm to package and deploy the MERN application on EKS.

---

## **Step 11: Monitoring and Logging**

1. **Set Up Monitoring**:

   - Use CloudWatch for monitoring and setting up alarms.

2. **Configure Logging**:

   - Use CloudWatch Logs or another logging solution for collecting logs.

---

This completes the deployment of a Dockerized MERN application on AWS EC2, including backend and frontend services.

