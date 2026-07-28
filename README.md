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
<img width="1210" height="673" alt="image" src="https://github.com/user-attachments/assets/8458ec42-44c5-44d0-aa69-5e39907e9404" />

#### Add the following Internet route to the Routes of the Public Route table created:

<img width="1209" height="517" alt="image" src="https://github.com/user-attachments/assets/572ee0b3-e075-4dd3-bb89-301a65805a4c" />

<img width="1200" height="685" alt="image" src="https://github.com/user-attachments/assets/d0953ac2-c696-454e-8e49-6bb12b7072d1" />

###### What does adding 0.0.0.0/0 to the route table do?
###### Adding a route with the destination 0.0.0.0/0 allows resources within the associated subnet to communicate with any IPv4 address on the internet. By pointing this route to the Internet Gateway, AWS knows to forward all outbound internet traffic through the gateway. For example, when the Elastic Server needs to download Docker packages, retrieve Elastic Stack components, or connect to external repositories, this route enables those internet connections.

#### Add the following subnet associations to the Public route table created:

<img width="1918" height="712" alt="image" src="https://github.com/user-attachments/assets/0dcca8dd-e1a4-420c-b2e6-58db57972b20" />

##### Associate Public Subnets (previously created) to public route table:

<img width="1918" height="711" alt="image" src="https://github.com/user-attachments/assets/d1ff6e02-7721-4bbc-9fee-7d9b9d39b58d" />

###### Now the public route table is associated with the public subnets so that resources deployed within them inherit its routing rules. This allows internet-facing resources, such as the Elastic Stack and security scanning servers, to communicate with the internet through the Internet Gateway.


#### Create Private route table:

<img width="1917" height="729" alt="image" src="https://github.com/user-attachments/assets/a55b5dc4-6821-4612-a6af-f327e372ed71" />

##### Verify the private routes within the private route table:

<img width="1918" height="723" alt="image" src="https://github.com/user-attachments/assets/20c6c85c-cd08-4ff2-93e0-6114b4297725" />

__Do not add an Internet Gateway route.__ Keeping the private subnets isolated is intentional. This reflects how game backend services are typically deployed.

#### Associate Private Subnets (previously created) to private route table:

<img width="1918" height="727" alt="image" src="https://github.com/user-attachments/assets/b898b944-53de-4a20-bf76-112105c01f98" />

The private route table is associated with the private subnets to ensure that backend resources follow a separate set of routing rules from public-facing services. This network segmentation will help isolate Amazon EKS worker nodes and internal game services, providing a more secure and realistic cloud architecture.



### Launch the Elastic server EC2 instance

#### Objective:
Deploy an Ubuntu EC2 instance that will host:
- Elasticsearch
- Kibana
- Fleet Server
- Elastic Agent management

This server will become the central Security Operations platform for this project, where all Infrastructure-as-Code (IaC) security findings from Checkov and Trivy will be ingested, stored, and visualized.

#### Architecture:

<img width="340" height="558" alt="image" src="https://github.com/user-attachments/assets/98e569e6-30f0-412f-b648-3f08456e319a" />

#### Launching EC2 Instance:

<img width="1918" height="766" alt="image" src="https://github.com/user-attachments/assets/be25cdbc-6a01-45ab-8a39-bd9da1d47d45" />

##### Create the Key Pair

<img width="1918" height="766" alt="image" src="https://github.com/user-attachments/assets/48889158-dc8b-4d15-b31e-a2481e9d60e0" />

##### Configure the Network settings of the EC2 instance:

###### Set the VPC to GameInfraGuard-VPC (previously created) and the subnet to public-subnet-a

<img width="1918" height="721" alt="image" src="https://github.com/user-attachments/assets/fadcf69e-f877-46f6-99f2-80ddeeb08b89" />

##### Create security groups necessary 

###### SSH inbound rule: (for testing purposes I will be using Source type 0.0.0.0/0)
<img width="602" height="335" alt="image" src="https://github.com/user-attachments/assets/69f20972-ff89-413d-8a7e-5d9fdc13e7f9" />

###### Kibana webinterface:
<img width="599" height="152" alt="image" src="https://github.com/user-attachments/assets/2ee5788c-b0d3-42c1-90f4-ddfad1c95108" />

###### Elasticsearch: (Used for API access and testing.)
<img width="596" height="138" alt="image" src="https://github.com/user-attachments/assets/1b04ea57-2e51-44ac-88ca-bb5415d0ae98" />

###### Fleet Server (Agent Server):
<img width="601" height="147" alt="image" src="https://github.com/user-attachments/assets/7804e3a9-176c-4687-89cf-8fd90736d9ce" />


##### Configure Storage (volumes):
###### Increase the root/EBS volume storage capacity.
<img width="959" height="361" alt="image" src="https://github.com/user-attachments/assets/5e1ca559-0c7a-4d48-aa2e-4c1ee456427e" />

###### Elastic stores logs, dashboards, indices, and scan results, so extra storage prevents you from running out of space during the project.
