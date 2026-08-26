AWS IAM Least-Privilege EC2 Access Control

Overview

This project demonstrates the design and implementation of an AWS IAM least-privilege access model for Amazon EC2.

The objective was to create an IAM group whose members can launch EC2 instances without being granted broad EC2 administrative permissions such as stopping or terminating instances.

Instead of using the AWS managed `AmazonEC2FullAccess` policy, I created a custom IAM policy specifically designed around the required access.

The configuration was then verified using the AWS CLI IAM Policy Simulator to confirm that the intended permissions were allowed and restricted actions were denied.

Project Objective

The goal of this project was to implement the following access model:

```text

                   IAM Group

                       |

                       v

            Custom EC2 IAM Policy

                       |

         +-------------+-------------+

         |             |             |

         v             v             v

      Describe     RunInstances   CreateTags

       EC2 data       ALLOW       During launch

|

         +-----------------------------+

                                       |

                             +---------+---------+

                             |                   |

                             v                   v

                      StopInstances       TerminateInstances

                          DENIED               DENIED
```
The primary requirement was:

> Allow users to launch EC2 instances while following the principle of least privilege and preventing them from stopping or terminating instances.

Why Custom IAM Policy?

AWS provides managed policies such as:

```text

AmazonEC2FullAccess
```

However, a broad managed policy can provide significantly more EC2 permissions than a user needs for a specific responsibility.

For this project, the requirement was not to give users full EC2 administration.

The intended permission model was:

- Launch EC2 instances: Allowed
- View required EC2 information: Allowed
- Create tags during launch: Allowed
- Stop instances: Denied
- Terminate instances: Denied

Therefore, a custom IAM policy was created instead of assigning `AmazonEC2FullAccess`.

This demonstrates the AWS security principle of least privilege.

## AWS Services and Tools

| Service / Tool | Purpose |
|---|---|
| AWS IAM | Identity and access management |
| Amazon EC2 | Compute service whose permissions were controlled |
| AWS Management Console | Created users, groups, and policies |
| AWS CLI | Audited and tested IAM permissions |
| IAM Policy Simulator | Verified effective permissions |
| JSON | Defined the custom IAM policy |
| Git / GitHub | Version control and portfolio documentation |

## IAM Architecture

The project uses IAM users and groups to demonstrate centralized permission management.

```text

                   AWS Account

                       |

                       v

                   IAM Group

                       |

             Custom IAM Policy

                       |

            +----------+----------+

            |                     |

            v                     v

       IAM User A             IAM User B

            |                     |

            +----------+----------+

                       |

                       v

                EC2 Permissions
```                

Users receive permissions through membership in the IAM group rather than requiring the same policy to be individually configured for every user.

This makes group-based permission management easier to maintain and audit.

Custom IAM Policy

The custom policy was designed with four main permission areas:

1. Read-only EC2 information

2. Launch EC2 instances

3. Create tags during instance launch

4. Explicitly deny EC2 instance termination

The policy does not restrict the user to a specific:

- AWS Region

- Availability Zone

- AMI

- VPC

- Subnet

- Security group

- Instance type

This was intentional.

The requirement was for the user to be able to launch instances using the available EC2 configuration rather than restricting the launch to a single environment.

## Policy

```json

{

 "Version": "2012-10-17",

 "Statement": \[

   {

     "Sid": "AllowEC2ReadOnlyInformation",

     "Effect": "Allow",

     "Action": \[

       "ec2:DescribeImages",

       "ec2:DescribeInstances",

       "ec2:DescribeInstanceTypes",

       "ec2:DescribeKeyPairs",

       "ec2:DescribeSecurityGroups",

       "ec2:DescribeSubnets",

       "ec2:DescribeTags",

       "ec2:DescribeVpcs",

       "ec2:DescribeAvailabilityZones"

     ],

     "Resource": "\*"

   },

   {

     "Sid": "AllowLaunchEC2Instances",

     "Effect": "Allow",

     "Action": "ec2:RunInstances",

     "Resource": "\*"

   },

   {

     "Sid": "AllowTagsOnlyDuringInstanceLaunch",

     "Effect": "Allow",

     "Action": "ec2:CreateTags",

     "Resource": "\*",

     "Condition": {

       "StringEquals": {

         "ec2:CreateAction": "RunInstances"

       }

     }

   },

   {

     "Sid": "DenyEC2InstanceTermination",

     "Effect": "Deny",

     "Action": "ec2:TerminateInstances",

     "Resource": "\*"

   }

 ]

}
```

