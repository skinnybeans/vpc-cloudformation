# VPCs

Started this as and exercise to layout and create a VPC from scratch using cloudformation.

As the layout of this kind of thing is somewhat complicated, the readme acts as a design doc to build against.

I'm not keen on having access from the public tier to the data tier, but as long as jump boxes are required for access this has to be there. The alternative might be splitting out a separate subnet for jump boxes. Ideally the app and data tiers would be accessable via a VPN, then no hosts need to run in the public tier at all.

## VPC cidr

10.0.0.0/19 (8192 available IPs)

First address: 10.0.0.1
Last address:  10.0.31.254

## Subnets

Each tier will have 3 subnets; one for each availability zone.

- Public
- Application
- Data
- NAT

### Public

#### Purpose

- Hosting load balancers.
- No instances should run here.
  - Possibly need to host a jump box here.

#### Connectivity

##### External

- Incoming for common web serving ports.
- May need SSH for jump boxes

##### Internal

- Ideally only communicate with app tier.
- May need SSH to data tier for jump boxes.
- No need for connection to NAT tier.

### App

#### Purpose

- Hosting EC2 instances
- Hosting private load balancers

#### Connectivity

##### External

- No incoming external traffic.
- All outgoing traffic routed though the NAT tier.

##### Internal

- Public
- Data
- NAT

### Data

#### Purpose

- Hosting data stores

#### Connectivity

##### External

- No external connectivity

##### Internal

- Public possibly needed for jump box access
- App
- NAT

### NAT

#### Purpose

- Hosting NAT gateways

#### Connectivity

##### External

- Outbound on common web ports
- No inbound

##### Internal

- App
- Data
- No public access required.

 ## Subnet CIDR ranges

 ### Public

 IPs allocated: 1536

| Name | CIDR | First address | Last address |
| --- | --- | --- | --- |
| pub-a: | 10.0.0.0/23 | 10.0.0.1 | 10.0.1.254 |
| pub-b: | 10.0.2.0/23 | 10.0.2.1 | 10.0.3.254 |
| pub-c: | 10.0.4.0/23 | 10.0.4.1 | 10.0.5.254 |
| reserved: | 10.0.6.0/23 | 10.0.6.1 | 10.0.7.254 |

### App

IPs allocated: 1536

| Name | CIDR | First address | Last address |
| --- | --- | --- | --- |
| app-a: | 10.0.8.0/23 | 10.0.8.1 | 10.0.9.254 |
| app-b: | 10.0.10.0/23 | 10.0.10.1 | 10.0.11.254 |
| app-c: | 10.0.12.0/23 | 10.0.12.1 | 10.0.13.254 |
| reserved: | 10.0.14.0/23 | 10.0.14.1 | 10.15.254 |

### Data

IPs allocated: 1536

| Name | CIDR | First address | Last address |
| --- | --- | --- | --- |
| data-a: | 10.0.16.0/23 | 10.0.16.1 | 10.0.17.254 |
| data-b: | 10.0.18.0/23 | 10.0.18.1 | 10.0.19.254 |
| data-c: | 10.0.20.0/23 | 10.0.20.1 | 10.0.21.254 |
| reserved: | 10.0.22.0/23 | 10.0.22.1 | 10.23.254 |

### NAT

IPs allocated: 48

| Name | CIDR | First address | Last address |
| --- | --- | --- | --- |
| nat-a: | 10.0.24.0/28 | 10.0.24.1 | 10.0.24.14 |
| nat-b: | 10.0.24.16/28 | 10.0.24.17 | 10.0.24.30 |
| nat-c: | 10.0.24.32/28 | 10.0.24.33 | 10.0.24.46 |
| reserved: | 10.0.24.48/28 | 10.0.24.48 | 10.24.62 |

## Route tables

### Public subnets

Name: public-table

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | igw |

### Application subnets

Each applications subnet will route to the NAT gateway it the corresponding AZ.

Name: app-table-a

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | nat-gateway-a |

Name: app-table-b

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | nat-gateway-b |

Name: app-table-c

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | nat-gateway-c |

### Data subnets

Each data subnet will route to the NAT gateway it the corresponding AZ.

Name: data-table-a

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | nat-gateway-a |

