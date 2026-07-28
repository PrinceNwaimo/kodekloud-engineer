## Task: Integrating AWS SQS and SNS for Reliable Messaging
The Nautilus DevOps team needs to implement priority queuing using Amazon SQS and SNS. The goal is to create a system where messages with different priorities are handled accordingly. You are required to use AWS CloudFormation to deploy the necessary resources in your AWS account. The CloudFormation template should be created on the AWS client host at `/root/datacenter-priority-stack.yml`, the stack name must be `datacenter-priority-stack` and it should create the following resources:

1. Two SQS queues named `datacenter-High-Priority-Queue` and `datacenter-Low-Priority-Queue`.
2. An SNS topic named `datacenter-Priority-Queues-Topic`.
3. A Lambda function named `datacenter-priorities-queue-function` that will consume messages from the SQS queues. The Lambda function code is provided in `/root/index.py` on the AWS client host.
4. An IAM role named `lambda_execution_role` that provides the necessary permissions for the Lambda function to interact with SQS and SNS.
Once the stack is deployed, to test the same you can publish messages to the SNS topic, invoke the Lambda function and observe the order in which they are processed by the Lambda function. The high-priority message must be processed first.
```bash
topicarn=$(aws sns list-topics --query "Topics[?contains(TopicArn, 'datacenter-Priority-Queues-Topic')].TopicArn" --output text)

aws sns publish --topic-arn $topicarn --message 'High Priority message 1' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"high"}}'

aws sns publish --topic-arn $topicarn --message 'High Priority message 2' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"high"}}'

aws sns publish --topic-arn $topicarn --message 'Low Priority message 1' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"low"}}'

aws sns publish --topic-arn $topicarn --message 'Low Priority message 2' --message-attributes '{"priority" : { "DataType":"String", "StringValue":"low"}}'
```

---

## Solution

### Step 1: Create the `/root/datacenter-priority-stack.yml` file on AWS client host:
```yml
AWSTemplateFormatVersion: '2010-09-09'
Description: SQS priority queues template

Resources:
  SQSHighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 180
      QueueName: nautilus-High-Priority-Queue

  SQSLowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 180
      QueueName: nautilus-Low-Priority-Queue

  PriorityQueuesTopic:
    Type: AWS::SNS::Topic
    Properties: 
      TopicName: nautilus-Priority-Queues-Topic 

  SQSHighQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref SQSHighPriorityQueue
      PolicyDocument:
        Id: AllowIncomingMessageFromSNS
        Statement:
          -
            Effect: Allow
            Principal: '*'
            Action:
              - sqs:SendMessage
            Resource:
              - !GetAtt SQSHighPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PriorityQueuesTopic

  SQSLowQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref SQSLowPriorityQueue
      PolicyDocument:
        Id: AllowIncomingMessageFromSNS
        Statement:
          -
            Effect: Allow
            Principal: '*'
            Action:
              - sqs:SendMessage
            Resource:
              - !GetAtt SQSLowPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PriorityQueuesTopic

  SNSHighSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityQueuesTopic
      Endpoint: !GetAtt SQSHighPriorityQueue.Arn
      Protocol: sqs
      RawMessageDelivery: true
      FilterPolicy: {"priority": ["high"]}

  SNSLowSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityQueuesTopic
      Endpoint: !GetAtt SQSLowPriorityQueue.Arn
      Protocol: sqs
      RawMessageDelivery: true
      FilterPolicy: {"priority": ["low"]}

  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Statement:
          - Action:
            - sts:AssumeRole
            Effect: Allow
            Principal:
              Service:
              - lambda.amazonaws.com
        Version: 2012-10-17
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSQSFullAccess
      Path: /

  LambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: nautilus-priorities-queue-function
      Description: Priority queue function
      Runtime: python3.9
      Code:
        ZipFile: >
          import boto3 
          import os
          sqs = boto3.client('sqs')
          def delete_message(queue_url, receipt_handle, message):
              response = sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)
              return "Message " + "'" + message + "'" + " deleted"
              
          def poll_messages(queue_url):
              QueueUrl=queue_url
              response = sqs.receive_message(
                  QueueUrl=QueueUrl,
                  AttributeNames=[],
                  MaxNumberOfMessages=1,
                  MessageAttributeNames=['All'],
                  WaitTimeSeconds=3
              )
              if "Messages" in response:
                  receipt_handle=response['Messages'][0]['ReceiptHandle']
                  message = response['Messages'][0]['Body']
                  delete_response = delete_message(QueueUrl,receipt_handle,message)
                  return delete_response
              else:
                  return "No more messages to poll"

          def lambda_handler(event, context):
              response = poll_messages(os.environ['high_priority_queue'])
              if response == "No more messages to poll":
                  response = poll_messages(os.environ['low_priority_queue'])
              return response

      Handler: index.lambda_handler
      MemorySize: 128
      Timeout: 10
      Role:
        Fn::GetAtt:
          - LambdaRole
          - Arn
      Environment:
        Variables:
          high_priority_queue: !Ref SQSHighPriorityQueue
          low_priority_queue: !Ref SQSLowPriorityQueue

  HighPriorityEventSource:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt SQSHighPriorityQueue.Arn
      FunctionName: !Ref LambdaFunction
      BatchSize: 1
      Enabled: true

Outputs:
  SNSTopicARN:
    Value: !Ref PriorityQueuesTopic
```

