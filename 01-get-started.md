# Getting Started with AWS

## Module Objectives

Before getting started, you first have to prepare the environment for the workshop.

1. Get an AWS account from the instructor
1. Connect to the AWS Cloud9 IDE using the AWS account
1. Download the lab source code from GitHub

---

## Amazon Web Services Overview

- Managed by Amazon
- Provides basic resources like compute, storage and network
- Also provides services like Relational Database Service and Managed Kubernetes Service
- All operations can be done through the API
- SLAs define reliability guarantees for the APIs
- Three ways of access
- API calls
- SDK commands
- Amazon Web Console

Amazon Web Services product groups:

- Compute
- Storage
- Database
- Migration & Transfer
- Networking & Content Delivery
- Developer Tools
- Robotics
- Blockchain
- Satellite
- Management & Governance
- Media Services
- Machine Learning
- Analytics
- Security, Identity & Compliance
- Mobile Services
- AR & VR
- Application Integration
- Internet of Things
- Game Development

You will use these services while doing the lab:

- Amazon Elastic Container Service for Kubernetes: Create Kubernetes cluster
- Identity and Access Management: Manage users and permissions
- Elastic Compute Cloud: Run virtual machines for worker nodes
- Virtual Private Cloud: Connectivity between the nodes
- Elastic Load Balancing: Create Ingress of LoadBalancer type
- Elastic Block Store: Persistent volume for Jenkins
- CodeCommit: Hosting source code for an app
- CodeBuild: Build Docker containers
- Elastic Container Registry: Storing versioned Docker images of an app

Amazon Web Console is the admin user interface for Amazon Web Services. With Amazon Web Console you can find and manage your resources through a secure administrative interface.

Cloud Console features:

- Resource Management
- Billing
- SSH in Browser
- Activity Stream
- Cloud9 Console

Cloud SDK provides essential tools for cloud platform.

- Manage Virtual Machine instances, networks, firewalls, and disk storage
- Spin up a Kubernetes Cluster with a single command

---

## Amazon Web Services (AWS) Account

In this training you will run Kubernetes in AWS. We have created a separate account for each student. You should receive an email with the credentials to log in.

We recommend using Google's Chrome browser during the workshop.

1. Go to <https://console.aws.amazon.com/>
1. Enter the username
1. Enter the user password

    > Note: Sometimes AWS asks for a verification code when it detects logins from unusual locations. It is a security measure to keep the account protected. If this happens, please ask the instructor for the verification code.

1. In the top right corner select the ap-northeast-1 (Tokyo) region.

    > Note: Some AWS features are not be available in all regions.

## AWS Cloud9 Console

AWS Cloud9 is a cloud-based integrated development environment (IDE) for managing cloud resources. Most of the exercises in this course are done from the command line, so you will need a terminal and an editor.

We can create the Cloud9 development environment via CloudFormation. This CloudFormation template will spin up the Cloud9 IDE, as well as configure the IDE environment for the rest of the workshop.

Click on the [Deploy to AWS](https://console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/new?stackName=k8s-workshop&templateURL=https://s3.amazonaws.com/aws-eks-workshop-altoros-artifacts/lab-ide-vpc.template) button and follow he CloudFormation prompts to begin.

Accept the default stack name and Click *Next*. You can give Tags such as Key=Name, Value=k8s-workshop, and click *Next*. Make sure to check *I acknowledge that AWS CloudFormation might create IAM resources with custom names* and click *Create*.

CloudFormation creates nested stacks and builds several resources that are required for this workshop. Wait until all the resources are created. Once the status for *k8s-workshop* changes to *CREATE_COMPLETE*, you can open Cloud9 IDE. To open the Cloud9 IDE environment, click on the "Outputs" tab in CloudFormation Console and click on the "Cloud9IDE" URL.

![](../../img/eks/cloudformation-output-tab.png)

You should see an environment similar to this:

![](../../img/eks/cloud9-development-environment-welcome.png)

### Cloud9 Instance Role

The Cloud9 IDE needs to use the assigned IAM Instance profile. Open the "AWS Cloud9" menu, go to "Preferences", go to "AWS Settings", and disable "AWS managed temporary credentials" as depicted in the diagram here:

![](../../img/eks/cloud9-disable-temp-credentials.png)

### Build Script

Once your Cloud9 is ready, download the build script and install in your IDE. This will prepare your IDE for running tutorials in this workshop. The build script installs the following:

* jq
* kubectl
* heptio/authenticator
* updates/configures the AWS CLI and stores necessary environment variables in bash_profile
* kops
* creates an SSH key

To install the script, run this command in the "bash" terminal tab of the Cloud9 IDE:

```sh
aws s3 cp s3://aws-eks-workshop-altoros-artifacts/lab-ide-build.sh . && \
chmod +x lab-ide-build.sh && \
./lab-ide-build.sh
```

![](../../img/eks/cloud9-build-script.png)

## Download the Lab Source Code from GitHub

Clone the lab repository in your cloud shell, then `cd` into that directory:

```shell
git clone -b samsung-training https://github.com/Altoros/kubernetes-training.git
```
```shell
cd kubernetes-training
```

---

Next: [Containers](02-containers.md)
