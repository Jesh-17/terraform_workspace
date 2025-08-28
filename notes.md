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

terraform destroy -auto-approve
```
---

```sh
# vi install_nginx.sh

#!/bin/bash
sudo apt-get update
sudo apt-get install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
echo "<h1>The nginx is installed!!<h1/>" | sudo tee /var/www/html/index.html

```
```
# ec2 instance
resource "aws_instance" "my_instance" {
  key_name = aws_key_pair.my_key.key_name
  security_groups = [aws_security_group.my_security_group.name]
  instance_type = var.ec2_instance_type
  ami           = var.ec2_ami_id  # amazon machine image
  user_data = file("install_nginx.sh")    # It allows the script runs automatically when a new server first boots.
  root_block_device{
    volume_size = var.ec2_root_storage_size
    volume_type = "gp3"

  }

  tags = {
    Name = "my-ubuntu-ec2"
  }
}

```
```bash
terraform fmt
terraform validate
terraform plan  # 'plan.out' file would be created by the terraform to store the execution plan.
terraform apply
ssh -i terra-key-ec2 ubuntu@your_ec2_public_ip     # check nginx running or not: 'sudo systemctl status nginx' and if you go to the browser and give: http://your_ec2_public_ip  and you will see the application running 'The nginx is installed!!'

terraform state list # list all the resources currently tracked in the state file
terraform destroy --target=aws_instance.my_instance  # particular resource when we want to delete
terraform destroy -auto-approve
```

```
# ec2 instance
resource "aws_instance" "my_instance" {
  count = 2   # meta argument
  key_name = aws_key_pair.my_key.key_name
  security_groups = [aws_security_group.my_security_group.name]
  instance_type = var.ec2_instance_type
  ami           = var.ec2_ami_id  # amazon machine image
  user_data = file("install_nginx.sh")    # It allows the script runs automatically when a new server first boots.
  root_block_device{
    volume_size = var.ec2_root_storage_size
    volume_type = "gp3"

  }

  tags = {
    Name = "my-ubuntu-ec2-${count.index+1}" # count.index starts from 0
  }
}

```
```
# outputs.tf for count
output "ec2_public_ip"{
  value = aws_instance.my_instance[*].public_ip
}
output "ec2_public_dns"{
  value = aws_instance.my_instance[*].public_dns
}
output "ec2_private_ip"{
  value = aws_instance.my_instance[*].private_ip # we can see attributes in terraform plan like public_ip...etc
}
```
```bash
terraform validate
terraform plan
terraform apply
ssh -i terra-key-ec2 ubuntu@your_any_ec2_public_ip

terraform destroy -auto-approve
```
---
```
# ec2 instance
resource "aws_instance" "my_instance" {
  for_each = tomap({  # meta argument
     my_ubuntu_1 = "t2.micro"
     my_ubuntu_2 = "t2.medium"
  })
  
  depends_on = [aws_security_group.my_security_group, aws_key_pair.my_key]        # It is a meta argument, tells this resource block will be depends on the given resources in-order to create.

  key_name = aws_key_pair.my_key.key_name
  security_groups = [aws_security_group.my_security_group.name]
  instance_type = each.value
  ami           = var.ec2_ami_id  # amazon machine image
  user_data = file("install_nginx.sh")    # It allows the script runs automatically when a new server first boots.
  root_block_device{
    volume_size = var.ec2_root_storage_size
    volume_type = "gp3"

  }

  tags = {
    Name = each.key
  }
}

```
```
# outputs.tf for For each

output "ec2_public_ip"{
  value = {
    for i in aws_instance.my_instance :  i.public_ip
  }
}
output "ec2_public_dns"{
  value = {
    for i in aws_instance.my_instance :  i.public_dns
  }
}
output "ec2_private_ip"{
  value = {
    for i in aws_instance.my_instance :  i.private_ip
  }
}