Name: data-table-b

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | nat-gateway-b |

Name: data-table-c

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | nat-gateway-c |

### NAT subnets

Name: nat-table

Destination | Target |
| --- | --- |
| 10.0.0.0/19 | local |
| 0.0.0.0/0 | igw |

## NAT gateways

- One nat gateway present for each availability zone.
- Sits in the NAT subnet.

## NACLs

### Public subnet

- Prevent subnets that shouldn't be communicating directly with the public tier.
- Only permit incoming connections on desired ports.

- Need to open ephermal ports inbound 1024-65535

#### Inbound

- Ephermal ports at top so response traffic for SSH into data tier will work.

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.18.0/26 | Deny | Deny access from NAT tier |
| TCP | 1024-65535 | Allow | ephermal ports for response traffic|
| Custom | all | 10.0.12.0/21 | Deny | Deny access from data tier |
| HTTP | 80 | 0.0.0.0/0 | Allow | Accept all http traffic |
| HTTPS | 443 | 0.0.0.0/0 | Allow | Accept all https traffic |
| SSH | 22 | 0.0.0.0/0 | Allow | Accept all ssh traffic. <br/> Only required for jumpboxes |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Destination | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.18.0/26 | Deny | Deny access to NAT tier |
| SSH | 22 | 0.0.0.0/0 | Allow | Allow SSH for jumpboxes |
| Custom | all | 10.0.12.0/21 | Deny | Deny access to data tier |
| HTTP | 80 | 0.0.0.0/0 | Allow | Allow HTTP traffic |
| HTTPS | 443 | 0.0.0.0/0 | Allow | Allow HTTPS traffic |
| TCP | 1024-65535 | 0.0.0.0/0 | Allow | Allow ephermal ports |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

### App

- Public internet traffic is not routed to the app tier.
- Hence the NACLs can be more relaxed.

#### Inbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| SSH | 22 | 0.0.0.0/0 | Allow | Accept all ssh traffic. <br/> Only required for jumpboxes |
| HTTP | 80 | 0.0.0.0/0 | Allow | Allow HTTP traffic |
| HTTPS | 443 | 0.0.0.0/0 | Allow | Allow HTTPS traffic |
| TCP | 1024-65535 | 0.0.0.0/0 | Allow | Allow ephermal ports |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Destination | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| HTTP | 80 | 0.0.0.0/0 | Allow | Allow HTTP traffic |
| HTTPS | 443 | 0.0.0.0/0 | Allow | Allow HTTPS traffic |
| TCP | 1024-65535 | 0.0.0.0/0 | Allow | Allow ephermal ports |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

### Data

- Prevent subnets that shouldn't be communicating directly with the data tier.
- Allow http and https for dynamo

#### Inbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| SSH | 22 | 10.0.0.0/21 | Allow | Allow ssh from jumpbox in public tier |
| Custom | all | 10.0.0.0/21 | Deny | Deny all traffic from public tier |
| HTTP | 80 | 10.0.8.0/21 | Allow | Allow HTTP traffic for DynamoDB |
| HTTPS | 443 | 10.0.8.0/21 | Allow | Allow HTTPS traffic for DynamoDB |
| MySQL | 3306 | 10.0.8.0/21 | Allow | Allow access on default MySQL port |
| TCP | 1024-65535 | 0.0.0.0/0 | Allow | Allow ephermal ports |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Destination | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| HTTP | 80 | 10.0.8.0/21 | Allow | Allow HTTP traffic |
| HTTPS | 443 | 10.0.8.0/21 | Allow | Allow HTTPS traffic |
| TCP | 1024-65535 | 0.0.0.0/0 | Allow | Allow ephermal ports 
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

### NAT

- Prevent subnets that shouldn't be communicating directly with the data tier.
- Incoming connections should only be on http and https ports

#### Inbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.0.0/21 | Deny | Deny access to public tier |
| HTTP | 80 | 0.0.0.0/0 | Allow | Accept all http traffic |
| HTTPS | 443 | 0.0.0.0/0 | Allow | Accept all https traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.0.0/21 | Deny | Deny access to public tier |
| Custom | all | 0.0.0.0/0 | Allow | Allow all traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |
