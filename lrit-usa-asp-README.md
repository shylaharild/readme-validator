CloudFormation Stack for LRIT-USA-ASP
===============================

A guideline to know how to use the CloudFormation YAML Templates to spin up the LRIT-USA-ASP service. 

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

> - lrit-usa-asp.yaml - The template that creates all necessary resources for the application infrastructure

# lrit-usa-asp.yaml
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
> - Scaling Policies and CPU alarms
> - CodeDeploy Application
> - Deployment Group (based on Environment)

This template gives output values of the resources and their attributes which can be used for application use and for Cross-Stack Reference into other templates (if any).

*Note*: Care should be taken when giving values to the parameters. It is used to configure all the resources in the template.

Metadata of Auto Scale Launch Configuration:
----

The metadata of the launch configuration comprises of 2 configsets. 

> - install\_dependencies
> - configure\_application\_info

install_dependencies:

* This section has service init commands to start ntpd, codedeploy-agent and cfn-hup. These are already made available when we create the AMI (Part 1) using packer.
* This consists of cfn-hup configuration which is used to update instances with the changes in the Metadata. The cfn-hup updates run at regular intervals (every 15 minutes) checking the Metadata changes.

configure\_application\_info:
This section basically creates the files necessary for the application to run in the instance. They are as follows

* /etc/profile.d/environment - File with necessary environment values created by the Stack. This is widely used in many places, especially in the CodeDeploy Appspec scripts that is used for deployment.
* /etc/profile.d/polestar.sh - Script to alter the design of profile view
* Network and Hostname related files (/etc/sysconfig/network, /etc/hostname, /etc/hosts)

User Data:
---

The userdata is also responsible for making few configurations like

* Removing default postgresql server and installing the PostgreSQL 9.6 to match with RDS instance
* Setting static hostname for the instances
* Creating necessary directories for the application at /opt, /var/log and /var/run
* Running initial puppet using the script /opt/dependencies/puppet_modules_update.sh
* Creating a /etc/success file. This file is used by the CodeDeploy to know if the instances are ready for the deployment or not

Scaling Policy & Alarms
---

As a step to monitoring and scaling, we have introduces the Scaling policies and CPU alarms as starters. This scales the autoscale group with the increased or decreased number of instances running the application in parallel so that the traffic is served without applications hanging or crashing due to High CPU usage. This uses the simple scaling type with cool down period of 5 minutes. This scaling process is triggered by the Alarm configured with the CPU threshold to 80% (for now). If the average CPU of the autoscale group reaches to 80%, then a new instance will be launched by itself. Once the CPU goes back to normal, the autoscale group will manage to terminate one of its instance to bring it to Minimum Capacity.

*Note*: The AutoScale will not just terminate the newly created instance. In fact, there is no guarantee that which instance will be terminated during the Scale Down event. So it is advised not to store any important files inside any of the instances. 

CodeDeploy Resource:
---

The template is responsible to create the following CodeDeploy resources. The template will not proceed with the deployment of the application unless it is triggered by Jenkins or manually using AWS console.

* CodeDeploy Application
* Environment based Deployment Group
* CodeDeploy IAM Role to give Deployment group access to the S3 bucket and instances.

*Note*: The template is designed such that the codedeploy application will be created only for test and stage stack. And in the event of stack deletion, it is mandated to retain so that we will not lose the revision list that was deployed earlier.

# Jenkins

Jenkins is used for 2 different purposes in the new DevOps V2 world. 

* To create a Packer AMI with all the software dependencies installed that are necessary for the application to run.
* To deploy the application into the test and stage instances using Code Deploy. 

# Jenkins job
---

The Jenkins job used to deploy the application into the instance uses 2 different Github repositories.

* Application Repository (lrit-usa-asp)
* devops-aws repo with the appspec yaml and scripts

Create a job as Maven project. Do the following in the settings. Make sure that the Global Tool Setting contains the *MAVEN_HOME* set to `/opt/maven` and *JAVA_HOME* set to `/opt/java8`

