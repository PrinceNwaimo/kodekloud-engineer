## Task: Managing EC2 Access with S3 Role-based Permissions
The Nautilus DevOps team needs to set up an application on an EC2 instance to interact with an S3 bucket for storing and retrieving data. To achieve this, the team must create a private S3 bucket, set appropriate IAM policies and roles, and test the application functionality.

**Task:**
1. **EC2 Instance Setup:**
    - An instance named `datacenter-ec2` already exists.
    - The instance requires access to an S3 bucket.
2. **Setup SSH Keys:**
    - Create new SSH key pair (id_rsa and id_rsa.pub) on the `aws-client` host and add the public key to the `root` user's authorized keys on the EC2 instance.
3. **Create a Private S3 Bucket:**
    - Name the bucket `datacenter-s3-13685`.
    - Ensure the bucket is private.
4. **Create an IAM Policy and Role:**
    - Create an IAM policy allowing `s3:PutObject`, `s3:ListBucket` and `s3:GetObject` access to `datacenter-s3-13685`.
    - Create an IAM role named `datacenter-role`.
    - Attach the policy to the IAM role.
    - Attach this role to the `datacenter-ec2` instance.
5. **Test the Access:**
    - SSH into the EC2 instance and try to upload a file to `datacenter-s3-13685` bucket using following command:
      `aws s3 cp <your-file> s3://datacenter-s3-13685/`
    - Now run following command to list the upload file:
      `aws s3 ls s3://datacenter-s3-13685/`

---

## Solution

### Step 1: Set Variables
```bash
BUCKET="datacenter-s3-13685"
IAM_ROLE="datacenter-role"
INSTANCE="datacenter-ec2"
```

### Step 2: Create SSH Key on aws-client & Enable Passwordless Access
Create SSH key
```bash
ssh-keygen
```
Copy the public key
```bash
cat /root/.ssh/id_rsa.pub
```

**Note:** Make sure `SSH` access is allowed by the EC2 security group.  
Login to the EC2 instance using **EC2 Instance Connect** from the AWS Management Console.
- On the EC2 instance switch to `root` user:
  ```bash
  sudo su -
  ```
- Add the public key copied earlier from aws-client to `/root/.ssh/authorized_keys` file on the EC2 instance
- Make a note of the EC2 instance public IP while in the AWS management console. We'll use it to `ssh` to it from the `aws-client` host.

### Step 3: Create Private S3 Bucket 
**From the `aws-client` host**
```bash
aws s3api create-bucket \
  --bucket $BUCKET
```
Bucket is private by default.

### Step 4: Create IAM Policy for S3 Access
```bash
cat > s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::$BUCKET"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::$BUCKET/*"
    }
  ]
}
EOF
```
Create IAM Policy
```bash
POLICY_ARN=$(aws iam create-policy \
  --policy-name s3-policy \
  --policy-document file://s3-policy.json \
  --query "Policy.Arn" \
  --output text)
```

### Step 5: Create IAM Role & Attach Policy
Trust policy for EC2
```bash
cat > ec2-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```
Create role
```bash
aws iam create-role \
  --role-name $IAM_ROLE \
  --assume-role-policy-document file://ec2-trust-policy.json
```
Attach policy
```bash
aws iam attach-role-policy \
  --role-name $IAM_ROLE \
  --policy-arn "$POLICY_ARN"
```

### Step 6: Attach IAM Role to EC2 Instance
Create instance profile
```bash
aws iam create-instance-profile \
  --instance-profile-name $IAM_ROLE
```
Add role to instance profile
```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name $IAM_ROLE \
  --role-name $IAM_ROLE
```
Get EC2 instance ID
```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=$INSTANCE" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text)
```
Attach profile to EC2
```bash
aws ec2 associate-iam-instance-profile \
  --instance-id $INSTANCE_ID \
  --iam-instance-profile Name=$IAM_ROLE
```

**Note:** While using AWS console selecting a role automatically creates & attaches an instance profile behind the scenes. Since we're perfoming the task using AWS CLI, we'll have to create an instance profile → add role to instance profile → attach instance profile to EC2.

