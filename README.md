# LevelUp! Lab for Serverless

## Lab Overview And High Level Design

Let's start with the High Level Design.

![High Level Design](./pictures/high-level-design.jpg)

An Amazon API Gateway is a collection of resources and methods. For this tutorial, you create one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). That is, when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

* Create, update, and delete an item.
* Read an item.
* Scan an item.
* Other operations (echo, ping), not related to DynamoDB, that you can use for testing.

The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}
```
The following is a sample request payload for a DynamoDB read item operation:
```json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```

## Setup

### Create Lambda IAM Role 
Create the execution role that gives your function permission to access AWS resources.

To create an execution role
First Create the custom policy with permission to DynamoDB and CloudWatch Logs. If this policy is already present then create the new role. If the policy is not present then create it with the below json 
1. Open the policies page in the IAM console. Choose Create Policy. Click on the json editor. Copy paste the below json policy in the editor. This policy has permissions that the function needs to write data to DynamoDB and upload logs.
   ```json
    {
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "Stmt1428341300017",
      "Action": [
        "dynamodb:DeleteItem",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "",
      "Resource": "*",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow"
    }
    ]
    }
    ```
2. Create this policy with the name **customPolicy-dynamoDB-CloudWatch**
3. Go to the roles page in the IAM console.
4. Choose Create role.
5. Create a role with the following properties.
    * Trusted entity – Lambda.
    * Role name – **lambda-apigateway-role**.
    * Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy **customPolicy-dynamoDB-CloudWatch** has been created in step 1 or it already exists in your account. Attach that policy here.

![Role and Policies Screenshot](./pictures/1.jpg)

![Role and Policies Screenshot](./pictures/2.jpg)

### Create Lambda Function

**To create the function**
1. Click "Create function" in AWS Lambda Console

![Create function](./pictures/3.png)

2. Select "Author from scratch". Use name **LambdaFunctionOverHttps** , select **Python 3.12** as Runtime. Under Permissions, select **Use an existing role**, and select **lambda-apigateway-role** that we created, from the drop down

3. Click "Create function"

![Lambda basic information](./pictures/4.jpg)

4. Replace the boilerplate coding with the following code snippet and click "Save"

**Example Python Code**
```python
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
```
![Lambda Code](./pictures/5.jpg)

### Test Lambda Function

Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.
1. Click the arrow on "Select a test event" and click "Configure test events"

2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output. Click "Save" to save
```json
{
    "operation": "echo",
    "payload": {
        "somekey1": "TestValue1",
        "somekey2": "TestValue2"
    }
}
```
![Save test event](./pictures/6.jpg)

3. Click "Test", and it will execute the test event. You should see the output in the console

![Execute test event](./pictures/7.jpg)

We're all set to create DynamoDB table and an API using our lambda as backend!

### Create DynamoDB Table

Create the DynamoDB table that the Lambda function uses.

**To create a DynamoDB table**

1. Open the DynamoDB console.
2. Choose Create table.
3. Create a table with the following settings.
   * Table name – lambda-apigateway
   * Primary key – id (string)
4. Choose Create.

![create DynamoDB table](./pictures/8.jpg)


### Create API

**To create the API**
1. Go to API Gateway console
2. Click on the three lines on the extreme left side of the webpage to expand the menu. Then click on APIs.
3. Scroll down to REST API. Select "Build" 

![create API](./pictures/9.jpg) 

4. Give the API name as **DynamoDBOperations**, keep everything as is, click "Create API"

![Create REST API](./pictures/10.jpg)

Now the page changes to the API GW API resource view for this newly created API.

![Create REST API](./pictures/11.jpg)

5. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next. 

Click "Create Resource" in the middle pane under "Resources" heading. A resource detail entry page opens up. 

6. Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"

![Create resource](./pictures/12.jpg)

7. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Create Method" button in the right side below pane of the page labeled "Methods". 

![Create resource method](./pictures/13.jpg)

8. Select "POST" from drop down menu , then choose the lamdba function name in the search bar. Search for the lambda create "LambdaFunctionOverHttps". Select it. Leave everything as is and select "Create Method".
The screen will refresh and below screen will be shown. This new screen has the integration directly with Lambda.

Click on the Lambda icon. A popup shows the name of the lambda function which is integrated. 

![Create lambda integration](./pictures/14.jpg)

Our API-Lambda integration is done!

### Deploy the API

In this step, you deploy the API that you created to a stage called prod.

1. Click the "Deploy API" button on the top right corner of the screen. A pop-up comes up.

2. Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"

    ![Deploy API to Prod Stage](./pictures/15.jpg)

   Now the screen refreshes and the staging screen is shown.

    ![Staging screen](./pictures/16.jpg)

3. We're all set to run our solution! To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen

![Copy Invoke Url](./pictures/17.jpg)


### Running our solution

1. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "FirstNumberInserted",
            "number": 25
        }
    }
}
```
2. To execute our API from local machine, we are going to use Postman and Curl command. You can choose either method based on your convenience and familiarity. 
    * To run this from Postman, select "POST" , paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.

    ![Execute from Postman](./pictures/18.jpg)

    * To run this from terminal using Curl, run the below
    ```
    $ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
    ```   
3. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select "lambda-apigateway" table, select "Items" tab, and the newly inserted item should be displayed.

![Dynamo Item](./pictures/19.jpg)

4. Run the step 1 of inserting records into DynamoDB table a few times each time modifying the **id** value and the **number** value in the above json.

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "<NewValue>",
            "number": <NewValue>
        }
    }
}
```

4. Now post running the record 5/6 times, to get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```

![List Dynamo Items](./pictures/20.jpg)

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Performance testing the API using Postman
The performance testing of API can be very easily done through Postman. The steps to do that are listed below.

## Setup for Creating Collection and POST Request
1. Create a blank collection in Postman. Rename that to **ServerlessMicroServicePerformanceTesting**. The screen of Postman will look like this now.
   
   ![Creating Blank Collection](./pictures/21.jpg)
   
2. Now click on the three dots right adjacent to the named collection **ServerlessMicroServicePerformanceTesting**. A dropdown comes. Select **New Request**. 

   ![Creating Blank Collection](./pictures/22.jpg)

   This will create a new request. The Page will look like this now.

   ![Creating Blank Collection](./pictures/23.jpg)
   
3. Now in this new request section, select the dropdown and choose **POST**. Paste the invoke URL from the POST method used above to list the records. Also add the list operation JSON in the body in raw format.
   
   ```json
   {
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
   }
   ```
  Save this request by clicking save.

## Creating a Custom Domain Name 
### Step 1 Create a custom domain name
### Step 2 Create a certificate in ACM
### Step 3 Map the custom domain name to the API GW
### Step 4 Trigger the API GW through the Custom domain name

## Performance Testing 

### Running the load testing scenario via Postman

### Changing the configuration of Lambda

### Re-running the load testing scenario via Postman

## Cleanup

Let's clean up the resources we have created for this lab.

### Cleaning up DynamoDB

To delete the table, from DynamoDB console, select the table "lambda-apigateway", and click "Delete table"

![Delete Dynamo](./pictures/delete-dynamo.jpg)

To delete the Lambda, from the Lambda console, select lambda "LambdaFunctionOverHttps", click "Actions", then click Delete 

![Delete Lambda](./pictures/delete-lambda.jpg)

To delete the API we created, in API gateway console, under APIs, select "DynamoDBOperations" API, click "Actions", then "Delete"

![Delete API](./pictures/delete-api.jpg)
