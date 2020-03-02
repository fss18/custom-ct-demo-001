
---
# Custom Guardrail and Service Catalog Demo
## Overview ##

AWS Control Tower provides several built in [Guardrails](https://docs.aws.amazon.com/controltower/latest/userguide/guardrails-reference.html) that provides ongoing governance for your overall AWS environments. AWS Control Tower implements *preventive* and *detective* guardrails to help you govern your resources and monitor compliance across groups of AWS accounts.

AWS Service Catalog allows organization to create and manage catalog of IT services. AWS Service Catalog allows you to centrally manage commonly deployed IT services and helps you achieve consistent governance and meet your compliance requirements.

Customers and partners often ask how to add custom guardrail in Control Tower and how to achieve consistent governance going forward.

In this lab, you will:
- Enable optional guardrail built-in from AWS Control Tower.
- Deploy AWS Config rules using CloudFormation StackSet to apply custom *detective* guardrail.
- Deploy Service Control Policies in AWS Organizations to enforce custom *preventive* guardrail.
- Deploy custom AWS Service Catalog to suit your organization needs and compliance.

## Prerequisites
- This lab requires an account with Administrator privileges and Control Tower.
- Use `CloudFormation New Console`. All instructions of this lab are based off new console.
- If you are still on old console, expand **CloudFormation** on top left and choose **New Console**
- Stay in one of the supported regions, `us-west-2`, `us-east-2`, `us-east-1`, `eu-west-1` for this lab.
- Choose one **Organization Unit** as target for custom guardrail deployment.
-- Click [**HERE**](https://console.aws.amazon.com/controltower/home/organizationunits) to view detail of your AWS Organizational units.
-- Click on the Organizational units that you choose as target for deployment.
-- Take note of the ID, it will be in format such as `ou-xxxxxxxxxx`

## Lab A - Enable Optional Guardrail ##
In this section of the lab, you will select and enable three optional guardrails.

| Guardrail Name | Outcomes |
|---|---|
| [Enable encryption for EBS volumes attached to EC2 instances](https://docs.aws.amazon.com/controltower/latest/userguide/strongly-recommended-guardrails.html#ebs-enable-encryption) | Detects whether EBS volume created without encryption |
| [Disallow Public Read Access to Amazon S3 Buckets](https://docs.aws.amazon.com/controltower/latest/userguide/strongly-recommended-guardrails.html#s3-disallow-public-read) | Detects whether public read access is allowed to Amazon S3 buckets |
|[Disallow Amazon RDS Database Instances That Are Not Storage Encrypted](https://docs.aws.amazon.com/controltower/latest/userguide/strongly-recommended-guardrails.html#disallow-rds-storage-unencrypted) | Detects whether your Amazon RDS database instances are not encrypted at rest, along with their automated backups, Read Replicas, and snapshots. |

Please follow instruction in the link [**HERE**](https://controltower.aws-management.tools/core/cttasks/#3-2-enable-disable-guardrails) to enable these guardrails.

**NOTE**:
> Don't forget to explore and try to create guardrail violation based on three setup mentioned above, observe how you can find the status of compliance in AWS Control Tower dashboard.

## Lab B - Deploy Custom AWS Config Rules ##
In this section of the lab, you will create new Detective guardrail to alert you when S3 bucket does not have default encryption enabled. AWS Config has built-in rules for this check called: `S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED`, we will utilize this rules and deploy a new custom guardrail.

#### Step 1 - Launch stackset to deploy custom detective guardrail
You will launch a CloudFormation Stackset to deploy this custom detective guardrail into your target OUs.

1.1. Log in to your AWS Control Tower `master` account.

1.2. Choose the appropriate region.

1.3. Click on the Launch Stackset button below to start deploying the stack.

[![LaunchStack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation#/stacksets/create)

1.4. While on **Create Stackset** page, enter the **Amazon S3 URL**: `https://s3.amazonaws.com/wellysiauw-deployment.us-east-1/config/s3_bucket_serverside_encrypt_config.yml`

1.5 Choose **NEXT**.

1.6. On **Specify Stackset details** page, enter your choice for **StackSet name** and choose **NEXT**

1.7. On **Configure StackSet options** under  **Permission** section, select **Service managed permissions** and choose **NEXT**

1.8. On **Deployment targets** section, select **Deploy to organizational units (OUs)** and enter the `AWS OU ID` according to notes you took in the **Prerequisite** stage.

1.9. On **Specify regions** section, select the `AWS Regions` where you would like to deploy this custom config. Control Tower supported regions can be found [here](https://aws.amazon.com/controltower/faqs/#Availability)

1.10 Choose **NEXT** to proceed. Review the selection and choose **SUBMIT** to deploy the Stackset.

1.11. Wait until the stackset operations change to **SUCCEEDED**

#### Step 2 - Verify the custom detective guardrail
As of time of writing, custom detective guardrail (Config) is not visible in the Control Tower dashboard, to verify the compliance of custom guardrail you will use AWS Config dashboard at the individual account level.
**NOTE** :

> you need to perform this action on another AWS account that is member of the AWS Control Tower, consider using separate browser session.

2.1. Log in to one of the linked AWS account that belongs to the `AWS OU ID` that you specified previously.

2.2 Click [**HERE**](https://console.aws.amazon.com/config/home#/rules/view) to jump to the AWS Config rules view.

2.3 In the **AWS Config Rules** page, click the newly created config rule `AWSControlTower_CUSTOM-GR_S3-SSE`

2.4 Observe the compliance status, try to create incompliance resource (S3 bucket without default encryption enabled) and check how long it takes for AWS Config to detect the incompliance resource.

#### Step 3 - Receive warning for custom detective guardrail

AWS Control Tower automatically aggregates all security notification into the `Audit` account. You can subscribe to the SNS notification in the `Audit` account to receive warning for incompliance resources.

**NOTE** :

> you need to perform this action on your `Audit` account, consider using separate browser session.

3.1. Log in to Control Tower `Audit` account using Administrator role.

3.2 Click [**HERE**](https://console.aws.amazon.com/sns/v3/home#/topics) to jump to the AWS Simple Notification Service (SNS).

3.3 Find topic with name `aws-controltower-AggregateSecurityNotifications` and click on the topic name.

3.4 Under **Subscriptions** tab, click **Create Subscription**

3.5 On the **Create subscription** page, select protocol of your choice, for this lab, choose `Email`.

3.6 On the **Endpoint** field, enter your email address and click **Create Subscription**

3.7 Check your email inbox for SNS confirmation email, don't forget to click **Confirm subscription**

3.8 Now, repeat test that you did previously on Step 2 and review the alert that you receive in your email.

## Lab C - Deploy Custom AWS SCP ##
In this section of the lab, you will create new Preventive guardrail that will prevent you from creating resource that did not align with the compliance requirements.

#### Step 4 - Download sample SCP
**Download** all SCP's below and save it in your computer before you proceed.

| Guardrail Name | Outcomes |
|---|---|
| [SCP-S3-SSE](https://s3.amazonaws.com/wellysiauw-deployment.us-east-1/scp/SCP-S3-SSE.json) | Enforce all objects in S3 bucket must have server side encryption enabled |
| [SCP-S3-NO-PUBLIC](https://s3.amazonaws.com/wellysiauw-deployment.us-east-1/scp/SCP-S3-NO-PUBLIC.json) | Enforce S3 bucket with block public access enabled |
| [SCP-RDS-ENCRYPT](https://s3.amazonaws.com/wellysiauw-deployment.us-east-1/scp/SCP-RDS-ENCRYPT.json) | Enforce RDS instance with encryption enabled |
| [SCP-EBS-ENCRYPT](https://s3.amazonaws.com/wellysiauw-deployment.us-east-1/scp/SCP-EBS-ENCRYPT.json) | Enforce EBS volume with encryption enabled |

#### Step 5 - Create new SCP
First you need to build the new SCP based on the example JSON file that you downloaded earlier.

5.1. Log in to Control Tower `Master` account using Administrator role.

5.2. Click [**HERE**](https://console.aws.amazon.com/organizations/)  to sign in to the Organizations console.

 - On the **Policies** tab, choose **Service control policies**.
 - On the **Service control policies** page, choose **Create policy**.
 - On the **Create policy** page, enter a name and description for the policy. Use the table above as reference for the SCP name.

5.3. On the **Policy** statement page, copy the content of the JSON SCP file that you downloaded earlier and paste it into the policy statement field.

5.4. Choose **Create Policy** to confirm.

5.5. Repeat step 5.2 and 5.4 for all SCP document.

**NOTE**:

> AWS Organization has limit of 5 (five) SCP that you can attach to root, OUs or Accounts. In real world scenario, you may want to create single SCP that combine all the guardrails.

#### Step 6 - Apply SCP to Organizational Unit (OU)
Next you will apply this SCP to one target OU of your choice. Make sure you have AWS account under your target OU to test the effectiveness of your SCP.

**WARNING**:

> We strongly recommends that you don't attach SCPs to the root of your organization and the Core OU without thoroughly testing the impact that the policy has on accounts.

6.1. Click [**HERE**](https://console.aws.amazon.com/organizations/)  to sign in to the Organizations console.

6.2 On the **Organize accounts** tab, choose the target `AWS OU` for test from the left navigation panel.
 - In the details pane on the right side, choose **Service control policies**.
 - On the **Service control policies** page, select the SCP that you want and choose **Attach**.

6.3 Repeat step 6.2 for other SCP that you would like to apply and test.

#### Step 7 - Test the SCP effect to Accounts under the target OU

As of time of writing, custom preventive guardrail (SCP) is not visible in the Control Tower dashboard, to verify the enforcement of custom guardrail you will use attempt to perform action at the individual account level.  

**NOTE** :
> you need to perform this action on another AWS account that is member of the AWS Control Tower, consider using separate browser session.

7.1. Log in to one of the linked AWS account that belongs to the `AWS OU ID` that you specified previously.  

7.2. Perform action such as:
- [Create EBS volume without encryption](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-creating-volume.html) or launch EC2 instance without enabling EBS encryption.
- [Create S3 Bucket without default encryption](https://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html) and then try to put object without any encryption specified.

7.3 Observe the warning that you receive, try to de-attach the SCP from your Control Tower `Master` account and observe the behavior.

## Lab D - Deploy Customized Service Catalog Portfolio ##

In this section of the lab, you will deploy new AWS Service Catalog portfolio that encompass all governance discussed above (encryption and public access). For this demo purpose, this service catalog porfolio will be deployed directly to the individual target account.

#### Step 8 - Deploy Service Catalog Portfolio

**NOTE** :
> you need to perform this action on another AWS account that is member of the AWS Control Tower, consider using separate browser session.

8.1. Log in to one of the linked AWS account that belongs to the `AWS OU ID` that you specified previously.  

8.2 Click [**HERE**](https://console.aws.amazon.com/iam/home#/roles) to jump to IAM role console.

8.3. Use the search box and type `AWSReservedSSO_AWSAdministratorAccess`

8.4 Click on the role name to open the role **Summary** page.

8.5. Take a note of the **Role Name**, it should be in format `AWSReservedSSO_AWSAdministratorAccess_xxxxxxxxxx`

8.6. Click on the Launch Stack button below to start deploying the stack.

[![LaunchStack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation#/stacks/create?templateURL=https://s3.amazonaws.com/wellysiauw-deployment.us-east-1/service-catalog/sc-portfolio-encrypt.json)

8.7. While on **Create Stackset** page, Choose **NEXT**.

8.8. On **Specify Stackset details** page, enter your choice for **Stack name**

8.9. Under parameter **LinkedRole1** enter the **IAM Role Name** that you copied on step 8.5.

8.10. Choose **NEXT**

8.11. On **Configure stack options** page, leave the default settings and choose **NEXT**

8.12. On **Review** page, make sure to select checkbox for

 - **I acknowledge that AWS CloudFormation might create IAM resources with custom names.**
 - **I acknowledge that AWS CloudFormation might require the following capability: CAPABILITY_AUTO_EXPAND**

8.13. Choose **Create Stack**


#### Step 9 - Provision Service Catalog Product
Now you can use the Service Catalog product from the portfolio and provision certain resource that align with the organization governance and compliance.

**NOTE**
> You must login using AWS SSO with role AWSReservedSSO_AWSAdministratorAccess_xxxxxxxxxx in order to access service catalog product

9.1. Log in to one of the linked AWS account that belongs to the `AWS OU ID` that you specified previously.  

9.2. Click [**HERE**](https://console.aws.amazon.com/servicecatalog/home?isSceuc=true#/products) to jump to Service Catalog product console.

9.3.  Select one of the product that you want to provision. Choose **Launch Product**

9.4. On the **Product Version** page, enter the **Provisioned product** name and select the default version. Choose **Next**.

9.5. On the **Parameters** page, enter the requested parameters (differ per each product type). Choose **Next**.

9.6. On the **TagOptions** page, enter any optional tags if you want. Choose **Next**.

9.7. On the **Notification** page, leave the default setting and choose **Next**.

9.8. On the **Review** page, review the selection and choose **Launch**.

9.9. After the provision product launched successfully, explore the newly created resource and confirm if it match the mandatory requirements (encryption enabled by default).

## Conclusion ##
You have successfully deployed optional and custom guardrail into AWS Control Tower to manage governance and compliance requirement of your organization. See additional link below for ideas on how to further customize your deployment.

#### Custom Guardrails ####
For deploying custom guardrail (SCP and AWS Config) across multiple accounts with automation, consider using [**Customizations for AWS Control Tower**](https://aws.amazon.com/solutions/customizations-for-aws-control-tower/) to build CI/CD pipeline for custom guardrails.

#### Service Catalog Pipeline
For deploying Service Catalog portfolio across multiple accounts, you can use:
- [**CloudFormation Stackset**](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html) to manage deployment in multiple accounts. Utilizing AWS Organization built-in support to target specific OUs.
- [**Multi-account Service Catalog pipeline**](https://controltower.aws-management.tools/infrastructure/resource/sc-multiaccount/) for implementing CI/CD on the service catalog portfolio.
- [**Portfolio Sharing**](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/catalogs_portfolios_sharing.html) for centralizing portfolio and share it across AWS Organization.
