# Awesome AWS CLI

As someone who works on Cloud Integrations at an Independent Software Vendor (ISV), I very often find myself diving deep into AWS APIs using the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html). In this repository I've compiled tips and tricks, example AWS CLI one-liners and useful [aliases](cli/alias) to help you get the most out of the AWS CLI. 

To learn about AWS CLI configuration tips, visit the [Getting Started](docs/GettingStarted.md) documentation.

# Useful commands

## EC2 

### List instances that have a specific tag key, show its tag value and a subset of relevant data

The following command uses server-side filtering (`--filter`) to list EC2 instances that have the tag `Name`. Then, using client-side filtering reduces the output to a the fields `InstanceId`, `InstanceType`, `ÀvailabilityZone`, `PrivateIP`, `PublicIP` and `<Value of the given tag key>`, sorted by `AvailabilityZone`.

> [!TIP]
> Always prefer server-side filtering when working with a large number of elements. 

```bash
export TAG_KEY=Name

aws ec2 describe-instances \
    --filter "Name=tag-key,Values=$TAG_KEY" \
    --query "Reservations[].Instances[].{Id: InstanceId, Type: InstanceType, AZ: Placement.AvailabilityZone, PrivateIP: PrivateIpAddress, PublicIP: PublicIpAddress, Name: Tags[?Key=='$TAG_KEY'] | [0].Value} | sort_by(@,&AZ) " \
    --output table
```

**Output:**

```bash
------------------------------------------------------------------------------------------------------------------------------------
|                                                         DescribeInstances                                                        |
+------------+----------------------+---------------------------------------------+----------------+-----------------+-------------+
|     AZ     |         Id           |                    Name                     |   PrivateIP    |    PublicIP     |    Type     |
+------------+----------------------+---------------------------------------------+----------------+-----------------+-------------+
|  us-east-1a|  i-xxxxxxxxxxxxxxxxx |  isaac-aws-sdk-instrumentation-testing      |  10.0.1.60     |  54.x.y.z       |  t3.medium  |
|  us-east-1a|  i-yyyyyyyyyyyyyyyyy |  isaac-ssm-test                             |  172.31.33.245 |  None           |  t2.micro   |
|  us-east-1a|  i-zzzzzzzzzzzzzzzzz |  isaac-aws-ug                               |  10.0.1.102    |  3.x.y.z        |  t3.medium  |
```

### Launch an EC2 instance from the latest AMI

When launching a new ephemeral EC2 instance for testing purposes, oftentimes you want to use the latest AMI provided by AWS. As these change frequently, you can leverage the public SSM parameters that AWS publishes referencing the latest AMIs.

The following command gives you a list of Amazon Linux of Amazon Linux AMI ids and its latest update date ordered by parameter name (i.e. AMI type: x86_64, arm64...) 

```bash
aws ssm get-parameters-by-path \
    --path '/aws/service/ami-amazon-linux-latest/' \
    --query 'Parameters[].{Name: Name, AMI: Value, LastModified: LastModifiedDate} | sort_by(@,&Name)' \
    --output table
```

**Output:**

```bash
------------------------------------------------------------------------------------------------------------------------------------------------
|                                                              GetParametersByPath                                                             |
+-----------------------+------------------------------------+---------------------------------------------------------------------------------+
|          AMI          |           LastModified             |                                      Name                                       |
+-----------------------+------------------------------------+---------------------------------------------------------------------------------+
|  ami-0f37c4a1ba152af46|  2025-02-22T00:20:02.735000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-arm64               |
|  ami-05b10e08d247fb927|  2025-02-22T00:20:03.118000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64              |
|  ami-0f37c4a1ba152af46|  2025-02-22T00:20:04.191000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64           |
|  ami-05b10e08d247fb927|  2025-02-22T00:20:04.543000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64          |
|  ami-0df19145e878164b6|  2025-02-22T00:20:03.469000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-minimal-kernel-6.1-arm64       |
|  ami-0320f10e7326a3e68|  2025-02-22T00:20:03.841000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-minimal-kernel-6.1-x86_64      |
|  ami-0df19145e878164b6|  2025-02-22T00:20:04.885000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-minimal-kernel-default-arm64   |
|  ami-0320f10e7326a3e68|  2025-02-22T00:20:05.214000+01:00  |  /aws/service/ami-amazon-linux-latest/al2023-ami-minimal-kernel-default-x86_64  |
| ...                   |   ...                              |  ...                                                                            |
```

Select the parameter name for the appropriate AMI type you need, then you can simply embed the command to resolve the AMI id into the `RunInstances` API call.

```bash
export AMI_ID=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query 'Parameter.Value' --output text)

aws ec2 run-instances \
    --instance-type t4g.micro \
    --image-id $AMI_ID
```

