# VPCs

Started this as and exercise to layout and create a VPC from scratch using cloudformation.

As the layout of this kind of thing is somewhat complicated, the readme acts as a design doc to build against.

## VPC cidr

10.0.0.0/19 (8192 available IPs)

First address: 10.0.0.1
Last address:  10.0.31.254

## Subnets

Each tier will have 3 subnets; one for each availability zone.

Tier are:

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

### App

IPs allocated: 1536

| Name | CIDR | First address | Last address |
| --- | --- | --- | --- |
| app-a: | 10.0.6.0/23 | 10.0.6.1 | 10.0.7.254 |
| app-b: | 10.0.8.0/23 | 10.0.8.1 | 10.0.9.254 |
| app-c: | 10.0.10.0/23 | 10.0.10.1 | 10.0.11.254 |

### Data

IPs allocated: 1536

| Name | CIDR | First address | Last address |
| --- | --- | --- | --- |
| data-a: | 10.0.12.0/23 | 10.0.12.1 | 10.0.13.254 |
| data-b: | 10.0.14.0/23 | 10.0.14.1 | 10.0.15.254 |
| data-c: | 10.0.16.0/23 | 10.0.16.1 | 10.0.17.254 |

### NAT

IPs allocated: 48

| Name | CIDR | First address | Last address |
| --- | --- | --- | --- |
| nat-a: | 10.0.18.0/28 | 10.0.18.1 | 10.0.18.14 |
| nat-b: | 10.0.18.16/28 | 10.0.18.17 | 10.0.18.30 |
| nat-c: | 10.0.18.32/28 | 10.0.18.33 | 10.0.18.46 |

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

out
- 123 ntp
- 80 http
- 443 https
- 8080 http

in
- 80
- 443
- 8080

### Public subnet

- Prevent subnets that shouldn't be communicating directly with the public tier.
- Only permit incoming connections on desired ports.

#### Inbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.12.0/23 | Deny | Deny access from data tier |
| Custom | all | 10.0.14.0/23 | Deny | Deny access from data tier |
| Custom | all | 10.0.16.0/23 | Deny | Deny access from data tier |
| Custom | all | 10.0.18.0/28 | Deny | Deny access from NAT tier |
| Custom | all | 10.0.18.16/28 | Deny | Deny access from NAT tier |
| Custom | all | 10.0.18.32/28 | Deny | Deny access from NAT tier |
| HTTP | 80 | 0.0.0.0/0 | Accept | Accept all http traffic |
| HTTPS | 443 | 0.0.0.0/0 | Accept | Accept all https traffic |
| SSH | 22 | 0.0.0.0/0 | Accept | Accept all ssh traffic. <br/> Only required for jumpboxes |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.12.0/23 | Deny | Deny access to data tier |
| Custom | all | 10.0.14.0/23 | Deny | Deny access to data tier |
| Custom | all | 10.0.16.0/23 | Deny | Deny access to data tier |
| Custom | all | 10.0.18.0/28 | Deny | Deny access to NAT tier |
| Custom | all | 10.0.18.16/28 | Deny | Deny access to NAT tier |
| Custom | all | 10.0.18.32/28 | Deny | Deny access to NAT tier |
| Custom | all | 0.0.0.0/0 | Allow | Allow all traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

### App

- Public traffic is not routed to the app tier.
- Hence the NACLs can be more relaxed.

#### Inbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 0.0.0.0/0 | Allow | Allow all traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 0.0.0.0/0 | Allow | Allow all traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

### Data

- Prevent subnets that shouldn't be communicating directly with the data tier.

#### Inbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.0.0/23 | Deny | Deny access from public tier |
| Custom | all | 10.0.2.0/23 | Deny | Deny access from public tier |
| Custom | all | 10.0.4.0/23 | Deny | Deny access from public tier |
| Custom | all | 0.0.0.0/0 | Allow | Allow all traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.0.0/23 | Deny | Deny access to public tier |
| Custom | all | 10.0.2.0/23 | Deny | Deny access to public tier |
| Custom | all | 10.0.4.0/23 | Deny | Deny access to public tier |
| Custom | all | 0.0.0.0/0 | Allow | Allow all traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

### NAT

- Prevent subnets that shouldn't be communicating directly with the data tier.
- Incoming connections should only be on http and https ports

#### Inbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.0.0/23 | Deny | Deny access to public tier |
| Custom | all | 10.0.2.0/23 | Deny | Deny access to public tier |
| Custom | all | 10.0.4.0/23 | Deny | Deny access to public tier |
| HTTP | 80 | 0.0.0.0/0 | Allow | Accept all http traffic |
| HTTPS | 443 | 0.0.0.0/0 | Allow | Accept all https traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |

#### Outbound

| Type | Port range | Source | Action | Description |
| ---- | ---------- | ------ | ------ | ----------- |
| Custom | all | 10.0.0.0/23 | Deny | Deny access to public tier |
| Custom | all | 10.0.2.0/23 | Deny | Deny access to public tier |
| Custom | all | 10.0.4.0/23 | Deny | Deny access to public tier |
| Custom | all | 0.0.0.0/0 | Allow | Allow all traffic |
| Custom | all | 0.0.0.0/0 | Deny | Deny all traffic |
