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
  <img width="913" alt="image" src="https://github.com/user-attachments/assets/f3099a02-c135-4b1f-8f93-21ab6a7b61bb">
 
   - **Role 2: APIGateway-SQS**
     - **Purpose:** Grants permissions to send data to the SQS queue and push logs to Amazon CloudWatch for troubleshooting.
     - **Trusted entity type:** AWS service
     - **Common use cases:** API Gateway
     - **Attach policies:** 
       - `AmazonAPIGatewayPushToCloudWatchLogs`
<img width="811" alt="image" src="https://github.com/user-attachments/assets/953aedef-d472-40bc-b43c-8e6673ae81d5">

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
<img width="461" alt="image" src="https://github.com/user-attachments/assets/cd0cec3e-350d-4674-9e19-633002e255cc">

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
<img width="602" alt="image" src="https://github.com/user-attachments/assets/f8e7f84b-273b-4a17-95ab-2aa815f4df2e">

5. **Complete the Queue Creation:**
   - Click **Create queue** to finalize the creation of the SQS queue.
<img width="851" alt="image" src="https://github.com/user-attachments/assets/4e383c43-7a6a-429d-82a5-ab0709ee4d1f">

### Step 4: Creating a Lambda Function and Setting Up Triggers

1. **Create a Lambda Function:**
   - Go to the AWS Management Console and open the Lambda service.
   - Choose "Create function" and configure the following:
     - **Function name:** `POC-Lambda-1`
     - **Runtime:** Python 3.9
     - **Execution role:** Use the existing role `Lambda-SQS-DynamoDB`.
<img width="585" alt="image" src="https://github.com/user-attachments/assets/62024350-12fe-482d-9b9b-a2c1fe263a09">

2. **Set Up SQS as a Trigger:**
   - Add SQS as a trigger for the Lambda function.
   - Choose `POC-Queue` as the SQS queue.
<img width="443" alt="image" src="https://github.com/user-attachments/assets/9881a289-3b6d-48b4-b94f-9fc4b3b4db0e">
<img width="555" alt="image" src="https://github.com/user-attachments/assets/2a65a440-24ff-41d5-86db-0f4c80de5d3c">

3. **Deploy the Lambda Function Code:**
   - Replace the default code with the provided Python code that reads from SQS and writes to DynamoDB.
     ```python
      import boto3, uuid
      
      client = boto3.resource('dynamodb')
      table = client.Table("orders")
      
      def lambda_handler(event, context):
          for record in event['Records']:
              print("test")
              payload = record["body"]
              print(str(payload))
              table.put_item(Item= {'orderID': str(uuid.uuid4()),'order':  payload})
     ```
   - Deploy the code.
<img width="429" alt="image" src="https://github.com/user-attachments/assets/8e0f1c8c-8af3-4fa9-a707-714877b0a517">

4. **Test the Lambda Function:**
   - Create a test event in the Lambda console using the SQS template.
     <img width="404" alt="image" src="https://github.com/user-attachments/assets/b005e40b-a31f-452f-99b8-6f4197f999c2">

   - Verify the test message "Hello from SQS!" is written to the DynamoDB table.
<img width="694" alt="image" src="https://github.com/user-attachments/assets/a6a75795-fbe9-4e49-9b28-e5a6e682ccf9">

### Step 5: Enabling DynamoDB Streams

1. **Enable DynamoDB Streams:**
   - In the DynamoDB console, enable streams for the `orders` table.
     <img width="784" alt="image" src="https://github.com/user-attachments/assets/3beb2540-48fa-4de2-af8b-0a04883e0b41">

   - Set the view type to "New image."
<img width="441" alt="image" src="https://github.com/user-attachments/assets/5ce3e090-c050-46f0-888f-65513379224d">

### Step 6: Creating an SNS Topic and Setting Up Subscriptions

1. **Create an SNS Topic:**
   - Open the SNS service in the AWS Management Console.
   - Create a new topic named `POC-Topic` and save its ARN.
<img width="805" alt="image" src="https://github.com/user-attachments/assets/323d7f4c-90db-407a-b8ab-84b5a5f3e5a5">

2. **Subscribe to Email Notifications:**
   - Create a subscription for the `POC-Topic` using the email protocol.
   - Confirm the subscription via the email sent to your inbox.
<img width="589" alt="image" src="https://github.com/user-attachments/assets/d52ac59c-551a-474d-ade1-0cf239b3bc27">
<img width="668" alt="image" src="https://github.com/user-attachments/assets/9feab5e5-a4fe-4241-b9a6-928a6eb9c95f">

### Step 7: Creating an AWS Lambda Function to Publish a Message to the SNS Topic

1. **Create a Second Lambda Function:**
   - Create a Lambda function named `POC-Lambda-2` using the existing role `Lambda-DynamoDBStreams-SNS`.

2. **Setting up DynamoDB as a trigger to invoke a Lambda function**
    - In the Function overview section, choose Add trigger and configure the following settings:
    - Trigger configuration: Enter DynamoDB and from the list, choose DynamoDB.
      DynamoDB table: orders
    - Keep the remaining default settings and choose Add.

In the Configuration tab, make sure that you are in the Triggers section and that the DynamoDB state is “Enabled.”
<img width="436" alt="image" src="https://github.com/user-attachments/assets/aed87266-6a66-4d8e-889a-990abb71284e">

