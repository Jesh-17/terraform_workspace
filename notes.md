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
3. Permission options->Attach policies directly->check 'AmazonS3FullAccess' and click on next
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