## Policy Breakdown

1. EC2 Read Permissions

The policy allows users to retrieve information about EC2 resources.

Examples include:

```text

ec2:DescribeImages

ec2:DescribeInstances

ec2:DescribeInstanceTypes

ec2:DescribeSecurityGroups

ec2:DescribeSubnets

ec2:DescribeVpcs

ec2:DescribeAvailabilityZones
```

These permissions allow the user to view information required when working with EC2 without granting permissions to modify those resources.

### 2. Launch EC2 Instances

The key permission is:

```json

"Action": "ec2:RunInstances"
```

This allows the user to launch EC2 instances.

No condition was added to restrict the user to a specific Region, AMI, subnet, VPC, security group, or instance type.

This reflects the project's requirement that the user should be able to launch instances using different available EC2 configurations.

### 3. Create Tags During Launch

The policy also allows:

```text

ec2:CreateTags
```

but limits the permission with:

```json

"ec2:CreateAction": "RunInstances"
```

This is intended to allow tags to be created as part of launching an EC2 instance rather than providing unrestricted tagging capabilities.

### 4. Explicit Termination Deny

The policy explicitly denies:

```text

ec2:TerminateInstances
```

This creates a clear security boundary:

```text

RunInstances          → ALLOWED

TerminateInstances    → DENIED
```

The explicit Deny also provides protection if another policy were later to grant `TerminateInstances`, because an explicit Deny takes precedence over an Allow.

## AWS CLI Administration

AWS CLI was used to inspect the IAM environment and verify the configuration.

### List IAM Users

```bash

aws iam list-users
```

### List IAM Groups

```bash

aws iam list-groups
```

### Find Groups Associated With a User

```bash

aws iam list-groups-for-user --user-name USER_NAME
```

### List Inline Policies Attached to a Group

```bash

aws iam list-group-policies --group-name GROUP_NAME
```

### List Managed Policies Attached to a Group

```bash

aws iam list-attached-group-policies --group-name GROUP_NAME
```
### List Inline Policies Attached to a User

```bash

aws iam list-user-policies --user-name USER_NAME
```

### List Managed Policies Attached to a User

```bash

aws iam list-attached-user-policies --user-name USER_NAME
```

These commands were used to inspect users, groups, and the permissions associated with them.

## IAM Policy Testing

After implementing the policy, the effective permissions were tested using the AWS CLI IAM Policy Simulator.

The following command was used:

```bash

aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::ACCOUNT\_ID:user/TEST\_USER --action-names ec2:RunInstances ec2:TerminateInstances ec2:StopInstances --query "EvaluationResults\[\*].\[EvalActionName,EvalDecision]" --output table
```

The account ID and test username were substituted with the appropriate values for the AWS environment.

## Test Results

The policy simulation produced the following results:

| EC2 Action               | Result    |

| ------------------------ | --------- |

| `ec2:RunInstances`       | ✅ Allowed |

| `ec2:StopInstances`      | ❌ Denied  |

| `ec2:TerminateInstances` | ❌ Denied  |

### Result

The test confirmed that the IAM principal was authorized to launch EC2 instances while being denied permission to stop or terminate EC2 instances.

This provided verification that the intended access-control model was being enforced.

## Security Model

The resulting permission model can be summarized as:

