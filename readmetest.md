# Servian Tech Challenge Submission

- [Application live version](http://servian-tech-challenge-lb-1445237091.eu-west-2.elb.amazonaws.com)

---

## 1. Technology Stack Used for Solution

### Amazon Web Services (AWS)

AWS was chosen as the cloud provider. As while doing certification for Terraform last year, I had used the examples for IaC on AWS. Also it's easier to find information on AWS, although in past I have majorly used IBM Cloud Platform as stategic cloud partner in my project work.

### Terraform

Terraform is used as IaC (Infrastructure as code) tool to manage cloud services and due to its declarative syntax. Here Terraform is used to create, manage and destroy AWS resources.

### Amazon Simple Storage Service (S3)

Just a single bucket to store the Terraform state file as a Terraoform Backend Resource.

### Amazon Elastic Container Service (ECS)

ECS was used for manage containers on a cluster running tasks in a service. Fargate was used to remove the need of EC2 intances as it's a serverless compute engine.

### Amazon Elastic Load Balancing (ALB)

ALB was used to cover application **high availability and auto-scaling**. The incomming traffic is distributed in the target group to route requests from the listeners in port 80 HTTP.

### Amazon Relational Database Service (RDS)

Aurora PostgreSQL was used to migrate the database with version 10.14 as older versions were also not available in EU-WEST-2 region.

### Amazon Route 53

To route end users to the application website creating a CNAME for the public ALB DNS. This is an optional resource.

### AWS Identity and Access Management (IAM)

To access control across services and resources of AWS. An User was created to give local and GitHub Action access to the AWS account and also a Role to give the ECS Task access to write logs to CloudWatch.

### Amazon Security Groups

To control inbound and outbound traffic, 3 security groups were added, on the server with inbound 3000 from the load balancer and outbound to everywhere, on the database with inbound 5432 from server, with no outbound, and on the load balancer with inbound from 80 and outbound 3000 for all for healthchecks.

### Amazon CloudWatch

Because it is integrated with ECS, it was used to monitor and collect data in the form of logs to troubleshoot and keep the application running.

### 3Musketeers

It makes reference to the use of the 3 technologies, Docker, Make and Docker Compose, used to test, build, run and deploy the application. 3Musketeers mainly helps with consistency across environments, running the same commands no matter if you are running the deployment locally or from a pipeline.

### GitHub and GitHub Actions

GitHub Actions were used to automate the Infrastructure provisioning and deployment adding workflow automation as well. GitHub was used as Source code management tool. It was configured in this case to run everytime a GitHub event like pull request (plan) and push/merge (apply).

[GitHub Actions](https://github.com/AnuragSharma2011/servian-tech-challenge/actions/runs/2253676377)



---

## 2. How to deploy

### Dependencies 

- [AWS CLI](https://aws.amazon.com/cli/)
- [Terraform 1.1.2](https://www.terraform.io/)
- [Docker](https://www.docker.com/)
- [Docker-Compose](https://docs.docker.com/compose/)
- Make

**Pre-Requisites**

### AWS account authentication

To run below commands, you will need to make sure to be authenticated to an AWS account. That can be done either exporting an AWS IAM User key/secret or by using roles if you have that setup.

[Configure AWS cli credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html#cli-configure-files-where)

### Manually create an S3 bucket in your AWS account

No extra configuration is needed, just make sure your AWS credentials have access to the S3 bucket.

[Create a S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html)

### Configure Terraform backend and variables

Before running the Terraform commands, you will need to make sure to configure your backend to point to your own S3 Bucket and have all following parameters configured as environment variables.

To configure the backend, you will need to edit the file [config.tf](/IaC/config.tf) with below on line 11 and 13:

    bucket = "<your-bucket-name>"
    region = "<regoin here>"
    
once copied/forked you will also need to go to Settings > Secrets in your repository and create below secret variable:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
VPC_ID
POSTGRESQL_PASSWORD
DOMAIN_NAME*
```

These variables are being reference by the workflow as per below in :

```yaml
name: "Terraform Plan"
    runs-on: ubuntu-latest
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

- name: Terraform Plan
id: plan
env:
    TF_VAR_vpc_id: ${{ secrets.VPC_ID }}
    TF_VAR_postgresql_password: ${{ secrets.POSTGRESQL_PASSWORD }}
    TF_VAR_domain_name: ${{ secrets.DOMAIN_NAME }}
    TF_VAR_production: false
run: make plan
```


### Run Terraform using GitHub Actions Automation to provision infra and deploy application container

With all variables configured, as soon as any push or merge request is raised under /IaC folder, will trigger the GitHub Actions depending on last statement in workflow file as below: 

File - https://github.com/AnuragSharma2011/servian-tech-challenge/blob/master/.github/workflows/terraform-infra-and-deployment.yml

### Initial Deployment of App container
`make apply` - This will apply the terraform.plan file (it won't ask for approval!!) created in the previous step to deploy resources to your AWS account and create the terraform.tfstate file in your previously manually created S3 bucket.
After the creation, it will return some outputs with the information of the resources created in the cloud. Make sure use `alb_dns_name` in the browser to check the application or if you have dns configured use `app_dns_name`.

### Run update on database
`make update_db` - This will populate the application's database, a script will perform updates and migrations on the application's database.

The script runs a standalone ECS task to update/migrate the application database. 

### Delete the stack
`make destroy` - Once you have tested this stack, it is recommended to delete all resources created on your AWS account to avoid any extra costs. Databases running 24/7 can get quite expensive.

The workflow example will run if any changes to `/IaC/**` files are commited and below rules are met:

- On pull requests to master
    - make init
    - make plan
- On push to master (merge)
    - make init
    - make plan
    - make apply

You can either check my own repository to see some pipeline runs or fork this repo and setup from your side.

https://github.com/AnuragSharma2011/servian-tech-challenge/actions


---

## 6. Challenges

- It was really challenging to deploy the entire stack on AWS. It was first time, I was deploying a whole application with frontend and backend using ECS (used EC2 a few times in past)So I had to learn to run containers on AWS, Load Balancers.

- It was quite difficult to make update_db command to work as well, as I was getting mutiple errors/issues, at last decided to run ECS task run command to make it work.

- Having all security groups open only the ports they need and keeping everything running.

---

## 7. Future Recommendations

- Make the URL secure and accessible on https.

- Create new VPC using Terraform and utilize that inside all terraform files, as intially started with default VPC.

- Move application database from AWS Aurora provisioned to serverless to save money.

- Fix update_db script to switch from shell script to actual depoyment automation.

[Access old README](/oldreadme.md)

