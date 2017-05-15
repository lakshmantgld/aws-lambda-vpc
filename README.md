# aws-lambda-vpc
AWS Lambda is revolutionizing the cloud. Developers are creating interesting use cases day-by-day with the Lambda. In our use case, we are using Lambda as the Web backend. Lambdas and APIGateway together form a scalable, highly available and cost-efficient backend for your web application.

When we use Lambda as our backend, we need our Lambda to talk to the databases. Usually databases reside inside an VPC, so that they are secured and not exposed to public world. Initially, if you want your Lambda to talk to an RDS(Relational Database Service), then you need to expose the RDS with the publicIP which helps Lambda to perform CRUD operations with the database. This exposing of database to the internet will be vulnerable. So, the obvious solution will be adding the Lambda inside an VPC. Lets see how we can add the Lambda within the VPC.

## Technical Architecture:

This high level architecture would help us understand how our VPC and other resources inside the VPC interact with each other.

![VPC Lambda Architecture](https://raw.githubusercontent.com/lakshmantgld/aws-lambda-vpc/stable/readmeFiles/architecture.png)

### Overview of Lambda inside an VPC:
- In our case, we want our lambda to talk to the mongoDB instance which is in the private subnet.
- The drawback of adding the lambda to the VPC, it can perform calls to the internet. Even though we attach our Lambda to the public subnet that in turn is connect to an IGW(Internet Gateway), the lambda still cannot make calls to the internet. This is because, lambda is given an private ENI(Elastic Network Interface), when attached to an VPC.
- The solution is to introduce a NAT(Network Address Translation) gateway in the public subnet, which enable Lambda and instances in the private subnet to connect to the Internet or other AWS services, but prevent the Internet from initiating a connection with those resources.
- So, adding the NAT gateway to the public subnet makes the Lambda accessible to both the VPC resources and also make calls to the internet.

### Setting up Lambda with a VPC:
- When we want to put lambda inside a VPC, we need to provide the subnets and the security groups where the lambda will reside. Think Lambda as an EC2 instance, as lambda will behave more like an instance when added inside the VPC. Make sure you open the appropriate ports in lambda's security group, so that lambda can communicate with the database in the VPC.
- I use serverless to automate the Lambda deployment, so here is how it looks, when we add the VPC configuration to the lambda.

```
functions:
  graphql:
    handler: handler.graphql
    iamRoleStatements:
      - Effect: Allow
        Resource: "*"
        Action:
          - ec2:CreateNetworkInterface
          - ec2:DescribeNetworkInterfaces
          - ec2:DetachNetworkInterface
          - ec2:DeleteNetworkInterface
          - logs:CreateLogGroup
          - logs:CreateLogStream
          - logs:PutLogEvents
    vpc:
      securityGroupIds:
        - sg-*******
      subnetIds:
        - subnet-*******
        - subnet-*******
    events:
      - http:
          path: graphql
          method: post
          integration: lambda
          memorySize: 256
          timeout: 10
          cors: true
          response:
            headers:
              Access-Control-Allow-Origin: "'*'"

```

- You should also add permissions to create, delete and describe (ENI)Network Interfaces to the function. The above configuration will make the Lambda to reside inside an VPC and communicate with the database inside the VPC.
- The source code inside lambda should have the privateIP of your database. The privateIP that is allocated by your VPC and subnet. Now, lambda can perform CRUD operations with your database inside the VPC, yet cannot make calls to the internet.

### Setting up NAT gateway in the public subnet:
