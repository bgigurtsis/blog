---
layout: post
title: "A secure alternative to traditional RDS access using AWS Identity Center (SSO), SSM, and IAM with ephemeral tokens"
date: 2025-02-18 12:00:00
summary: "A story of frustration and (lack of) documentation"
categories: aws rds documentation
comments: true
---

# Part 0: Introduction

Accessing RDS from human endpoints traditionally involves managing static credentials, VPN connections, and/or exposing bastion hosts to the internet.

This guide attempts to demonstrate a more secure alternative. It uses a combination of AWS Identity Center (SSO), Systems Manager (SSM), and Identity and Access Management (IAM) to leverage ephemeral login tokens to authenticate into RDS.

This ensures the only inbound internet traffic is from your endpoint to the bastion host that accepts the SSM port forwarding connection. Additionally, this can only be initiated via your SSO (Single Sign On) from your IDP (Identity Provider). This will be explained further below.

The creation of this guide, and indeed the original setup, required the correlation of multiple sources which will be linked at the bottom.

**NOTE: This guide uses an Aurora PostgreSQL DB, but the principles *should* be able to be applied to any RDS DB.**

## Full Disclosure

This project involved using two things that are quite possibly my weakest areas: databases and AWS networking. Because of this, it was and is very much a learning experience.

**Parts of this guide may be inaccurate or incorrect. I** would greatly appreciate advice on what could be done better here!

On the other hand, I think I managed to set up something and correlate information that several guides have failed to communicate effectively. On that basis, I hope that this guide can help someone looking to achieve a rather complex form of RDS authentication.

## Out of Scope

This guide does not include:

1. Setting up observability and logging for the above setup, which would be a necessary next step to ensure you can establish a baseline for access and audit/alert as appropriate.
2. Setting up IAM Identity Center (IDC) from scratch. The guide also presumes you have foundational knowledge of IDC, and know how to edit groups, permission sets, etc.
3. Service accounts. However, the setup for these would be relatively similar.
    1. For services inside AWS, you can use a role that has permission to call `rds-db:connect`.
    2. For services outside of AWS, you can use a federated identity or IAM user that has permission to call `generate-db-auth-token`, and then use that to authenticate.

## Prerequisites

Before you begin, you should have:

- An intermediate level of knowledge/familiarity about AWS, and the previously mentioned AWS services.
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed
- The [Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) installed
- PostgreSQL client tools installed - `brew install libpq && brew link --force libpq`
- Administrative access to AWS accounts that have:
  - Systems Manager
  - IAM Identity Center
  - The account where you want the RDS DB and EC2 Bastion to live
  - Systems Manager Host Management enabled in that account/the org account to automatically enroll all EC2 instances.

Estimated setup time: **45 - 90 mins.**

## Architecture

