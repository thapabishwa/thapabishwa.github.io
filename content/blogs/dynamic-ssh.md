---
title: "Dynamic SSH Config with ProxyCommand"
date: 2022-09-01T00:00:00+05:45
author: "thapabishwa"
tags: ["ssh", "jumpbox"]
summary: "Automate Jumpbox SSH config"
---

This article describes how to avoid hardcoding the IP/DNS addresses of the jumpbox in the SSH config file. And, is useful when you have multiple jumpboxes in different regions and you want to access them using the same SSH config file.

Some familiarity with Unix/Linux command line tools, AWS Cloud(incl. aws-cli) is assumed as a basic skill and is seldom discussed.

Some of the benefits of using ProxyCommand based ssh config are:

1. No need to update the SSH config file when the jumpbox IP/DNS changes
2. No need to update the SSH config file when you add a new region or environment
3. No need to update the SSH config file when you add a new resource in a subnet

# Background Context
The fundamental problem is to access resources in private subnets of a VPC. Like any other AWS resource, the EC2 instances in private subnets are not accessible from the internet. To access them, we need to use a jumpbox. The jumpbox is an EC2 instance in a public subnet of the same VPC. The jumpbox is the only instance accessible from the internet and has access to the resources in private subnets. The setup is in the following diagram:

{{< figure src="/image/plantuml/jumpbox/jumpbox.png"  height="auto" width="100%">}}

Normally, we would hardcode the IP/DNS address of the jumpbox in the SSH config file and then use it to jump to the private resources. This is not a good practice as the IP/DNS address of the jumpbox is likely to change. And, we would have to update the config file every time the IP/DNS address changes. Also, we would have to update the SSH config file when we add a new region or environment or a subnet, or a VPC. This is error-prone and tedious. 

This is where ProxyCommand-based dynamic ssh config comes to the rescue. Not only it avoids hardcoding of the jumpbox address, but it also avoids manual maintenance of the SSH config file.

# What is SSH ProxyCommand?
ProxyCommand is a specific command used to connect to a remote server. This allows you to specify a command to run when the SSH client is invoked. The specified command then determines and connects the users to the actual host. This is useful when you have multiple jumpboxes in different regions and you want to access them using the same SSH config file.

If that is hard, think of a function that takes an `AWS Region` and an `EC2 Instance Identifier` as input and returns the DNS address of the EC2 Instance. Think of another function that takes the result from the previous one and returns an SSH connection to the EC2 Instance. This is what Dynamic SSH does. 

In this article, we will implement Dynamic SSH by doing the following:
1. Fetch the DNS address of the jumpbox EC2 Instance at runtime
2. Use the DNS address to connect to the jumpbox instance itself
3. Then, use that connection to jump to the private instances

Let's talk about these steps in detail.

## Fetching the address

Since we are using AWS, we can use the `aws-cli` to fetch the DNS address of any EC2 instance. The following command will fetch the DNS address of the jumpbox in the `us-east-1` region:
{{< highlight bash >}}
aws ec2 describe-instances --region us-east-1 --filters "Name=tag:Name,Values=jumpbox" --query "Reservations[*].Instances[*].PublicDnsName" --output text
{{< /highlight >}}

As you can see, the script above does not take any input. It is hard coded to fetch the DNS address of the jumpbox in the `us-east-1` region. We need to pass the region as input to make it dynamic. We can do that by using the `REGION` environment variable.

{{< highlight bash >}}
aws ec2 describe-instances --region $REGION --filters "Name=tag:Name,Values=jumpbox" --query "Reservations[*].Instances[*].PublicDnsName" --output text
{{< /highlight >}}

Similarly, we can pass the jumpbox identifier as an input. We can do that by using the `JUMPBOX_IDENTIFIER` environment variable.

{{< highlight bash >}}
aws ec2 describe-instances --filters Name=tag:Name,Values="$JUMPBOX_IDENTIFIER" --region $REGION --query 'Reservations[*].Instances[*].[PublicDnsName]' --output text
{{< /highlight >}}

Finally, this script is not specific to a particular AWS region or environment. It can access any jumpbox in any AWS region or development environment. We must set the `JUMPBOX_IDENTIFIER` and `REGION` environment variables accordingly. Also, it can extend to support different cloud providers like Azure, GCP, etc. 

Very important to note that the script assumes that the jumpbox is tagged with the following tags:
- `Name` - the jumpbox identifier

## SSH to jumpbox

The following configuration is a wrapper around the previous script. It takes the region and jumpbox identifier as input and returns the SSH connection to the jumpbox. Then, this connection is used to jump to private resources.

{{< highlight bash >}}
Host jumpbox
	StrictHostKeyChecking no
	User ec2-user
	CheckHostIP/DNS no
	ProxyCommand bash -c "ssh ec2-user@$(aws ec2 describe-instances --filters Name=tag:Name,Values="$JUMPBOX_IDENTIFIER" --region $REGION --query 'Reservations[*].Instances[*].[PublicDnsName]' --output text) -W $(aws ec2 describe-instances --filters Name=tag:Name,Values="$JUMPBOX_IDENTIFIER" --region $REGION --query 'Reservations[*].Instances[*].[PublicDnsName]' --output text):22"
	UserKnownHostsFile /dev/null

