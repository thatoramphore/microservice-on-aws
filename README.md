# AWS Serverless Microservice

## Overview And High Level Design



![High Level Design](./images/high-level-design.png)

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
            "name": "Thato"
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

1. Open the roles page in the IAM console.
2. Choose Create role.
3. Create a role with the following properties.
    * Trusted entity – Lambda.
    * Role name – **lambda-apigateway-role**.
    * Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that  the function needs to write data to DynamoDB and upload logs. 
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

### Create Lambda Function

**To create the function**
1. Click "Create function" in AWS Lambda Console

![Create function](./images/create-lambda-1.png)


![Create function](./images/create-lambda-2.png)

2. Select “Author from scratch”. Use the name **LambdaFunctionOverHttps** , select **Python** as Runtime. Under Permissions, select "Use an existing role", and select **lambda-apigateway-role** that was created, from the drop down

3. Click "Create function"

4. Replace the boilerplate code with the following code snippet and click "Save"

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
![Lambda Code](./images/lambda-code.png)

### Test Lambda Function

Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.
1. Click the arrow on "Select a test event" and click "Configure test events"

![Configure test events](./images/lambda-test-event-create.png)

2. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output. Click "Create" to save
```json
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
```
![Save test event](./images/save-test-event.png)

3. Click "Test", and it will execute the test event. You should see the output in the console

![Execute test event](./images/execute-test.png)

We're all set to create DynamoDB table and an API using our lambda as backend!

### Create DynamoDB Table

Create the DynamoDB table that the Lambda function uses.

**To create a DynamoDB table**

1. Open the DynamoDB console.
2. Click ‘Create table’

![create DynamoDB table](./images/create-dynamo-table1.png)

3. Create  a table with the following settings:

    - Table name - lambda-apigateway
    - Primary key - id(string)

![create DynamoDB table](./images/create-dynamo-table2.png)

Click 'Create table' for the last time.


### Create API

**To create the API**
1. Open the API Gateway console
2. Scroll down and click "Build" for REST API

![create API](./images/build-api-button.png) 

3. Give the API name as "DynamoDBOperations", keep everything as is, and click "Create API"

![Build REST API](./images/create-new-api.png) 

4. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next

5. Click on your newly created API and then click ‘Create resource’

6. On the next screen, input "DynamoDBManager" in the Resource Name, Resource Path will get populated automatically. Click "Create resource"

![Create resource](./images/create-api-resource.png)

7. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, click "Create Method" 

![Create resource method](./images/create-method-button.png)

8. From the dropdown select ‘POST’ as the method type

![Create resource method](./images/create-post-method.png)

9. The integration will come up automatically with "Lambda Function" option selected. Select "LambdaFunctionOverHttps" function that we created earlier. As you start typing the name, your function name will show up. Select and click "Create method". 
Click the checkmark to allow Lambda Integration.

![Create lambda integration](./images/create-lambda-integration.png)

Our API-Lambda integration is done!

### Deploy the API

In this step, you deploy the API that you created to a stage called Production.

1. Click on the ‘POST’ method created above. Click ‘Deploy API’

2. A popup window will come up to add a stage, choose “New Stage '' and for Stage Name type “Production”. Click "Deploy".

![Deploy API](./images/deploy-api-stage.png)



<p align="center">
  &#x25BE;
</p>



![Deploy API to Prod Stage](./images/deploy-api-button.png)

3. We're all set to run our solution! To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen

![Copy Invoke Url](./images/copy-invoke-url.png)


### Run The Solution

1. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
```
Before running the operation to create an item in the DynamoDB table, the item count is 0. Upon successful execution, the will change to 1.

![Initial Item Count](./images/initial-item-count.png)


2. To execute our API from local machine, we are going to use Postman and Curl command. You can choose either method based on your convenience and familiarity.

    * To run this from Postman, select "POST" , paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.

    ![Execute from Postman](./images/create-from-postman.png)

    * To run this from terminal using Curl, run the below snippet
    ```
    $ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Thato\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
    ```   
3. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select "lambda-apigateway" table,  click 'Explore table items', and the newly inserted item id(s) should be displayed.

![Dynamo Item](./images/dynamo-item.png)

4. To get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "paload": {

    }
}
```
![List Dynamo Items](./images/dynamo-item-list.png)


Optionally you can run other operations, for this demo I will test the 'ping' operation, from which I am expecting to get 'pong' as a response.

To request this operation, use the following JSON:


```json
{
    "operation": "ping",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234567ABCDEFG",
            "number": 6
        }
    }
}
```


Ping Pong! Successfully got ‘pong’ as a response. [Doing a little dance at my desk 🕺]


![Ping Dynamo Items](./images/dynamo-item-ping.png)

To execute other operations follow the same steps as for the 'create' and 'ping' operations above and rename the operation to your desired operation.

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Cleanup

Let's clean up the resources we have created for this demo.


### Cleaning up DynamoDB

To delete the table, from DynamoDB console, select the table "lambda-apigateway", and click "Delete table"

![Delete Dynamo](./images/delete-dynamo-1.png)

![Delete Dynamo](./images/delete-dynamo-2.png)

### Cleaning up Lambda

To delete the Lambda, from the Lambda console, select lambda "LambdaFunctionOverHttps", click "Actions", then click Delete 

![Delete Lambda](./images/delete-lambda.png)

### Cleaning up API Gateway

To delete the API we created, in API gateway console, under APIs, select "DynamoDBOperations" API, click "Actions", then "Delete"

![Delete API](./images/delete-api.png)


## Conclusion

This demo showcased how to create your own microservice on AWS Cloud using Amazon API Gateway, AWS Lambda and Amazon DynamoDB. We also tested the microservice using Postman and cleaned up AWS resources to curb any unnecessary costs.
