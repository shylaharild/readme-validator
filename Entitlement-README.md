Cloud Formation for LRIT-ENTITLEMENT
===============================

A guideline to know how to use the CloudFormation YAML Templates to spin the LRIT-Entitlement service up. 

----------

Ingredients Used
-------------
The current infrastructure also uses the AWS Cloud Service. Therefore, there are various resources already available or in place for the application. The main challenge is to use the existing resources into the templates and bring the infrastructure up and running. The following are the list of services that we are trying to use.
 > - Github Repository (E)
 > - Jenkins Job (E)
 > - Packer AMI (N)
 > - EC2 Auto Scale Resources(N)
 >  - IAM Roles (E)
 >  - Security Groups (E)
 >  - VPC & Subnets (E)
 >  - Key Pairs (E)
 >  - AMI (N)
 > - Code Deploy Resources (N)

**Here, (E) stands for Existing Resource and (N) New Resource


Templates
-------------
The Cloud Formation Stack works with Json & YAML Templates. 

> **List of Files**

> - Entitlement.yaml - The template that creates all necessary resources for the application infrastructure

# Entitlement.yaml
---

Template creates all the necessary resources for the application to be deployed. The list of resources created by this template are

> - SecurityGroups (EC2, ELB, DB & Redis)
> - IAM (Role, Instance Profile & Policy)
> - Subnet Groups (RDS & Redis)
> - RDS Instance
> - Elastic Load Balancer
> - Route53 Entries (RDS, Redis & ELB)
> - Master EC2 Instance
> - Auto Scale Group
> - Launch Configuration
> - CodeDeploy Application
This template gives output values of the resources and their attributes that can be used into other template using Cross-Stack Reference.

Metadata of Auto Scale and EC2:
----

The metadata for LRIT-Entitlement service includes the installation of nginx, postgresql server, client, devel and libs. It also ensures that the nginx and codedeploy-agent are running when the instance starts. The resource is launched using Instance AMI that is generated using Packer-Jenkins combination.

The following files are created during the launch

> - /etc/profile.d/environment - File with necessary environment values created by the Stack
> - Network and Hostname related files

User Data:
---

The userdata also makes few configurations like creating the developers group, set hostname, installing PostgreSQL 9.5 to match with RDS, creates directories for application logs, run and socket files, change ownership of the created necessary directories to devops user and at the end create a /et/success file. The 'success' file is important as it is used by the CodeDeploy to know if the instances is ready for the deployment or not.

```
P.S.: The above list of files & setup may change in future. Some of the files cab be baked in the AMI depending on its use.
```

CodeDeploy Resource:
---

The template is responsible to create the following CodeDeploy resources. The template will not proceed with the deployment of the application unless it is triggered by Jenkins or manually using AWS console.

* CodeDeploy Application
* Environment based Deployment Group
* CodeDeploy IAM Role to give Deployment group access to the S3 bucket and instances.

*Note*: The template is designed such that the codedeploy application will be created only for test and stage stack. And in the event of stack deletion, it is mandated to retain so that we will not lose the revision list that was deployed earlier.

# CIDR Ingress Rules
---

This template is slightly different from the other templates that have been created. The LRIT EC2 Instances are launched using AutoScale Group inot the private subnet. But the Elastic Load Balancer is a public or internet facing one. Usually, when we use a public facing resource it is always advises to access through the SSL. But in the LRIT case, we don't have the luxury to use the SSL and make the connection over HTTPS. The Production LRIT Server pool tries to connect to these servers through HTTP and over port 8080 - as of current situation (This may change in future). So we need to allow only the selected CIDR IPs to access the public facing ELB. The following are the list of IPs that are currently allowd ingress through the port 8080. [In Template](https://github.com/polestarglobal/devops-aws/blob/master/cloudformation/LRIT/Entitlement.yaml#L163-L225)

Office IPs
---
 > - 62.252.146.234/29   LON HQ
 > - 212.126.149.18/28   LON HQ
 > - 50.245.26.185/29    Boston
 > - 210.177.93.214/30   HK
 > - 203.186.109.113/28  HK
 > - 200.46.219.114/29   Pan
 > - 83.217.111.130/32   IX

