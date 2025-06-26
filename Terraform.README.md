1. Diff Cloudformation and Terraform (See the screenshot for ans.)

1. Sentinel provides three enforcement modes.
============================================
- Hard-mandatory: requires that the policy passes. If a policy fails, the run is halted and may not be applied until the failure is resolved.

- Soft-mandatory: is similar to hard-mandatory, but allows an administrator to override policy failures on a case-by-case basis.

- Advisory: will never interrupt the run, and instead will only surface policy failures as informational to the user.

2. Can you provide a few examples where we can use for Sentinel policies?
================================================
- Enforce explicit ownership in resources
- Enforce mandatory tagging on resources 
- Restrict how modules are used in the Private Module Registry
- Restrict roles the cloud provider can assume

3. Built-In provisioners in terrafom
====================================
- File Provisioner
- Puppet provisioner
- Chef Provisioner
- Remote-exec provisioner

4. Create ec2 and encrypt the ebs
5. How to lock tfstate file
=================================
	terraform {
		backend "s3"{
			bucket = "tf_bucket",
			key = "terrform_state.tfstate",
			region = "us-east-1"
			dynamodb_table = "tfstate_lock"
		}
	}
	
	terraform plan -lock = true   OR	terraform apply -lock = true


7. How to define the dependencies in terraform
==============================================
- depends_on keyword used to define the resource dependencies.
- Usefule when you want create the resource in particular order

resource "aws_vpc" "my_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "my_subnet" {
  cidr_block = "10.0.1.0/24"
  vpc_id = aws_vpc.my_vpc.id
  depends_on = [aws_vpc.my_vpc]
}



10. What is null_resource?
==========================
The null_resource resource implements the standard resource lifecycle but takes no further action.

11. Looping in terraform
=============================
1. for_each 
-------------
locals {
  avenger_names = ["ironman", "captain america", "thor","doctor strange","spider man","hulk","black panther","black widow"]
}

resource "aws_iam_user" "avengers" {
  for_each = toset(var.avenger_names)
  name     = each.value
}
output "avengers" {
  value = aws_iam_user.avengers
}


2. count - index
------------------
resource "aws_instance" "server" {
  count = 4 # create four similar EC2 instances

  ami           = "ami-a1b2c3d4"
  instance_type = "t2.micro"

  tags = {
    Name = "Server ${count.index}"
  }
}

12. Terraform Vault
----------------------
- Used to store the sesitive information similar like AWS secret manager
- You can store the API keys, DB username, DB password.
- While terraform plan and apply execution, terraform will commuicate with Vault and get the secrets.

provider "vault" {
  address = "https://your-vault-server-url"
  token   = "your-vault-access-token"
}

data "vault_generic_secret" "my_secret" {
  path = "secret/path/to/your/secret"
}

resource "aws_instance" "example" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t2.micro"

  tags = {
    Name = "${data.vault_generic_secret.my_secret.data.username}-instance"
  }
}


6. How to unlock the tfstate file
====================================
- Once terraform apply or destroy is complete, tf state file automatically unlocked
- But sometimes due to some issue tfstate file not unlocked so you can use following command
	terraform force-unlock tf_file_id DIR

Another ways to lock and unlock
---------------------
lock:
terraform state lock my-state-file.tfstate

unlock:
terraform state unlock my-state-file.tfstate

8. Can we modify infrsturcure using terraform which is created manually
========================================================================
If you modify infrastructure outside of Terraform, it is recommended to import the existing resources into the Terraform state before attempting to manage them with Terraform. This ensures that Terraform is aware of the current state of the infrastructure and can make accurate decisions about what changes need to be made. To import an existing resource into the Terraform state, you can use the "terraform import" command.

9. Can we use terraform for OnPremises resource provision
==========================================================
- If you're using VMware vSphere as your on-premises virtualization platform.
- You can use the Terraform vSphere provider to provision and manage virtual machines, storage, and networking resources.

13. Diff between terraform refresh and import
--------------------------------------------
refresh - will refresh the state file
import  - will import the current state of infrastructure

What is use of "terraform init -upgrade" command?
---------------------------------------------------
- If you run terraform init with aws provider version 5.0.0, version will be locked into .terraform.lock.hcl file
- When you just hit terraform init it will not allow to initialize due to mismatch version in lock file and your provider file

14. I want access the value of one module into another module how to do that?
------------------------------------------
1. Create IAM role in module
2. Create EC2 in secod module

15. How do you ensure your Terraform state file is not accidentally committed to version control?
-----------------------------------------------
- Add the .terraform directory to your .gitignore file.

16. You encounter a circular dependency between Terraform resources. How can you resolve it?
--------------------------------------------------
- 1. Introduce an Intermediate Resource:
- 2. Create separate modules for resources that have circular dependencies. Module can manage internal dependencies
- 3. Utilize Local Values

17. How can you prevent remote-state manipulation and ensure data integrity?
------------------------
- Implement Terraform Cloud backend with Sentinel policies for access control.


18. How can you prevent Terraform from modifying specific resources during a planned change?
---------------------------------------------
Two ways
---------
- 1. prevent_destroy 
resource "aws_instance" "my_instance" {
  # ... other configuration options

  lifecycle {
    prevent_destroy = true
  }
}

- 2. ignore_changes 
resource "aws_db_instance" "my_database" {
  # ... other configuration options

  lifecycle {
    ignore_changes = ["tags.drift_example"]
  }
}
