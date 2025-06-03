# Optimizing ODCR usage through AI-powered capacity insights

Efficient resource management is crucial for organizations seeking to optimize cloud costs while making sure of seamless access to compute capacity. [Amazon Elastic Compute Cloud](https://aws.amazon.com/ec2/) (Amazon EC2) [On-Demand Capacity Reservations (ODCRs)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html) provide the flexibility to reserve compute capacity within a specific Availability Zone (AZ) for any duration. This makes sure that critical workloads always have the necessary resources available, minimizing the risk of capacity shortages.

However, managing ODCRs across multiple teams and accounts presents several challenges:

- Limited visibility across teams: In large organizations, tracking ODCR usage across teams and business units can be difficult, often leading to underused or duplicate reservations.
- Complexity in optimization: Without clear insights, adjusting, releasing, or modifying reservations to align with changing workloads becomes a cumbersome process.
- Cost management challenges: Unused or oversized ODCRs contribute to unnecessary expenses, making it essential to continuously monitor and optimize usage.

In this post, we demonstrate how [Amazon Bedrock Agents](https://aws.amazon.com/bedrock/agents/) can help organizations gain actionable insights into ODCR usage across their Amazon Web Services (AWS) environment. Unlike traditional approaches that rely on predefined patterns or manual tracking, this solution dynamically retrieves the latest ODCR usage data and provides intelligent, query-based recommendations to fulfill the capacity needs. This solution is serverless and pay-per-use and incurs minimal operational cost. 

This approach allows IT leaders, cloud architects, and finance teams to optimize reservations, control costs, and enhance overall resource management—without the complexity of traditional analysis methods.

## Solution overview

This solution addresses two specific use cases:

1. The team must create a new ODCR to accommodate an upcoming project.
2. The team needs to expand the capacity of their existing ODCR.

The system consists of three essential components:
- **ODCRSupervisorAgent:** Functions as the main coordinator, handling user inquiries and managing requests to specialized subordinate agents.
- **CapacityPlanningAgent:** Reviews existing ODCRs throughout the organization, recommending SPLIT and MOVE operations to optimize capacity allocation for new projects.
- **AugmentationAgent:** Identifies opportunities to increase existing ODCR capacity by recommending MOVE operations from other organizational ODCRs.

The following figure shows the high-level architecture for this solution.

<img width="2276" alt="1-SolutionsOverview-ODCR" src="https://github.com/user-attachments/assets/c720a783-7e50-4ea6-b602-7d76b2660823" />

The system architecture uses AWS services for comprehensive ODCR management, as shown in the following figure. The platform combines [Amazon Cognito](https://aws.amazon.com/cognito/) for authentication, [AWS Amplify](https://aws.amazon.com/amplify/) for front-end delivery, and [AWS Lambda](https://aws.amazon.com/lambda/) for serverless computing. [AWS Resource Explorer](https://aws.amazon.com/resource-explorer/) enables efficient discovery and tracking of ODCRs across AWS Regions, while Lambda functions periodically query Resource Explorer to retrieve detailed ODCR information and usage metrics. [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) plays a crucial role in securing the system, managing fine-grained access controls and permissions across all components. Amazon Bedrock Agents and its multi-agent capabilities generate intelligent recommendations through coordinated agent interactions, allowing organizations to optimize ODCR usage and implement data-driven capacity planning strategies. 

## How the solution functions

At the heart of this system is an Amazon Bedrock Agent powered by [Action Groups](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-create.html), which map user queries to backend tasks using Lambda. These Lambda functions act as the system's intelligence layer: fetching, filtering, and formatting ODCR data for the agent to reason over. The following three Lambda functions are deployed as part of the [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template:

**get_az_mapping_info Lamba function:** This Lambda function takes an AWS account ID and Region as input. It returns a mapping of AZ names to their corresponding AZ IDs for that account and AWS Region. This helps the Capacity Planning Agent align AZs across accounts.

**find_eligiable_odcrs Lambda function:** This Lambda function supports the Capacity Planning Agent by identifying suitable ODCRs that meet the specified instance type and AZ. It uses Resource Explorer to search for ODCRs across the organization. Then, the function assumes cross-account roles to access ODCRs in different AWS accounts to gather detailed information. After retrieving the necessary data, it filters the ODCRs based on instance type, tenancy, and AZ. Finally, it returns a formatted list of eligible ODCRs, which are sourced from either the specified account or an alternative account with adequate capacity.

**find_eligible_odcrs_for_move Lambda function:** This Lambda function assists the Augmentation Agent by finding all capacity reservations within the same account as the specified ODCR. It uses Resource Explorer to discover ODCRs and, when necessary, assumes cross-account roles to gather ODCR information. The function filters ODCRs based on matching instance type, tenancy, and AZ, then returns a formatted list of eligible ODCRs that can provide more capacity through MOVE operations.

Each Lambda function receives structured inputs from the Amazon Bedrock Agent, executes its designated task, and returns structured responses. Then, these responses are used by the agent to generate intelligent, actionable answers to user queries. 

Together, these Lambda functions extend the capabilities of the Amazon Bedrock Agent by integrating with external data sources and automating complex ODCR management logic—allowing the system to deliver accurate, dynamic recommendations.

## Prerequisites 

You must have the following in place to implement the solution in this post: 

- An [AWS account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup) or multiple accounts with [AWS Organizations](https://aws.amazon.com/organizations/) configured.
  - Enable [Trusted Access](https://docs.aws.amazon.com/resource-explorer/latest/userguide/security-trusted-access.html) for Resource Explorer in Organizations.
- Foundational Model (FM) [access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) in Amazon Bedrock for Anthropic's Claude 3.5 Haiku v1 in the same Region where you plan to deploy this solution.
- Turn on Resource Explorer.
  - [Setting up and configuring Resource Explorer](https://docs.aws.amazon.com/resource-explorer/latest/userguide/getting-started-setting-up.html)
  - We recommend assigning a [delegated administrator account](https://docs.aws.amazon.com/resource-explorer/latest/userguide/manage-service-multi-account.html#getting-started-prerequisites)—the account where you plan to deploy the ODCR solution—and creating the view in that account. Note this account number because it is needed later while deploying cross account role.
  - Create a [Resource Explorer view](https://docs.aws.amazon.com/resource-explorer/latest/userguide/configure-views-create.html) that filters for `resourcetype:ec2:capacity-reservation` to enable searching for ODCRs in the delegated account.
  - After setting up Resource Explorer and creating a view, note the [Amazon Resource Name (ARN)](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html). You need this ARN when deploying the ODCR CloudFormation template.
- Active ODCRs configured and maintained across multiple accounts within your Organization.
- The accompanying CloudFormation template downloaded from the [aws-samples GitHub repo](https://github.com/aws-samples/sample-odcr-ai-intelligence).
  - [Cross Account IAM Role](https://github.com/aws-samples/sample-odcr-ai-intelligence/blob/main/ODCRCrossAccountAccessRole.yaml)
  -	[ODCR Solution](https://github.com/aws-samples/sample-odcr-ai-intelligence/blob/main/ODCR-MultiAgent-with-Cognito.yaml)

## Deploy solution resources using CloudFormation

This solution is designed to be deployed in a single Region. If deploying outside us-east-1, then you must modify the CloudFormation parameters to reference the correct FM version and adjust for [Regional availability](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html), such as any necessary [cross-Region inference profile](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html) configurations.

- **Deploy the Cross Account IAM Role template:** To access ODCR data across your Organization, deploy a [CloudFormation StackSet](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html) from your payer account. During deployment, you must provide the account number where you plan to deploy the ODCR solution. This is the same account number that you used as delegate account for Resource Explorer under the prerequisites. This account number is added to the trust policy of the IAM role, allowing the solution in that account to assume the role created by the StackSet. The deployment automatically provisions a standardized IAM role with the necessary permissions (for example `ec2:DescribeCapacityReservations` and `ec2:DescribeAvailabilityZones`) in each target account.

- **Deploy the ODCR Solution template:** Deploy the provided template in the delegated administrator account that you chose for Resource Explorer. This sets up the core components of the solution and integrates with your payer account for organization-wide ODCR insights.


## Resources deployed by the CloudFormation template

After you have deployed the CloudFormation template, the following AWS resources are created in your account. 

### Cross Account IAM Role template

- IAM resource
  - `CrossAccountODCRAccessRole`

### ODCR Solution template

- Amazon Cognito resources: 
  - User Pool: `ODCRAgentUserPool`
  - App Client: `ODCRAgentApp`
  - Identity Pool: `odcr-identity-pool`
  - Groups: `ODCRAdmins`
  - User: `ODCR User`

- IAM resources:
  - IAM roles: 
    - `GetAZMappingFunctionRole`
    - `LambdaODCRAccessRole`
    - `BedrockAgentExecutionRole`
    - `ODCRSupervisorAgentExecutionRole`
    - `CognitoAuthRole`

- Lambda functions:
  - `get_az_mapping_info`
  - `find_eligible_odcrs`
  - `find_eligible_odcrs_for_move`

- Amazon Bedrock Agents
  - `ODCRSupervisorAgent`
  - `CapacityPlanningAgent` with action groups: 
    - `get_az_mapping_info`
    - `find_eligible_odcrs`

- `AugmentationAgent` with action groups 
  - `find_eligible_odcrs_for_move`

After you deploy the final CloudFormation template, copy the following from the **Outputs** tab on the CloudFormation console to use during the configuration of your application after it’s deployed in Amplify:

- `AWSRegion`
- `BedrockAgentAliasId`
- `BedrockAgentId`
- `BedrockAgentName`
- `IdentityPoolId`
- `UserPoolClientId`
- `UserPoolId`

The following screenshot shows you what the **Outputs** tab looks like.  

<img width="1144" alt="2-CFN-Output-ODCR" src="https://github.com/user-attachments/assets/a213691e-2f70-4e15-8235-f706833881aa" />


## Deploy the Amplify application for front-end 

You need to manually deploy the Amplify application using the front-end code found on GitHub. Complete the following steps, as shown in the following figure:

1. Download the front-end code AWS-Amplify-Frontend.zip from [GitHub](https://github.com/aws-samples/sample-odcr-ai-intelligence/blob/main/AWS-Amplify-Frontend.zip).
2. Use the .zip file to manually [deploy](https://docs.aws.amazon.com/amplify/latest/userguide/manual-deploys.html) the application in Amplify.
3. Return to the Amplify page and use the domain that it automatically generated to access the application.

<img width="1505" alt="4-AmplifyDeployed" src="https://github.com/user-attachments/assets/f3d45364-1165-4c8b-ac36-ebe88cbea975" />

After completing these steps, your environment is ready to analyze ODCR usage and receive recommendations powered by Amazon Bedrock Agents.

## Solution walkthrough 

After deploying the application in Amplify, return to the Amplify console. Use the **auto-generated domain URL** shown there to access the front-end. Upon accessing the application URL, you’re prompted to provide information related to Amazon Cognito and Amazon Bedrock Agents. This information is necessary to securely authenticate users and allow the front end to interact with the Amazon Bedrock Agent. It enables the application to manage user sessions and make authorized API calls to AWS services on behalf of the user.

You can enter information with the values that you collected from the CloudFormation stack outputs, as shown in the following screenshot:

![3-AmplifyODCRConfiguration](https://github.com/user-attachments/assets/9cecbea0-dcca-4832-acb5-89f8bebad14b)

A temporary password was automatically generated during deployment and sent to the email address provided when launching the CloudFormation template. At first sign-in, you’re prompted to reset your password.When you’re signed in, you can begin interacting with the application to analyze and manage your ODCR usage. The interface supports natural language queries to streamline capacity planning. The following are some example questions that you can ask:

- I require [X units] of capacity for the [INSTANCE TYPE] instance type in Availability Zone [AZ ID], within account [ACCOUNT NUMBER]. Could you please advise if there are any existing On-Demand Capacity Reservations (ODCRs) that can fulfill this requirement? Additionally, what operations would be necessary to utilize the available capacity?

![image-5](https://github.com/user-attachments/assets/5949bf5d-051b-4a7a-a585-20e5b7401f63)

- I have a requirement to add [X additional units] of capacity to an existing On-Demand Capacity Reservation with ID [cr-0012abcd123456789]. Could you please advise on the steps or operations required to fulfill this request?

![image-6](https://github.com/user-attachments/assets/dd0a521d-12ae-4edd-b09c-858384e035cf)


## Cleaning up

If you decide to discontinue using this solution, then you can follow these steps to remove it, its associated resources deployed using CloudFormation, and the Amplify deployment:

1. Delete the CloudFormation StackSet:
  - On the CloudFormation console, choose **StackSets** in the navigation pane.
  - Locate your StackSet for the Cross-Account IAM Role (you assigned a name to it).
  - Choose **Delete stacks from StackSet** to remove all stack instances.
  - After all instances are deleted, choose **StackSet**.
  - Choose **Delete StackSet** from the **Actions** menu.

2. Delete the CloudFormation stack:
  - On the CloudFormation console, choose **Stacks** in the navigation pane. 
  - Locate the stack you created during the deployment process (you assigned a name to it).
  - Choose the stack and choose **Delete**.

3. Delete the Amplify application and its resources. For instructions, refer to [Clean Up Resources](https://docs.aws.amazon.com/amplify/latest/userguide/cleanup.html).

## Conclusion 

In this post, we demonstrated how to use Amazon Bedrock Agents and AWS services to build an intelligent ODCR management solution. Combining AWS Resource Explorer for organization-wide visibility, Amazon Bedrock Agents for intelligent recommendations, and a secure front-end powered by AWS Amplify, organizations can now make data-driven decisions about their capacity reservations. This solution helps teams optimize ODCR usage, reduce costs, and efficiently manage compute capacity across their AWS Organization. The artificial intelligence (AI)-powered approach eliminates manual tracking and analysis, allowing teams to focus on strategic capacity planning rather than operational overhead.

Get started today by deploying this solution in your AWS environment, and unlock the power of AI-driven capacity insights for more efficient ODCR management.