![Source: https://aws.amazon.com/blogs/database/securely-connect-to-amazon-rds-for-postgresql-with-aws-session-manager-and-iam-authentication/](https://i.imgur.com/zFNO7HN.png)

Source: [AWS Blog](https://aws.amazon.com/blogs/database/securely-connect-to-amazon-rds-for-postgresql-with-aws-session-manager-and-iam-authentication/)

To expand on the above, the connection flow, when complete, will look something like this:

1. In one terminal window:
    1. Use `aws sso login --profile myDBProfile` to SSO into AWS with a profile that lets you create an ephemeral DB token and start an SSM port forwarding session.
    2. Start an SSM port forwarding session with:
    
        ```bash
        aws ssm start-session --target {instance-id} --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":["5432"],"localPortNumber":["1053"]}' --profile myDBProfile
        ```
2. In a second terminal window:
    1. Obtain an ephemeral RDS login token using:
    
        ```bash
        export PGPASSWORD=$(aws rds generate-db-auth-token ...)
        ```
    
    2. Connect to your DB using that token through the local port you just opened up with the SSM command in the other window:
    
        ```bash
        psql "host=localhost port=1053 dbname=appdb user=$USER_NAME sslmode=verify-ca sslcert=rds-cert.pem"
        ```

# Part 1: Setting Up the Foundation

## VPC Configuration

1. Create or select a VPC with:
    1. Two private subnets (for RDS, EC2, and VPC endpoints)
    2. Two public subnets
2. Attach the following VPC Endpoint Interfaces to a private subnet (herein `Private Subnet 1`):
    - `ssm.*region*.amazonaws.com`
    - `ssmmessages.*region*.amazonaws.com`
    - `ec2messages.*region*.amazonaws.com`

You may be asking, "Well Billy, why not just have a VPC with two private subnets and no public subnets?" After all, the whole point of this configuration is to only use private subnets and limit public exposure by design. That’s also what the AWS guide on this topic details!

The answer is that **for some reason VPC interface endpoints cannot be placed on a private subnet *without a NAT gateway*.** This is not something I discovered until after setting everything up. And as you may know, NAT gateways have a flat fee regardless of usage.

![image.png](https://i.imgur.com/gzRJ7BF.png)

![Above: trying to create an interface endpoint on a VPC without a NAT gateway.](https://i.imgur.com/9OMeE6P.png)

Above: trying to create an interface endpoint on a VPC without a NAT gateway.

Instead of paying, we can simply not use the public subnets, not attach an internet gateway, and limit internet access that way. I think this is still a secure way of doing things? But please feel free to tell me otherwise!

# Part 2: Implementing Secure Access

## Bastion Host Configuration

Launch an EC2 instance with the following settings:

1. Ubuntu 22.04 (distro doesn’t matter, [so long as it has the SSM agent pre-installed](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html))
2. t3 / t2.micro
3. Placed in `Private Subnet 1` (i.e., the subnet created above with the VPC endpoints attached).
4. **Public IP is required for SSM connections**  
    - Could be wrong here, but until I added a public IP address I wasn't able to connect to SSM. The security group will ensure no inbound access regardless.
5. Select this IAM role: `AmazonSSMRoleForInstancesQuickSetup`
6. Create/attach a security group with:

    ```json
    {
      "Inbound": "No rules",
      "Outbound": "All traffic (this could probably be limited to SSM (443) and the RDS SG but I haven't had time to test it)."
    }
    ```

## RDS Instance Configuration

Launch a new RDS PostgreSQL instance with:

1. PostgreSQL 14 or later
2. Disabled public access
3. Select "Enable IAM database authentication"
4. Deploy in private subnet `Private Subnet 1`
5. Attach the RDS security group and edit it as below:

    ```json
    {
      "RDS Security Group": {
        "Inbound": "PostgreSQL (5432) from Bastion Security Group ID only",
        "Outbound": "RDS (5432) and HTTPS (443) for SSM"
      }
    }
    ```

## Identity Center Configuration

Create a new permission that will be attached to the account where your RDS database lives. For the permission set you have two options.

### Option 1: User-Based Permission Sets

Depending on what database you're securing, what your threat model is, and how many users will be using your database, I have considered two ways of setting up access. For me, with a database that few users will be using, I’ve decided to create individual permission sets per user. This will allow me to very easily audit access usage and see what user does what action in the database.

### Option 2: Role-Based Permission Sets

The other option is to create permission sets based on roles (admin, developer, readonly) and audit access by correlating:

- CloudTrail IAM Identity Center login logs
- Database connection logs
- SSM Session Manager logs

This is inherently more complex when it comes to auditing, but depending on your use case might be the way to go. The in-line Identity Center permission set policy for either use case is the same and can be found below:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds-db:connect"
      ],
      "Resource": [
        "arn:aws:rds-db:{region}:{account-id}:dbuser:{cluster-id}/{username}"
      ]
    },
    {
      "Sid": "SSM",
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession"
      ],
      "Resource": [
        "arn:aws:ec2:{region}:{account-id}:instance/{instance-id}",
        "arn:aws:ssm:{region}::document/AWS-StartPortForwardingSessionToRemoteHost"
      ]
    }
  ]
}
```

Where the RDS `$username` will be the RDS IAM user that we will create later on in this guide. For now you can set this as your `$firstName-admin`. `$instance-id` is the bastion host instance that we previously created. The remaining variables are left as an exercise to the reader.

You can call the permission set whatever you want, just note that you’ll have to reference it in the below AWS CLI config for `sso_role_name`.

Create a new profile in your AWS config for this new role, e.g.:

```ini
[profile RDS-Profile]
sso_start_url = https://your-sso-url.awsapps.com/start
sso_region = {region}
sso_account_id = {account-id}
sso_role_name = RDS-MyDBAccess-$firstName
region = {region}
```

Up until now, we have:

1. A VPC that has three VPC endpoints connected to a private subnet.
2. An EC2 instance bastion host, and an RDS database, both in the same private subnet, with security groups that allow communication between each other.
3. An AWS account that has been assigned an Identity Center user/group with the policy as described above. Take note of the profile name, as we will be using this below. If all of this is done, then let’s continue.

# Part 3: Initial Database Setup

When you initially create any database in RDS, it will automatically create and store an admin-level username/password in Secrets Manager. We will leverage these credentials to create our RDS IAM users.

## Prerequisites

1. Set up environment variables in your shell (add to `~/.bashrc` or `~/.zshrc`):

    ```bash
    export AWS_REGION="{region}"
    export RDSHOST="{your-cluster-endpoint}"
    export EC2_INSTANCE_ID="{your-bastion-id}"
    ```

2. Download the RDS .pem certificate bundles for your region, and store these in the same directory as where you’ll run the below setup process: [AWS RDS Certificates](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.CertificatesAllRegions)

## Setup Process

1. Login to your newly created SSO role:

    ```bash
    aws sso login --profile RDS-Profile
    ```

2. In Terminal 1 (keep this terminal open), establish SSM port forwarding. Note: this forwards all traffic from local port 1053 to the PostgreSQL port of 5432. (1053 is an arbitrary port and can be any open port on your device.)

    ```bash
    aws ssm start-session \
      --region "$AWS_REGION" \
      --target "$EC2_INSTANCE_ID" \
      --document-name "AWS-StartPortForwardingSessionToRemoteHost" \
      --parameters "{\"host\":[\"$RDSHOST\"],\"portNumber\":[\"5432\"],\"localPortNumber\":[\"1053\"]}" \
      --profile RDS-Profile
    ```

3. Retrieve your master username/password from Secrets Manager in the DB account.

4. In Terminal 2, open a PostgreSQL connection using those secrets. Note that specifying `hostaddr=127.0.0.1` avoids having to set your [localhost](http://localhost) to your RDS endpoint in your hosts file, as the AWS documentation advises. Also, ensure you have the appropriate `.pem` file (e.g., `eu-west-2-bundle-1.pem`) in the same directory.

    ```bash
    psql "host=$RDSHOST \
          hostaddr=127.0.0.1 \
          port=1053 \
          dbname=postgres \
          user=$UsernameYouGotFromSecretsManager \
          password=$PasswordYouGotFromSecretsManager \
          sslmode=verify-full \
          sslrootcert=./eu-west-2-bundle-1.pem"
    ```

5. Create IAM users. These RDS users need to be exactly the same as the `$username` you specified in your Identity Center permission set, otherwise the authentication won’t be successful:

    ```sql
    -- Create user
    CREATE USER $username;

    -- Grant IAM authentication and admin permissions
    GRANT rds_iam TO $username;
    ALTER USER $username WITH CREATEDB CREATEROLE;
    ```

6. Exit this PostgreSQL connection and, in the same terminal, test the RDS IAM user you just created:

    ```bash
    # Generate an authentication token
    export PGPASSWORD="$(aws rds generate-db-auth-token \
      --hostname "$RDSHOST" \
      --port 5432 \
      --region "$AWS_REGION" \
      --username "$USER_NAME" \
      --profile RDS-Profile)"
    ```

    ```bash
    psql "host=$RDSHOST \
          hostaddr=127.0.0.1 \
          port=1053 \
          dbname=postgres \
          user=$USER_NAME \
          sslmode=verify-full \
          sslrootcert=./eu-west-2-bundle-1.pem"
    ```

# Part 4: Making It User-Friendly(ish)

To make logging into this database more user friendly, you can provide your end users with a zip file containing a few scripts and the certificate bundle. Additionally, give them instructions on how to:

1. Set up their AWS profile.
2. Set up the config file with their environment variables.
3. Run `chmod +x` on the scripts when they first use them.
4. Run the scripts in two separate terminal windows.

Effectively, this means they just have to run three commands:

```bash
cd /ScriptDirectory
sh ./start-ssm.sh
sh ./connect-psql.sh
```

### Connection Scripts Package

Create a ZIP archive containing:

1. **config.sh**

    ```bash
    # config.sh - Edit these values for your environment when connecting via SSO 

    AWS_PROFILE="RDS-Profile"          # Your AWS SSO profile name
    AWS_REGION="eu-west-2"             # Your AWS region
    RDSHOST="rds.cluster-xxxx.eu-west-2.rds.amazonaws.com"  # Your RDS endpoint
    EC2_INSTANCE_ID="i-xxxxxxxxxxx"    # Your bastion host ID
    DB_USER="firstName_admin"          # Your database username
    DB_NAME="postgres"                 # Database name 
    CERT_BUNDLE="./eu-west-2-bundle.pem"  # Your region's cert bundle
    ```

2. **start-ssm.sh**

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail

    # Load configuration (assumes config.sh is in the same directory)
    source "$(dirname "$0")/config.sh"

    # Check for required commands
    command -v aws >/dev/null 2>&1 || { echo "Error: AWS CLI is required. Try running brew install awscli"; exit 1; }

    command -v session-manager-plugin >/dev/null 2>&1 || { echo "Error: AWS SSM plugin is required. Try running brew install --cask session-manager-plugin"; exit 1; }

    # Log in to AWS SSO
    echo "Logging in to AWS SSO using profile '$AWS_PROFILE'..."
    aws sso login --profile "$AWS_PROFILE"

    # Start SSM session with port forwarding
    echo "Starting SSM session: forwarding local port 1053 to remote $RDSHOST:5432..."
    aws ssm start-session \
        --region "$AWS_REGION" \
        --target "$EC2_INSTANCE_ID" \
        --document-name "AWS-StartPortForwardingSessionToRemoteHost" \
        --parameters "{\"host\":[\"$RDSHOST\"],\"portNumber\":[\"5432\"],\"localPortNumber\":[\"1053\"]}" \
        --profile "$AWS_PROFILE"
    ```

