# AWS Serverless Stack - Employee App

This stack provides:
- **Complete user authentication** with Cognito
- **Real-time GraphQL API** with AppSync
- **Scalable photo upload** with Lambda + S3
- **Leave request management** with automated notifications
- **Efficient data storage** with DynamoDB
- **Infrastructure as code** for easy deployment

Each component works together to create a fully functional employee management system without managing any servers.

## 1. AWS Cognito - User Authentication

### Setup Cognito User Pool
```bash
# AWS CLI command to create user pool
aws cognito-idp create-user-pool \
  --pool-name "EmployeeAppUserPool" \
  --policies "PasswordPolicy={MinimumLength=8,RequireUppercase=true,RequireLowercase=true,RequireNumbers=true}" \
  --auto-verified-attributes email \
  --username-attributes email
```

### Frontend Integration (React/JavaScript)
```javascript
// cognito-config.js
import { CognitoUserPool, CognitoUser, AuthenticationDetails } from 'amazon-cognito-identity-js';

const poolData = {
  UserPoolId: 'us-east-1_XXXXXXXXX', // Your User Pool ID
  ClientId: 'your-client-id-here'
};

const userPool = new CognitoUserPool(poolData);

// User Registration
export const signUp = (email, password, name) => {
  const attributeList = [
    {
      Name: 'email',
      Value: email
    },
    {
      Name: 'name',
      Value: name
    }
  ];
  
  return new Promise((resolve, reject) => {
    userPool.signUp(email, password, attributeList, null, (err, result) => {
      if (err) reject(err);
      else resolve(result);
    });
  });
};

// User Login
export const signIn = (email, password) => {
  const user = new CognitoUser({
    Username: email,
    Pool: userPool
  });
  
  const authDetails = new AuthenticationDetails({
    Username: email,
    Password: password
  });
  
  return new Promise((resolve, reject) => {
    user.authenticateUser(authDetails, {
      onSuccess: (result) => resolve(result),
      onFailure: (err) => reject(err)
    });
  });
};
```

## 2. AppSync GraphQL - API Layer

### GraphQL Schema
```graphql
# schema.graphql
type User @model @auth(rules: [{allow: owner}]) {
  id: ID!
  email: String!
  name: String!
  profilePicture: String
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
  leaveRequests: [LeaveRequest] @hasMany
}

type LeaveRequest @model @auth(rules: [{allow: owner}]) {
  id: ID!
  userId: ID!
  user: User @belongsTo
  startDate: AWSDate!
  endDate: AWSDate!
  reason: String!
  status: LeaveStatus!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

enum LeaveStatus {
  PENDING
  APPROVED
  REJECTED
}

type Photo @model @auth(rules: [{allow: owner}]) {
  id: ID!
  userId: ID!
  url: String!
  caption: String
  createdAt: AWSDateTime!
}

# Custom mutations
type Mutation {
  uploadPhoto(input: UploadPhotoInput!): Photo
  submitLeaveRequest(input: LeaveRequestInput!): LeaveRequest
}

input UploadPhotoInput {
  caption: String
  imageData: String! # base64 encoded image
}

input LeaveRequestInput {
  startDate: AWSDate!
  endDate: AWSDate!
  reason: String!
}
```

### Frontend GraphQL Queries
```javascript
// graphql-operations.js
import { gql } from '@apollo/client';

export const CREATE_USER = gql`
  mutation CreateUser($input: CreateUserInput!) {
    createUser(input: $input) {
      id
      email
      name
      profilePicture
      createdAt
    }
  }
`;

export const GET_USER_PROFILE = gql`
  query GetUser($id: ID!) {
    getUser(id: $id) {
      id
      email
      name
      profilePicture
      leaveRequests {
        items {
          id
          startDate
          endDate
          reason
          status
          createdAt
        }
      }
    }
  }
`;

export const SUBMIT_LEAVE_REQUEST = gql`
  mutation SubmitLeaveRequest($input: LeaveRequestInput!) {
    submitLeaveRequest(input: $input) {
      id
      startDate
      endDate
      reason
      status
      createdAt
    }
  }
`;

export const UPLOAD_PHOTO = gql`
  mutation UploadPhoto($input: UploadPhotoInput!) {
    uploadPhoto(input: $input) {
      id
      url
      caption
      createdAt
    }
  }
`;
```

## 3. Lambda Functions - Business Logic

### Photo Upload Lambda
```python
# photo_upload_lambda.py
import json
import boto3
import base64
import uuid
from datetime import datetime

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

def lambda_handler(event, context):
    try:
        # Parse input
        input_data = event['arguments']['input']
        user_id = event['identity']['sub']  # From Cognito
        image_data = input_data['imageData']
        caption = input_data.get('caption', '')
        
        # Decode base64 image
        image_bytes = base64.b64decode(image_data)
        
        # Generate unique filename
        photo_id = str(uuid.uuid4())
        filename = f"photos/{user_id}/{photo_id}.jpg"
        
        # Upload to S3
        bucket_name = 'employee-app-photos'
        s3.put_object(
            Bucket=bucket_name,
            Key=filename,
            Body=image_bytes,
            ContentType='image/jpeg'
        )
        
        # Get public URL
        photo_url = f"https://{bucket_name}.s3.amazonaws.com/{filename}"
        
        # Save to DynamoDB
        table = dynamodb.Table('Photo')
        table.put_item(Item={
            'id': photo_id,
            'userId': user_id,
            'url': photo_url,
            'caption': caption,
            'createdAt': datetime.utcnow().isoformat()
        })
        
        return {
            'id': photo_id,
            'url': photo_url,
            'caption': caption,
            'createdAt': datetime.utcnow().isoformat()
        }
        
    except Exception as e:
        raise Exception(f"Photo upload failed: {str(e)}")
```

### Leave Request Lambda
```python
# leave_request_lambda.py
import json
import boto3
import uuid
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
ses = boto3.client('ses')

def lambda_handler(event, context):
    try:
        # Parse input
        input_data = event['arguments']['input']
        user_id = event['identity']['sub']
        
        start_date = input_data['startDate']
        end_date = input_data['endDate']
        reason = input_data['reason']
        
        # Generate leave request ID
        request_id = str(uuid.uuid4())
        
        # Save to DynamoDB
        table = dynamodb.Table('LeaveRequest')
        table.put_item(Item={
            'id': request_id,
            'userId': user_id,
            'startDate': start_date,
            'endDate': end_date,
            'reason': reason,
            'status': 'PENDING',
            'createdAt': datetime.utcnow().isoformat()
        })
        
        # Send email notification to HR
        send_hr_notification(user_id, start_date, end_date, reason)
        
        return {
            'id': request_id,
            'startDate': start_date,
            'endDate': end_date,
            'reason': reason,
            'status': 'PENDING',
            'createdAt': datetime.utcnow().isoformat()
        }
        
    except Exception as e:
        raise Exception(f"Leave request submission failed: {str(e)}")

def send_hr_notification(user_id, start_date, end_date, reason):
    # Get user details
    user_table = dynamodb.Table('User')
    user = user_table.get_item(Key={'id': user_id})['Item']
    
    # Send email
    ses.send_email(
        Source='noreply@yourcompany.com',
        Destination={'ToAddresses': ['hr@yourcompany.com']},
        Message={
            'Subject': {'Data': 'New Leave Request'},
            'Body': {
                'Text': {
                    'Data': f"""
                    New leave request submitted:
                    Employee: {user['name']} ({user['email']})
                    Dates: {start_date} to {end_date}
                    Reason: {reason}
                    
                    Please review and approve/reject.
                    """
                }
            }
        }
    )
```

## 4. DynamoDB - Database Tables

### Table Schemas
```json
// User Table
{
  "TableName": "User",
  "KeySchema": [
    {
      "AttributeName": "id",
      "KeyType": "HASH"
    }
  ],
  "AttributeDefinitions": [
    {
      "AttributeName": "id",
      "AttributeType": "S"
    },
    {
      "AttributeName": "email",
      "AttributeType": "S"
    }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "email-index",
      "KeySchema": [
        {
          "AttributeName": "email",
          "KeyType": "HASH"
        }
      ],
      "Projection": {
        "ProjectionType": "ALL"
      }
    }
  ]
}

// LeaveRequest Table
{
  "TableName": "LeaveRequest",
  "KeySchema": [
    {
      "AttributeName": "id",
      "KeyType": "HASH"
    }
  ],
  "AttributeDefinitions": [
    {
      "AttributeName": "id",
      "AttributeType": "S"
    },
    {
      "AttributeName": "userId",
      "AttributeType": "S"
    }
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "userId-index",
      "KeySchema": [
        {
          "AttributeName": "userId",
          "KeyType": "HASH"
        }
      ],
      "Projection": {
        "ProjectionType": "ALL"
      }
    }
  ]
}
```

### DynamoDB Operations
```javascript
// dynamodb-operations.js
import AWS from 'aws-sdk';

const dynamodb = new AWS.DynamoDB.DocumentClient();

export const getUserLeaveRequests = async (userId) => {
  const params = {
    TableName: 'LeaveRequest',
    IndexName: 'userId-index',
    KeyConditionExpression: 'userId = :userId',
    ExpressionAttributeValues: {
      ':userId': userId
    },
    ScanIndexForward: false // Most recent first
  };
  
  const result = await dynamodb.query(params).promise();
  return result.Items;
};

export const updateLeaveRequestStatus = async (requestId, status) => {
  const params = {
    TableName: 'LeaveRequest',
    Key: { id: requestId },
    UpdateExpression: 'SET #status = :status, updatedAt = :updatedAt',
    ExpressionAttributeNames: {
      '#status': 'status'
    },
    ExpressionAttributeValues: {
      ':status': status,
      ':updatedAt': new Date().toISOString()
    }
  };
  
  await dynamodb.update(params).promise();
};
```