output "ec2_public_ip"{
  value = {
    for idx,inst in aws_instance.my_instance : "my-ubuntu-ec2-${idx+1}" => inst.public_ip
  }
}
output "ec2_public_dns"{
  value = {
    for idx,inst in aws_instance.my_instance : "my-ubuntu-ec2-${idx+1}" => inst.public_dns
  }
}
output "ec2_private_ip"{
  value = {
    for idx,inst in aws_instance.my_instance : "my-ubuntu-ec2-${idx+1}" => inst.private_ip # we can see attributes in terraform plan like public_ip...etc
  }
}

```
```bash
terraform validate
terraform plan
terraform apply
ssh -i terra-key-ec2 ubuntu@your_any_ec2_public_ip

terraform destroy -auto-approve
```
---
```
# variables.tf
variable "ec2_instance_type"{
  default = "t2.micro"
  type = string
}
variable "ec2_default_root_storage_size"{
  default = 10
  type = number
}
variable "env"{
  default = "prod"
  type = string
}
variable "ec2_ami_id"{
  default = "paste_ami_id"
  type = string
}

```
```
# ec2 instance
resource "aws_instance" "my_instance" {
  for_each = tomap({  # meta argument
     my_ubuntu_1 = "t2.micro"
     my_ubuntu_2 = "t2.medium"
  })
  
  depends_on = [aws_security_group.my_security_group, aws_key_pair.my_key]        # It is a meta argument, tells this resource block will be depends on the given resources in-order to create.

  key_name = aws_key_pair.my_key.key_name
  security_groups = [aws_security_group.my_security_group.name]
  instance_type = each.value
  ami           = var.ec2_ami_id  # amazon machine image
  user_data = file("install_nginx.sh")    # It allows the script runs automatically when a new server first boots.
  root_block_device{
    volume_size = var.env == "prod" ? 20 : var.ec2_default_root_storage_size   # Ternary operator or conditional statement
    volume_type = "gp3"
  }

  tags = {
    Name = each.key
  }
}

```
```bash
terraform validate
terraform plan
terraform apply
ssh -i terra-key-ec2 ubuntu@your_any_ec2_public_ip

terraform destroy -auto-approve
```
---
### Terraform State:
- It keeps a record of all resources terraform has created, updated, or destroyed. It is stored in a file called terraform.tfstate (by default, in the project folder)
- Without this terraform don't know what it already built.

```bash
# If you go to AWS and if you change the status of ec2 from running to stopped then, 
terraform refresh # It updates your local state file to match the real infrastructure in the cloud, It only makes sure the state file is accurate
terraform apply -refresh-only  # It only refreshes state, doesn't change resources.

terraform state list  # lists all resources terraform is tracking in the state file
terraform state show aws_key_pair.my_key   # show details of a resource.
terraform state rm aws_key_pair.my_key    # removes a resource from a state without deleting it in the cloud. Here exmaple still the keypair exists in the AWS

```
### now you removed so terraform does not know about it so you need to import the resource from cloud
```
# just create the dummy resource block of it in the local

resource aws_key_pair my_key {
                    
}

```
```bash
terraform import aws_key_pair.my_key key_resource_name_give  # EC2->key pairs i.e terra-key-ec2
terraform state list
terraform state show aws_key_pair.my_key
```
```
# Now copy the correct important attributes from terraform state show into .tf file. Since terrafrom always considers .tf file as the source of truth 
resource aws_key_pair my_key {
  key_name   = "terra-key-ec2"
  public_key= file("terra-key-ec2.pub")                  
}

```
```bash
terraform plan # if terraform shows no changes, your infrastructure matches the configuration
```

```
# This is simple example
resource "aws_instance" "my_new_instance"{
  ami = "unknown"
  instance_type = "unknown"
}

