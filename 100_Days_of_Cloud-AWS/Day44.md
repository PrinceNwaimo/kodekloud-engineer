## Task: Implementing Auto Scaling for High Availability in AWS
The DevOps team is tasked with setting up a highly available web application using AWS. To achieve this, they plan to use an Auto Scaling Group (ASG) to ensure that the required number of EC2 instances are always running, and an Application Load Balancer (ALB) to distribute traffic across these instances. The goal of this task is to set up an ASG that automatically scales EC2 instances based on CPU utilization, and an ALB that directs incoming traffic to the instances. The EC2 instances should have Nginx installed and running to serve web traffic.

1. Create an EC2 launch template named `devops-launch-template` that specifies the configuration for the EC2 instances, including the **Amazon Linux 2 AMI**, **t2.micro** instance type, and a security group that allows **HTTP traffic on port 80**.
2. Add a `User Data` script to the launch template to install `Nginx` on the EC2 instances when they are launched. The script should install Nginx, start the Nginx service, and enable it to start on boot.
3. Create an Auto Scaling Group named `devops-asg` that uses the launch template and ensures a minimum of **1 instance**, desired capacity is **1 instance** and a maximum of **2 instances** are running based on **CPU utilization**. Set the target **CPU utilization** to **50%**.
4. Create a target group named `devops-tg`, an Application Load Balancer named `devops-alb` and configure it to listen on **port 80**. Ensure the ALB is associated with the Auto Scaling Group and distributes traffic across the instances.
5. Configure health checks on the ALB to ensure it routes traffic only to healthy instances.
6. Verify that the ALB's DNS name is accessible and that it displays the default Nginx page served by the EC2 instances.


---

## Solution

### Step 1: Set Variables
```bash
LAUNCH_TEMPLATE="devops-launch-template"
INSTANCE_TYPE="t2.micro"
ASG="devops-asg"
TARGET_GROUP="devops-tg"
ALB="devops-alb"
```

### Step 2: Get default VPC, subnets, and create a Security Group
Get default VPC
```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' \
  --output text)
```
Get two subnets (for ALB high availability)
```bash
SUBNETS=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID \
  --query 'Subnets[0:2].SubnetId' \
  --output text)
```
Create security group
```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name web-sg \
  --description "Allow HTTP traffic" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)
```
Allow HTTP on port 80
```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### Step 3: Create Launch Template
User Data script
```bash
cat > userdata.sh <<EOF
#!/bin/bash
yum update -y
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
EOF
```
Encode user data
```bash
USER_DATA=$(base64 -w 0 userdata.sh)
```
Get Amazon Linux 2 AMI ID
```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'Images | sort_by(@,&CreationDate)[-1].ImageId' \
  --output text)
```
Create launch template
```bash
aws ec2 create-launch-template \
  --launch-template-name $LAUNCH_TEMPLATE \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"$INSTANCE_TYPE\",
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"UserData\": \"$USER_DATA\"
  }"
```

### Step 4: Create Target Group
```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name $TARGET_GROUP \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --health-check-path / \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)
```

### Step 5: Create Application Load Balancer
```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name $ALB \
  --subnets $SUBNETS \
  --security-groups $SG_ID \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)
```
Create listener
```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### Step 6: Create Auto Scaling Group
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name $ASG \
  --launch-template LaunchTemplateName=$LAUNCH_TEMPLATE,Version=1 \
  --min-size 1 \
  --desired-capacity 1 \
  --max-size 2 \
  --vpc-zone-identifier "$(echo $SUBNETS | tr ' ' ',')" \
  --target-group-arns $TG_ARN
```

### Step 7: Configure CPU-based Auto Scaling
```bash
aws autoscaling put-scaling-policy \
  --policy-name cpu-scaling-policy \
  --auto-scaling-group-name devops-asg \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0
  }'
```

### Step 8: Verify setup
Check target health, it should show `healthy`(it usually takes 3-5 mins)
```bash
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN
```
![tg status](assets/day44_01.png)

Get ALB DNS name
```bash
aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text
```
Open the ALB DNS name in a browser, you should see the default `nginx` page




To Launch Template
aws ec2 create-launch-template \
  --launch-template-name $LAUNCH_TEMPLATE \
  --version-description "v1" \
  --launch-template-data "{
    \"ImageId\":\"$AMI_ID\",
    \"InstanceType\":\"$INSTANCE_TYPE\",
    \"SecurityGroupIds\":[\"$SG_ID\"],
    \"UserData\":\"$USER_DATA\"
  }"
