# Servian Tech Challenge Submission

- [Application live version](https://servian-tech-challenge.vanessadenardin.com/)

---

## 1. Technology Stack Used for Solution

### AWS (Amazon Web Services)

AWS was chosen as the cloud provider as while doing certification for Terraform, I had used the examples for IaC on AWS only, also it's easier to find information, although in past I have majorly used IBM Cloud Platform.

### Amazon Elastic Container Service (ECS)

ECS was used for manage containers on a cluster running tasks in a service. Fargate was used to remove the need of EC2 intances as it's a serverless compute engine.

### Amazon Elastic Load Balancing (ALB)

ALB was used to cover application high availability and auto-scaling. The incomming traffic is distributed in the target group to route requests from the listeners in port 80 HTTP and in port 443 HTTPS with the TLS protocol using certificate.

### Amazon Relational Database Service (RDS)

Aurora PostgreSQL was used to migrate the database.

### Amazon Certificate Manager (ACM)

To increase security, an SSL/TLS certificate was acquired to be able to encrypt network communications on the application website.

### Amazon Route 53

To route end users to the application website creating a CNAME for the public ALB DNS. This is an optional resource.

### Amazon Simple Storage Service (S3)

Just a single bucket to store the Terraform state file.

### AWS Identity and Access Management (IAM)

To access control across services and resources of AWS. An User was created to give local and GitHub Action access to the AWS account and also a Role to give the ECS Task access to write logs to CloudWatch.

### Amazon Security Groups

To control inbound and outbound traffic, 3 security groups are added, on the server with inbound 3000 from the load balancer and outbound to everywhere, on the database with inbound 5432 from server, with no outbound, and on the load balancer with inbound from 80 and 443 for all and outbound 3000 for all for healthchecks.

### Amazon CloudWatch

Because it is integrated with ECS, it was used to monitor and collect data in the form of logs to troubleshoot and keep the application running.

### GitHub and GitHub Actions

Source code management and to automate the application workflow. It was configured in this case to run everytime a GitHub event like pull request (plan) and push/merge (apply).

[GitHub Actions Plan](https://github.com/vanessadenardin/servian-tech-challenge/runs/4710428483?check_suite_focus=true)

[GitHub Actions Apply](https://github.com/vanessadenardin/servian-tech-challenge/runs/4710317126?check_suite_focus=true)

### 3Musketeers

It makes reference to the use of the 3 technologies, Docker, Make and Docker Compose, used to test, build, run and deploy the application. 3Musketeers mainly helps with consistency across environments, running the same commands no matter if you are running the deployment locally or from a pipeline.

### Terraform

Infrastructure as a code tool is used to manage cloud services and due to its declarative syntax, it is easier to track changes through source code. In this case, it was used to create, manage and destroy AWS resources.

---

## 4. How to deploy

### Dependencies

- [Docker](https://www.docker.com/)
- [Docker-Compose](https://docs.docker.com/compose/)
- Make
- [Terraform 1.1.2](https://www.terraform.io/)
- [AWS CLI](https://aws.amazon.com/cli/)

### AWS account authentication

To run below commands, you will need to make sure to be authenticated to an AWS account. That can be done either exporting an AWS IAM User key/secret or by using roles if you have that setup.

[Configure AWS cli credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html#cli-configure-files-where)

### Manually create an S3 bucket in your AWS account

No extra configuration is needed, just make sure your AWS credentials have access to the S3 bucket.

[Create a S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html)

### Configure Terraform backend and variables

Before running the Terraform commands, you will need to make sure to configure your backend to point to your own S3 Bucket and have all following parameters configured as environment variables.

To configure the backend, you will need to edit the file [config.tf](/infra/config.tf) with below:

```terraform
    backend "s3" {
    bucket = "<your-bucket-name"
    key    = "terraform.tfstate"
    region = "ap-southeast-2"
    }
```

```bash
# AWS VPC ID to deploy the stack
export TF_VAR_vpc_id=<value>
# Database password, default username is 'postgres'
export TF_VAR_postgresql_password=<value>
# OPTIONAL: an existing domain name on your AWS account to create a CNAME record for the Load Balancer. If not specified, you will need to access through the LB public dns name
export TF_VAR_domain_name=<value>
# OPTIONAL: an ACM certificate ARN to be used by the 443 listener on the Load Balancer. If not specified, only port 80 will work
export TF_VAR_certificate_arn=<value>
# true/false (default false) -- If true, will prevent to deploy ECS Tasks with Public IPS. This in theory would only work with some re-work to add support to private subnets. I only used this for now to demonstrate some conditional values on Terraform.
export TF_VAR_production=<value>
```

### Run Terraform

With all variables configured, you can run the following Terraform commands:

- `make init`
    This will configure the backend in the config.tf file and download the cloud provider being used, in this case AWS.
- `make plan`
    This will show you which AWS resources will be deployed and save the result in a file called `terraform.plan`.
- `make apply`
    This will apply the `terraform.plan` file (it won't ask for approval!!) created in the previous step to deploy resources to your AWS account and create the `terraform.tfstate` file in your previously manually created S3 bucket.

    After the creation, it will return some outputs with the information of the resources created in the cloud. Make sure use `alb_dns_name` in the browser to check the application or if you have dns configured use `app_dns_name`.

    You can use `https://` if you have provided an ACM Certificate.

### Run update on database

Run `make update_db`

To populate the application's database, a script will perform updates and migrations on the application's database.

The script runs a standalone ECS task to update/migrate the application database. At this stage, it is important to make sure that Terraform (v1.1.2) and the AWS cli are installed on your local machine as the script is not using the 3Musketeers pattern. Unfortunately, I couldn't find a way to get the terraform variable values using containers.

### Delete the stack

Once you have tested this stack, it is recommended to delete all resources created on your AWS account to avoid any extra costs. Databases running 24/7 can get quite expensive.

For that you just need to run `make destroy`.

---

## 5. GitHub Actions

There is a provided example of a Github Actions Workflow under [/.github/workflows/terraform-infra.yml](/.github/workflows/terraform-infra.yml) file.

The workflow example will run if any changes to `/infra/**` files are commited and below rules are met:

- On pull requests to master
    - make init
    - make plan
- On push to master (merge)
    - make init
    - make plan
    - make apply

You can either check my own repository to see some pipeline runs or fork this repo and setup from your side.

https://github.com/vanessadenardin/servian-tech-challenge/actions

Once you fork this repository, you will need to go to Settings > Secrets and create a secret variable for each of the below variables:

```
VPC_ID
POSTGRESQL_PASSWORD
DOMAIN_NAME
CERTIFICATE_ARN
```

Those variables are being reference by the workflow as per below:

```yaml
- name: Terraform Plan
id: plan
env:
    TF_VAR_vpc_id: ${{ secrets.VPC_ID }}
    TF_VAR_postgresql_password: ${{ secrets.POSTGRESQL_PASSWORD }}
    TF_VAR_domain_name: ${{ secrets.DOMAIN_NAME }}
    TF_VAR_certificate_arn: ${{ secrets.CERTIFICATE_ARN }}
    TF_VAR_production: false
run: make plan
```

---

## 6. Challenges

- It was really challenging to deploy the entire stack on AWS, It was first time I was deploying a whole application with frontend and backend using ECS (used EC2 a few times in past)So I had to learn to run containers on AWS, Load Balancers.

- It was quite difficult to make update_db command to work as well, as I was getting mutiple errors/issues, at last decided to run ECS task run command to make it work.

- Having all security groups open only the ports they need and keeping everything running.

---

## 7. Future Recommendations

- Make the URL secure and accessible on https.

- Create new VPC using Terraform and utilize that inside all terraform files, as intially started with default VPC.

- Move application database from AWS Aurora provisioned to serverless to save money.

- Fix update_db script to switch from shell script to actual depoyment automation.

[Access old README](/readme_old.md)