## 5. Frontend Integration Example

### React Component
```javascript
// EmployeeApp.js
import React, { useState, useEffect } from 'react';
import { useMutation, useQuery } from '@apollo/client';
import { SUBMIT_LEAVE_REQUEST, UPLOAD_PHOTO, GET_USER_PROFILE } from './graphql-operations';

const EmployeeApp = ({ userId }) => {
  const [leaveForm, setLeaveForm] = useState({
    startDate: '',
    endDate: '',
    reason: ''
  });
  
  const { data: userData, loading } = useQuery(GET_USER_PROFILE, {
    variables: { id: userId }
  });
  
  const [submitLeaveRequest] = useMutation(SUBMIT_LEAVE_REQUEST);
  const [uploadPhoto] = useMutation(UPLOAD_PHOTO);
  
  const handleLeaveSubmit = async (e) => {
    e.preventDefault();
    try {
      await submitLeaveRequest({
        variables: { input: leaveForm }
      });
      alert('Leave request submitted successfully!');
      setLeaveForm({ startDate: '', endDate: '', reason: '' });
    } catch (error) {
      alert('Error submitting leave request');
    }
  };
  
  const handlePhotoUpload = async (e) => {
    const file = e.target.files[0];
    if (!file) return;
    
    const reader = new FileReader();
    reader.onload = async (event) => {
      const base64Data = event.target.result.split(',')[1];
      try {
        await uploadPhoto({
          variables: {
            input: {
              imageData: base64Data,
              caption: 'Profile photo'
            }
          }
        });
        alert('Photo uploaded successfully!');
      } catch (error) {
        alert('Error uploading photo');
      }
    };
    reader.readAsDataURL(file);
  };
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div>
      <h1>Employee Portal</h1>
      
      {/* Photo Upload */}
      <div>
        <h2>Upload Photo</h2>
        <input type="file" accept="image/*" onChange={handlePhotoUpload} />
      </div>
      
      {/* Leave Request Form */}
      <div>
        <h2>Submit Leave Request</h2>
        <form onSubmit={handleLeaveSubmit}>
          <input
            type="date"
            value={leaveForm.startDate}
            onChange={(e) => setLeaveForm({...leaveForm, startDate: e.target.value})}
            required
          />
          <input
            type="date"
            value={leaveForm.endDate}
            onChange={(e) => setLeaveForm({...leaveForm, endDate: e.target.value})}
            required
          />
          <textarea
            placeholder="Reason for leave"
            value={leaveForm.reason}
            onChange={(e) => setLeaveForm({...leaveForm, reason: e.target.value})}
            required
          />
          <button type="submit">Submit Request</button>
        </form>
      </div>
      
      {/* Leave Requests History */}
      <div>
        <h2>Your Leave Requests</h2>
        {userData?.getUser?.leaveRequests?.items?.map(request => (
          <div key={request.id}>
            <p>Dates: {request.startDate} to {request.endDate}</p>
            <p>Reason: {request.reason}</p>
            <p>Status: {request.status}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default EmployeeApp;
```

## 6. Deployment Configuration

### AWS CDK Stack (Infrastructure as Code)
```typescript
// app-stack.ts
import * as cdk from '@aws-cdk/core';
import * as cognito from '@aws-cdk/aws-cognito';
import * as appsync from '@aws-cdk/aws-appsync';
import * as lambda from '@aws-cdk/aws-lambda';
import * as dynamodb from '@aws-cdk/aws-dynamodb';

export class EmployeeAppStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Cognito User Pool
    const userPool = new cognito.UserPool(this, 'EmployeeUserPool', {
      selfSignUpEnabled: true,
      signInAliases: { email: true },
      passwordPolicy: {
        minLength: 8,
        requireUppercase: true,
        requireLowercase: true,
        requireDigits: true,
      },
    });

    // DynamoDB Tables
    const userTable = new dynamodb.Table(this, 'UserTable', {
      partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
    });

    const leaveRequestTable = new dynamodb.Table(this, 'LeaveRequestTable', {
      partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
    });

    // Lambda Functions
    const photoUploadLambda = new lambda.Function(this, 'PhotoUploadLambda', {
      runtime: lambda.Runtime.PYTHON_3_9,
      handler: 'photo_upload_lambda.lambda_handler',
      code: lambda.Code.fromAsset('lambda'),
    });

    // AppSync API
    const api = new appsync.GraphqlApi(this, 'EmployeeApi', {
      name: 'employee-api',
      schema: appsync.Schema.fromAsset('schema.graphql'),
      authorizationConfig: {
        defaultAuthorization: {
          authorizationType: appsync.AuthorizationType.USER_POOL,
          userPoolConfig: { userPool },
        },
      },
    });

    // Connect Lambda to AppSync
    const photoUploadDataSource = api.addLambdaDataSource('PhotoUploadDataSource', photoUploadLambda);
    photoUploadDataSource.createResolver({
      typeName: 'Mutation',
      fieldName: 'uploadPhoto',
    });
  }
}
```
