# OCP 4.2 Install AWS UPI

This process has been generated using the following installation manual:

https://docs.openshift.com/container-platform/4.2/installing/installing_aws_user_infra/installing-aws-user-infra.html

## Content

- [Environment](#environment)
- [Parameters](#parameters)
- [Installation](#installation)
    - [Final tasks](#final-tasks)
- [Ansible Roles](#ansible-roles)
    - [Prerequisites](#prerequisites)
    - [VPC](#vpc)
    - [Networking and Load Balancing](#networking-and-load-balancing)
    - [Security Group and Roles](#security-group-and-roles)
    - [Bootstrap Node](#bootstrap-node)
    - [Control Plane machines](#control-plane-machines)
    - [Worker Node](#worker-node)
    - [Install](#install)
- [TODO](#todo)


## Environment

This information/configuration is needed to start the installation:

- An empty installarion directory
- AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv1.html (configured)
- An SSH Key
```sh
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa_os
```
- An Openshift installation token: https://cloud.redhat.com/openshift/install/aws/user-provisioned
- An AWS Route 53 (base domain)
- jq installed
- OC CLi: https://docs.openshift.com/container-platform/4.2/cli_reference/openshift_cli/getting-started-cli.html 

## Parameters

These are the installation parametes (defined in hosts.yaml):

| Parameter   |      Description      |
|----------|-------------|
| *PREREQUISITES* |
| installation_dir | Local directory for the installation (empty) |
| ssh_key | SSH private key local url |
| aws_base_domain | AWS Route 53 domain |
| aws_hosted_zone_id | AWS Route 53 hosted zone |
| aws_region | AWS region for the cluster|
| aws_cluster_name | AWS cluster name |
| os_install_url | OS install binary url |
| os_token | OS installation token (between '') |
| *VPC* |
| vpc_stack_name | VPC stack name
| vpc_cidr | The CIDR block for the VPC |
| availability_zonecount | The number of availability zones to deploy the VPC in | 
| subnet_bits | The size of each subnet in each availability zone |
| *NET* |
| net_stack_name | NET stack name |
| *SEC* |
| sec_stack_name | SEC stack name |
| *BOOT* |
| boot_stack_name | BOOT stack name |
| rhcos_ami | Current Red Hat Enterprise Linux CoreOS (RHCOS) AMI to use for the bootstrap node (depends on selected region) |
| *CONTROL* |
| control_stack_name | CONTROL stack name |
| master_instance_type | The type of AWS instance to use for the control plane machines |
| *WORKER* |
| worker_stack_name | WORKER stack name |
| worker_instance_type | The type of AWS instance to use for the worker nodes |
| *INSTALL* |
| installation_log_level | Installation output log level |

## Installation

Use this command to run the installation process
```sh
ansible-playbook -i hosts.yaml site.yaml
```
!! Use "--tag" to execute installation roles separately (roles explained below)

### Final tasks
The ansible installation process ends with the openshift-install bootstrap process, these are the steps needed to complete the cluster installation:

- Generate k8s configuration and validate it
```sh
export KUBECONFIG=<installation_dir>/files/auth/kubeconfig
oc whoami
# system:admin
```
- Validate OCP installation
```sh
# Nodes are ready
oc get nodes
# certificates signing request validation (Approved,Issued)
oc get csr
# Wait until cluster operators are available
watch oc get clusteroperators
```
- Delete boot stack
```sh
aws cloudformation delete-stack --stack-name ocp-boot
```
- Complete installation
```sh
./openshift-install --dir=<installation_dir>/files wait-for install-complete
```
- Use console url and kubeadmin password to login and create an admin user
```sh
# Create user htpasswd
htpasswd -c -B -b users.htpasswd USER PASS
# Add user using ocp cweb console (identity provider)
# Provide admin rights and delete kubeadmin user
oc adm policy add-cluster-role-to-user cluster-admin <user>
oc delete secrets kubeadmin -n kube-system
```


## Ansible Roles

### Prerequisites

```sh
ansible-playbook -i hosts.yaml site.yaml --tag="prerequisites"
```
Prerequisites role executes the following actions:

- Download openshift-install and create files folder
- Unzip openshift-install
- Start ssh agent and add ssh key
- Store ssh_key as a variable
- Copy install-config to the installation folder
- Create and configure Kubernetes manifest
- Generate the ignition config files

### VPC

```sh
ansible-playbook -i hosts.yaml site.yaml --tag="vpc"
```
VPC role executes the following actions:
- Validate the template
- Create the stack

Stack output:
- VpcId: ID of the new VPC
- PublicSubnetIds: Subnet IDs of the public subnets
- PrivateSubnetIds: Subnet IDs of the private subnets

### Networking and Load Balancing

```sh
ansible-playbook -i hosts.yaml site.yaml --tag="net"
```
Net role executes the following actions:
- Validate Net template
- Get OCP cluster name
- Get VpcId from VPC Stack
- Get PublicSubnets from VPC Stack
- Get PrivateSubnets from VPC Stack
- Create Net stak

Stack output:
- ApiServerDnsName: Full hostname of the API server, which is required for the Ignition config files
- ExternalApiLoadBalancerName: Full name of the External API load balancer created
- ExternalApiTargetGroupArn: ARN of External API target group
- InternalApiLoadBalancerName: Full name of the Internal API load balancer created
- InternalApiTargetGroupArn: ARN of Internal API target group
- InternalServiceTargetGroupArn: ARN of internal service target group
- PrivateHostedZoneId: Hosted zone ID for the private DNS, which is required for private records
- RegisterNlbIpTargetsLambda: Lambda ARN useful to help register or deregister IP targets for these load balancers

### Security Group and Roles

```sh
ansible-playbook -i hosts.yaml site.yaml --tag="sec"
```
Sec role executes the following actions:
- Validate Net template
- Get OCP cluster name
- Get VpcId from VPC Stack
- Get PrivateSubnets from VPC Stack
- Create Sec stak

Stack output:
- MasterInstanceProfile: Master IAM Instance Profile
- MasterSecurityGroupId: Master Security Group ID
- WorkerInstanceProfile: Worker IAM Instance Profile
- WorkerSecurityGroupId: Worker Security Group ID

### Bootstrap Node

```sh
ansible-playbook -i hosts.yaml site.yaml --tag="boot"
```
Boot role executes the following actions:
- Validate Boot template
- Get OCP cluster name
- Get VpcId from VPC Stack
- Get PublicSubnets from VPC Stack
- Get MasterSecurityGroupId from SEC Stack
- Get MasterSecurityGroupId from NET Stack
- Get ExternalApiTargetGroupArn from NET Stack
- Get InternalApiTargetGroupArn from NET Stack
- Get InternalServiceTargetGroupArn from NET Stack
- Check if s3 bucket for cluster infra already exists
- Create s3 bucket if not exists
- Upload bootstrap ignition file
- Create Boot stak

Stack output:
- BootstrapInstanceId: Instance ID
- BootstrapPrivateIp: The bootstrap node private IP address
- BootstrapPublicIp: The bootstrap node public IP address

### Control Plane machines

```sh
ansible-playbook -i hosts.yaml site.yaml --tag="control"
```
Control role executes the following actions:
- Validate Control template
- Get OCP cluster name
- Get MasterSecurityGroupId from SEC Stack
- Get PrivateHostedZoneId from NET Stack
- Get MasterSecurityGroupId from NET Stack
- Get ExternalApiTargetGroupArn from NET Stack
- Get InternalApiTargetGroupArn from NET Stack
- Get InternalServiceTargetGroupArn from NET Stack
- Get PrivateSubnets from VPC Stack
- Get the base64 encoded certificate authority string to use
- Get MasterInstanceProfile from SEC Stack
- Create Control stak

Stack output:
- PrivateIPs: The control-plane node private IP addresses.

### Worker Node

! This Role creates a worker on each availability zone

```sh
ansible-playbook -i hosts.yaml site.yaml --tag="worker"
```
Worker role executes the following actions:
- Validate Worker template
-  Get OCP cluster name
- Get WorkerSecurityGroupId from SEC Stack
- Get WorkerInstanceProfile from SEC Stack
- Get the base64 encoded certificate authority string to use
- Get PrivateSubnets from VPC Stack
- Create Worker stak


Stack output:
- PrivateIPs: The control-plane node private IP addresses.

### Install
```sh
ansible-playbook -i hosts.yaml site.yaml --tag="install"
```
After this role, go to [Final tasks](#final-tasks) to complete the installation.

## TODO
- Add to ansible the manual tasks after bootstrap
- Add update/delete action on cloudformation
- Less than 3 availability zones ansible management
- Add tags to templates