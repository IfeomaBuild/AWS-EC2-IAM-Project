&#x20;**AWS IAM Least-Privilege EC2 Access Control**



&#x20;**Overview**



This project demonstrates the design and implementation of an AWS IAM least-privilege access model for Amazon EC2.



The objective was to create an IAM group whose members can launch EC2 instances without being granted broad EC2 administrative permissions such as stopping or terminating instances.



Instead of using the AWS managed `AmazonEC2FullAccess` policy, I created a custom IAM policy specifically designed around the required access.



The configuration was then verified using the AWS CLI IAM Policy Simulator to confirm that the intended permissions were allowed and restricted actions were denied.







**Project Objective**



The goal of this project was to implement the following access model:



text

&#x20;                   IAM Group

&#x20;                       |

&#x20;                       v

&#x20;            Custom EC2 IAM Policy

&#x20;                       |

&#x20;         +-------------+-------------+

&#x20;         |             |             |

&#x20;         v             v             v

&#x20;      Describe     RunInstances   CreateTags

&#x20;       EC2 data       ALLOW       During launch

&#x20;         

&#x20;         |

&#x20;         +-----------------------------+

&#x20;                                       |

&#x20;                             +---------+---------+

&#x20;                             |                   |

&#x20;                             v                   v

&#x20;                      StopInstances       TerminateInstances

&#x20;                          DENIED               DENIED





**The primary requirement was**:



> Allow users to launch EC2 instances while following the principle of least privilege and preventing them from stopping or terminating instances.







Why Custom IAM Policy?



AWS provides managed policies such as:



&#x20;text



AmazonEC2FullAccess





However, a broad managed policy can provide significantly more EC2 permissions than a user needs for a specific responsibility.



For this project, the requirement was not to give users full EC2 administration.



The intended permission model was:



&#x20;text



Launch EC2 instances

&#x20;       +

View required EC2 information

&#x20;       +

Create tags during launch

&#x20;       -

Stop instances

&#x20;       -

Terminate instances





Therefore, a custom IAM policy was created instead of assigning `AmazonEC2FullAccess`.



This demonstrates the AWS security principle of least privilege.







AWS Services and Tools



| Service / Tool         | Purpose                                           |

| ---------------------- | ------------------------------------------------- |

| AWS IAM                | Identity and access management                    |

| Amazon EC2             | Compute service whose permissions were controlled |

| AWS Management Console | Created users, groups, and policies               |

| AWS CLI                | Audited and tested IAM permissions                |

| IAM Policy Simulator   | Verified effective permissions                    |

| JSON                   | Defined the custom IAM policy                     |

| Git / GitHub           | Version control and portfolio documentation       |







**IAM Architecture**



The project uses IAM users and groups to demonstrate centralized permission management.



&#x20; text

&#x20;                   AWS Account

&#x20;                       |

&#x20;                       v

&#x20;                   IAM Group

&#x20;                       |

&#x20;             Custom IAM Policy

&#x20;                       |

&#x20;            +----------+----------+

&#x20;            |                     |

&#x20;            v                     v

&#x20;       IAM User A             IAM User B

&#x20;            |                     |

&#x20;            +----------+----------+

&#x20;                       |

&#x20;                       v

&#x20;                EC2 Permissions





Users receive permissions through membership in the IAM group rather than requiring the same policy to be individually configured for every user.



This makes group-based permission management easier to maintain and audit.







&#x20;**Custom IAM Policy**



The custom policy was designed with four main permission areas:



1\. Read-only EC2 information

2\. Launch EC2 instances

3\. Create tags during instance launch

4\. Explicitly deny EC2 instance termination



The policy does not restrict the user to a specific:



\* AWS Region

\* Availability Zone

\* AMI

\* VPC

\* Subnet

\* Security group

\* Instance type



This was intentional.



The requirement was for the user to be able to launch instances using the available EC2 configuration rather than restricting the launch to a single environment.







&#x20;**Policy**



Json