3. **connect-psql.sh**

    ```bash
    #!/usr/bin/env bash
    set -euo pipefail

    # Load configuration (assumes config.sh is in the same directory)
    source "$(dirname "$0")/config.sh"

    # Check for required commands
    command -v psql >/dev/null 2>&1 || { echo "Error: psql is required. Try running brew install libpq && brew link --force libpq"; exit 1; }

    command -v aws >/dev/null 2>&1 || { echo "Error: AWS CLI is required."; exit 1; }

    # Check for the SSL certificate file
    if [ ! -f "$CERT_BUNDLE" ]; then
        echo "Error: SSL root certificate (rds-cert.pem) not found in the current directory."
        exit 1;
    fi

    # Generate the authentication token for the RDS connection
    echo "Generating authentication token for RDS..."
    PG_TOKEN=$(aws rds generate-db-auth-token \
        --hostname "$RDSHOST" \
        --port 5432 \
        --region "$AWS_REGION" \
        --username "$DB_USER" \
        --profile "$AWS_PROFILE")

    if [ -z "$PG_TOKEN" ]; then
        echo "Error: Failed to generate the authentication token."
        exit 1;
    fi

    export PGPASSWORD="$PG_TOKEN"

    # Connect to PostgreSQL using the local forwarded port (1053) while
    # specifying the original RDS host for SSL verification.
    echo "Connecting to PostgreSQL database '$DB_NAME' as user '$DB_USER'..."
    psql "host=$RDSHOST \
        hostaddr=127.0.0.1 \
        port=1053 \
        dbname=${DB_NAME} \
        user=$DB_USER \
        sslmode=verify-full \
        sslrootcert=$CERT_BUNDLE"
    ```