> [!NOTE]
> AWS publishes public parameters for multiple Operating Systems AMIs, e.g. Bottlerocket, Ubuntu, Debian, Windows... For more information, go to the AWS SSM [documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-finding-public-parameters.html).

> [!TIP]
> Normally, companies generate periodically their own Golden AMIs. Using shared [SSM parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-shared-parameters.html) you can publish in SSM the latest Golden AMI Ids of your company.  

### Create an AMI from an instance, then wait until its status is *Completed* so we can copy it to another region

The following bash script creates an image from a given EC2 instance and then uses a CLI waiter to wait until the AMI is completed.

```bash
export INSTANCE_ID="i-xxxxxxxxx"
export AMI_ID=$(aws ec2 create-image --instance-id $INSTANCE_ID --name cli-playground-example --query 'ImageId' --output text)

aws ec2 wait image-available $AMI_ID

if [ $? -eq 0 ] ; then
    echo "Image $AMI_ID for $INSTANCE_ID completed."
else
    echo "The image $AMI_ID is not on a completed state."
fi
```


## CloudWatch

### List all metric names and dimensions 

You might want to take a look at the available CloudWatch metrics in your account in general, or within a given namespace. You can also concatenate the output with Linux command line utilities to get the number of occurrences for each.

```bash
aws cloudwatch list-metrics --query 'Metrics[].{Namespace: Namespace, Key: MetricName}' --output text | sort | uniq -c
```

**Output:**

```bash
   2 4XXError	AWS/ApiGateway
 141 4xxErrors	AWS/S3/Storage-Lens
   2 5XXError	AWS/ApiGateway
 141 5xxErrors	AWS/S3/Storage-Lens
   1 AccountMaxReads	AWS/Cassandra
```

## S3

### Delete a large number of S3 objects from an S3 bucket with versioning enabled

When you're working with an S3 bucket that has versioning enabled, you need to explicitly delete each object version, otherwise, a DeleteMarker is created for each deleted object. 

The `aws s3api list-object-versions` command provides you a list of object versions. You can then use the `aws s3api delete-objects` command to pass a list of up to 1,000 object versions in a JSON structure like the following:

```json
{
    "Objects": [
        {
            "Key": "key",
            "VersionId": "version-id"
        },
        ...
    ]
}
```

You can then concatenate the two commands to delete all the objects:

```bash
export BUCKET_NAME=your_bucket_name
aws s3api delete-objects --bucket $BUCKET_NAME --delete "$(aws s3api list-object-versions --bucket $BUCKET_NAME --query '{"Objects": Versions[].{Key:Key,VersionId:VersionId}}')"
```

If your bucket has more than 1,000 object versions, you can disable pagination on `aws s3api list-object-versions` and iterate in blocks of 1,000 objects with the `--max-keys` parameter like the example script below (requires `jq`):

```bash
list_object_versions() {
    echo $(aws s3api list-object-versions --bucket $BUCKET_NAME --no-cli-pager --max-keys 50 --query '{"Objects": Versions[].{Key:Key,VersionId:VersionId}}')
}
delete_object_versions() {
    aws s3api delete-objects --bucket $BUCKET_NAME --delete "$1"; 
}

delete_object_versions "$(list_object_versions)"

continue=1
while [ $continue -eq 1 ];
do
    OBJECT_VERSIONS=$(list_object_versions)
    NUM_OBJECTS=$(echo $OBJECT_VERSIONS | jq '.Objects | length')
    if [ $NUM_OBJECTS -gt 0 ];
    then
        delete_object_versions "$OBJECT_VERSIONS";
    else
        continue=0
        echo "All Object versions have been deleted";
    fi
done
```

> [!NOTE]
> The `list_object_versions` limits the page size to 50 S3 Object versions to simulate multiple pages without requiring 1,000+ objects. To use this on a large number of objects just remove the --max-keys parameter to iterate in 1,000 object chunks. 

## ECR

### Execute docker login in your ECR private registry

When working with Amazon ECR and docker cli, you need to first get a temporary token for docker to authenticate against your ECR registry in a given region. You can add the below bash function to your `~/.aws/cli/alias` file for easy log in.

```bash
docker-login-in-region() {
    accountid=$(aws ecr describe-registry --region ${1} --query 'registryId' --output text)
    aws ecr get-login-password --region ${1} | docker login --username AWS --password-stdin ${accountid}.dkr.ecr.${1}.amazonaws.com
}
docker-login-in-region eu-west-1
```

## Systems Manager

AWS provides several Systems Manager [public parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-finding-public-parameters.html) to help you find useful information, like the latest published AMIs for different Operating Systems, or the availability of services per region.

### Get Amazon Linux AMIs

```bash
aws ssm get-parameters-by-path \
    --path '/aws/service/ami-amazon-linux-latest/' \
    --query 'Parameters[].{Name: Name, AMI: Value, LastModified: LastModifiedDate} | sort_by(@,&Name)' 
```
