# Custodian policies against multiple accounts using c7n-org

- [c7n-org](#c7n-org)
   - [Overview](#overview)
   - [Cross Account Access Role](#cross-account-access-role)
       - [step-1: Create Role on Target Account](#Create-Role-on-Target-Account)
       - [step-2: Assuming the created role on source account](#Assuming-the-created-role-on-source-account)
   - [Example](#example)
		- [executing c7n-org manually](#executing-c7n-org-manually)
		- [executing c7n-org using docker](#executing-c7n-org-using-docker)

### Overview

`c7n-org` is a tool to run custodian against multiple AWS accounts, Azure subscriptions, or GCP projects in parallel.

**Note:** Steps explained in the subsequent sections are valid only for `AWS` cloud platform. for other cloud providers such as for `Azure`, `GCP` refer to [docs](https://cloudcustodian.io/docs/tools/c7n-org.html).

### Cross Account Access Role

We need to create roles with required permissions(policies) on target accounts then we need to assume those roles on source account from where you will be executing c7n-org commands. 

To illustrate this better. Lets consider 2 accounts.

* Source account - 123456789012
* target account - 210987654321

### step-1: Create Role on Target Account

Create a policy named "remove-unused-ebs-volumes" with the following permissions.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ec2:DeleteVolume",
                "ec2:DescribeVolumeStatus",
                "ec2:DeleteTags",
                "ec2:DescribeSnapshotAttribute",
                "ec2:CreateTags",
                "ec2:CreateSnapshots",
                "ec2:DescribeVolumes",
                "ec2:CreateSnapshot",
                "ec2:DescribeSnapshots"
            ],
            "Resource": "*"
        }
    ]
}
```

Create a role named "ebs-garbage-collector" with created policy(remove-unused-ebs-volumes).

Go to IAM **->** Select Role **->** Click \"Create Role" **->** Select "Another AWS Account" **->** enter source accountid i.e 123456789012 in AccountID option **->** Filter Policies with "Customer Managed" **->** select "remove-unused-ebs-volumes" **->** Tags(Optional) **->** Enter Role name as "ebs-garbage-collector" **->** Click "Create Role".

### step-2: Assuming the created role on source account

create a iam user named "custodian" in IAM Section. Process is as follows.

IAM -> ADD User -> Enter Username as "custodian" -> Select "Programmatic access" -> Click "Next Permissions" -> Next Tags(optional) -> Click "Review" -> click "Create User"

If the user creation is successfull then you will get a "AWS Access key id" & "AWS Secret access key" make a note of it.

Create a policy named "c7n-org-policy-ebs-garbage-collector" with the following permissions.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "<ARN-of-Target-Account-Role>",
            "Condition": {
                "StringEquals": {
                    "aws:username": "<username>"
                }
            }
        }
    ]
}
```

**Note**: Replace \<ARN-of-Target-Account-Role\> in the above policy to your target role ARN. In our case \<ARN-of-Target-Account-Role\> is - arn:aws:iam::210987654321:role/ebs-garbage-collector.

**Note**: Replace \<username\> in the above policy to "custodian"( since created username is "custodian"). Here we are restricting the policy to be used only by user "custodian". In case if this policy is attached to any other user, they will not get the required privileges.

Now we need to attach the above policy to "custodian" IAM user.

IAM -> Users -> Select user "custodian" -> click "Add Permissions" -> Select "Attach Existing Policies Directly" -> Filter with "Customer Managed Policies" -> Select the policy "c7n-org-policy-ebs-garbage-collector" -> Click "Review" -> click "Add Permissions".

### Example

Create an "EC2 instance" for testing. SSH into that instance. 

### executing c7n-org manually

**configure the aws profile**

execute aws configure which prompts for "aws access key id" & "aws secret acces key" enter the values correctly(these were created during "custodian" user creation).

**setting up custodian environment**

python3 -m venv c7n-org
source c7n-org/bin/activate
pip install c7n-org

**execution**

copy the accounts.yml & ebs-garbage-collector.yml files to current directory.

```
c7n-org run -c accounts.yml -s output -u ebs-garbage-collector.yml --dryrun
```

### executing c7n-org using docker

**installing docker**

if you are into Amazon EC2 instance then run the below commands. for other operating system refer to [docs](https://docs.docker.com/get-docker/)

```
yum install -y docker
systemctl start docker
systemctl enable docker
```

**set the AWS environment variables**

export AWS_ACCESS_KEY_ID=\<AWS-ACCESS-KEY-ID\>
export AWS_SECRET_ACCESS_KEY=\<AWS-SECRET-ACCESS-KEY\>
export AWS_DEFAULT_REGION=\<AWS-DEFAULT_REGION\>

**Execution**

Make sure you copy  accounts.yml & ebs-garbage-collector.yml files to current directory before running the below command.

```
docker run -it -u `id -u $USER` -v $(pwd)/output:/home/custodian/output -v $(pwd)/ebs-garbage.yml:/home/custodian/ebs-garbage.yml -v $(pwd)/accounts.yml:/home/custodian/accounts.yml  --env-file <(env | grep "^AWS") cloudcustodian/c7n-org run -v -c accounts.yml -s output -u ebs-garbage.yml
```