### Step 7: Test Access
SSH to the EC2 instance using its public IP
```bash
ssh root@<VM_PUBLIC_IP>
```
Create a test file and upload it to S3 from the EC2 instance
```bash
# create a test file
touch test.txt

# upload to s3 bucket


Task Summary: EC2 Accessing a Private S3 Bucket Using IAM Roles
What you were asked to build

The Nautilus DevOps team needed an application running on an EC2 instance to securely store and retrieve files from an S3 bucket.

The goal was:

EC2 Instance (Application)
        |
        |  AWS IAM Role (temporary credentials)
        |
        v
Private S3 Bucket
(datacenter-s3-431380056739)

Instead of giving the application AWS access keys, you configured IAM role-based access.

Steps you completed and why
1. Created SSH keys and configured EC2 access

You created:

~/.ssh/id_rsa       → private key
~/.ssh/id_rsa.pub   → public key

Then you added the public key to the EC2 instance:

/root/.ssh/authorized_keys
Why?

SSH keys allow secure administration of the EC2 server without passwords.

You needed access to the server so you could test whether the application/instance could communicate with S3.

The private key stays on your machine:

aws-client
   |
   | SSH private key
   v
EC2 instance
2. Created a private S3 bucket

Bucket:

datacenter-s3-431380056739

You ensured it remained private.

Why?

S3 buckets are storage services. By default, you do not want application data exposed publicly.

A private bucket means:

No anonymous users can read files.
Only authorized AWS identities can access objects.

Example:

Public Internet
       X
       |
       v
Private S3 Bucket
       ^
       |
Authorized IAM Role only
3. Created an IAM Policy

You created a policy allowing:

s3:PutObject
s3:GetObject
s3:ListBucket
What these permissions mean
Permission	Purpose
s3:PutObject	Upload files
s3:GetObject	Download/read files
s3:ListBucket	See files inside the bucket

The policy defines what actions are allowed.

Example:

"This identity is allowed to upload and download objects from this bucket."

4. Created an IAM Role

You created:

datacenter-role

and attached the policy to it.

Why a role?

An EC2 instance should not store AWS access keys like:

AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY

inside configuration files or application code.

That is dangerous because:

Keys can leak.
Developers may accidentally commit them to GitHub.
Rotating keys becomes difficult.

Instead, AWS provides IAM Roles for EC2.

The flow becomes:

Application
     |
     |
EC2 Metadata Service
     |
     |
IAM Role
(datacenter-role)
     |
     |
Temporary AWS credentials
     |
     |
S3 Bucket

AWS automatically provides temporary credentials to the EC2 instance.

5. Attached the IAM Role to the EC2 instance

You attached:

datacenter-role
        |
        |
        v
datacenter-ec2
Why?

An IAM policy cannot do anything by itself.

The relationship is:

Policy
  |
  | attached to
  v
Role
  |
  | attached to
  v
EC2 Instance

The EC2 instance inherits the permissions from the role.

6. Tested uploading to S3

Inside the EC2 instance:

aws s3 cp test.txt s3://datacenter-s3-431380056739/

Then:

aws s3 ls s3://datacenter-s3-431380056739/

Output:

2026-07-13 11:11:34 15 test.txt
Why test this?

This proves:

✅ EC2 can authenticate with AWS
✅ The IAM role is attached correctly
✅ The policy permissions work
✅ The bucket is reachable
✅ The application can store data in S3

The bigger DevOps lesson

The architecture follows AWS security best practice:

❌ Bad approach
Application
    |
    |
Hardcoded AWS keys
    |
    |
S3

Problems:

Secrets can leak.
Manual key rotation.
Difficult auditing.
✅ Recommended approach
Application
    |
    |
EC2 IAM Role
    |
    |
Temporary Credentials
    |
    |
Private S3 Bucket

Benefits:

No secrets stored on servers.
Automatic credential rotation.
Least privilege access.
Easier auditing.
More secure production deployments.

In a real production environment, this same pattern is used for:

Web applications storing user uploads.
Backup systems writing to S3.
Log collectors sending logs to S3.
Data pipelines storing processed files.
Kubernetes workloads using IAM roles instead of AWS keys.
aws s3 cp test.txt s3://<s3_bucket_name>/

# list the upload file

Assuming you're on the aws-client host with the AWS CLI already configured, these commands will complete the task.

1. Get the EC2 instance details
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query "Reservations[].Instances[].[InstanceId,PublicIpAddress]" \
  --output table

If that returns nothing, try:

aws ec2 describe-instances \
  --query "Reservations[].Instances[].[InstanceId,Tags[?Key=='Name']|[0].Value,PublicIpAddress]" \
  --output table

Save the values:

INSTANCE_ID=<instance-id>
EC2_IP=<public-ip>
2. Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

Copy the public key:

cat ~/.ssh/id_rsa.pub

Connect to the EC2 instance using the existing key supplied by the lab:

ssh -i <existing-key.pem> ec2-user@$EC2_IP

or

ssh -i <existing-key.pem> ubuntu@$EC2_IP

or

ssh -i <existing-key.pem> centos@$EC2_IP

Become root:

sudo su -

Create the ssh directory if necessary:

mkdir -p ~/.ssh
chmod 700 ~/.ssh

Append the public key you copied:

echo "<contents-of-id_rsa.pub>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

Exit.

Now verify the new key works:

ssh -i ~/.ssh/id_rsa root@$EC2_IP
3. Create the private S3 bucket

If the region is us-east-1:

aws s3api create-bucket \
    --bucket datacenter-s3-431380056739

For every other region:

REGION=$(aws configure get region)

aws s3api create-bucket \
    --bucket datacenter-s3-431380056739 \
    --region $REGION \
    --create-bucket-configuration LocationConstraint=$REGION

Enable bucket ownership controls:

aws s3api put-public-access-block \
    --bucket datacenter-s3-431380056739 \
    --public-access-block-configuration \
BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
4. Create the IAM policy

Create the policy document.

cat > s3-policy.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":[
        "s3:ListBucket"
      ],
      "Resource":"arn:aws:s3:::datacenter-s3-431380056739"
    },
    {
      "Effect":"Allow",
      "Action":[
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource":"arn:aws:s3:::datacenter-s3-431380056739/*"
    }
  ]
}
EOF

Create it:

aws iam create-policy \
    --policy-name datacenter-policy \
    --policy-document file://s3-policy.json

Save the ARN:

POLICY_ARN=$(aws iam list-policies \
    --scope Local \
    --query "Policies[?PolicyName=='datacenter-policy'].Arn" \
    --output text)

echo $POLICY_ARN
5. Create the IAM role

Trust policy:

cat > trust-policy.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Principal":{
        "Service":"ec2.amazonaws.com"
      },
      "Action":"sts:AssumeRole"
    }
  ]
}
EOF

Create the role:

aws iam create-role \
    --role-name datacenter-role \
    --assume-role-policy-document file://trust-policy.json

Attach the policy:

aws iam attach-role-policy \
    --role-name datacenter-role \
    --policy-arn $POLICY_ARN
6. Create the instance profile
aws iam create-instance-profile \
    --instance-profile-name datacenter-role

Add the role:

aws iam add-role-to-instance-profile \
    --instance-profile-name datacenter-role \
    --role-name datacenter-role

Wait a few seconds:

sleep 15

Associate it with the EC2 instance:

aws ec2 associate-iam-instance-profile \
    --instance-id $INSTANCE_ID \
    --iam-instance-profile Name=datacenter-role

If the instance already has a profile attached:

aws ec2 describe-iam-instance-profile-associations

Then replace it instead:

aws ec2 replace-iam-instance-profile-association \
    --association-id <association-id> \
    --iam-instance-profile Name=datacenter-role
7. Test access

SSH using the new key:

ssh -i ~/.ssh/id_rsa root@$EC2_IP

Create a file:

echo "Hello Nautilus" > test.txt

Upload:

aws s3 cp test.txt s3://datacenter-s3-431380056739/

List objects:

aws s3 ls s3://datacenter-s3-431380056739/

You should see something similar to:

2026-07-13 12:34:56         16 test.txt

If you encounter an error at any step, paste the exact output (especially from the IAM policy creation, role attachment, or S3 upload), and I can help diagnose it.
aws s3 ls s3://<s3_bucket_name>/
```
![s3 upload](assets/day37_01.png)
