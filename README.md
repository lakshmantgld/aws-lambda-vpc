# aws-lambda-vpc
**AWS Lambda** is **revolutionizing** the cloud. Developers are creating interesting use cases day-by-day with the Lambda. In our use case, we are using Lambda as the **Web backend**. Lambdas and APIGateway together form a scalable, highly available and cost-efficient backend for your web application.

When we use Lambda as our backend, we need our **Lambda** to talk to the **databases**. Usually databases reside inside an VPC, so that they are secured and not exposed to public world. Initially, if you want your Lambda to talk to an RDS(Relational Database Service), then you need to expose the RDS with the publicIP which helps Lambda to perform CRUD operations with the database. This **exposing of database** to the internet will be **vulnerable.** So, the obvious solution will be adding the **Lambda inside an VPC.** Lets see how we can add the Lambda within the VPC.

## Technical Architecture:

This high level architecture would help us understand how our VPC and other resources inside the VPC interact with each other.

![VPC Lambda Architecture](https://raw.githubusercontent.com/lakshmantgld/aws-lambda-vpc/master/readmeFiles/architecture.png)

### Overview of Lambda inside an VPC:
- In our case, we want our **Lambda** to talk to the **mongoDB** instance which is in the **private subnet.**
- The drawback of adding the lambda to the VPC, it **cannot make calls to the internet*.** Even though we attach our **Lambda** to the **public subnet** that in turn is connected to an IGW(Internet Gateway), the Lambda still cannot make calls to the internet. This is because, **Lambda** is given an **private ENI**(Elastic Network Interface), when attached to an VPC.
- The **solution** is to introduce a **NAT**(Network Address Translation) Gateway in the **public subnet**, which enable Lambda and instances in the private subnet to connect to the Internet or other AWS services, but prevent the Internet from initiating a connection with those resources.
- So, adding the **NAT gateway to the public subnet** makes the **Lambda** accessible to both the VPC resources and also **make calls to the internet**.

### Setting up Lambda with a VPC:
- When we want to put Lambda inside a VPC, we need to provide the subnets and the security groups where the Lambda will reside. Think Lambda as an EC2 instance, as it will behave more like an instance when added inside the VPC. Make sure you open the appropriate ports in Lambda's security group, so that it can communicate with the database in the VPC.
- I use [Serverless framework](https://github.com/serverless/serverless/) to automate the Lambda deployment, so here is how it looks, when we add the **VPC configuration to the Lambda**.

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

- You should also add appropriate **permissions** to create, delete and describe (ENI)Network Interfaces to the function. The above configuration will make the Lambda to reside inside an VPC and communicate with the database inside the VPC.
- The **source code inside Lambda** should have the **privateIP of your database**. The privateIP that is allocated by your VPC and subnet. Now, lambda can perform CRUD operations with your database inside the VPC, yet cannot make calls to the internet.

### Setting up NAT gateway in the public subnet:

To create a NAT Gateway, you must specify a subnet and an Elastic IP address. Here are the steps:

1. Navigate to **AWS Dashboard** -> **VPC.**
2. In the navigation pane, choose **NAT Gateways** and then select **Create a NAT Gateway.**
3. In the dialog box, specify the **subnet** in which to create NAT Gateway and select an Elastic IP address, which you can associate with the NAT Gateway.
4. Now, In sometime, your **NAT Gateway** will be **available**.
5. Once you have created your NAT Gateway, you must **update** your **route tables** for your private subnets to point Internet traffic to the NAT Gateway.
6. In the navigation pane of VPC, choose **Route tables**. Select the Route table associated with the private subnet and click Edit.
7. Choose Add another route. For Destination, enter **0.0.0.0/0.** For **Target**, select the **ID of your NAT Gateway.**
9. Now your **private subnet** uses **NAT gateway** for the **internet traffic.**
10. Make sure that your **public subnet** in which NAT resides, is actually connected to **IGW**(Internet Gateway).

Doing so, now our **Lambda** can talk to **both the VPC resources and also make calls to the Internet.**

### Advantages of VPC-enabled Lambda:
1. The **database** (in our case it is mongoDB) is **secured** and only resources inside the VPC will be able to access the database.
2. When we use publicIP for our mongoDB and when Lambda calls mongo using publicIP, there will be a small time penalty for DNS translation. So adding Lambda inside the VPC will also **reduce this time penalty.**

### Best practices while setting up Lambda with VPC:
1. Ensure that your private subnet(Lambda's subnet) has **enough IP addresses.** Whenever a Lambda is created, it creates an ENI(Elastic Network Interface) or a privateIP inside the subnet. So, when Lambda scales, make sure that your subnet has enough IP addresses to scale along with the lambda.
2. Specify at least one subnet in each availability zone, so that your Lambdas are highly available. For example, say you are setting up the **VPC** in **Tokyo(ap-northeast-1)** region which has three availability zone(AZs). Create a private subnet for each AZ, so that even when an AZ goes down, your Lambda will be created in a different AZ.
