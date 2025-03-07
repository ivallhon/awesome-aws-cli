# AWS CLI Pro-level

As someone who works on Cloud Integrations at an Independent Software Vendor (ISV), I find myself diving deep into AWS API behavior as well as data extraction using the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html). This repository contains explanations, tips and tricks, and example one-liners to master the AWS CLI. 

To jump straight into example useful commands, go to the [Useful AWS CLI Commands](/docs/UsefulAWSCLICommands.md) document.

To learn about AWS CLI configuration tips, visit the [Getting Started](docs/GettingStarted.md) documentation.

## Common levers to modify CLI behavior

The AWS CLI behaves according to its default setup or to explicit settings. The AWS CLI honors configuration in the following order:

1. Command Parameter
2. Environment Variable
3. Profile setting in`~/.aws/config`

This means that you can keep your commonly preferred settings in your profiles configuration, alter specific settings on a shell session with environment variables (if running multiple commands) or simply use an explicity parameter on commands as needed. 

> [!NOTE]
> The AWS CLI v2 also pulls configuration from the EC2 Metadata Service (e.g. set the default AWS region to the one where the Instance is running on) 

| Configuration | Command Parameter | Environment Variable(s) | Profile setting parameter | Examples |
|---|---|---|---|---|
| Credentials | N/A | `AWS_ACCESS_KEY_ID`  `AWS_SECRET_ACCESS_KEY`  `AWS_SESSION_TOKEN` | `aws_access_key_id`   `aws_secret_access_key`   `aws_session_token`  (in `~/aws.credentials`) | export `AWS_ACCESS_KEY`=`AKIAxxxx` |
| CLI Profile to use (defined in `~/.aws/config`) | `--profile` | `AWS_PROFILE` | N/A | `aws ec2 describe-instances --profile development-account` |
| AWS Region | `--region` | `AWS_REGION` | `region` | `aws ec2 describe-instances --region us-east-1` |
| Output Format | `--output` | `AWS_DEFAULT_OUTPUT` | `output` | `aws ec2 describe-instances --output yaml` For more info, see [output formats](#output-formats) |
| Client-side filtering | `--query` | N/A | N/A | `aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName` For more info, see [client-side filtering](#client-side-filtering) |
| Client-side Pager (default Linux: `less` Windows:`more`) | `--no-cli-pager` | `AWS_PAGER` |  `cli_pager` | export `AWS_PAGER`="" |
| Full auto-prompt | `--cli-auto-prompt` | `AWS_CLI_AUTO_PROMPT` | `cli_auto_prompt` | export `AWS_CLI_AUTO_PORMPT`=`on` |
| Debug output | `--debug` | N/A | N/A | `aws ec2 describe-instances --debug` |

For more information, read [Configuration and credentials precedence](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#configure-precedence).

## Output Formats

The AWS CLI supports multiple output formats for the commands, being the most commonly used `json`. However, there are a couple of very interesting formats to highlight:

* text
* table

### text

Plain text output. This is very useful to concatenate AWS CLI commands and iterate with one command throught the output of another.

**Example:** List the availability zones in each AWS region, showing only the `ZoneName` and `ZoneId`.

**Command:**

```bash
for region in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do aws ec2 describe-availability-zones --region $region --query 'AvailabilityZones[].[ZoneName, ZoneId]' --output text; done
```

**Output:**

```bash
ap-south-2a     aps2-az1
ap-south-2b     aps2-az2
ap-south-2c     aps2-az3
ap-south-1a     aps1-az1
ap-south-1b     aps1-az3
ap-south-1c     aps1-az2
eu-south-1a     eus1-az1
eu-south-1b     eus1-az2
eu-south-1c     eus1-az3
eu-south-2a     eus2-az1
eu-south-2b     eus2-az2
eu-south-2c     eus2-az3
...
```

### table

ASCII table output. This is very useful for easier visualization of data.

**Example:** Same query as before but structure the output in table format with custom column names.

**Command:**

```bash
$ aws ec2 describe-availability-zones --query 'AvailabilityZones[].{AZName: "ZoneName", AZ_Id: "ZoneId"}' --output table
```

**Output:**

```bash
----------------------------
| DescribeAvailabilityZones|
+-------------+------------+
|   AZName    |   AZ_Id    |
+-------------+------------+
|  us-east-1a |  use1-az6  |
|  us-east-1b |  use1-az1  |
|  us-east-1c |  use1-az2  |
|  us-east-1d |  use1-az4  |
|  us-east-1e |  use1-az3  |
|  us-east-1f |  use1-az5  |
+-------------+------------+
```

For more information on the different AWS CLI output formats, visit the AWS CLI [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-output-format.html).

## Client-side filtering

The AWS CLI uses [JMESPath](https://jmespath.org/) for client-side filtering CLI output via the global `--query` parameter. This allows for very powerful output filtering and manipulation capabilities without the need of piping the output to additional tools. 

For more detailed information on client-side filtering, visit the AWS CLI [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-filter.html#cli-usage-filter-client-side).

> [!IMPORTANT]
> If you're working with a large amount of data, consider using server-side filtering first if supported by the API you're using, then do finer-grained filtering client-side.

Examples:

### Filter entries of a list based on a specific field value

#### String value

**Example:** Describe all non-default AWS regions (explicitely opted-in)

**Command:**

```bash
aws ec2 describe-regions --query 'Regions[?OptInStatus==`opted-in`].RegionName'
```

Output:

```bash
[
    "ap-south-2",
    "eu-south-1",
    "eu-south-2"
]
```

#### Integer value

**Example:** Find EBS volumes bigger than 50 GB

Command:

```bash
aws ec2 describe-volumes --query 'Volumes[?Size>`50`].VolumeId'
```

Output:

```bash
[
    "vol-xxxxxxxxxxxxxxxxx",
    "vol-yyyyyyyyyyyyyyyyy"
]`
```

#### Date value

**Example:** Find Snapshots older than `date`

**Command:**

```bash
aws ec2 describe-snapshots --owner self --query 'Snapshots[?StartTime<=`2025-01-01`].SnapshotId
```

**Output:**

```bash
[
    "snap-xxxxxxxxxxxxxxxxx"
]`
```

### Manipulate/modify the command output


#### Select a subset of fields to return in the command output

**Example:** Describe all instances. Return a table with the instance list containing: AvailabilityZone, InstanceId, InstanceType, VPCId and SubnetId.

**Command:**

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[].[Placement.AvailabilityZone, InstanceId, InstanceType, VpcId, SubnetId]' --output table
```

**Output:**

```bash
---------------------------------------------------------------------------------------------------------
|                                           DescribeInstances                                           |
+------------+----------------------+------------+------------------------+-----------------------------+
|  us-east-1d|  i-00000000000000000 |  t2.small  |  vpc-xxxxxxxxxxxxxxxxx |  subnet-abcdef12345678901   |
|  us-east-1b|  i-11111111111111111 |  t3.medium |  vpc-xxxxxxxxxxxxxxxxx |  subnet-1234567890abcdef0   |
|  us-east-1d|  i-99999999999999999 |  t2.large  |  vpc-xxxxxxxxxxxxxxxxx |  subnet-abcdef12345678901   |
|  us-east-1a|  i-66666666666666666 |  t3.medium |  vpc-yyyyyyyyyyyyyyyyy |  subnet-abcdef12345678901   |
|  us-east-1c|  i-abcdef12345678910 |  t2.medium |  vpc-xxxxxxxxxxxxxxxxx |  subnet-9abcdef1234567890   |
|  us-east-1a|  i-012345678910abcde |  t2.micro  |  vpc-xxxxxxxxxxxxxxxxx |  subnet-91234567890fedbca   |
+------------+----------------------+------------+------------------------+-----------------------------+
```

#### Modify the output format

**Example:** Describe all instances. Return a list in JSON in the following format: { "AvailabilityZone": "", "Id": "", "Type": "", "VPC": "", "Subnet": "" }

**Command:**

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[] | [].{"AvailabilityZone":Placement.AvailabilityZone, "Id": InstanceId, "Type": InstanceType, "VPC": VpcId, "Subnet": SubnetId}' --output json
```

**Output:**

```bash
[
    {
        "AvailabilityZone": "us-east-1d",
        "Id": "i-abcdef12345678910",
        "Type": "t2.small",
        "VPC": "vpc-xxxxxxxxxxxxxxxxx",
        "Subnet": "subnet-abcdef1234567890"
    },
    {
        "AvailabilityZone": "us-east-1b",
        "Id": "i-012345678910abcde",
        "Type": "t3.medium",
        "VPC": "vpc-xxxxxxxxxxxxxxxxx",
        "Subnet": "subnet-1234567890abcdef"
    },
    {
        "AvailabilityZone": "us-east-1d",
        "Id": "i-11111111111111111",
        "Type": "t2.large",
        "VPC": "vpc-xxxxxxxxxxxxxxxxx",
        "Subnet": "subnet-abcdef1234567890"
    }
]
```

#### Sort the output based on a specific field

**Example:**: Same command as in the previous section, but sort the output by AvailabilityZone.

> [!NOTE]
> This command uses JMESPath functions. For a complete list of available functions, visit the JMESPath [documentation](https://jmespath.org/specification.html#built-in-functions).

**Command:**

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[] | sort_by([].{"AvailabilityZone":Placement.AvailabilityZone, "Id": InstanceId, "Type": InstanceType, "VPC": VpcId, "Subnet": SubnetId},&AvailabilityZone)' --output json
```

**Output:**

```bash
[
    {
        "AvailabilityZone": "us-east-1b",
        "Id": "i-012345678910abcde",
        "Type": "t3.medium",
        "VPC": "vpc-xxxxxxxxxxxxxxxxx",
        "Subnet": "subnet-1234567890abcdef"
    },
    {
        "AvailabilityZone": "us-east-1d",
        "Id": "i-abcdef12345678910",
        "Type": "t2.small",
        "VPC": "vpc-xxxxxxxxxxxxxxxxx",
        "Subnet": "subnet-abcdef1234567890"
    },
    {
        "AvailabilityZone": "us-east-1d",
        "Id": "i-11111111111111111",
        "Type": "t2.large",
        "VPC": "vpc-xxxxxxxxxxxxxxxxx",
        "Subnet": "subnet-abcdef1234567890"
    }
]
```
