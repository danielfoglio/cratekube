[![License](http://img.shields.io/badge/license-apache%202.0-yellow)](http://choosealicense.com/licenses/apache-2.0/)
[![Sponsored By Cisco](https://img.shields.io/badge/sponsored%20by-Cisco-blue)](https://www.cisco.com/c/en/us/solutions/cloud/multicloud-solutions.html)
[![Coded with Groovy](https://img.shields.io/badge/language-Groovy-green)](https://github.com/apache/groovy)
[![12 Factor App](https://img.shields.io/badge/app-12--factor-yellow)](https://12factor.net/)

# Cloud Management Service
_A service to manage and monitor assets in cloud platforms_

## Introduction
This **_cloud management service_** is part of an [MVaP architecture](https://github.com/cratekube/cratekube/blob/master/docs/Architecture.md) and set of [requirements](https://github.com/cratekube/cratekube/blob/master/docs/Requirements.md) for [CrateKube](https://cratekube.github.io/) that creates infrastructure [VPC](https://aws.amazon.com/vpc/)s, bootstraps, and configures [Kubernetes](https://kubernetes.io/) clusters on AWS [EC2](https://aws.amazon.com/ec2/pricing/) using [CloudFormation](https://aws.amazon.com/cloudformation/) templates and [Terraform](https://www.terraform.io/).  The underlying objective of our product is to provide default secure, ephemeral, cloud-ready instances that will launch on [AWS](https://aws.amazon.com/ec2/).  This approach is based on an Open Source Software initiative within Cisco called [NoOps](https://www.cio.com/article/3407714/what-is-noops-the-quest-for-fully-automated-it-operations.html), which takes a first iterative step towards modifying the IT lifecycle in order for zero human intervention to be necessary for mundane orchestration tasks.

## What does this service do?
The cloud-mgmt-service is in charge of provisioning and monitoring all cloud resources and services. Cloud resources are loosely defined as managed virtual infrastructure, or "Infrastructure as code".

In AWS, this is expected to provision any and all resources necessary to prepare for the creation of a [Kubernetes cluster](https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/), including but not limited to [IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html), [VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html), [EC2 instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html), [subnets](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html#vpc-subnet-basics), [security groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html), and security policies. 

When utilized as a component, this service can act as a stand-alone service for creating infrastructure as code, and can be easily extended by forking and [developing](https://github.com/cratekube/cratekube/blob/master/docs/Development.md) as a [CrateKube contributor](https://github.com/cratekube/cratekube/blob/master/CONTRIBUTING.md).

## Requirements for deployment on AWS
- Requires SuperUser permission to an AWS account using the IAM keys of your choice.   
- Terraform templates will need to have configuration options that support the compliance rules implemented in the [policy-mgmt-service](https://github.com/cratekube/policy-mgmt-service). 
- The state files generated by Terraform must be persisted.  If running this service on EC2, you must persist the data in a volume in order for Docker to attach to it and save the state.

## How this service can be used
To run this service locally, one may simply fork this repo and clone down to your local machine:
- Visit the [cloud-mgmt-service](https://github.com/cratekube/cloud-mgmt-service)
- Press the `fork` button at the top-right of the page
- Clone down the repo locally:
```html
git clone git@github.com:<yourUsername>/cloud-mgmt-service.git
```
If you were to use this service on a host, you could perform similar steps.  In both cases, you will want to be sure and persist any state data in a volume.  See below for more information.

### Building locally with Docker
We strive to have our builds repeatable across development environments so we also provide a Docker build to generate 
the Dropwizard application container.  The examples below should be executed from the root of the project.

#### Configuration
The DropWizard [app.yml](https://github.com/cratekube/cloud-mgmt-service/blob/master/app.yml) can be configured dynamically using environment variables:

```html
$ CONFIG_DIR=/app/config
$ SSH_PUBLIC_KEY=<public key>
$ ADMIN_APIKEY=<api key>
```
These environment variables are the preferred method of configuration at runtime.

##### Run the base docker build:
```bash
docker build -t cloud-mgmt-service:local --target build .
```
Note: This requires docker 19.03.x or above.  Docker 18.09 will throw errors for mount points and the `--target` flag.

##### Build the package target:
```bash
docker build -t cloud-mgmt-service:local --target package .
```
### Running locally with Docker
##### Run the docker application locally on port 8080:
```bash
docker run -p 8080:9000 -v /home/user/tfstate:/app/config -d cloud-mgmt-service:local
```
Note: We are bind mounting the `/home/user/tfstate` directory inside of the container to preserve the state of the Terraform installation locally on the host.  If you don't do that, whenever your container dies you will lose your infrastructure state files.  **That's bad!**  _Without state, the next time Terraform runs it could remove infrastructure it doesn't know it exists._

If you don't want to preserve state, you can just remove the volume flag.

##### Fire up the Swagger specification by visiting the following URL in a browser:
```bash
http://localhost:8080/swagger
```

### Running on AWS
This microservice can also run on an Amazon EC2 instance:
- Find an [Ubuntu Cloud AMI](https://cloud-images.ubuntu.com/locator/ec2/) and spin it up in your AWS EC2 console.
- [Install Docker](https://docs.docker.com/engine/install/ubuntu/) 
- [Push](https://docs.docker.com/docker-hub/) your locally built docker image to a registry
- [Configure](https://github.com/cratekube/cratekube/blob/master/docs/user/ServiceCloudManagement.md#configuration) the necessary environment variables
- Set up your [EC2 security groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html) to protect this app from the outside world
- Pull the docker image from dockerhub i.e.- `docker pull yourUser/yourImage:tag`
- [Run](https://github.com/cratekube/cratekube/blob/master/docs/user/ServiceCloudManagement.md#run-the-docker-application-locally-on-port-8080) the container

The cloud management service will become available at:
```html
http://<ec2 instance public dns name>:8080/swagger
```


### Using the API locally
The API has endpoints that allow you to create, read, and delete environments.  In order for these operations to be successful, your app.yml file must have been properly configured to utilize your AWS account with the appropriate key.  That API key must have access to all the resources necessary to configure a full VPC.

The resulting operations exist as REST endpoints, which you can simply hit in your browser or with a tool such as [Postman](https://www.postman.com/downloads/).

| HTTP Verb | Endpoint | Payload | Function |
| --- | --- | --- | --- |  
| GET | /environment | None | Get a list of all environments |
| POST | /environment | <code>{ name": "string"}</code>  | Create an environment by specifying a name |
| GET | /environment/{environmentName} | None | Get a specific environment by name |
| DELETE | /environment/{environmentName} | None | Delete a specific environment by name |