### Step 2: Create stack using AWS CLI
```bash
aws cloudformation create-stack \
  --stack-name datacenter-priority-stack \
  --template-body file:///root/datacenter-priority-stack.yml \
  --capabilities CAPABILITY_NAMED_IAM
```
Wait for the stack to get created
```bash
aws cloudformation wait stack-create-complete \
  --stack-name datacenter-priority-stack    
```
Test it by running the commands provided in the task description


reat! The CloudFormation stack has been successfully created. The wait command completed without errors, which means the stack creation is complete and successful.

Let's verify the stack status and resources:
1. Check stack status
bash
aws cloudformation describe-stacks \
  --stack-name datacenter-priority-stack \
  --query "Stacks[0].[StackName, StackStatus, CreationTime]" \
  --output table
2. List all resources created by the stack
bash
aws cloudformation list-stack-resources \
  --stack-name datacenter-priority-stack \
  --query "StackResourceSummaries[*].[LogicalResourceId, ResourceType, ResourceStatus]" \
  --output table
3. View outputs from the stack (if any were defined)
bash
aws cloudformation describe-stacks \
  --stack-name datacenter-priority-stack \
  --query "Stacks[0].Outputs" \
  --output table
4. Get detailed stack information
bash
aws cloudformation describe-stacks \
  --stack-name datacenter-priority-stack \
  --output json
5. Check for any events (to see what happened during creation)
bash
aws cloudformation describe-stack-events \
  --stack-name datacenter-priority-stack \
  --max-items 10 \
  --query "StackEvents[*].[LogicalResourceId, ResourceStatus, ResourceStatusReason]" \
  --output table
Quick one-liner to confirm everything is healthy:
bash
aws cloudformation describe-stacks \
  --stack-name datacenter-priority-stack \
  --query "Stacks[0].[StackName, StackStatus]" \
  --output text
This should return:

text
datacenter-priority-stack    CREATE_COMPLETE
Important: Keep this stack ID for reference
text
arn:aws:cloudformation:us-east-1:048824521363:stack/datacenter-priority-stack/9e071820-8a5e-11f1-8d5f-0affd5483141
You can use this ARN for any future operations on this stack, such as updates or deletion.

If you need to delete the stack later:
bash
aws cloudformation delete-stack \
  --stack-name datacenter-priority-stack

# Wait for deletion to complete
aws cloudformation wait stack-delete-complete \
  --stack-name datacenter-priority-stack
