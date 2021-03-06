# Setup - Creating a Multi-Region workshop environment using AWS CloudFormation

To simplify the getting started experience with the labs, we have created foundational templates for <a href="https://aws.amazon.com/cloudformation/" target="_blank">AWS CloudFormation</a> that provisions the resources needed for the lab environment. The template is designed to deploy a consistent networking infrastructure, and client-side experience of software packages and components used in the workshop.

## Conventions

Due to the __multi-region__ nature of this Global Database workshop, you will often be switching between the two regions that has been assigned to you. You can always confirm and change the region on the top of the AWS Console Navigation Menu. __Please be mindful__ that you are performing the actions in the proper region, as some of the resources created are very similar between the two regions. The instructions will clearly label the AWS Region in which you will be performing the actions in, as indicated by these unique header labels when switching between the regions. In AWS hosted events, your workshop staff should also have identified the 2 regions specifically for you:

> **`Region 1 (Primary)`**    &nbsp;&nbsp;&nbsp;&nbsp;and&nbsp;&nbsp;&nbsp;&nbsp;    **`Region 2 (Secondary)`**

<span class="image">![Console Region Change](region-change.png)</span>

We will also provide commands for you to run in the terminal <a href="https://aws.amazon.com/systems-manager/features/#Session_Manager" target="_blank">AWS Systems Manager Session Manager</a>. These commands will look like this, with <b><i>highlighted text</i></b> usually replaced by values that are unique to your account settings and resource names:

<pre>
$ ssh -i <b><i>PRIVATE_KEY.PEM</i></b> ec2-user@<b><i>EC2_PUBLIC_DNS_NAME</i></b>
</pre>

You will often also see some sections that are expandable/collapsible to make the instructions more readable:

<details>
<summary><b>Click here to expand!</b></summary>
Good job! You expanded for more details! Click again to hide/collapse.
</details>


## Workshop Setup - Primary Region

If you are at an AWS public event, you are likely provided a 12-digit team code that gives access to a temporary AWS account dedicated for this workshop. If that is the case, the accounts already have the infrastructure in the assigned primary region pre-deployed.

> **`Region 1 (Primary)`**

* Open <a href="https://console.aws.amazon.com/cloudformation" target="_blank">CloudFormation</a> in the AWS Management Console.

* On the next screen, click on the stack with the Description *Amazon Aurora Labs - Global Database Workshop (Region 1)*, you should see that the status shows *CREATE_COMPLETE*.

* Click on the **Outputs** tab. The values here will be critical to the completion of the remainder of the lab.  Please take a moment to copy and save these values somewhere (like a notepad application) that you will have easy access to them during the remainder of the lab. The names that appear in the **Key** column are referenced directly in the instructions in subsequent steps, using the parameter value format: ``=[outputKey]=``

* This confirms that we have resources ready in the primary region. We can skip the step below and go directly to **Workshop Setup - Secondary Region**.

<details>
<summary><b>Manual Deployment for Primary Region <span style="color:red;">(expand only for testing or if Stack was missing)</span></b></summary>

If this is for AWS staff testing, or if you do not have the stack auto-deployed in your primary region. <a href="./manual-cfn1/">Click here for instructions for manual deployment.</a>

</details>

## Workshop Setup - Secondary Region

On the upper right corner of the AWS Console, click on your existing secondary region name and switch over to the secondary region that is assigned to you.

> **`Region 2 (Secondary)`**

| Region 2 | Region 2 Location | Deploy |
| --- | --- | --- |
| us-east-1 | N. Virginia |  <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?stackName=gdb2&templateURL=https://s3.amazonaws.com/ams-labs-prod-content-us-east-1/templates/lab-gdb2-no-cluster.yml" target="_blank"><img src="../../assets/images/cloudformation-launch-stack.png" alt="Deploy - Secondary"></a> |
| us-east-2 | Ohio | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/review?stackName=gdb2&templateURL=https://s3.amazonaws.com/ams-labs-prod-content-us-east-1/templates/lab-gdb2-no-cluster.yml" target="_blank"><img src="../../assets/images/cloudformation-launch-stack.png" alt="Deploy - Secondary"></a> |
| us-west-2 | Oregon | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?stackName=gdb2&templateURL=https://s3.amazonaws.com/ams-labs-prod-content-us-east-1/templates/lab-gdb2-no-cluster.yml" target="_blank"><img src="../../assets/images/cloudformation-launch-stack.png" alt="Deploy - Secondary"></a> |

Choose the region above matching your assigned secondary region to deploy the workshop environment. You can also download the [lab-gdb2-no-cluster.yml](/templates/lab-gdb2-no-cluster.yml) template and manually upload it to CloudFormation in your secondary region.

**Important**: Remember your primary and secondary region should be different. Refer back to the regions that has been assigned to you if you forget.

The desired template should be filled under **Template URL**. In the field named **Stack Name**, enter the value `gdb2`.

Scroll to the bottom, check the box that reads: **I acknowledge that AWS CloudFormation might create IAM resources with custom names** and then click **Create stack**.

<span class="image">![Region 2 CFN Launch](setup-cfn-gdb2a.png)</span>

In about 2-3 minutes, the status of the stack will change to `CREATE_COMPLETE`, click on the **Outputs** tab. The values here will be critical to the completion of the remainder of the lab.  Please take a moment to save these values somewhere that you will have easy access to them during the remainder of the lab. The names that appear in the **Key** column are referenced directly in the instructions in subsequent steps, using the parameter value format: ``=[outputKey]=``


## Checkpoint

At this point, you should have the base resources provisioned in the following:

> **`Region 1 (Primary)`**

* Basic VPC with public and private subnets across 3 availability zones; security groups and NAT Gateway
* VPC NACLs (for failure injection)
* Amazon EC2 Instance with Apache Superset (Our BI Application)
* Amazon Aurora Regional Cluster (running MySQL 5.7 compatible version)
* Amazon RDS DB subnet group for Aurora
* Amazon RDS DB and DB Cluster parameter groups for Aurora

> **`Region 2 (Secondary)`**

* Basic VPC with public and private subnets across 3 availability zones; security groups and NAT Gateway
* Amazon EC2 Instance with Apache Superset (Our BI Application)
* Amazon RDS DB subnet group for Aurora
* Amazon RDS DB and DB Cluster parameter groups for Aurora

You should now have also gathered the CloudFormation **Outputs** values from both regions. Below is an architectural representation of the resources that has been deployed so far.

![Setup Architecture Diagram](setup-arch.png)
