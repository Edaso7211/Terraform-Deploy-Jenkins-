# Terraform-Deploy-Jenkins:
Deploy an EC2 instance bootstrapped with a script to install and run Jenkins.
Project Scenario:
Your team would like to start using Jenkins as their CI/CD tool to create pipelines for DevOps projects. They need you to create the Jenkins server using Terraform so that it can be used in other environments and so that changes to the environment are better tracked.
Project Objective:
Deploy 1 EC2 Instance in your Default VPC.
Bootstrap the EC2 instance with a script that will install and start Jenkins. Review the official Jenkins Documentation for more information: Linux
Create and assign a Security Group to the Jenkins EC2 that allows traffic on port 22 and allows traffic from port 8080.
Create a S3 bucket for your Jenkins Artifacts that is not open to the public.
Verify that you can reach your Jenkins install via port 8080 in your browser.
Project Pre-Requisites
Basic knowledge of Terraform commands.
Terraform installed within your CLI.
AWS CLI configured in your IDE to allow Terraform access to resources via your account.
General knowledge of cloud computing services including virtual machines, images, security groups, & virtual networks.
Step One: Initializing Terraform
Login to your CLI/IDE I am using AWS Cloud9 and create a new directory for your project.
mkdir tf-project-jenkins
cd tf-project-jenkins
https://miro.medium.com/v2/resize:fit:4800/format:webp/1*jHxazlmbKFWd2akgf6KS8A.png![image](https://github.com/Edaso7211/Terraform-Deploy-Jenkins-/assets/126200469/43b11187-f487-4d6d-be32-9d5a16373c67)
Next we want to pass through our AWS credentials into Terraform, we could do this by writing our AWS credentials into our terraform code, however, it is not best practise to hardcode your credentials into Terraform scripts. Alternatively we can pass through “Environment Variables”. This allows us to bypass the need to write a “providers block” which defines which cloud-based service to deploy into.
Run the following code and substitute you personal details as necessary:
export AWS_ACCESS_KEY_ID=<id>
export AWS_SECRET_ACCESS_KEY=<id>
export AWS_REGION=<region>
  Step Two: Create Terraform Files
Create Variables.tf file
We’ll start by defining the variables to use in our configuration. Save the following in a variables.tf file.
  
  variable "aws_region" {
  description = "regions my resources will be deployed"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "ami" {
  type    = string
  default = "ami-04581fbf744a7d11f"
}

variable "key_name" {
  type    = string
  default = "myec2key"
}

variable "s3bucket" {
  type    = string
  default = "jenkins-bucket-31032023369432"
}

variable "acl" {
  type    = string
  default = "private"
}

variable "vpc_id" {
  type    = string
  default = "vpc-084f2b2f58db22b55"
}
  Create main.tf file
  
  # Configure the AWS Provider
provider "aws" {
  region = var.aws_region
}

#Create EC2 instance in default VPC and bootstrap instance to start and install Jenkins
resource "aws_instance" "jenkins-instance" {
  ami                    = var.ami
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.jenkins-sg.id]
  user_data              = <<-EOF
                #!/bin/bash
                sudo yum update –y 
                sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo 
                sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
                sudo yum upgrade
                sudo amazon-linux-extras install java-openjdk11 -y 
                sudo yum install jenkins -y
                sudo systemctl enable jenkins
                sudo systemctl start jenkins
                EOF

  user_data_replace_on_change = true
  tags = {
    Name = "jenkins-EC2"
  }
}

#Create and assign a security group to Jenkins EC2 to allow traffic on port 22 and 8080
resource "aws_security_group" "jenkins-sg" {
  name        = "jenkins-sg"
  description = "Allow traffic on port 22 and port 8080"
  vpc_id = var.vpc_id

  ingress {
    description = "Allow for SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Port 8080 used for web servers"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "virtual port that computers use to divert network traffic"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

#Create S3 bucket for your Jenkins Artifacts not open to the bucket
resource "aws_s3_bucket" "jenkins-bucket" {
  bucket = var.s3bucket
  acl    = var.acl
  tags = {
    description = "Private bucket to hold Jenkins artifacts"
    name        = "jenkins-bucket"
  }
}


#Create IAM role for EC2 to allow S3 read/write access
resource "aws_iam_role" "s3-jenkins-role" {
  name = "s3-jenkins-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })
}

#IAM instance profile for EC2 instance
resource "aws_iam_instance_profile" "s3-jenkins-instance-profile" {
  name = "s3-jenkins-instance-profile"
  role = aws_iam_role.s3-jenkins-role.name
}

#IAM policy for S3 access
resource "aws_iam_policy" "s3-jenkins-policy" {
  name = "s3-jenkins-policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3Access"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = [
          "${aws_s3_bucket.jenkins-bucket.arn}/*",
          "${aws_s3_bucket.jenkins-bucket.arn}"
        ]
      }
    ]
  })
}

#Attaches IAM policy to IAM role
resource "aws_iam_role_policy_attachment" "s3-jenkins-policy-attachment" {
  policy_arn = aws_iam_policy.s3-jenkins-policy.arn
  role       = aws_iam_role.s3-jenkins-role.name
}
  
  Step Three: Deploy our Infrastructure
The next step is to begin the terraform workflow and deploy our infrastructure.
Firstly, run the command:
  
  terraform init
  
  terraform init downloads plugins needed to execute config files.
  
  Next run the command:
  
  terraform fmt
  
  terraform fmt This command will validate the syntax and ensure the format and spacing of our file is correct.
Now run the command:
  
  terraform plan
  
  terraform plan creates an execution plan, listing resources that will be created based on the Terraform file.
  
  Finally run the command:
  
  terraform apply -auto-approve
  
  erraform apply creates the resource infrastructure as defined in the Terraform file.
Step Four: Verify Infrastructure
Navigate to the management console and confirm that you can see your instance, your s3 bucket, your security groups.
  
  Step Five: Verify Jenkins Deployment
SSH into your instance and run the following command to verify Jenkins has successfully been installed:
  
  sudo systemctl status jenkins
  
  Now let’s verify that we can access Jenkins via a web browser. In your web browser, input the following address:
  
  http://<instance_ip>:8080
  
  Step Six: Verify S3 Permissions
We will verify that our S3 permissions have been configured correctly by testing some read and write commands in our instance CLI.
First configure the AWS CLI on the instance. You will also need to input your AWS Access Key ID, Secret Access Key, default region, and output format
  
  aws configure
  
  Run the following command to test that we can list the s3 buckets in our AWS account:
  
  aws s3 ls
  
  Success! Our Read permissions work. Now let us echo a new file and upload it to our s3 bucket that we created in our terrafrom file to verify we can use the PutItem API call.
  
  echo "I love Terrafrom" > test.txt
  
  Run the following command to upload the file to our s3 bucket:
  
  aws s3 cp test.txt s3://<bucket_name>
  
  Success!
Now, Navigate to your S3 Bucket in the management console and verify that you can see your file within the s3 bucket.
  
  Step Seven: Clean up
Finally let us clean up our environment by running the command:
terraform destroy -auto-approve
All your resources will be destroyed except for the s3 bucket as we have an object inside. Manually delete the object and run the terraform destroy command again.
  
  The End
Thank you following along with this Terraform project.