{

&#x20; "Version": "2012-10-17",

&#x20; "Statement": \[

&#x20;   {

&#x20;     "Sid": "AllowEC2ReadOnlyInformation",

&#x20;     "Effect": "Allow",

&#x20;     "Action": \[

&#x20;       "ec2:DescribeImages",

&#x20;       "ec2:DescribeInstances",

&#x20;       "ec2:DescribeInstanceTypes",

&#x20;       "ec2:DescribeKeyPairs",

&#x20;       "ec2:DescribeSecurityGroups",

&#x20;       "ec2:DescribeSubnets",

&#x20;       "ec2:DescribeTags",

&#x20;       "ec2:DescribeVpcs",

&#x20;       "ec2:DescribeAvailabilityZones"

&#x20;     ],

&#x20;     "Resource": "\*"

&#x20;   },

&#x20;   {

&#x20;     "Sid": "AllowLaunchEC2Instances",

&#x20;     "Effect": "Allow",

&#x20;     "Action": "ec2:RunInstances",

&#x20;     "Resource": "\*"

&#x20;   },

&#x20;   {

&#x20;     "Sid": "AllowTagsOnlyDuringInstanceLaunch",

&#x20;     "Effect": "Allow",

&#x20;     "Action": "ec2:CreateTags",

&#x20;     "Resource": "\*",

&#x20;     "Condition": {

&#x20;       "StringEquals": {

&#x20;         "ec2:CreateAction": "RunInstances"

&#x20;       }

&#x20;     }

&#x20;   },

&#x20;   {

&#x20;     "Sid": "DenyEC2InstanceTermination",

&#x20;     "Effect": "Deny",

&#x20;     "Action": "ec2:TerminateInstances",

&#x20;     "Resource": "\*"

&#x20;   }

&#x20; ]

}









&#x20;**Policy Breakdown**



1\. EC2 Read Permissions



The policy allows users to retrieve information about EC2 resources.



Examples include:



&#x20;  text



ec2:DescribeImages

ec2:DescribeInstances

ec2:DescribeInstanceTypes

ec2:DescribeSecurityGroups

ec2:DescribeSubnets

ec2:DescribeVpcs

ec2:DescribeAvailabilityZones

