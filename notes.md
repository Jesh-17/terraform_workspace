### Terraform
- It is a infrastructure as code tool (IaC)

```bash
# This is the file created manually.
vi new.txt 
    This is the file created manually.
```
```
# vi main.tf 
resource local_file my_file{  
   filename = "new_automate.txt"
   content = "This is the file created automatically."
} 

# local is provider, local_file is resource type, my_file is resource name.

```
```bash
terraform init 
terraform validate
terraform plan
terraform apply # without yes you can give- terraform apply -auto-approve

terraform destroy # without yes you can give- terraform destroy -auto-approve
```
---


```bash
# Provider is like a driver that tells terraform how to talk to a service. without this terraform does not know how to create things in AWS, Azure, Google Cloud..etc
# to check the providers
ls -a
cd .terraform/providers/registry.terraform.io/hashicorp/
ls -ltr
```

```
# vi terraform.tf

terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "6.10.0"
    }
  }
}
```
```bash
terraform init
cd .terraform/providers/registry.terraform.io/hashicorp/
ls -ltr    # Here we will 'aws' provider also
```
---
```bash
# To connect to the aws account
sudo apt-get install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

```
### Go to aws and create IAM (identity access management)
1. IAM->Users->Create a User
2. Username: 'terra-admin' and click on next
3. Permission options->Attach policies directly->check 'AmazonS3FullAccess' or 'AdministratorAccess' and click on next
4. And click create user
5. IAM->Users->terra-admin
6. Click on 'Security credentials'-> Access key-> create access key
7. give Use case as 'Command Line Interface(CLI)'->check confirmation-> click on next
8. click on Create access key-> copy Access key, Secret access key
```bash
aws configure # AWS Access Key ID,AWS Secret Access Key give which you copied in the above and remaining next..next
aws s3 ls # Show all the s3 buckets in the aws
```

```
# vi provider.tf
provider "aws" {
  # Configuration options
  region = "us-east-2"
}
```

```
# This is s3 bucket, vi s3.tf
resource aws_s3_bucket my_bucket_ref{
    bucket = "aws-s3-bucket"
}

```
```bash
terraform plan
terraform apply
terraform destroy
```

```
# vi terraform.tf
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "6.10.0"
    }
  }
}
```
```
# vi provider.tf
provider "aws" {
  # Configuration options
  region = "us-east-2"
}
```
```bash
# generate keys
ssh-keygen # Enter this later, enter file in which to save the key: terra-key-ec2 and give next..next. (we will get private and public key)
```
```
# vi ec2.tf


# key pair (login)
resource aws_key_pair my_key {
  key_name   = "terra-key-ec2"
  public_key= file("terra-key-ec2.pub")                  # (or) public_key = "Give_here_above_generated_public_key"
}


# VPC and Security Group
resource "aws_default_vpc" "default" {
  tags = {
    Name = "Default VPC"
  }
}

resource "aws_security_group" "my_security_group" {
  name        = "ec2-sg"
  description = "This is ec2 security group"
  vpc_id      = aws_default_vpc.default.id     # getting the values from other block is interpolation

   
  # inbound rules
  ingress{
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # classless inter-domain routing
    description = "SSH open"
  }
  ingress{
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # classless inter-domain routing, In this alows which IP ranges are allowed to access resources. Here allow traffic from any IP address in the world.
    description = "HTTP open"
  }
  ingress{
    from_port = 8000
    to_port = 8000
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # classless inter-domain routing
    description = "my app port open"
  }


  # outbound rules
  egress{
    from_port        = 0
    to_port          = 0
    protocol         = "-1" # all protocols
    cidr_blocks      = ["0.0.0.0/0"]
    description = "all access open outbound"
  }
  

  tags = {
    Name = "ec2-sg"
  }
}


# ec2 instance
resource "aws_instance" "my_instance" {
  key_name = aws_key_pair.my_key.key_name
  security_groups = [aws_security_group.my_security_group.name]
  instance_type = "t2.micro"
  ami           = "paste_ami_id"  # amazon machine image
  root_block_device{
    volume_size = 15
    volume_type = "gp3"


  }

  tags = {
    Name = "my-ubuntu-ec2"
  }
}
```
```bash
terraform init
terraform validate
terraform plan
terraform apply

chmod 400 terra-key-ec2
ssh -i terra-key-ec2 ubuntu@...... # you will be in ec2 instance


terraform destroy -auto-approve
```
---

```
# variables.tf

variable "ec2_instance_type"{
  default = "t2.micro"
  type = string
}
variable "ec2_root_storage_size"{
  default = 15
  type = number
}
variable "ec2_ami_id"{
  default = "paste_ami_id"
  type = string
}

```
```
# ec2 instance
resource "aws_instance" "my_instance" {
  key_name = aws_key_pair.my_key.key_name
  security_groups = [aws_security_group.my_security_group.name]
  instance_type = var.ec2_instance_type
  ami           = var.ec2_ami_id  # amazon machine image
  root_block_device{
    volume_size = var.ec2_root_storage_size
    volume_type = "gp3"

  }

  tags = {
    Name = "my-ubuntu-ec2"
  }
}

```
```
# outputs.tf
output "ec2_public_ip"{
  value = aws_instance.my_instance.public_ip
}
output "ec2_public_dns"{
  value = aws_instance.my_instance.public_dns
}
output "ec2_private_ip"{
  value = aws_instance.my_instance.private_ip # we can see attributes in terraform plan like public_ip...etc
}
```
```bash
terraform validate
terraform plan
terraform apply
ssh -i terra-key-ec2 ubuntu@your_ec2_public_dns
```


