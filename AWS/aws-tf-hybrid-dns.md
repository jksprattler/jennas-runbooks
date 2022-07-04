# Hybrid DNS between AWS and On-prem with Terraform

_Last updated: July 03, 2022_

## Overview
The purpose of this runbook is to demonstrate the implementation of an AWS Hybrid DNS design and architecture between an AWS region hosting private only subnets and an on prem private corporate data center. The intent of this design is to simulate this architecture while the actual implementation will provide private DNS resolution over an established inter-region AWS VPC Peering connection in addition to various Route 53 and Bind DNS server components as detailed below.

### Pre-requisites

- GitHub Account
- AWS Account
- AWS configuration and credentials [setup](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- Terraform [installed](https://learn.hashicorp.com/tutorials/terraform/install-cli)

### Design Artifacts

- YouTube demo recording: [FIXME](FIXME)
- Simulated AWS Hybrid DNS Network Design Architecture: [FIXME](FIXME)
- Actual AWS Hybrid DNS Network Design Architecture: [FIXME](FIXME)
- Clone/Fork the repo containing the terraform artifacts for the AWS Hybrid DNS design here: [jksprattler/aws-networking](https://github.com/jksprattler/aws-networking.git)
  - relevant files are under `/aws-terraform-hybrid-dns`

### Simulated design diagram

### Actual design diagram

### Architecture
The us-east-1 region will be hosting the `micros4l-aws` VPC with a prefix of 10.10.0.0/16 containing 2 private subnets. Two basic t2.micro EC2 instances, `micros4l-awsec2b/b` will be deployed here for testing DNS resolution into our simulated Corporate on-prem datacenter (which actually lives in us-east-2). Each instance is deployed in a separate subnet/availability zone.

The us-east-2 region will be hosting the `micros4l-onprem` VPC with a prefix of 192.168.10.0/24 also containing 2 private subnets - note the non-overlapping private IP space as this is a requirement for VPC peering in addition to DirectConnect (DX) connectivity which is being simulated. Two t2.micro EC2 instances running bind and configured with zone `corp.microgreens4life.org` called  `micros4l-onpremdnsa/b` will be deployed here. A third basic t2.micro EC2 instance which we'll use for testing DNS resolution from "on prem" to AWS (us-east-1) called `micros4l-onpremapp` will be deployed. Each DNS instance is deployed in a separate subnet/availability zone.

The `micros4l-aws` VPC will host the private zone for `aws.microgreens4life.org` with an A record of `web.aws.microgreens4life.org`. This VPC will also host a Route 53 Inbound and Outbound endpoint where each endpoint gets associated with both of the private subnets. The Outbound endpoint will have a forwarding rule associated with it which targets the `micros4l-onpremdnsa/b` Bind servers hosted "on prem" (us-east-2).

All instances will be deployed with settings configured to allow Systems Manager connectivity as this is the only way to connect to these private instances in this environment as none of them will be deployed with a public IP address nor is there any internet gateway created.

### Procedure

#### Deploy the us-east-2 resources
1. Navigate to the `/global/iam` directory and run terraform plan/apply:
```scss
cd aws-terraform-hybrid-dns/global/iam
terraform plan
terraform apply
```
Resources deployed in this terraform module:
- `roles.tf` - IAM instance policy, roles and policy attachments which all EC2 instances in this design will utilize
- `s3.tf` - S3 bucket for storing the terraform state files. Update your bucket name here as it must be globally unique
2. Navigate to the `/us-east-2` directory and run terraform plan/apply:
```scss
cd aws-terraform-hybrid-dns/us-east-2
terraform plan
terraform apply
```
Resources deployed in this terraform module:
- `ec2.tf` -  micros4l-onpremdnsa/b simulating on prem Linux Bind/DNS servers and micros4l-onpremapp Linux server
- `vpc.tf` - VPC with prefix 192.168.10.0/24, 2x private subnets, private route table associated with the 2x subnets, Security Group and rules allowing SSM access and DNS requests, VPC Endpoints for SSM connectivity
3. Capture the outputs from the `/us-east-2` module deployment and save them in a temp text file for use as input in the next step. For example:
```scss
onprem-private-rt_id = "rtb-0fae5266503453da8"
onpremdnsa_ip = "192.168.10.11"
onpremdnsb_ip = "192.168.10.236"
onpremvpc_id = "vpc-0ab74c12320891aa3"
```
4. Navigate to the `/us-east-1` directory and run terraform plan/apply:
Resources deployed in this terraform module:
- `ec2.tf` - micros4l-awsec2a/b AWS instances
- `route53.tf` - aws.microgreens4life.org Route 53 private zone, web.aws.microgreens4life.org A record, Route 53 Inbound endpoint, Route 53 Outbound endpoint for the corp.microgreens4life.org domain with a Forwarding rule pointing to the Corp on prem environment (us-east-2)
- `vpc.tf` - VPC with prefix 10.10.0.0/16, 2x private subnets, private route table associated with the 2x subnets, VPC Peering connectivity between the AWS us-east-1 region to the "on prem" us-east-2 region, Security Group and rules allowing SSM access and DNS requests, VPC Endpoints for SSM connectivity 
5. Capture the outputs from the `/us-east-1` module deployment and save them in a temp text file for use as input in the next step. Note you'll only need the "ip" address output from each of the 2 endpoints. For example:
```scss
FIXME
```
6. In the [awszone.forward](https://github.com/jksprattler/aws-networking/blob/main/aws-terraform-hybrid-dns/us-east-2/awszone.forward) file, replace the INBOUND_ENDPOINT_IP1 and INBOUND_ENDPOINT_IP2 values of the forwarders with the Endpoint IP addresses from the outputs of the previous step.
7. From the AWS console, navigate to the EC2 instances in the us-east-2 region, select `micros4l-onpremdnsa` and initiate a connection to it via Session Manager. Enter `sudo -i` and with your editor of choice, vi or nano into the `/etc/named.conf` file. Scroll to the end of the file and paste the contents of your updated [awszone.forward](https://github.com/jksprattler/aws-networking/blob/main/aws-terraform-hybrid-dns/us-east-2/awszone.forward) file, save and exit. Run the following command to restart bind service:
```scss
systemctl restart named 
systemctl status named 
```
Using dig or nslookup, test that your local DNS server is resolving the AWS Route 53 private zone for `aws.microgreens4life.org` now that you've applied the Route 53 endpoint IP addresses into the Bind server's named.conf file. You should see it resolve to the 10.10.x.x private IP space of the us-east-1 AWS VPC where the Route 53 inbound endpoints are hosted. For example:
```scss
sh-4.2$ dig web.aws.microgreens4life.org @127.0.0.1 +short
10.10.0.172
10.10.10.31
```
From the AWS console, navigate to the Route 53 service in the us-east-1 region and validate the A record hosted in your private Route 53 zone is using the same IP addresses that your dns server in us-east-2 just resolved to.
8. Navigate to the EC2 instances in us-east-2 and select `micros4l-onpremapp` and initiate a connection to it via Session Manager. Enter `sudo -i` and with your editor of choice, vi or nano into the `/etc/sysconfig/network-scripts/ifcfg-eth0` file. Scroll to the end of the file and paste the following contents replacing the `THE_PRIVATE_IP_OF_ONPREM_DNS_A/B` values with the actual private IP addresses of the onprem DNS servers which were given in the outputs of your terraform apply for the us-east-2 implementation in step 3.):
```scss
DNS1=THE_PRIVATE_IP_OF_ONPREM_DNS_A
DNS2=THE_PRIVATE_IP_OF_ONPREM_DNS_B
```
Restart the network services: `systemctl restart network`
Run a test ping/dig from the `micros4l-onpremapp` instance to the AWS route 53 hosted domain: 
```scss
ping web.aws.microgreens4life.org
dig web.aws.microgreens4life.org +short
```
Navigate back to the ec2 instances in us-east-1 and initiate a systems manager session on micros4l-awsec2a/b and test dns resolution of the onprem hosted domain: 
```scss
ping app.corp.microgreens4life.org
dig app.corp.microgreens4life.org +short
```

### Summary
Upon completion of the above procedure, you should now have 2 separate private environments with fully integrated DNS resolution between them. The private AWS VPC instances in us-east-1 are successfully resolving the corp domains hosted in the private "on prem" VPC in us-east-2 via the outbound endpoint and forwarding rule and routed via the vpc peering connection. Conversely, the private "on prem" instances in us-east-2 are successfully resolving the web domain hosted in the private AWS VPC us-east-1 via the inbound endpoint.

**Note** the original idea for this design came from Cloud Trainer, Adrian Cantrill. You can find the CFT stack, procedure steps, and videos for his lab [here](https://github.com/acantril/learn-cantrill-io-labs/tree/master/aws-hybrid-dns)

#### Highlighting key differences between our deployments where I:
- Coded **all** infrastructure steps in *Terraform* including vpc peering, vpc peering inter region routes, route 53 inbound and outbound endpoints and forward rules, etc. instead of CloudFormation for initial/base infrastructure.
- Coded outputs.tf files to provide values for the input variable strings for the deployments to the separate regions/modules to prevent needing to hunt down id's and ip addresses from within the AWS console (ie vpc peering id, route table id for peering connection, onpremdnsa/b private ip addresses for the aws zone file)
- Deployed the simulation of the on prem / DX environment in a completely separate region (us-east-2) instead of all in the same us-east-1 region in an attempt to increase complexity, validate inter-region vpc peering works with DNS resolution against private Route 53 endpoints and just for overall better visualization of connectivity between 2 separate environments/regions
- Chose microgreens4life for my Domain/zone instead of animals4life - nothing against animals, I just really love microgreens. I also replaced all resource names with my variation of m4l*/micros4l*/microgreens4life* which allowed me the opportunity to deeply review the code line by line so I wasn't just copy/pasting pieces of Adrian's CFT stack code for the lab (ie a4l*/animals4l*/animals4life*).

#### Highlighting a few issues I ran into that prevented DNS outbound requests and Systems Manager from working:
- Outbound (egress) rules for DNS (tcp/udp) - needed for DNS requests to make it outbound from the onprem vpc to the aws inbound endpoints for Route 53 hosted zones and vice versa for the aws vpc to make outbound dns requests to the corp hosted zones in the onprem vpc - Applied these to both aws and onprem security groups 
- Outbound (egress) rules for HTTPS - needed for SSM traffic in order to have Systems Manager connection for private IP space hosts in VPC - Applied these to both aws and onprem security groups
- VPC Endpoint of com.amazonaws.<region>.s3 Gateway type is required on both aws and onprem VPC's in order for Systems Manager to connect successfully to ec2 instances: [Step 6: Create a VPC endpoint - AWS Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html)
