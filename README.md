# Serverless Architecture with AWS: API Gateway, Lambda, SQS, DynamoDB, and SNS

This README provides detailed instructions on setting up a serverless architecture using AWS services. The architecture involves an API Gateway, SQS, Lambda functions, DynamoDB, and SNS for handling data processing and notifications.

In this example, Customer wants to ingest orders through a portal and requests are served via REST api calls. Later in the backend application, need to store the order details and process it to further downstream systems to consume and act upon.


![ServerlessSolution](https://github.com/user-attachments/assets/8f0660b8-06ad-45f4-8007-1aef6ab2fe98)


- API Gateway : Since customer needed scalable and operationally less dependent. And existing implementation of thier application is built on REST protocol, so had to choose API gateway. Also for the Sale day and hight traffic day API gateway was best for the scalability reason.
- SQS - Amazon SQS is used in this architecture to decouple the backend business logic to store process , customers dont need to wait for that to happen. So Api gateway can return success to customer once the order is accepted. And SQS can hold messages up to 14 days based on the configuration.
- AWS Lambda - Customers wanted pay as you use compute model and didnt want to manage servers, so AWS lambda was choosen here to support this requirement.
- Amazon Dynamo DB - Since one of customer feedback on the data type for order was just simple and non relational for further systems or business logic needed. So adding Dynamo DB also helps during the Peak traffic , sales event to accept DB updates with milliseconds throughput.
- DynamoDBstreams - This was needed to setup Bridge between store Orders and downstream systems for processing newly added orders.
- AWS Lambda & SNS - Amazon Lambda & Amazon SNS is added between DynamoDB streams and downstream systems , Lambda would read the dynamoDB streams for newly added orders and publish messsge to SNS topic. And later SNS topic notifies the Downstream systems or email/sms .
  



## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [Step 1: Create IAM Policies and Roles](#step-1-create-iam-policies-and-roles)
  - [Step 2: Creating a DynamoDB Table](#step-2-creating-a-dynamodb-table)
  - [Step 3: Creating an SQS Queue](#step-3-creating-an-sqs-queue)
  - [Step 4: Creating a Lambda Function and Setting Up Triggers](#step-4-creating-a-lambda-function-and-setting-up-triggers)
  - [Step 5: Enable DynamoDB Streams](#step-5-enabling-dynamodb-streams)
  - [Step 6: Creating an SNS Topic and Setting Up Subscriptions](#step-6-creating-an-sns-topic-and-setting-up-subscriptions)
  - [Step 7: Creating an AWS Lambda Function to Publish a Message to the SNS Topic](#step-7-creating-an-aws-lambda-function-to-publish-a-message-to-the-sns-topic)
  - [Step 8: Creating an API with Amazon API Gateway](#step-8-creating-an-api-with-amazon-api-gateway)
  - [Step 9: Testing the Architecture by Using API Gateway](#step-9-testing-the-architecture-by-using-api-gateway)
  - [Step 10: Cleaning Up](#step-10-cleaning-up)


## Architecture Overview

The architecture is composed of the following components:

1. **API Gateway**: Accepts HTTP requests and forwards them to an SQS queue.
2. **SQS Queue**: Acts as a buffer and triggers the first Lambda function.
3. **Lambda Function 1**: Processes messages from SQS and stores data in DynamoDB.
4. **DynamoDB Streams**: Tracks data changes in DynamoDB and triggers the second Lambda function.
5. **Lambda Function 2**: Sends notifications via SNS when new data is inserted into DynamoDB.
6. **SNS**: Sends notifications, such as emails, when triggered by the Lambda function.

## Prerequisites

- An AWS account
- Familiarity with AWS services: IAM, Lambda, DynamoDB, SQS, SNS, API Gateway
- AWS CLI (optional but recommended for certain steps)

## Setup Instructions

### Step 1: Create IAM Policies and Roles

#### Create IAM Policies

- **Lambda-Write-DynamoDB**
  
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "dynamodb:PutItem",
          "dynamodb:DescribeTable"
        ],
        "Resource": "*"
      }
    ]
  }
**Lambda-SNS-Publish**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish",
        "sns:GetTopicAttributes",
        "sns:ListTopics"
      ],
      "Resource": "*"
    }
  ]
}
```

**Lambda-DynamoDBStreams-Read**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetShardIterator",
        "dynamodb:DescribeStream",
        "dynamodb:ListStreams",
        "dynamodb:GetRecords"
      ],
      "Resource": "*"
    }
  ]
}
```

**Lambda-Read-SQS**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:DeleteMessage",
        "sqs:ReceiveMessage",
        "sqs:GetQueueAttributes",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": "*"
    }
  ]
}
```

### Step 1.2: Creating IAM Roles and Attaching Policies

To follow the principle of least privilege, AWS recommends providing role-based access to only the resources required to perform a specific task. In this step, you'll create IAM roles and attach policies to these roles.

## Creating IAM Roles

1. **Navigate to IAM Dashboard:**
   - In the AWS Management Console, go to the IAM dashboard.
   - In the navigation pane, select **Roles**.

2. **Create a Role for Lambda to Interact with DynamoDB and SQS:**
   - Click on **Create role**.
   - On the **Select trusted entity** page, configure the following:
     - **Trusted entity type:** AWS service
     - **Common use cases:** Lambda
   - Click **Next**.
   - On the **Add permissions** page, select the following policies:
     - `Lambda-Write-DynamoDB`
     - `Lambda-Read-SQS`
   - Click **Next**.
   - For **Role name**, enter `Lambda-SQS-DynamoDB`.
   - Click **Create role**.

3. **Create Additional IAM Roles:**
   - **Role 1: Lambda-DynamoDBStreams-SNS**
     - **Purpose:** Allows Lambda to obtain records from DynamoDB Streams and send them to Amazon SNS.
     - **Trusted entity type:** AWS service
     - **Common use cases:** Lambda
     - **Attach policies:** 
       - `Lambda-SNS-Publish`
       - `Lambda-DynamoDBStreams-Read`
   
   - **Role 2: APIGateway-SQS**
     - **Purpose:** Grants permissions to send data to the SQS queue and push logs to Amazon CloudWatch for troubleshooting.
     - **Trusted entity type:** AWS service
     - **Common use cases:** API Gateway
     - **Attach policies:** 
       - `AmazonAPIGatewayPushToCloudWatchLogs`

### Step 2: Creating a DynamoDB Table

In this task, you will create a DynamoDB table to ingest data passed through API Gateway.

## Steps to Create the DynamoDB Table

1. **Navigate to DynamoDB:**
   - In the AWS Management Console, locate the search box at the top.
   - Type **DynamoDB** and select the **DynamoDB** service from the list.

2. **Create a New Table:**
   - On the DynamoDB dashboard, find the **Get started** card and click on **Create table**.
   - Configure the table with the following settings:
     - **Table name:** `orders`
     - **Partition key:** `orderID`
     - **Data type:** String (default)
   - Leave all other settings at their default values.

3. **Complete the Table Creation:**
   - Click **Create table** to finalize the creation of the DynamoDB table.


### Step 3: Creating an SQS Queue

In this task, you will create an Amazon SQS queue. This queue will receive data records from API Gateway, store them, and send them to a database.

## Steps to Create the SQS Queue

1. **Navigate to Simple Queue Service (SQS):**
   - In the AWS Management Console, locate the search box at the top.
   - Type **SQS** and select **Simple Queue Service** from the list.

2. **Create a New Queue:**
   - On the SQS dashboard, find the **Get started** card and click on **Create queue**.
   - The **Create queue** page will appear.

3. **Configure Queue Settings:**
   - **Name:** Enter `POC-Queue`.
   - **Access Policy:** Choose **Basic**.

4. **Define Message Permissions:**
   - **Who can send messages to the queue:**
     - Select **Only the specified AWS accounts, IAM users, and roles**.
     - Paste the Amazon Resource Name (ARN) for the `APIGateway-SQS` IAM role.
       - Example: `arn:aws:iam::<account_ID>:role/APIGateway-SQS`
   - **Who can receive messages from the queue:**
     - Select **Only the specified AWS accounts, IAM users, and roles**.
     - Paste the ARN for the `Lambda-SQS-DynamoDB` IAM role.
       - Example: `arn:aws:iam::<account_ID>:role/Lambda-SQS-DynamoDB`

5. **Complete the Queue Creation:**
   - Click **Create queue** to finalize the creation of the SQS queue.

### Step 4: Creating a Lambda Function and Setting Up Triggers

1. **Create a Lambda Function:**
   - Go to the AWS Management Console and open the Lambda service.
   - Choose "Create function" and configure the following:
     - **Function name:** `POC-Lambda-1`
     - **Runtime:** Python 3.9
     - **Execution role:** Use the existing role `Lambda-SQS-DynamoDB`.

2. **Set Up SQS as a Trigger:**
   - Add SQS as a trigger for the Lambda function.
   - Choose `POC-Queue` as the SQS queue.

3. **Deploy the Lambda Function Code:**
   - Replace the default code with the provided Python code that reads from SQS and writes to DynamoDB.
   - Deploy the code.

4. **Test the Lambda Function:**
   - Create a test event in the Lambda console using the SQS template.
   - Verify the test message "Hello from SQS!" is written to the DynamoDB table.

### Step 5: Enabling DynamoDB Streams

1. **Enable DynamoDB Streams:**
   - In the DynamoDB console, enable streams for the `orders` table.
   - Set the view type to "New image."

### Step 6: Creating an SNS Topic and Setting Up Subscriptions

1. **Create an SNS Topic:**
   - Open the SNS service in the AWS Management Console.
   - Create a new topic named `POC-Topic` and save its ARN.

2. **Subscribe to Email Notifications:**
   - Create a subscription for the `POC-Topic` using the email protocol.
   - Confirm the subscription via the email sent to your inbox.

### Step 7: Creating an AWS Lambda Function to Publish a Message to the SNS Topic

1. **Create a Second Lambda Function:**
   - Create a Lambda function named `POC-Lambda-2` using the existing role `Lambda-DynamoDBStreams-SNS`.

2. **Set Up DynamoDB as a Trigger:**
   - Add the `orders` table as a trigger for the new Lambda function.

3. **Configure the Lambda Function:**
   - Replace the default code with the provided Python code that publishes a message to SNS.
   - Deploy the code.

4. **Test the Lambda Function:**
   - Test the function with a DynamoDB event, and verify that you receive an email notification.

### Step 8: Creating an API with Amazon API Gateway

1. **Create a REST API:**
   - Open API Gateway and create a new REST API named `POC-API`.

2. **Configure the API:**
   - Create a POST method and integrate it with the SQS service to send messages to `POC-Queue`.

3. **Add Mapping Templates:**
   - Configure the API to map the request body and headers correctly for SQS.

### Step 9: Testing the Architecture by Using API Gateway

1. **Test the API:**
   - Use API Gateway's test feature to send a mock request.
   - Verify that the request is processed through SQS, Lambda, DynamoDB, and SNS.

### Step 10: Cleaning Up

1. **Delete AWS Resources:**
   - Delete the DynamoDB table, Lambda functions, SQS queue, SNS topic, API Gateway, and associated IAM roles and policies.
