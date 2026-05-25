## Task: Configure NAT Gateway for Internet Access in a Private VPC
The Nautilus DevOps team is tasked with enabling internet access for an EC2 instance running in a private subnet. This instance should be able to upload a test file to a public S3 bucket once it can access the internet. To achieve this, the team must set up a NAT Gateway in a public subnet within the same VPC.


1. A VPC named `datacenter-priv-vpc` and a private subnet `datacenter-priv-subnet` have already been created.
2. An EC2 instance named `datacenter-priv-ec2` is already running in the private subnet.
3. The EC2 instance is configured with a cron job that uploads a test file to a bucket `datacenter-nat-976947076` once internet is accessible.

Your task is to:
- Create a public subnet named `datacenter-pub-subnet` in the same VPC.
- Create an Internet Gateway and attach it to the VPC.
- Create a route table `datacenter-pub-rt` and associate it with the public subnet.
- Allocate an Elastic IP and create a NAT Gateway named `datacenter-natgw`.
- Update the private route table to route `0.0.0.0/0` traffic via the NAT Gateway.  

Once complete, verify that the EC2 instance can reach the internet by confirming the presence of the test file in the S3 bucket `datacenter-nat-976947076`. After completing all the configuration, please wait a few minutes for the test file to appear in the bucket, as it may take 2–3 minutes.

---

## Solution

### Step 1: Set Variables
```bash
VPC_NAME="datacenter-priv-vpc"
PUB_SUBNET_NAME="datacenter-pub-subnet"
PUB_RT_NAME="datacenter-pub-rt"
NAT_GW_NAME="datacenter-natgw"
PRIVATE_SUBNET_NAME="datacenter-priv-subnet"
TEST_BUCKET="datacenter-nat-976947076"
```

### Step 2: Get the VPC ID
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=$VPC_NAME" \
  --query "Vpcs[0].VpcId" \
  --output text)

echo $VPC_ID
```

### Step 3: Create a Public Subnet
Create a public subnet in the same VPC.
```bash
aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$PUB_SUBNET_NAME}]"
```
Get the subnet ID:
```bash
PUB_SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=$PUB_SUBNET_NAME" \
  --query "Subnets[0].SubnetId" \
  --output text)

echo $PUB_SUBNET_ID
```
Enable auto-assign public IPs:
```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_ID \
  --map-public-ip-on-launch
```

### Step 4: Create and Attach an Internet Gateway
Create the Internet Gateway:
```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

echo $IGW_ID
```
Attach it to the VPC:
```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```
Add a Name tag:
```bash
aws ec2 create-tags \
  --resources $IGW_ID \
  --tags Key=Name,Value=datacenter-igw
```

### Step 5: Create Public Route Table
Create route table:
```bash
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo $PUB_RT_ID
```
Tag the route table:
```bash
aws ec2 create-tags \
  --resources $PUB_RT_ID \
  --tags Key=Name,Value=$PUB_RT_NAME
```
Create internet route:
```bash
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
```
Associate the route table with the public subnet:
```bash
aws ec2 associate-route-table \
  --subnet-id $PUB_SUBNET_ID \
  --route-table-id $PUB_RT_ID
```

### Step 6: Allocate Elastic IP
```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)

echo $EIP_ALLOC_ID
```

### Step 7: Create NAT Gateway
Create the NAT Gateway in the public subnet:
```bash
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET_ID \
  --allocation-id $EIP_ALLOC_ID \
  --tag-specifications "ResourceType=natgateway,Tags=[{Key=Name,Value=$NAT_GW_NAME}]" \
  --query 'NatGateway.NatGatewayId' \
  --output text)

echo $NAT_GW_ID
```
Wait until the NAT Gateway becomes available:
```bash
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids $NAT_GW_ID
```  

### Step 8: Update the Private Route Table
Get the private subnet ID:
```bash
PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=$PRIVATE_SUBNET_NAME" \
  --query "Subnets[0].SubnetId" \
  --output text)

echo $PRIV_SUBNET_ID
```
Get the private route table associated with the private subnet:
```bash
PRIV_RT_ID=$(aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$PRIV_SUBNET_ID" \
  --query "RouteTables[0].RouteTableId" \
  --output text)

echo $PRIV_RT_ID
```
Add default route through NAT Gateway:
```bash
aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID
```

### Step 9: Verify Internet Connectivity
Wait 2–3 minutes for the cron job running on datacenter-priv-ec2 to upload the test file.  
Check the S3 bucket contents:
```bash
aws s3 ls s3://$TEST_BUCKET
```
If the setup is correct, you should see the uploaded test file in the bucket.