```
```bash
terraform import aws_instance.my_new_instance instance_id_give
terraform state list
terraform state show aws_instance.my_new_instance
```
---
### What is state conflict
- Terraform state conflict: if two people editing the same state file at once then state conflict would be occured. It is sloved by using ```'state locking'```
- Case 1:
  - local state (default):
    - terraform.tfstate is stored on your system. if you push code to github, your teammate has their own state file on their laptop.
    - Problem- Terraform does not know what the other person did this may results in duplicate or delete resources on cloud. 
    - Since the .tfstate file is local, there is no lock. So both you and your team mate apply changes and conflict occur(state file overwritten)
- Case 2:
  - Remote Backend(S3 without locking):
    - If state is stored in S3 bucket. Multiple people/CI pipelines can access the same state. if two people run terraform apply at the same time then conflicts occurs(last write wins and it is dangerous)
- Solution: With S3 + DynamoDB statefile locking & release mechanism- Terraform blocks second person until the first person finishes. So whenever first person tries the change the statefile in S3 bucket, a trigger will go to the DynamoDB and Here a LockID would be generated with info. Till this lock ID exists second person or other person won't allow to access the State file in S3 bucket. Means When the first person's terraform apply finishes, terraform automatically removes the lock entry and then others can run terraform safely.

Note:
  - DynamoDB how the data is stored? key-value way(json)
    - Table: like a folder that holds all the data
    - Item: A single record(like a row in sql)
    - Attributes: like columns in sql
    - you must define a primary key called Partition key or hash_key. Attributes you define are part of primary key 

  Eg:
  
    [
      {
        "UserID": "U123",
        "Name": "Ram"
      },
      {
        "UserID": "U124",
        "Name": "Max"
      }
    ]

    Final Table:
      UserID Name
      U123   Ram
      U124   Max

```
# vi s3.tf

resource "aws_s3_bucket" "remote_s3" {
  bucket = "my-tf-state-bucket"

  tags = {
    Name        = "my-tf-state-bucket"
    # Environment = "Dev"
  }
}

```
```
# vi dynamodb.tf

resource "aws_dynamodb_table" "basic-dynamodb-table" {
  name           = "my-tf-state-dynamodb-table"  # Table name
  # billing_mode   = "PROVISIONED"    # you fix how many reads/writes per second you want. You pay for this capacity whether you use it or not. If you need more than this, DynamoDB will block extra requests.
  # read_capacity  = 20
  # write_capacity = 20
  billing_mode   = "PAY_PER_REQUEST"  # you don't set any limits, Aws automatically adjusts to your traffic. You pay only for what you use.
  hash_key       = "LockId"  # tells table's primary key is LockID and this column use as primary key

  attribute {
    name = "LockId"  # Terraform also needs to tell AWS what type LockID is so we should define an attribute block. So It defines the datatype of the above column.
    type = "S"   # type is string, N (number), B (binary).
  }

  tags = {
    Name        = "my-tf-state-dynamodb-table"
    # Environment = "production"
  }
}

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

  backend "s3"{
    bucket = "my-tf-state-bucket"
    key    = "terraform.tfstate"     # path with statefile name in the s3 bucket
    region = "us-east-2"
    dynamodb_table = "my-tf-state-dynamodb-table"
  }
}



```
```
# vi providers.tf

provider "aws" {
  # Configuration options
  region = "us-east-2"
}

```
```bash
terraform init
terraform plan
terraform apply

terraform state list
terraform refresh

# Note: Get terraform state from backup just renaming the backup ex: terraform.tfstate.backup->terraform.tfstate, In local backend terraform always keeps a .backup file of your last state for safety.
terraform state list
terraform apply

# Now you can delete the statefile and  as well as backups in local backend, but still latest statefile is sync and connected in S3 bucket (Remote backend)
terraform state list  # Still it shows the resources state since it is in s3 bucket
```
### Now, in AWS if you go: DynamoDB->Tables->my-tf-state-dynamodb-table, Amazon S3->Buckets->my-tf-state-bucket->terraform.tfstate

```bash
# Person A
terraform apply  # Person A applied then LockID would be generated in DynamoDB.
```
```bash
# Person B
terraform apply  # Person B applied then he is waiting via saying  "Lock info like who is doing + Terraform acquires a state lock to protect the state from being written by mutiple users at the same time ", once person A finishes then only person B can modify the state file and that time LockID under Info of person A won't be available. We can see LockID under Info in the AWS path: DynamoDB->Explore items->my-tf-state-dynamodb-table ex: {"ID":".....",..}
```


