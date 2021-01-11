# Create EKS Cluster & Node Groups

## Step-00: Introduction
- Understand about EKS Core Objects
  - Control Plane
  - Worker Nodes & Node Groups
  - Fargate Profiles
  - VPC
- Create EKS Cluster
- Associate EKS Cluster to IAM OIDC Provider
- Create EKS Node Groups
- Verify Cluster, Node Groups, EC2 Instances, IAM Policies and Node Groups


## Step-01: Create EKS Cluster using eksctl
- It will take 15 to 20 minutes to create the Cluster Control Plane 
```
# Create Cluster
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 
# Example:                      
eksctl create cluster --name=eksdemo2 --region=ap-south-1 --zones=ap-south-1a,ap-south-1b --without-nodegroup 

#IF cluster delete is unclean you may encounter below error while creating cluster
eksctl-eksdemo2-cluster	DELETE_FAILED	2020-10-29 11:14:34 UTC+0530	EKS cluster (dedicated VPC: true, dedicated IAM: true) [created and managed by eksctl]

# Solution  - login AWS Console -> Cloudformation and delete eksctl-eksdemo2-cluster - 
#To clean up you may have to delete - VPC, subnets manually associated with it also manually. Check cloudformation stack events and delete manually

# Error when executing  eksctl and fix 

[ℹ]  eksctl version 0.35.0
[ℹ]  using region ap-south-1
[!]  retryable error (EC2MetadataError: failed to make EC2Metadata request
        status code: 404, request id:
caused by: <?xml version="1.0" encoding="iso-8859-1"?>

Login AWS - In the navigation bar on the upper right, choose your account name or number and then choose My Security Credentials.

Click on Create Access Key and get the ACCESS KEY ID and also download  csv which has Access key ID	and Secret access key

in terminal where u exectue eksctl  

> aws configure  and provide access key id and secreate access key





# Get List of clusters
eksctl get clusters

# Get any nodes present

kubectl get nodes
No resources found in default namespace.
```

#Check to see if every resource requested is created propertly

eksctl utils describe-stacks --region=ap-south-1 --cluster=eksdemo1

# get help to create cluster

eksctl create cluster --help

#what and all you can create

eksctl create --help

## Step-02: Create & Associate IAM OIDC Provider for our EKS Cluster
- To enable and use AWS IAM roles for Kubernetes service accounts on our EKS cluster, we must create &  associate OIDC identity provider.
- To do so using `eksctl` we can use the  below command. 
- Use latest eksctl version (as on today the latest version is `0.21.0`)
```                   
# Template
eksctl utils associate-iam-oidc-provider \
    --region region-code \
    --cluster <cluter-name> \
    --approve

# Replace with region & cluster name
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve
    
Example :
eksctl utils associate-iam-oidc-provider --region ap-south-1 --cluster eksdemo2  --approve
```



## Step-03: Create EC2 Keypair
- Create a new EC2 Keypair with name as `kube-demo`  (Stored in local : D:\Udemy_aws_kubernetes_Kalyan)
- This keypair we will use it when creating the EKS NodeGroup.
- This will help us to login to the EKS Worker Nodes using Terminal.

## Step-04: Create Node Group with additional Add-Ons in Public Subnets
- These add-ons will create the respective IAM policies for us automatically within our Node Group role.
 ```
# Create Public Node Group   
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-public1 \
                        --node-type=t3.medium \
                        --nodes=2 \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access 
                        
Example: 

eksctl create nodegroup --cluster=eksdemo2  --region=ap-south-1 --name=eksdemo2-ng-public2 --node-type=t2.micro --nodes=2 --nodes-min=2 --nodes-max=4 --node-volume-size=20  \
--ssh-access --ssh-public-key=kube-demo --managed --asg-access --external-dns-access --full-ecr-access --appmesh-access --alb-ingress-access

## Step-05: Verify Cluster & Nodes

kubectl get nodes

kubectl get nodes -o wide  (Get additional detail)


### Verify NodeGroup subnets to confirm EC2 Instances are in Public Subnet
- Verify the node group subnet to ensure it created in public subnets
  - Go to Services -> EKS -> eksdemo -> eksdemo1-ng1-public
  - Click on Associated subnet in **Details** tab
  - Click on **Route Table** Tab.
  - We should see that internet route via Internet Gateway (0.0.0.0/0 -> igw-xxxxxxxx)

### Verify Cluster, NodeGroup in EKS Management Console
- Go to Services -> Elastic Kubernetes Service -> eksdemo1

### List Worker Nodes
```
# List EKS clusters
eksctl get cluster

# List NodeGroups in a cluster
eksctl get nodegroup --cluster=<clusterName>

# List Nodes in current kubernetes cluster
kubectl get nodes -o wide

# Our kubectl context should be automatically changed to new cluster
kubectl config view --minify
```

### Verify Worker Node IAM Role and list of Policies
- Go to Services -> EC2 -> Worker Nodes
- Click on **IAM Role associated to EC2 Worker Nodes**

### Verify Security Group Associated to Worker Nodes
- Go to Services -> EC2 -> Worker Nodes
- Click on **Security Group** associated to EC2 Instance which contains `remote` in the name.

### Verify CloudFormation Stacks
- Verify Control Plane Stack & Events
- Verify NodeGroup Stack & Events

### Login to Worker Node using Keypai kube-demo
- Login to worker node
```
# For MAC or Linux or Windows10
ssh -i kube-demo.pem ec2-user@<Public-IP-of-Worker-Node>

- I used bash as i got permission issue with respect to kube-demo.pem

# For Windows 7
Use putty
```

###Check volume size in workernode
ssh into worker node and execute - df -h 

$ ssh -i kube-demo.pem ec2-user@13.233.48.107
load pubkey "kube-demo.pem": invalid format
Last login: Wed Oct  7 19:05:48 2020 from 205.251.233.50

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-192-168-52-1 ~]$ ls -ltr
total 0
[ec2-user@ip-192-168-52-1 ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        479M     0  479M   0% /dev
tmpfs           492M     0  492M   0% /dev/shm
tmpfs           492M  688K  491M   1% /run
tmpfs           492M     0  492M   0% /sys/fs/cgroup
/dev/xvda1      5.0G  2.2G  2.9G  43% /
tmpfs            99M     0   99M   0% /run/user/0
tmpfs            99M     0   99M   0% /run/user/1000

## worker node security group - inbound  all protocol , anywhere is added -- so if you get error while submiting worker node check this....


## Step-06: Update Worker Nodes Security Group to allow all traffic
- We need to allow `All Traffic` on worker node security group

## Additional References
- https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html
- https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html