```
General:
----
* Configure the Github Project url. 
* Enable `This project is parameterized` option and add **BUILD_WITH_BRANCH**  as a parameter. This parameter is used to run the job against a branch that is created in `devops-aws` repository.

Source Code Management:
----
* Select Multiple SCMs.
* Select git from the dropdown.
* In the Repositories section, add the `devops-aws` repository URL with the credentials.
* In the Brach Specifiers, mention `*/$BUILD_WITH_BRANCH` as value
* Add another repository by selecting git from the dropdown.
* In the Repositories section, add the `lrit-usa-asp` repository URL with the credentials.
* In the Brach Specifiers, mention `*/master` as value. Do not alter this unless you want to run the build against a branch.
* In Additional Behavious section, select `Check out to a sub-directory` from the dropdown. Provide `lrit-usa-asp` as its value. This will help to checkout the repository into the mentioned directory inside the workspace.

Build Environment:
----

Select the option 'Delete workspace before build starts' so that the images files don't fill up the server disk space (**Optional**). If you don't want to download the maven repository dependencies everytime, you can leave it without enabling this option

Build: 
----
* In Root POM, give the value `lrit-usa-asp/pom.xml`

Post Steps:
---
* Select the option `Run only if build succeeds`
* From the dropdown, select `Execute shell` option
* Add the following code in the commands

```
#!/bin/bash -x

/bin/bash $WORKSPACE/cloudformation/LRIT/${JOB_NAME}_appspec/scripts/JenkinsBuildScript.sh

#Archiving the applications
VERSION=`grep '^version=' ${WORKSPACE}/${JOB_NAME}/target/maven-archiver/pom.properties | cut --delimiter== --fields=2`
cd $WORKSPACE/images && zip -r $WORKSPACE/${JOB_NAME}-$VERSION.zip ./*

#Using AWS CLI to deploy
echo "*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*"
echo "*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*"
echo "The Code Deploy plugins is misbehaving. So using CLI to create revision and deploy into test environment"
echo "*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*"
echo "*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*"
[[ -f $WORKSPACE/${JOB_NAME}-$VERSION.zip ]] && /bin/aws s3 cp $WORKSPACE/${JOB_NAME}-$VERSION.zip s3://polestar-images/${JOB_NAME}/${JOB_NAME}-$VERSION.zip \
                                             || exit 1
                                             
ETAG=$(/bin/aws s3api head-object --bucket polestar-images --key ${JOB_NAME}/${JOB_NAME}-${VERSION}.zip --region us-east-1 | sed -n 's/.*"ETag": "\\\"\(.*\)\\\"",/\1/p')

/bin/aws deploy register-application-revision --application-name ${JOB_NAME} --description "Deployment to test-${JOB_NAME}" \
                --s3-location bucket=polestar-images,key=${JOB_NAME}/${JOB_NAME}-${VERSION}.zip,bundleType=zip,eTag=$ETAG \
                --region us-east-1

/bin/aws deploy create-deployment --application-name ${JOB_NAME} --deployment-config-name CodeDeployDefault.OneAtATime \
                --deployment-group-name test-${JOB_NAME} --description "Deployment to test-${JOB_NAME}}" \
                --s3-location bucket=polestar-images,key=${JOB_NAME}/${JOB_NAME}-${VERSION}.zip,bundleType=zip,eTag=$ETAG \
                --region us-east-1

```

Post-build Action:
----

This will be used usually to do the codedeploy. Since the dev-jenkins box is misbehaving, we are bound to use CLI method mentioned in the earlier section.

*Note*: Do not proceed with the steps below if you have mentioned the CLI code in the Execute Shell.

This is the important step. This connects the SCM, build with the Code Deploy. You need active Code Deploy application available or should start with the same names that you mention in this area.

* Select 'Deploy an application to AWS CodeDeploy' option in the list.
* Provide all the necessary values against their required fields.
* You don't have to select any options or provide access/secret keys if you have Jenkins running in an instance with IAM role 'CodeDeployDeployerAccess'.


How to run....
-------------------

In the Cloud Formation, create a stack with lrit-usa-asp.yaml with correct parameter values. Wait for the resource to be created which usually takes upto 15 to 20 minutes.

Once the stack has been completely created and Jenkins setup ready, you can trigger the build in the Jenkins which will complete the deployment to the environment.

# Deployment Rollback Check
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