CIDR of LRIT NDC & CDC Pool at IX
---
 > - 83.217.111.128/28
 > - 83.217.122.32/27
 > - 85.133.60.224/29
 > - 212.126.149.16/28
 > - 217.150.114.32/27

In addition to the above, we also give access to the 10.x.0.0/16, 192.168.0.0/16 & 172.16.0.0/19 where the *x* depends in the VPC IP range for the environment the application stack is launched in.

# Jenkins & Github

Github is the SCM used here for the source code versioning. Jenkins, as a CI tool, is responsible for generating the build archive from the SCM and Instance AMI using Packer for the application depending on the chnage in the md5sum of the requirements.txt file. This is to reduce the deployment time which again involves in rerunning the installation of requirements.txt as a double check process not to miss out any dependencies. Jenkins also acts as a pipeline module for the automated deployment. The post-build method connects with the mentioned Deployment Group of the CodeDeploy to deploy the application. In order to achieve this, you need the following plugins available in the Jenkins.

> - AWS SDK Plugin (1.10.50)
> - S3 publisher plugin
> - AWS Code Deploy Plugin
> - Github plugin
> - GitHub Authentication plugin
> - GitHub Branch Source Plugin
> - Github API plugin
> - Git plugin
> - Git Client plugin
> - GiHub Pull Request Builder

You need to add the appspec.yml file, application scripts and necessary service files for application and nginx to the LRIT-Entitlement Service Github Repository. These files are important for Code Deploy to run successfully. You also need to write a build-lrit-entitlement.sh file which creates the build image of the LRIT-Entitlement application. The script modifies the appspec.yml according to the version of the application. Refer: https://github.com/polestarglobal/lrit-entitlement/blob/master/README.md


####Jenkins job
---

Create a job as Freestyle project. Do the following in the settings

```
General:
----
 
Configure the Github Project

Source Code Management:
----

Select Git as SCM. In the Repositories add the Repository URL with the credentials. You have to mention branch to build else it will run for any checkins.

Build Triggers: 
----

Select 'Poll SCM' option and add 'H/05 * * * *' for Jenkins to poll every five minutes for the changes. **Optional**

Build Environment:
----

Select the option 'Delete workspace before build starts' so that the images files don't fill up the server disk space.

Build: 
----

Add build step with Execute Shell option. In the command area, add 'bash build-lrit-entitlement.sh'.

Post-build Action:
----

This is the important step. This connects the SCM, build with the Code Deploy. You need active Code Deploy application available or should start with the same names that you mention in this area.

Select 'Deploy an application to AWS CodeDeploy' option in the list. Provide all the necessary values against their required fields. 

You don't have to select any options or provide access/secret keys if you have Jenkins running in an instance with IAM role 'CodeDeployDeployerAccess'. Else, you need to create an user with that IAM role and S3 access role. You have to provide the user's access and secret keys in the 'Use Access/Secret keys' section.

```

How to run....
-------------------

In the Cloud Formation, create a stack with Entitlement.yaml with correct parameter values. Wait for the resource to be created which usually upto 15 to 20 minutes.

Once all the stack has been completely created and Jenkins setup ready, you can trigger the build in the Jenkins which will complete the deployment to the environment.

###Deployment Rollback Check
---

CodeDeploy supports Deployment Rollback to the last successful build if a deployment fails. To achive this, follow the steps... 

> - Go to CodeDeploy page
> - Select the application
> - Select the Deployment Group and Click on 'Actions ---> Edit'
> - Scroll to the bottom till 'Rollbacks' section
> - Disable the "Disable rollbacks" button
> - Enable the "Roll back when a deployment fails" checkbox
> - click on 'Save' button

After the above steps, you can find the following options displayed in the Deployment Group

```
Rollback enabled:  true
Rollback events :  DeploymentFailure
```

We need to do enable this because, broken release will be reverted to previous successful revision when there is a failure by AWS CodeDeploy automatically. This ensures only successful build to be live.

PS:
==
If there is any issue with the DB Migration, it has to be handled by the application. Deployment cannot revert this automatically as it is risky to handle any DB changes which may lead to data loss.