3. **Configure the Lambda Function:**
   - Replace the default code with the provided Python code that publishes a message to SNS.
       ```python
        import boto3, json
        
        client = boto3.client('sns')
        
        def lambda_handler(event, context):
        
            for record in event["Records"]:
        
                if record['eventName'] == 'INSERT':
                    new_record = record['dynamodb']['NewImage']    
                    response = client.publish(
                        TargetArn='<Enter Amazon SNS ARN for the POC-Topic>',
                        Message=json.dumps({'default': json.dumps(new_record)}),
                        MessageStructure='json'
                    )
     ```
   - Deploy the code.
<img width="553" alt="image" src="https://github.com/user-attachments/assets/de9edb08-3598-4f91-a113-33de3cf09f10">

4. **Test the Lambda Function:**
   - Test the function with a DynamoDB event, and verify that you receive an email notification.
<img width="410" alt="image" src="https://github.com/user-attachments/assets/be563646-79e3-49c1-8e3e-5e0c4d801bec">
<img width="688" alt="image" src="https://github.com/user-attachments/assets/7c41ff9f-c232-435e-92c9-d082271da08d">
<img width="864" alt="image" src="https://github.com/user-attachments/assets/a9c162da-0bfe-4ffc-ab52-6f8a881a6387">

### Step 8: Creating an API with Amazon API Gateway

1. **Create a REST API:**
   - Open API Gateway and create a new REST API named `POC-API`.
<img width="448" alt="image" src="https://github.com/user-attachments/assets/ef378d15-e92f-48f4-90c3-323f83129938">

2. **Configure the API:**
   - Create a POST method and integrate it with the SQS service to send messages to `POC-Queue`.
   - Integration type: AWS Service
   - AWS Region: us-east-1
   - AWS Service: Simple Queue Service (SQS)
   - AWS Subdomain: Keep empty
   - HTTP method: POST
   - Action Type: Use path override
   - Path override: Enter your account ID followed by a slash (/) and the name of the POC queue
   - Note: If POC-Queue is the name of the SQS queue that you created, this entry might look similar to the following: /<account ID>/POC-Queue
   - Execution role: Paste the ARN of the APIGateway-SQS role
      Note: For example, the ARN might look like the following: arn:aws:iam::<account ID>:role/APIGateway-SQS
   - Content Handling: Passthrough
   - Save your changes.
<img width="783" alt="image" src="https://github.com/user-attachments/assets/05194f91-2d7d-407a-9088-2309b2736c96">
<img width="443" alt="image" src="https://github.com/user-attachments/assets/1dc6837f-3b34-4cf0-943b-da889e38ee85">
<img width="427" alt="image" src="https://github.com/user-attachments/assets/eb36b288-14e4-47aa-9e8b-21631e13bb9f">

3. **Add Mapping Templates:**
   - Configure the API to map the request body and headers correctly for SQS.

   - Choose the Integration Request card.
    <img width="780" alt="image" src="https://github.com/user-attachments/assets/7784ea74-7efa-4893-b4c6-0ad28bc6ad7d">

   - Scroll to the bottom of the page and expand HTTP Headers.
    
   - Choose Add header.
    
   - For Name, enter Content-Type
    
   - For Mapped from, enter 'application/x-www-form-urlencoded'
    <img width="432" alt="image" src="https://github.com/user-attachments/assets/da6016f6-d3d1-4957-96a8-3f3e82b8dda7">

   - Save your changes to the HTTP Headers section by choosing the check mark.
    
   - Expand Mapping Templates and for Request body passthrough, choose Never.
    
   - Choose Add mapping template and for Content-Type , enter application/json
    <img width="412" alt="image" src="https://github.com/user-attachments/assets/3ddf9e41-a13d-4ca2-8461-24b3f0d8bf79">

   - Save your changes by choosing the check mark.
   

   - For Generate template, do not choose a default template from the list. Instead, enter the following command: Action=SendMessage&MessageBody=$input.body in a box.
    
   - Choose Save.
      <img width="607" alt="image" src="https://github.com/user-attachments/assets/71eb24cd-7f49-4bd5-b5b1-4879973d5b94">
### Step 9: Testing the Architecture by Using API Gateway

1. **Test the API:**
   - Use API Gateway's test feature to send a mock request.
      ```json
     {  "item": "latex gloves",
    "customerID":"12345"}
    ```
    <img width="775" alt="image" src="https://github.com/user-attachments/assets/18d3a0ad-99e7-4da4-ad61-370f52876397">
    <img width="546" alt="image" src="https://github.com/user-attachments/assets/c80d9ec4-36d1-495d-946f-329203f0164d">

   - Verify that the request is processed through SQS, Lambda, DynamoDB, and SNS.
<img width="731" alt="image" src="https://github.com/user-attachments/assets/bd382c5c-fbf9-4d9b-8152-274390e9b7c9">
<img width="763" alt="image" src="https://github.com/user-attachments/assets/e33e264c-7167-4fe7-a03d-1feadaae8293">

### Step 10: Cleaning Up

1. **Delete AWS Resources:**
   - Delete the DynamoDB table, Lambda functions, SQS queue, SNS topic, API Gateway, and associated IAM roles and policies.
