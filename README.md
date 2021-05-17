# Terraform AWS EC2 Windows Password Rotation using Lambda/SSM and secret Manager
When you create an EC2 windows the password will remain the same for the whole life of the machine. It can be rotated automatically with this module

## Requirement
* AWS Provider - 3.40
* Python - 3.9
* Terraform - 0.15.3

This module is a dependencie of https://github.com/rajeshwrn/terraform-aws-windows-password-rotation-secret-manager/

Thanks  <a href="https://github.com/giuseppeborgese">@giuseppeborgese</a>

It is the enhanced version of <a href="https://github.com/giuseppeborgese/terraform-aws-windows-password-rotation-secret-manager">giuseppeborgese/terraform-aws-windows-password-rotation-secret-manager</a> and <a href="https://github.com/giuseppeborgese/terraform-aws-windows-password-lambda-rotation">giuseppeborgese/terraform-aws-windows-password-lambda-rotation</a>

## Prerequisites

* The ec2 machine is already created and needs to have a role with a policy ssm arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM
* The SSM needs to work on the machines if it is not the password is not replaced and the old one with pem key remains, nothing brokes.
* To change a password it is not necessary to know the previous password because SSM runs as a daemon.

## Infrastructure and steps
Using the terraform modules it will be created

* One lambda function for account and region
* For each machine a secret manager record, all the records call the same functions.

### Creations steps:
Create one lambda function for each region you are working on. Using this code

``` hcl
module "windows-password-lambda-rotation" {
  source  = "github.com/rajeshwrn/terraform-aws-windows-password-lambda-rotation/"
  prefix  = "raj"
  application_id   = "application_id"
  application_name = "application_name"
  aws_region       = "aws_region"
  bu               = "bu"
  core_id          = "core_id"
  created_by       = "created_by"
  dept_id          = "dept_id"
  env              = "env"
  profile          = "profile"
  project_id       = "project_id"
}
```
* Before applying the rotation try to run a simple command like this to the machine, to see if ssm commands can run, check the output in the run command history

``` hcl
aws ssm send-command --instance-ids i-xxxxxxxxx --document-name AWS-RunPowerShellScript --parameters commands="dir c:"
``` 

* For each EC2 Windows machine create a new secret manager record and connected to the function using this code

``` hcl
module "windows-password-rotation-secret-manager" {
  source  = "github.com/rajeshwrn/terraform-aws-windows-password-rotation-secret-manager"
  secret_name_prefix = "vault_"
  instanceid = "i-xxxxxx"
  rotation_lambda_arn = "${module.windows-password-lambda-rotation.lambda_arn}"
  application_id   = "application_id"
  application_name = "application_name"
  aws_region       = "aws_region"
  bu               = "bu"
  core_id          = "core_id"
  created_by       = "created_by"
  dept_id          = "dept_id"
  env              = "env"
  profile          = "profile"
  project_id       = "project_id"
}
``` 
* You can rotate the password manually using the rotation button or wait the numbers of days defined
* You can still recover the old password from the web console but it will NOT work


so you can see all the details and don’t miss anything. There is also a troubleshooting phase.

# Rotation steps what happens behind the scene:
This image describes all the steps every time there is a rotation manual or automatic

![solution](https://raw.githubusercontent.com/rajeshwrn/terraform-aws-windows-password-lambda-rotation/main/solution.png)

Let's read in details:

1. The secret manager record triggers a lambda function passing its parameters.
2. The Lambda extract the instance id from the called record and generates a new password with the predefined criteria
3. The lambda runs a call to SSM run command this runs a AWSSupport-RunEC2RescueForWindowsTool command in the EC2 machine.
4. The EC2 machine recovers the password from parameter store by EC2Rescue tool.
5. The SSM command will reset password and store the password in SSM parameter store secret /EC2Rescue/Passwords/i-xxxxxxxxxxx
6. The lambda stores the new password in the original Secret Manager record using the default key aws/secretsmanager
7. The lambda deletes the password from the parameter store.

You cannot pass the password as a parameter in the shell script because it will be shown in cleartext the System Manager log, it has to “travel” between services always encrypted.

# Feedback
if you like this module leave a comment