4. **RDS SSL Certificate for Your Region**  
   Download from: [AWS RDS SSL Certificates](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.CertificatesDownload)

# Troubleshooting

### 1. Network & VPC

- **VPC Endpoints:** Ensure the three endpoints (`ssm.`, `ssmmessages.`, `ec2messages.`) are in the correct private subnet.
- **Subnets & SGs:** Verify that the bastion is in a subnet with a public IP and that SGs allow SSM (outbound 443) and PostgreSQL (inbound 5432 from the bastion).

### 2. Bastion Host & SSM

- **SSM Agent & Role:** Confirm the SSM agent is running on the bastion and it has the proper IAM role attached.
- **Session Start:** Run the SSM session command manually; use `aws ssm describe-instance-information` to check instance registration.

### 3. RDS & Database Connection

- **IAM DB Auth & Certs:** Make sure IAM authentication is enabled on RDS and you’re using the correct SSL certificate.
- **Token Generation:** Test `aws rds generate-db-auth-token` manually; verify the created database user matches the IAM policy.

### 4. IAM Identity Center & Permissions

- **Profile Settings:** Confirm your AWS SSO profile (in `config.sh`) is correct.
- **Permission Policy:** Ensure the permission set includes the correct ARNs and matches the DB user name.

# References

- [EC2 Systems Manager VPC Endpoints](https://repost.aws/knowledge-center/ec2-systems-manager-vpc-endpoints)
- [Create Interface Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/create-interface-endpoint.html)
- [Securely Connect to Amazon RDS for PostgreSQL](https://aws.amazon.com/blogs/database/securely-connect-to-amazon-rds-for-postgresql-with-aws-session-manager-and-iam-authentication/)
- [AWS Blog Comments](https://aws.amazon.com/blogs/database/securely-connect-to-amazon-rds-for-postgresql-with-aws-session-manager-and-iam-authentication/?commentId=2oPWKD2tTaMaB8nTu6rUEUuvpoS#Comments)
- [Secure Your Data with Amazon RDS for SQL Server](https://aws.amazon.com/blogs/database/secure-your-data-with-amazon-rds-for-sql-server-a-guide-to-best-practices-and-fortification/)
