# streamlit-cdk-fargate

You're one command away from deploying your [Streamlit](https://www.streamlit.io/) app on [AWS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)!

# TLDR: What is that one command you're teasing us with?

`git clone https://github.com/tzaffi/streamlit-cdk-fargate.git && cd streamlit-cdk-fargate && make deploy-streamlit`

The Streamlit App's URL will be printed at the very end of the process.

**DON'T FORGET TO [TEAR DOWN](#tear-it-down) YOUR DEPLOY AFTER USAGE !!!!**

## How long does that take?
Around 10-15 minutes. In particular:
1. Less than a minute to setup the project locally
2. 10 minutes or so to stand up the stack on AWS

## Caveats

Ok, so that was actually assuming the following pre-req's:
* you have an AWS account
* have configured the AWS CLI
* installed AWS CDK on your machine
* have the `git` and `make` commands available
* optional but recommended: `docker` and `docker-compose`

## What if I don't have those pre-reqs?
Here's [an explanation](#pre-requisites)

## Attribution - Standing on the Shoulders of Giants
* Maël Fabien's [excellent post](https://maelfabien.github.io/project/Streamlit/). I ripped off his very trim Streamlit app and Dockerfile.
* Nicolas Metalo's [very comprehensive tutorial](https://github.com/nicolasmetallo/legendary-streamlit-demo). I ripped off most of the rest of this code from there, except that I tried to streamline the CDK stack definition a bit, and summarized the various commands using `make`.


# Ok, that was a little sparse. I want to understand how this all works and what I should do with my streamlit app.
The basic steps are to:

1. [Setup](#setup-your-streamlit-docker-image) your Streamlit Docker image
2. [Test](#test-the-docker-image-locally) the Docker image locally
3. [Setup](#setup-the-cdk-fargate-stack) your CDK streamlit Fargate stack, copying the Docker project into the cdk project
4. [Deploy](#deploy-the-streamlit-docker-image-on-fargate-using-cdk) the CDK stack
5. [Tear-down](#tear-it-down) the CDK stack when you're done using it

## Setup your Streamlit Docker Image
This is the following sub-directory in the repo:
```bash
.
├── streamlit-docker
    ├── Dockerfile
    ├── docker-compose.yml
    |── requirements.txt
    ├── app.py
```

### UPSHOT: Tuck your streamlit `app.py` and `requirements.txt` right next to `Dockerfile` and `docker-compose.yml`

You won't need to modify `Dockerfile` or `docker-compose.yml` unless you want do change the configuration, such as running the app on a port different from 8501 (the typical local streamlit port).

## Test the Docker Image Locally

Run the following command:

`cd streamlit-docker && docker-compose up --build streamlit`

(this is also available as `make local-streamlit`)

You'll be able to run the app locally on port 8501: [http://localhost:8501](http://localhost:8501)

To terminate the app simply exit the docker interactive shell as usual: on Linux / Mac this is done using the `CTRL-C` key stroke.

## Setup the CDK Fargate Stack
Here you'll use a few command to recreate the cdk portion of this repo and also copy in the docker subdirectory. You resulting subdirectory structure will look like:

The steps and commands are as follows (_ASSUMING_ you have removed or renamed the `cdk` directory):

1. Create the cdk directory and enter it: 
> `mkdir cdk && cd cdk`
2. Create the CDK project skeleton with appropriate virtual env, latest Pip :
> `cdk init app --language python && python3 -m venv .env && source .env/bin/activate && pip install --upgrade pip && pip install -r requirements.txt`
3. Add the project specific CDK Python requirements. In our case these are `aws_ec2`, `aws_ecs`, and `aws_ecs_patterns`:
> `pip install aws_cdk.aws_ec2 aws_cdk.aws_ecs aws_cdk.aws_ecs_patterns && pip freeze > requirements.txt`
4. Copy the `streamlit-docker` file under `cdk` with 
> `cp -r ../streamlit-docker .`
5. **The non-trival Part**: you'll need to modify the auto-generated stack file `./cdk/cdk/cdk_stack.py` to define the needed infrastructure as code. Of all the technologies I've used so far, AWS CDK is the least painful -I only had to [add 3 imports](https://github.com/tzaffi/streamlit-cdk-fargate/blob/master/cdk_stack.py#L2) and write about [20 lines of code](https://github.com/tzaffi/streamlit-cdk-fargate/blob/master/cdk_stack.py#L14). I recommend you investigate the [AWS ECS Patterns](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_ecs_patterns.html) as there are a lot of pre-canned stacks. I used the `aws_ecs_patterns.ApplicationLoadBalancedFargateService` pattern to do most of the heavy lifting.


```bash
.
├── cdk
    ├── requirements.txt
    ├── ... a bunch of other autogenerated files you don't have to worry about for this POC ...
    ├── cdk
    |    ├── cdk_stack.py
    |    ├── __init__.py
    ├── streamlit-docker
    |    ├── Dockerfile
    |    ├── docker-compose.yml
    |    |── requirements.txt
    |    ├── app.py
```

## Deploy the Streamlit Docker Image on Fargate using CDK
With the driectory structure and stack definition in place run the command

>`cdk deploy` 

(NOTE: you may get [an error](https://github.com/aws/aws-cdk/issues/3091) the first time you run this command. If so, `cdk bootstrap` usually fixes this).

This will take around 10 minutes to complete. At the end of the process you will see an autogenerated "Outputs" URL printed out where the app has been deployed to. It will look something like:

```
 ✅  cdk

Outputs:
cdk.ZephFargateServiceServiceURL926BB193 = http://cdk-ZephFar-1VMAS051TQX7J-1682243052.us-east-1.elb.amazonaws.com
cdk.ZephFargateServiceLoadBalancerDNS9CEA5134 = cdk-ZephFar-1VMAS051TQX7J-1682243052.us-east-1.elb.amazonaws.com

Stack ARN:
... ELIDED ...
```

Simply copy/past that URL into your browser and you 

## Tear it Down!
Don't forget that [all the resources](#resources-that-cdk-configured-and-stood-up) that CDK stood up for you are costing you money and wasting energy when not used. Thankfully, tearing down the entire stack is as simple as:

> `cdk destroy`

## Explanation of the CDK Stack Definition `cdk_stack.py`

#### Set up a VPC (Virtual Private Cloud) where all the resources will live securely and add an EC2 Cluster to it
```python
vpc = ec2.Vpc(
    self, "ZephStreamlitVPC", 
    max_azs = 2, # default is all AZs in region, 
    )

cluster = ecs.Cluster(self, "ZephStreamlitCluster", vpc=vpc)
```

#### Build Dockerfile from local folder and push to ECR
```python
        image = ecs.ContainerImage.from_asset('streamlit-docker')
```

#### Use an ecs_patterns recipe to do all the rest!

You need configure the ALB (Application Load Balancer) to redirect traffic into the Streamlit Docker's port 8501:

```python
        ecs_patterns.ApplicationLoadBalancedFargateService(self, "ZephFargateService",
            cluster=cluster,            # Required
            cpu=256,                    # Default is 256
            desired_count=1,            # Default is 1
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=image, container_port=8501),  # Docker exposes 8501 for streamlit
            memory_limit_mib=512,       # Default is 512
            public_load_balancer=True,  # Default is False
        )  
```


## The Amazing Power of CDK

You can see from the [resources output](#resources-that-cdk-configured-and-stood-up), that those 20 lines delivered quite a punch. In particular, the following resources were all stood up and configured:
* EC2:
  * VPC
  * Subnet
  * RouteTable
  * SubnetRoutTableAssociation
  * Route
  * NatGateway
  * InternetGateway
  * Cluster
  * SecurityGroup
  * Service
  * ServiceGroup
* ELB
* IAM:
  * Role
  * Policy
* ECS:
  * TaskDefinition
* Logs:
  * LogGroups
  
## Resources that CDK Configured and Stood Up

```
Resources
[+] AWS::EC2::VPC ZephStreamlitVPC ZephStreamlitVPC40984819
[+] AWS::EC2::Subnet ZephStreamlitVPC/PublicSubnet1/Subnet ZephStreamlitVPCPublicSubnet1Subnet1A2AA945
[+] AWS::EC2::RouteTable ZephStreamlitVPC/PublicSubnet1/RouteTable ZephStreamlitVPCPublicSubnet1RouteTable583840F3
[+] AWS::EC2::SubnetRouteTableAssociation ZephStreamlitVPC/PublicSubnet1/RouteTableAssociation ZephStreamlitVPCPublicSubnet1RouteTableAssociationB4A02982
[+] AWS::EC2::Route ZephStreamlitVPC/PublicSubnet1/DefaultRoute ZephStreamlitVPCPublicSubnet1DefaultRoute082FB422
[+] AWS::EC2::EIP ZephStreamlitVPC/PublicSubnet1/EIP ZephStreamlitVPCPublicSubnet1EIP026E0E8A
[+] AWS::EC2::NatGateway ZephStreamlitVPC/PublicSubnet1/NATGateway ZephStreamlitVPCPublicSubnet1NATGateway44972270
[+] AWS::EC2::Subnet ZephStreamlitVPC/PublicSubnet2/Subnet ZephStreamlitVPCPublicSubnet2Subnet9BBD958C
[+] AWS::EC2::RouteTable ZephStreamlitVPC/PublicSubnet2/RouteTable ZephStreamlitVPCPublicSubnet2RouteTableC9BCFF80
[+] AWS::EC2::SubnetRouteTableAssociation ZephStreamlitVPC/PublicSubnet2/RouteTableAssociation ZephStreamlitVPCPublicSubnet2RouteTableAssociationC7002783
[+] AWS::EC2::Route ZephStreamlitVPC/PublicSubnet2/DefaultRoute ZephStreamlitVPCPublicSubnet2DefaultRoute6BA5157F
[+] AWS::EC2::EIP ZephStreamlitVPC/PublicSubnet2/EIP ZephStreamlitVPCPublicSubnet2EIP75476F55
[+] AWS::EC2::NatGateway ZephStreamlitVPC/PublicSubnet2/NATGateway ZephStreamlitVPCPublicSubnet2NATGatewayA8222148
[+] AWS::EC2::Subnet ZephStreamlitVPC/PrivateSubnet1/Subnet ZephStreamlitVPCPrivateSubnet1SubnetD20AE7C7
[+] AWS::EC2::RouteTable ZephStreamlitVPC/PrivateSubnet1/RouteTable ZephStreamlitVPCPrivateSubnet1RouteTable8052D12D
[+] AWS::EC2::SubnetRouteTableAssociation ZephStreamlitVPC/PrivateSubnet1/RouteTableAssociation ZephStreamlitVPCPrivateSubnet1RouteTableAssociation0FCA6A09
[+] AWS::EC2::Route ZephStreamlitVPC/PrivateSubnet1/DefaultRoute ZephStreamlitVPCPrivateSubnet1DefaultRoute41382E72
[+] AWS::EC2::Subnet ZephStreamlitVPC/PrivateSubnet2/Subnet ZephStreamlitVPCPrivateSubnet2SubnetBB9AB52B
[+] AWS::EC2::RouteTable ZephStreamlitVPC/PrivateSubnet2/RouteTable ZephStreamlitVPCPrivateSubnet2RouteTableE7CB2958
[+] AWS::EC2::SubnetRouteTableAssociation ZephStreamlitVPC/PrivateSubnet2/RouteTableAssociation ZephStreamlitVPCPrivateSubnet2RouteTableAssociation15975235
[+] AWS::EC2::Route ZephStreamlitVPC/PrivateSubnet2/DefaultRoute ZephStreamlitVPCPrivateSubnet2DefaultRoute8643277B
[+] AWS::EC2::InternetGateway ZephStreamlitVPC/IGW ZephStreamlitVPCIGW73E796BB
[+] AWS::EC2::VPCGatewayAttachment ZephStreamlitVPC/VPCGW ZephStreamlitVPCVPCGW683753A9
[+] AWS::ECS::Cluster ZephStreamlitCluster ZephStreamlitCluster66F13F7D
[+] AWS::ElasticLoadBalancingV2::LoadBalancer ZephFargateService/LB ZephFargateServiceLB23A42DAD
[+] AWS::EC2::SecurityGroup ZephFargateService/LB/SecurityGroup ZephFargateServiceLBSecurityGroupD2E76CD5
[+] AWS::EC2::SecurityGroupEgress ZephFargateService/LB/SecurityGroup/to streamlitZephFargateServiceSecurityGroup0445EB7E:8501 ZephFargateServiceLBSecurityGrouptostreamlitZephFargateServiceSecurityGroup0445EB7E850136068A37
[+] AWS::ElasticLoadBalancingV2::Listener ZephFargateService/LB/PublicListener ZephFargateServiceLBPublicListenerC7EDB90F
[+] AWS::ElasticLoadBalancingV2::TargetGroup ZephFargateService/LB/PublicListener/ECSGroup ZephFargateServiceLBPublicListenerECSGroupAAED681A
[+] AWS::IAM::Role ZephFargateService/TaskDef/TaskRole ZephFargateServiceTaskDefTaskRoleE745B9F4
[+] AWS::ECS::TaskDefinition ZephFargateService/TaskDef ZephFargateServiceTaskDef28DE8CEF
[+] AWS::Logs::LogGroup ZephFargateService/TaskDef/web/LogGroup ZephFargateServiceTaskDefwebLogGroupAD28A891
[+] AWS::IAM::Role ZephFargateService/TaskDef/ExecutionRole ZephFargateServiceTaskDefExecutionRole0B5B4D15
[+] AWS::IAM::Policy ZephFargateService/TaskDef/ExecutionRole/DefaultPolicy ZephFargateServiceTaskDefExecutionRoleDefaultPolicy48707106
[+] AWS::ECS::Service ZephFargateService/Service/Service ZephFargateService8DC10953
[+] AWS::EC2::SecurityGroup ZephFargateService/Service/SecurityGroup ZephFargateServiceSecurityGroupF8F38911
[+] AWS::EC2::SecurityGroupIngress ZephFargateService/Service/SecurityGroup/from streamlitZephFargateServiceLBSecurityGroup61AD5FB1:8501 ZephFargateServiceSecurityGroupfromstreamlitZephFargateServiceLBSecurityGroup61AD5FB185018651D62E
```


## Pre-requisites
* Get an [AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start) if you don't already have access to one
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) if you haven't already done so and make sure to configure the client to connect with your AWS account (use the command `aws configure` after installation)
* Install the [AWS CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html) on your machine. NOTE: You'll need to install Node.js as part of the process.
* Install `git` and `make` if you don't already have these on your machine. These will probably be installed if you're using Mac OS or Linux. On windows, you'll have to do a little more work. The last time I was using Windows (circa 2017), I enjoyed the [Chocalatey application](https://chocolatey.org/install) which made installing git and make as simple as running the following commands:
  * `choco install git`
  * `choco install make`
* Optional but highly recommended: For testing locally, you'll also need to install [docker-compose and docker](https://docs.docker.com/compose/install/)
* I'm also assuming that you have Python 3 and Pip installed. Otherwise, why would you have even been interested in this repo?


## Some Brief After Thoughts Comparing CDK with Other Infrastructure as Code Solutions
I was impressed by CDK's expressive power and choice of languages. It did create a lot of boilerplate. But once you knew _which_ boilerplate to modify, it was relatively painless. Playing around with [Pulumi](https://www.pulumi.com/), I would say it is more aesthetically pleasing (i.e. less boilerplate) than CDK, might even be more succinct, is available in several programming languages as well, and has the distinct advantage of working with all major cloud providers. Unfortunately, I feel that currently Pulumi is not ready for the prime time (i.e., I couln't get it to stand up the stack without any hitches). CDK is definitely less painful than the AWS CLI, AWS Cloudfront, and even Terraform. 