```text

                        EC2 Access

                            |

            +---------------+---------------+

            |               |               |

            v               v               v

        Describe       RunInstances      CreateTags

         Actions          ALLOW          Launch only

|

                            |

                   +--------+--------+

                   |                 |

                   v                 v

             StopInstances    TerminateInstances

                 DENIED             DENIED
```                 

This provides users with the ability to perform the required launch operation without granting broad EC2 management permissions.

## Least-Privilege Analysis

The project demonstrates several aspects of least-privilege access control.

### Permissions that are allowed

The user can:

- View relevant EC2 information

- Launch EC2 instances

- Create tags during instance launch

### Permissions that are not granted

The user cannot:

- Stop EC2 instances

- Terminate EC2 instances

- Perform arbitrary EC2 administrative actions that are not included in the policy

The policy therefore provides a more limited permission set than a broad EC2 administrative policy.

Why Group-Based Permissions?

Instead of configuring the same permissions individually for every user, the custom policy was associated with an IAM group.

Users were then added to the group.

```text

                   EC2 Launch Group

                          |

            +-------------+-------------+

            |             |             |

            v             v             v

          User 1        User 2        User 3

            |             |             |

            +-------------+-------------+

                          |

                          v

                 Custom IAM Policy
```                 

This approach makes permission management more scalable.

If the organization's EC2 launch requirements change, the group policy can be updated rather than modifying every user individually.

### Inline User Policy Demonstration

As part of the IAM exercise, an inline policy was also assigned directly to one user.

This was used to demonstrate the difference between:

- Direct user-level permissions

- Group-based permissions

The project therefore covers multiple methods of IAM permission assignment and provides practical experience with AWS IAM policy management.


## What I Learned

Through this project, I gained practical experience with:

- Creating AWS IAM users

- Creating IAM groups

- Adding users to groups

- Assigning policies to IAM users

- Assigning policies to IAM groups

- Writing custom IAM policies in JSON

- Understanding IAM Allow and Deny behaviour

- Applying the principle of least privilege

- Working with Amazon EC2 permissions

- Using AWS CLI to inspect IAM configuration

- Using IAM Policy Simulator to test permissions

- Understanding the difference between AWS managed and custom policies

- Documenting cloud infrastructure and security configurations

## Challenges and Security Considerations

One of the main challenges was determining how to provide enough permission to launch EC2 instances without simply assigning a broad EC2 administrative policy.

The solution was to identify the specific actions required for the intended task and create a custom policy around those actions.

The policy was then tested using the IAM Policy Simulator to verify the expected authorization behaviour.

Future Improvements

This project could be extended with additional AWS security controls.

Potential improvements include:

- Require MFA for IAM users.

- Introduce IAM roles instead of relying primarily on long-lived IAM users.

- Integrate AWS CloudTrail for API activity auditing.

- Use IAM Access Analyzer to identify unintended access.

- Further restrict EC2 launch permissions based on organizational requirements.

- Restrict resources using AWS resource-level permissions where appropriate.

- Add conditions based on required tags.

- Require specific EC2 instance tags.

- Implement the IAM infrastructure using Terraform.

- Automate IAM policy validation.

- Create separate administrator, developer, and read-only roles.

- Add automated security testing to the project.

## Project Structure

```text

aws-iam-least-privilege/

│

├── README.md

│

├── policies/

│   └── ec2-launch-restricted.json

│

└── cli-commands/

   └── iam-policy-testing.md
```

## Conclusion

This project demonstrates a practical AWS IAM least-privilege implementation in which users are granted the ability to launch EC2 instances without receiving broad EC2 administrative access.

A custom IAM policy was created and assigned through an IAM group. AWS CLI was then used to inspect the IAM configuration and the IAM Policy Simulator was used to verify the effective permissions.

The final test demonstrated:

```text

ec2:RunInstances        → ALLOWED

ec2:StopInstances       → DENIED

ec2:TerminateInstances  → DENIED
```

This project demonstrates practical knowledge of AWS IAM, EC2 authorization, custom IAM policies, group-based access management, least privilege, AWS CLI, and permission testing.

