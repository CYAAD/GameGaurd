# GameGaurd
> Continuous Infrastructure Security Monitoring for AWS EKS-Based Game Platforms

## Purpose

This project is designed to understand how security vulnerabilities and misconfigurations in cloud-native game infrastructure can be identified, monitored, and visualized using Elastic.

It simulates a modern AWS and Kubernetes-based gaming environment and demonstrates how Infrastructure-as-Code (IaC) security scanning tools like Checkov and Trivy can detect misconfigurations, and how these findings can be centralized into Elasticsearch and visualized through Kibana dashboards.

The goal is to gain hands-on experience in securing cloud-native game systems, building security monitoring pipelines, and translating IaC security findings into actionable insights for DevSecOps, game studios, and MSSP environments.


## Lab Setup
### What we need
- Create and configure a VPC
- Create and configure subnets
- Create Internet Gateway
- Create and configure Route tables
- Deploy EC2 instances ( 2 VMs - 1 for Elastic, 1 for Scanner like trivy and checkov)
- Create and configure EKS cluster
- Create and configure S3 bucket (this is where scan results will be stored)

### Create and Configure VPC on AWS

A dedicated Virtual Private Cloud (VPC) was created to simulate a realistic cloud-native gaming environment. The VPC provides network isolation and serves as the foundation for hosting the Elastic Stack, IaC scanning infrastructure, and the Amazon EKS cluster. Public and private subnets were configured to separate internet-facing resources from backend game services, following security best practices commonly used in modern game studio cloud architectures.


<img width="1512" height="870" alt="image" src="https://github.com/user-attachments/assets/f38a82ba-3b77-4566-938b-089a030420cf" />

<img width="1889" height="715" alt="image" src="https://github.com/user-attachments/assets/bbb4db4e-74e7-41d6-a20c-c20481360a9c" />



### Create and Configure multiple Subnets within the VPC

Public and private subnets were created across multiple Availability Zones to simulate a production-grade cloud environment for online gaming services. Public subnets to host the internet-facing components such as the Elastic Stack and security scanning infrastructure, while private subnets host the Amazon EKS worker nodes and backend game services. This network segmentation follows cloud security best practices and provides a realistic lab environment for monitoring, detecting, and visualizing infrastructure security risks.

#### Public Subnets will host
- Elastic Server EC2
- Scanner Server EC2

#### Private Subnets will host
- EKS Worker Nodes
- Redis
- Matchmaking Service
- Game API
- Leaderboard Service

#### Subnets created: (In my case)
- Public subnet a (10.0.1.0/24)
- Public subnet b (10.0.2.0/24)
- Private subnet a (10.0.11.0/24)
- Private subnet b (10.0.12.0/24)

<img width="1910" height="811" alt="image" src="https://github.com/user-attachments/assets/126115d2-7538-4d7c-aadf-b155a0f19468" />

<img width="459" height="617" alt="image" src="https://github.com/user-attachments/assets/2cd5713b-ee5c-4ff0-ac7c-78e40c4b049d" />,

#### Add Tags to the private subnets that will host EKS worker nodes and backend game services
> EKS uses subnet tags to determine where to place resources.

##### Tags to add to subnets created:
<img width="1300" height="620" alt="image" src="https://github.com/user-attachments/assets/e848ed46-c343-4232-abd7-fd8b75719403" />
<img width="1289" height="615" alt="image" src="https://github.com/user-attachments/assets/801c9cf5-fdc7-416a-ae8f-23d287c4619f" />

These tags will allow EKS to create public load balancers when later game services are exposed down the line in this project.


### Create Internet Gateway

An Internet Gateway (IGW) will need to be created and attached to the VPC to enable communication between resources within the VPC and the public internet. This component provides the connectivity required for internet-facing services, such as the Elastic Stack and security scanning infrastructure, while serving as a foundational networking element for the cloud-native gaming environment.

<img width="1300" height="672" alt="image" src="https://github.com/user-attachments/assets/b55defbd-1d20-40a7-8dbd-a3c470818dc2" />

<img width="987" height="527" alt="image" src="https://github.com/user-attachments/assets/277c40d6-b10f-49df-94ad-92a199b77aab" />


### Create Route Tables

Route tables will be created to control how network traffic flows within the VPC and between AWS resources and the internet. Separate route tables will be configured for public and private subnets to enforce network segmentation, ensuring that internet-facing services remain isolated from backend game infrastructure. This setup will provide a realistic cloud architecture for the project and enable the identification, monitoring, and visualization of network-related security misconfigurations within the AWS environment.

#### For this project we will create the following Route Tables:
- Public Route Table -> Elastic Server & Scanner VM
- Private Route Table -> EKS Worker Nodes

#### Create Public route table:
<img width="1307" height="730" alt="image" src="https://github.com/user-attachments/assets/08e097ce-df00-40a2-bdbe-eeb14efda395" />

#### Add the following Internet route to the Routes of the Public Route table created:

<img width="1907" height="813" alt="image" src="https://github.com/user-attachments/assets/d298a3b0-2ccc-427b-b777-4f44b4890dd7" />

<img width="1298" height="736" alt="image" src="https://github.com/user-attachments/assets/e43e3f4e-a052-4b59-968a-c060e0ddf5b4" />

###### What does adding 0.0.0.0/0 to the route table do?
###### Adding a route with the destination 0.0.0.0/0 allows resources within the associated subnet to communicate with any IPv4 address on the internet. By pointing this route to the Internet Gateway, AWS knows to forward all outbound internet traffic through the gateway. For example, when the Elastic Server needs to download Docker packages, retrieve Elastic Stack components, or connect to external repositories, this route enables those internet connections.

#### Add the following subnet associations to the Public route table created:

<img width="1918" height="712" alt="image" src="https://github.com/user-attachments/assets/0dcca8dd-e1a4-420c-b2e6-58db57972b20" />

##### Associate Public Subnets (previously created):

<img width="1918" height="711" alt="image" src="https://github.com/user-attachments/assets/d1ff6e02-7721-4bbc-9fee-7d9b9d39b58d" />

###### Now the public route table is associated with the public subnets so that resources deployed within them inherit its routing rules. This allows internet-facing resources, such as the Elastic Stack and security scanning servers, to communicate with the internet through the Internet Gateway.


#### Create Private route table:

<img width="1917" height="729" alt="image" src="https://github.com/user-attachments/assets/a55b5dc4-6821-4612-a6af-f327e372ed71" />

##### Verify the private routes within the private route table:

<img width="1918" height="723" alt="image" src="https://github.com/user-attachments/assets/f9976141-da48-4cde-a741-42176bee94a9" />

__Do not add an Internet Gateway route.__ Keeping the private subnets isolated is intentional. This reflects how game backend services are typically deployed.
