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
{"type":"excalidraw/clipboard","workspaceId":"uJOD6w10ad1INaNp3vkE","elements":[{"0":525,"1":340,"renderVersion":"20260727","strokeColor":"#d7d9dc","fillStyle":"solid","backgroundColor":"transparent","strokeWidth":1,"strokeStyle":"solid","roughness":1,"opacity":100,"strokeSharpness":"sharp","version":23,"isDeleted":false,"id":"EyzkBioyLsBKYrTA2tQ_","code":"","x":85,"y":15,"diagramType":"freeform-diagram","forceAiMode":false,"isBeingGenerated":false,"lastEditMode":"ai","scale":1,"type":"diagram","width":250,"height":690,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":162975552,"zIndex":0,"title":"System Architecture Diagram","modifiedAt":1785150009170,"isSyntaxMissing":false},{"strokeColor":"#1c1c1c","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":1,"strokeStyle":"solid","strokeSharpness":"round","opacity":100,"roughness":1,"shouldApplyRoughness":true,"isDeleted":false,"diagramId":"EyzkBioyLsBKYrTA2tQ_","figureId":null,"id":"3138606de89352a8193d8b89f1de7cca","x":100,"y":30,"diagramEntityId":"title","isContainer":false,"freeform":{"tag":"Textbox","text":"**Architecture**","fontSize":18,"hAlign":"left","fixedWidth":true},"compound":{"type":"parent","containerType":"freeform"},"type":"freeform","width":102.888905843099,"height":18.65625,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":1050337088,"version":30,"zIndex":1,"modifiedAt":1785150035511},{"strokeColor":"#1c1c1c","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":1,"strokeStyle":"solid","strokeSharpness":"round","opacity":100,"roughness":1,"shouldApplyRoughness":true,"isDeleted":false,"diagramId":"EyzkBioyLsBKYrTA2tQ_","figureId":null,"id":"388aff61a38769d8c4b9c34892b196d6","x":100,"y":90,"diagramEntityId":"checkov","isContainer":false,"freeform":{"tag":"Shape","shape":"rectangle","texts":[{"text":"Checkov & Trivy\n","fontSize":16}],"icon":"shield-check","bgColor":"#e8f0fe","borderColor":"#4a6fa5"},"compound":{"type":"parent","containerType":"freeform"},"type":"freeform","width":220,"height":70,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":880129216,"version":13,"zIndex":2,"modifiedAt":1785150009170},{"strokeColor":"#1c1c1c","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":1,"strokeStyle":"solid","strokeSharpness":"round","opacity":100,"roughness":1,"shouldApplyRoughness":true,"isDeleted":false,"diagramId":"EyzkBioyLsBKYrTA2tQ_","figureId":null,"id":"cb50fa8ea887b522e2bfcc28ac6d801e","x":100,"y":210,"diagramEntityId":"agent","isContainer":false,"freeform":{"tag":"Shape","shape":"rectangle","texts":[{"text":"Elastic Agent","fontSize":16}],"icon":"elastic","bgColor":"#fef3e8","borderColor":"#d4915a"},"compound":{"type":"parent","containerType":"freeform"},"type":"freeform","width":220,"height":70,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":1048377152,"version":3,"zIndex":3,"modifiedAt":1785150009170},{"id":"b2aa8823f62df34efc53d4029b17cc47","type":"arrow","x":210,"y":160,"points":[[0,0],[0,50]],"diagramId":"EyzkBioyLsBKYrTA2tQ_","diagramEntityId":"r1","backgroundColor":"transparent","fillStyle":"solid","strokeSharpness":"elbow","roughness":0,"opacity":100,"arrowHeadSize":12,"cardinalElbowData":{"isEnabled":true,"preferredSegmentDirections":["down"]},"freeform":{"tag":"Relationship","from":"checkov","fromPort":"bottom","to":"agent","toPort":"top","label":"IaC scan results","labelPlacement":{"x":-45.5,"y":17.5}},"strokeColor":"#1c1c1c","strokeWidth":0.75,"strokeStyle":"solid","startArrowhead":null,"endArrowhead":"triangle","lastCommittedPoint":null,"startBinding":{"elementId":"388aff61a38769d8c4b9c34892b196d6","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"down"}},"endBinding":{"elementId":"cb50fa8ea887b522e2bfcc28ac6d801e","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"up"}},"width":0,"height":50,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":849216704,"version":10,"isDeleted":false,"compound":{"type":"parent","containerType":"freeform-relationship"},"zIndex":4,"modifiedAt":1785150009170,"textGap":[-46.5,16.5,93,17]},{"strokeColor":"#1c1c1c","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":1,"strokeStyle":"solid","strokeSharpness":"round","opacity":100,"roughness":1,"shouldApplyRoughness":true,"isDeleted":false,"diagramId":"EyzkBioyLsBKYrTA2tQ_","figureId":null,"id":"6f0ffe443180fa1c550f523efc747cc4","x":100,"y":330,"diagramEntityId":"elasticsearch","isContainer":false,"freeform":{"tag":"Shape","shape":"cylinder","texts":[{"text":"Elasticsearch","fontSize":16}],"icon":"elasticsearch","bgColor":"#e8f7ee","borderColor":"#5aa570"},"compound":{"type":"parent","containerType":"freeform"},"type":"freeform","width":220,"height":80,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":761339072,"version":3,"zIndex":5,"modifiedAt":1785150009170},{"id":"608d3542aa0e538a81a48148eb0bcbe8","type":"arrow","x":210,"y":280,"points":[[0,0],[0,50]],"diagramId":"EyzkBioyLsBKYrTA2tQ_","diagramEntityId":"r2","backgroundColor":"transparent","fillStyle":"solid","strokeSharpness":"elbow","roughness":0,"opacity":100,"arrowHeadSize":12,"cardinalElbowData":{"isEnabled":true,"preferredSegmentDirections":["down"]},"freeform":{"tag":"Relationship","from":"agent","fromPort":"bottom","to":"elasticsearch","toPort":"top","label":"ingest","labelPlacement":{"x":-17.5,"y":17,"width":35,"height":16}},"strokeColor":"#1c1c1c","strokeWidth":0.75,"strokeStyle":"solid","startArrowhead":null,"endArrowhead":"triangle","lastCommittedPoint":null,"startBinding":{"elementId":"cb50fa8ea887b522e2bfcc28ac6d801e","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"down"}},"endBinding":{"elementId":"6f0ffe443180fa1c550f523efc747cc4","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"up"}},"width":0,"height":50,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":1371645760,"version":6,"isDeleted":false,"compound":{"type":"parent","containerType":"freeform-relationship"},"zIndex":6,"modifiedAt":1785150009171,"textGap":[-18.5,16,37,18]},{"strokeColor":"#1c1c1c","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":1,"strokeStyle":"solid","strokeSharpness":"round","opacity":100,"roughness":1,"shouldApplyRoughness":true,"isDeleted":false,"diagramId":"EyzkBioyLsBKYrTA2tQ_","figureId":null,"id":"63c66abbf60e4519df51b96dd14f7e62","x":100,"y":460,"diagramEntityId":"kibana","isContainer":false,"freeform":{"tag":"Shape","shape":"rectangle","texts":[{"text":"Kibana","fontSize":16}],"icon":"kibana","bgColor":"#f3e8fe","borderColor":"#8a5ad4"},"compound":{"type":"parent","containerType":"freeform"},"type":"freeform","width":220,"height":70,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":710676288,"version":3,"zIndex":7,"modifiedAt":1785150009171},{"id":"e886329643bf039886c8a8d8c8bfa6d6","type":"arrow","x":210,"y":410,"points":[[0,0],[0,50]],"diagramId":"EyzkBioyLsBKYrTA2tQ_","diagramEntityId":"r3","backgroundColor":"transparent","fillStyle":"solid","strokeSharpness":"elbow","roughness":0,"opacity":100,"arrowHeadSize":12,"cardinalElbowData":{"isEnabled":true,"preferredSegmentDirections":["down"]},"freeform":{"tag":"Relationship","from":"elasticsearch","fromPort":"bottom","to":"kibana","toPort":"top","label":"query","labelPlacement":{"x":-16,"y":17,"width":32,"height":16}},"strokeColor":"#1c1c1c","strokeWidth":0.75,"strokeStyle":"solid","startArrowhead":null,"endArrowhead":"triangle","lastCommittedPoint":null,"startBinding":{"elementId":"6f0ffe443180fa1c550f523efc747cc4","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"down"}},"endBinding":{"elementId":"63c66abbf60e4519df51b96dd14f7e62","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"up"}},"width":0,"height":50,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":1644129472,"version":6,"isDeleted":false,"compound":{"type":"parent","containerType":"freeform-relationship"},"zIndex":8,"modifiedAt":1785150009171,"textGap":[-17,16,34,18]},{"strokeColor":"#1c1c1c","backgroundColor":"transparent","fillStyle":"solid","strokeWidth":1,"strokeStyle":"solid","strokeSharpness":"round","opacity":100,"roughness":1,"shouldApplyRoughness":true,"isDeleted":false,"diagramId":"EyzkBioyLsBKYrTA2tQ_","figureId":null,"id":"64df2110b77bb28285db0cfdf74e1904","x":100,"y":580,"diagramEntityId":"outputs","isContainer":false,"freeform":{"tag":"Shape","shape":"rectangle","texts":[{"text":"**Dashboards**","fontSize":15},{"text":"**Alerts**","fontSize":15},{"text":"**Analytics**","fontSize":15}],"icon":"layout-dashboard","vAlign":"middle","bgColor":"#fef8e8","borderColor":"#c9a94a"},"compound":{"type":"parent","containerType":"freeform"},"type":"freeform","width":220,"height":110,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":1151304896,"version":3,"zIndex":9,"modifiedAt":1785150009171},{"id":"f8bed54dc04a14706480a8fa05e5ebca","type":"arrow","x":210,"y":530,"points":[[0,0],[0,50]],"diagramId":"EyzkBioyLsBKYrTA2tQ_","diagramEntityId":"r4","backgroundColor":"transparent","fillStyle":"solid","strokeSharpness":"elbow","roughness":0,"opacity":100,"arrowHeadSize":12,"cardinalElbowData":{"isEnabled":true,"preferredSegmentDirections":["down"]},"freeform":{"tag":"Relationship","from":"kibana","fromPort":"bottom","to":"outputs","toPort":"top","label":"visualize","labelPlacement":{"x":-25,"y":17,"width":50,"height":16}},"strokeColor":"#1c1c1c","strokeWidth":0.75,"strokeStyle":"solid","startArrowhead":null,"endArrowhead":"triangle","lastCommittedPoint":null,"startBinding":{"elementId":"63c66abbf60e4519df51b96dd14f7e62","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"down"}},"endBinding":{"elementId":"64df2110b77bb28285db0cfdf74e1904","bindingType":"portOrCenter","portLocationOptions":{"portLocation":"varying.CardinalDirection","preferredDirection":"up"}},"width":0,"height":50,"angle":0,"groupIds":[],"lockedGroupId":null,"seed":1368047424,"version":6,"isDeleted":false,"compound":{"type":"parent","containerType":"freeform-relationship"},"zIndex":10,"modifiedAt":1785150009171,"textGap":[-26,16,52,18]}],"diagramMetadata":{"settings":{},"diagramType":"freeform-diagram","diagramId":"EyzkBioyLsBKYrTA2tQ_","entitySettings":{}}}