**LRIT-Entitlement Deployment:**
===
The deployments for LRIT-Entitlement application is handled by Jenkins-CodeDeploy Combination. The deployment process depends on the type of environment.

 1. Test
 2. Prod

Test LRIT-Entitlement:
---
All the Test Servers are launched into the ***polestar-test*** AWS account. The Cloudformation stack is used to spin up the infrastructure. The Test LRIT-Entitlement stack can be found [here](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stack/detail?stackId=arn:aws:cloudformation:us-east-1:178847878690:stack%2Ftest-lrit-entitlement%2Fec14ba20-ad02-11e6-9627-50d5cd274ad2). 

The Jenkins used for test build & deployments is [Dev-Jenkins](https://dev-jenkins.polestar-testing.com/). The job associated with the application is configured with the necessary credentials and linked with the Code Deploy application that was created as a part of infrastructure. You can go to [Configure](https://dev-jenkins.polestar-testing.com/job/lrit-entitlement/configure) to change any configurations for the TEST LRIT-Entitlement Build. The process is very simple. The job pulls the source code from specified [Github](https://github.com/polestarglobal/lrit-entitlement) branch (usually master) and then runs a shell script to build the package. Post that it connects to the Code Deploy application to deploy the archive.

To deploy a new version into the test infrastructure, kindly follow the following steps. It is simple and straight forward.

 1. Open Dev-Jenkins LRIT Entitlement Job by clicking [here](https://dev-jenkins.polestar-testing.com/job/lrit-entitlement/).
 2. Click on "Build Now". 
 3. The above action will pull the code from the specified branch in the SCM configuration, builds the package and deploys to the instances using Code Deploy and S3 bucket.

Prod LRIT-Entitlement:
---
The Stage and Prod environments are maintained in the ***Polestar-Production*** AWS Account. Since there are no Stage Data Centres available for the LRIT, there is no Stage Environment needed here. As a process in DevOps, we will be launching a stage environment - a dummy stack - to mimic the Prod Infrastructure. There won't be any release made to this environment through Jenkins as it is just to test the Template & Infrastructure changes before going to Production.

The Deployment Pipeline for Production LRIT is similar to that of Test. The only differences are the deployment is carried from [DevOps Jenkins](https://jenkins.polestar-testing.com/job/lrit-entitlement/) and the build operation is Paramaterized. Meaning, you need to provide a valid Tag from the [Github Tags/Releases](https://github.com/polestarglobal/lrit-entitlement/tags) to proceed with the deployment. The Tag is usually the version number. eg: [0.14](https://github.com/polestarglobal/lrit-entitlement/releases/tag/0.14).

To deploy a new version into the Prod Environment, kindly follow the following steps. The deployment to Stage and Prod are restricted to DevOps Team.

 1. Open the  DevOps Jenkins LRIT Entitlement Job by clicking [here](https://jenkins.polestar-testing.com/job/lrit-entitlement/).
 2. Click on "Build with Parameters"
 3. Enter the release or tag version to be released to the Production. You can obtain it from the assigned JIRA Ticket or from [Github Tags](https://github.com/polestarglobal/lrit-entitlement/tags). It is advised to ask for JIRA Ticket for Production Releases.
 4. Click on "Build". 

Jenkins then pulls the Source Code from Github, updates it to the given Tag, packages the application using the build phase and deploys the version into the Production LRIT Entitlement instances using Code Deploy. 

> **Note:**
The DevOps Jenkins is launched in the **Polestar-Test** AWS Account. When configuring the Job for Stage/Prod Deployment, make sure you do the following in the Post-Build section. This is done so that the Jenkins can do Cross-AWS-Account Connectivity to do a successful deployment.
> - In Post Build Actions, select Deploy an application to AWS CodeDeploy
> - Fill up the text fields from the Prod LRIT CloudFormation Stack
> - Select the "Use temporary credentials" option
> - In the "IAM Role ARN", give the value **arn:aws:iam::205287409906:role/CrossAccountDeployment**
> - Click on the Test Connection. If the connection is successful, you can see the message "***Connection test passed.***"