Host *.compute.amazonaws.com
	StrictHostKeyChecking no
	AddKeysToAgent yes
	UseKeychain yes
	User ec2-user
	ForwardAgent yes
	IdentityFile ~/.ssh/ssh
	UserKnownHostsFile /dev/null
{{< /highlight >}}

The ssh config file listed above has the following sections:
1. `Host jumpbox` - the wrapper around the aws-cli script. It takes the `REGION` and `JUMPBOX_IDENTIFIER` as input and starts a new SSH command to connect to the jumpbox.
2. `Host *.compute.amazonaws.com` - this is the default section for the jumpbox. The ssh command executed in the previous step uses this configuration to connect to the jumpbox.

So, in short, the `jumpbox` section of the SSH config file is the wrapper around the aws-cli script. It takes the region and jumpbox identifier as input and starts a new SSH command to connect to the jumpbox. And the default section of the SSH config file is the wrapper around the `ssh` command executed by the `jumpbox` section. It takes the DNS address of the jumpbox as input and returns the actual SSH connection to the jumpbox. After setting the `JUMPBOX_IDENTIFIER` and `REGION` environment variables, we can use the following command to connect to the jumpbox:
### Sample Command
{{< highlight bash >}}
JUMPBOX_IDENTIFIER=jumpbox-1 REGION=us-east-1 ssh jumpbox
{{< /highlight >}}

### Sample output
{{< highlight bash >}}
~ >>>JUMPBOX_IDENTIFIER=jumpbox REGION=us-east-1 ssh jumpbox
Warning: Permanently added 'ec2-16-71-2-98.us-east-1.compute.amazonaws.com' (ED25519) to the list of known hosts.
Warning: Permanently added 'jumpbox' (ED25519) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|
[ec2-user@ip-10-0-0-1 ~]
{{< /highlight >}}

Viola !!! We have successfully connected to the jumpbox. Now we can enhance this setup to configure a `ProxyJump` to access the private resources.

## SSH to private resources

The following is the `Host *.compute.internal` section of the SSH config file, the wrapper that executes the `jumpbox` section and is responsible for jumping to private resources.

{{< highlight bash >}}
Host *.compute.internal
	ProxyJump jumpbox
	CheckHostIP no
	StrictHostKeyChecking no
	AddKeysToAgent yes
	UseKeychain yes
	User ec2-user
	ForwardAgent yes
	IdentityFile ~/.ssh/ssh
	UserKnownHostsFile /dev/null
{{< /highlight >}}

It works because the `ProxyJump jumpbox` section returns the SSH connection to the jumpbox. And, the `Host *.compute.internal` section uses this connection to jump to the private resources. Hence, we can access the private resources using the following command:
###  Sample Command
{{< highlight bash >}}
ssh private-ip.compute.internal
{{< /highlight >}}

### Sample output

{{< highlight bash >}}
~ >>>JUMPBOX_IDENTIFIER=jumpbox-1 REGION=us-east-1 ssh ip-10-0-0-2.compute.internal                          
Warning: Permanently added 'ec2-16-71-2-98.us-east-1.compute.amazonaws.com' (ED25519) to the list of known hosts.
Warning: Permanently added 'jumpbox' (ED25519) to the list of known hosts.
Warning: Permanently added 'ip-10-0-0-2.us-east-1.compute.internal' (ED25519) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-10-0-0-2 ~]$ 
{{< /highlight >}}


## Accessing PostgreSQL
This setup can also be used to access PostgreSQL. We can use the following command to connect to the PostgreSQL database:

### Sample Command
{{< highlight bash >}}
pgcli -h postgres.us-east-1.rds.amazonaws.com --password -d postgres --user postgres --ssh-tunnel jumpbox
{{< /highlight >}}<cite>pgcli[^1]</cite>
<cite>sshtunnel[^2]</cite>

[^1]: [Pgcli](https://www.pgcli.com/) is a command line interface for Postgres with auto-completion and syntax highlighting. It can be installed using the following command: `pip install pgcli`.
[^2]: [sshtunnel](https://pypi.org/project/sshtunnel/) is a Python library that allows you to create SSH tunnels from your local machine to a remote server. It can be installed using the following command: `pip install sshtunnel`.

### Sample output
{{< highlight bash >}}
~ >>>JUMPBOX_IDENTIFIER=jumpbox-1 REGION=us-east-1 pgcli --ssh-tunnel jumpbox -h my-postgres.us-east-1.rds.amazonaws.com --password -d postgres --user postgres
Password for postgres: ******************************* 
Server: PostgreSQL 13.4
Version: 3.4.1
Home: http://pgcli.com
postgres@127:postgres> select * from information_schema.columns
The result was limited to 1000 rows
Time: 2.499s (2 seconds), executed in: 1.953s (1 second)
postgres@127:postgres> quit
Goodbye!
{{< /highlight >}}


## Conclusion
Hence we have dynamically accessed the private resources using the `ProxyJump` and `ProxyConnect` features of the SSH. This setup is clean, easy, and straightforward to maintain and extend to support multiple cloud providers or environments.

<!--more--
## Code in action
{{<diagram name="component-diagram" type="plantuml">}}
top to bottom direction
@startuml
cloud aws {

}

aws .d> "ssh vmconnect"
aws <- "ssh vmconnect"

"ssh Bastion" -d> "ProxyCommand"  : triggers
"ProxyCommand" -d> "ssh vmconnect" : triggers
"ssh vmconnect" -d> "bastion host"
@enduml
{{</diagram>}}
-->