Is there anything specific about the stack resources you'd like to verify or any outputs you need to access?

What Problem Does This Solution Solve?
This priority queuing system using Amazon SQS and SNS solves the critical problem of processing workload prioritization in distributed systems. Here's when and why you would use this solution:

The Core Problem
In many business applications, not all tasks are equally important or time-sensitive. Without a priority system, messages are typically processed in FIFO (First-In-First-Out) order, which can cause:

Critical tasks being delayed behind less important ones

SLA violations for high-value customers

Inefficient resource allocation during traffic spikes

Poor user experience when urgent requests wait in queue

Real-World Use Cases
E-commerce Order Processing

High Priority: Premium members' orders, expedited shipping requests

Low Priority: Regular orders, inventory updates, analytics data

Customer Support Systems

High Priority: VIP customers, critical system outages, payment issues

Low Priority: General inquiries, feedback, feature requests

Healthcare Applications

High Priority: Emergency patient data, critical lab results

Low Priority: Routine checkups, administrative updates

Financial Services

High Priority: Large transactions, fraud alerts, regulatory reporting

Low Priority: Balance inquiries, transaction history requests

IoT/Device Management

High Priority: Safety alerts, system failures, security events

Low Priority: Telemetry data, routine status updates

How the Architecture Works
text
┌─────────────────┐
│   SNS Topic     │
│ (Priority Queue)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│ High Priority   │     │ Low Priority    │
│ SQS Queue       │     │ SQS Queue       │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
          ┌─────────────────────┐
          │   Lambda Function   │
          │ (Priority Processor)│
          └─────────────────────┘
Key Benefits of This Solution
Guaranteed Priority Processing

High-priority messages are always processed before low-priority ones

Prevents critical tasks from being stuck behind non-urgent work

Resource Optimization

System resources are allocated where they matter most

During peak loads, important work gets priority access to compute capacity

Scalability

Each queue can scale independently based on message volume

Lambda can process messages concurrently from both queues

Decoupling and Reliability

SNS decouples publishers from subscribers

SQS provides reliable message delivery with retry capabilities

Cost Efficiency

Process high-priority messages immediately

Batch or delay low-priority processing during off-peak hours

Pay only for what you process

When to Use This Solution
Ideal Scenarios:

✅ You have clear business priority tiers (e.g., Gold/Silver/Bronze customers)

✅ Urgent tasks must be processed within strict time constraints

✅ You need to maintain service level agreements (SLAs)

✅ Workloads are unpredictable but priorities are well-defined

✅ You want to implement "circuit breaker" pattern for critical paths

Not Recommended When:

❌ All messages have equal importance

❌ Order of processing doesn't matter (pure event streaming)

❌ You need exactly-once ordering across all messages

❌ Message volume is very low (single queue would suffice)

Testing the Solution
To test your deployment:

bash
# Publish a message with priority metadata
aws sns publish \
  --topic-arn "arn:aws:sns:us-east-1:048824521363:datacenter-Priority-Queues-Topic" \
  --message '{"priority": "high", "data": "Urgent request"}' \
  --message-attributes '{"priority": {"DataType": "String", "StringValue": "high"}}'

# Publish a low-priority message
aws sns publish \
  --topic-arn "arn:aws:sns:us-east-1:048824521363:datacenter-Priority-Queues-Topic" \
  --message '{"priority": "low", "data": "Routine update"}' \
  --message-attributes '{"priority": {"DataType": "String", "StringValue": "low"}}'
The Lambda function will consistently process the high-priority message first, demonstrating the system's effectiveness in ensuring critical workloads are handled promptly.

Conclusion
This priority queuing solution is essential for modern cloud applications where not all work is equally important. It provides a robust, serverless architecture that ensures business-critical operations receive preferential treatment while still processing lower-priority work efficiently. This pattern is widely used in production systems handling customer-facing applications where user experience and business outcomes depend on timely processing of high-value requests.