```



These permissions allow the user to view information required when working with EC2 without granting permissions to modify those resources.





2\. Launch EC2 Instances



The key permission is:



Json



"Action": "ec2:RunInstances"



This allows the user to launch EC2 instances.



No condition was added to restrict the user to a specific Region, AMI, subnet, VPC, security group, or instance type.



This reflects the project's requirement that the user should be able to launch instances using different available EC2 configurations.





3\. Create Tags During Launch



The policy also allows:



text



ec2:CreateTags



but limits the permission with:



Json



"ec2:CreateAction": "RunInstances"





This is intended to allow tags to be created as part of launching an EC2 instance rather than providing unrestricted tagging capabilities.







4\. Explicit Termination Deny



The policy explicitly denies:



text

ec2:TerminateInstances





This creates a clear security boundary:



text

RunInstances          → ALLOWED

TerminateInstances    → DENIED





The explicit Deny also provides protection if another policy were later to grant `TerminateInstances`, because an explicit Deny takes precedence over an Allow.





&#x20;AWS CLI Administration



AWS CLI was used to inspect the IAM environment and verify the configuration.



&#x20;List IAM Users



&#x20;bash

aws iam list-users





List IAM Groups



bash

aws iam list-groups





Find Groups Associated With a User



bash

aws iam list-groups-for-user --user-name USER\_NAME





List Inline Policies Attached to a Group



bash

aws iam list-group-policies --group-name GROUP\_NAME





List Managed Policies Attached to a Group



bash



aws iam list-attached-group-policies --group-name GROUP\_NAME





List Inline Policies Attached to a User



bash



aws iam list-user-policies --user-name USER\_NAME





List Managed Policies Attached to a User



bash



aws iam list-attached-user-policies --user-name USER\_NAME





These commands were used to inspect users, groups, and the permissions associated with them.







&#x20;**IAM Policy Testing**



After implementing the policy, the effective permissions were tested using the AWS CLI IAM Policy Simulator.



The following command was used:



bash

aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::ACCOUNT\_ID:user/TEST\_USER --action-names ec2:RunInstances ec2:TerminateInstances ec2:StopInstances --query "EvaluationResults\[\*].\[EvalActionName,EvalDecision]" --output table





The account ID and test username were substituted with the appropriate values for the AWS environment.







&#x20;Test Results



The policy simulation produced the following results:



| EC2 Action               | Result    |

| ------------------------ | --------- |

| `ec2:RunInstances`       | ✅ Allowed |

| `ec2:StopInstances`      | ❌ Denied  |

| `ec2:TerminateInstances` | ❌ Denied  |



&#x20;**Result**



The test confirmed that the IAM principal was authorized to launch EC2 instances while being denied permission to stop or terminate EC2 instances.



This provided verification that the intended access-control model was being enforced.







&#x20;**Security Model**



The resulting permission model can be summarized as:



text

&#x20;                        EC2 Access

&#x20;                            |

&#x20;            +---------------+---------------+

&#x20;            |               |               |

&#x20;            v               v               v

&#x20;        Describe       RunInstances      CreateTags

&#x20;         Actions          ALLOW          Launch only

&#x20;            

&#x20;                            |

&#x20;                            |

&#x20;                   +--------+--------+

&#x20;                   |                 |

&#x20;                   v                 v

&#x20;             StopInstances    TerminateInstances

&#x20;                 DENIED             DENIED





This provides users with the ability to perform the required launch operation without granting broad EC2 management permissions.







&#x20;**Least-Privilege Analysis**



The project demonstrates several aspects of least-privilege access control.



&#x20;Permissions that are allowed



The user can:



\* View relevant EC2 information

\* Launch EC2 instances

\* Create tags during instance launch



&#x20;**Permissions that are not granted**



The user cannot:



\* Stop EC2 instances

\* Terminate EC2 instances

\* Perform arbitrary EC2 administrative actions that are not included in the policy



The policy therefore provides a more limited permission set than a broad EC2 administrative policy.







&#x20;**Why Group-Based Permissions?**



Instead of configuring the same permissions individually for every user, the custom policy was associated with an IAM group.



Users were then added to the group.



**Text**

&#x20;                   EC2 Launch Group

&#x20;                          |

&#x20;            +-------------+-------------+

&#x20;            |             |             |

&#x20;            v             v             v

&#x20;          User 1        User 2        User 3

&#x20;            |             |             |

&#x20;            +-------------+-------------+

&#x20;                          |

&#x20;                          v

&#x20;                 Custom IAM Policy





This approach makes permission management more scalable.



If the organization's EC2 launch requirements change, the group policy can be updated rather than modifying every user individually.







&#x20;**Inline User Policy Demonstration**



As part of the IAM exercise, an inline policy was also assigned directly to one user.



This was used to demonstrate the difference between:



\* Direct user-level permissions

\* Group-based permissions



The project therefore covers multiple methods of IAM permission assignment and provides practical experience with AWS IAM policy management.



\---



&#x20;**What I Learned**



Through this project, I gained practical experience with:



\* Creating AWS IAM users

\* Creating IAM groups

\* Adding users to groups

\* Assigning policies to IAM users

\* Assigning policies to IAM groups

\* Writing custom IAM policies in JSON

\* Understanding IAM Allow and Deny behaviour

\* Applying the principle of least privilege

\* Working with Amazon EC2 permissions

\* Using AWS CLI to inspect IAM configuration

\* Using IAM Policy Simulator to test permissions

\* Understanding the difference between AWS managed and custom policies

\* Documenting cloud infrastructure and security configurations







&#x20;**Challenges and Security Considerations**



One of the main challenges was determining how to provide enough permission to launch EC2 instances without simply assigning a broad EC2 administrative policy.



The solution was to identify the specific actions required for the intended task and create a custom policy around those actions.



The policy was then tested using the IAM Policy Simulator to verify the expected authorization behaviour.







&#x20;**Future Improvements**



This project could be extended with additional AWS security controls.



Potential improvements include:



\* Require MFA for IAM users.

\* Introduce IAM roles instead of relying primarily on long-lived IAM users.

\* Integrate AWS CloudTrail for API activity auditing.

\* Use IAM Access Analyzer to identify unintended access.

\* Further restrict EC2 launch permissions based on organizational requirements.

\* Restrict resources using AWS resource-level permissions where appropriate.

\* Add conditions based on required tags.

\* Require specific EC2 instance tags.

\* Implement the IAM infrastructure using Terraform.

\* Automate IAM policy validation.

\* Create separate administrator, developer, and read-only roles.

\* Add automated security testing to the project.







&#x20;**Project Structure**



**Text**

aws-iam-least-privilege/

│

├── README.md

│

├── policies/

│   └── ec2-launch-restricted.json

│

└── cli-commands/

&#x20;   └── iam-policy-testing.md







&#x20;**Conclusion**



This project demonstrates a practical AWS IAM least-privilege implementation in which users are granted the ability to launch EC2 instances without receiving broad EC2 administrative access.



A custom IAM policy was created and assigned through an IAM group. AWS CLI was then used to inspect the IAM configuration and the IAM Policy Simulator was used to verify the effective permissions.



The final test demonstrated:



text



ec2:RunInstances        → ALLOWED

ec2:StopInstances       → DENIED

ec2:TerminateInstances  → DENIED





This project demonstrates practical knowledge of AWS IAM, EC2 authorization, custom IAM policies, group-based access management, least privilege, AWS CLI, and permission testing.

