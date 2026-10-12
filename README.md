# AWS End-to-End Cloud Solution: EC2, RDS MySQL, and S3

## Project Overview

This project demonstrates a basic end-to-end cloud application
architecture on Amazon Web Services (AWS). An Ubuntu Amazon EC2 instance
acts as the application/server machine, Amazon RDS hosts a private MySQL
database, and Amazon S3 stores application files and objects.

The project is designed for an AWS Cloud Computing practical. It covers
networking, compute, managed databases, object storage, IAM permissions,
security groups, and connectivity troubleshooting.

## Objectives

-   Create and configure a custom Amazon VPC.
-   Deploy an Ubuntu EC2 instance in a public subnet.
-   Create private subnets for the RDS database subnet group.
-   Configure security groups so MySQL is reachable only from the EC2
    instance.
-   Create an Amazon RDS for MySQL database with public access disabled.
-   Use an EC2 IAM role to access a specific S3 bucket.
-   Upload, list, download, and verify objects in S3.
-   Test the EC2-to-RDS connection and troubleshoot network errors.

## Architecture

``` text
                         Internet
                            |
                     Internet Gateway
                            |
                  Public Route Table
                            |
                 Public Subnet (10.0.1.0/24)
                            |
                  Ubuntu EC2 Instance
                  CloudApp-Server
                  Security Group:
                  CloudApp-EC2-SG
                      /          \
                     /            \
       MySQL TCP 3306              HTTPS/API calls
      allowed from EC2 SG                |
                   |                     v
                   v              Amazon S3 Bucket
           Private RDS MySQL      sri-cloud-storage-2026-001
           cloudapp-db            (Block Public Access ON)
           Public access: No
           Security Group:
           CloudApp-RDS-SG
```

The RDS DB subnet group must contain subnets in at least two different
Availability Zones. The database remains private; do not make it
publicly accessible to solve connection problems.

## AWS Resources

The following names reflect the resources used during implementation.
Verify actual settings in the AWS Console before relying on them.

  -----------------------------------------------------------------------
  Resource                            Name / configuration
  ----------------------------------- -----------------------------------
  AWS Region                          US East (N. Virginia), `us-east-1`

  VPC                                 `CloudApp-VPC`

  VPC CIDR                            `10.0.0.0/16`

  Public subnet                       `CloudApp-Public-Subnet`,
                                      `10.0.1.0/24`

  Private subnet 1                    `CloudApp-Private-Subnet-1`,
                                      `10.0.3.0/24`

  Private subnet 2                    `CloudApp-Private-Subnet-2`,
                                      `10.0.4.0/24`

  Additional private subnet           `CloudApp-Private-Subnet-3`,
                                      `10.0.5.0/24`

  EC2 instance                        `CloudApp-Server` (Ubuntu)

  EC2 security group                  `CloudApp-EC2-SG`

  RDS security group                  `CloudApp-RDS-SG`

  RDS DB subnet group                 `CloudApp-RDS-Subnet`

  RDS DB identifier                   `cloudapp-db`

  RDS engine                          MySQL

  RDS port                            `3306`

  S3 bucket                           `sri-cloud-storage-2026-001`

  EC2 IAM role                        `CloudApp-S3-Access`
  -----------------------------------------------------------------------

**Subnet note:** During setup, both original private subnets were found
in `us-east-1b`. An additional subnet was created to provide another
Availability Zone. Confirm that the DB subnet group uses at least two
actual subnets in different AZs. A subnet is not private merely because
its name contains "Private"; verify its route table has no direct
default route to an Internet Gateway.

## Prerequisites

-   AWS account with permission to create VPC, EC2, IAM, S3, and RDS
    resources.
-   An SSH key pair for connecting to the EC2 instance.
-   An Ubuntu EC2 instance in the public subnet.
-   A custom VPC and correctly configured route table.
-   A strong RDS master password stored securely.
-   Awareness that AWS resources may incur charges. Check the current
    pricing and account eligibility before creating resources.

## Implementation Steps

### 1. Create the VPC and subnets

1.  Open the VPC Console and create `CloudApp-VPC` with CIDR
    `10.0.0.0/16`.
2.  Create the public subnet `CloudApp-Public-Subnet` with CIDR
    `10.0.1.0/24`.
3.  Create private subnets using non-overlapping CIDRs, such as
    `10.0.3.0/24`, `10.0.4.0/24`, and `10.0.5.0/24`.
4.  Ensure the subnets selected for the RDS DB subnet group span at
    least two Availability Zones.
5.  Attach `CloudApp-IGW` to the VPC.
6.  Configure the public route table with:
    -   `10.0.0.0/16` → local
    -   `0.0.0.0/0` → Internet Gateway
7.  Associate the public subnet with the public route table and enable
    auto-assignment of public IPv4 addresses for the public subnet if
    required.
8.  Ensure private subnets do not route directly to the Internet
    Gateway.

### 2. Configure the EC2 security group

The EC2 security group is named `CloudApp-EC2-SG`.

-   Allow SSH (TCP 22) only from your current public IP when practical.
-   Allow HTTP (TCP 80) from the internet only if hosting a public web
    service.
-   Add HTTPS (TCP 443) only if required.
-   Keep outbound access sufficient for required updates, S3 access, and
    database connections.

Avoid leaving SSH open to `0.0.0.0/0` longer than necessary.

### 3. Launch the EC2 instance

1.  Launch an Ubuntu EC2 instance in `CloudApp-Public-Subnet`.
2.  Attach `CloudApp-EC2-SG`.
3.  Attach the IAM role `CloudApp-S3-Access`.
4.  Connect through SSH using the correct private key.
5.  Update packages:

``` bash
sudo apt update
```

### 4. Create the S3 bucket

1.  Open Amazon S3 and create `sri-cloud-storage-2026-001` in
    `us-east-1`.
2.  Keep S3 Block Public Access enabled.
3.  Enable bucket versioning if required for the practical.
4.  Create an `uploads/` prefix by uploading an object to that path.

Example commands from EC2:

``` bash
echo "AWS Cloud Practical - Version 1" > test.txt
aws s3 cp test.txt s3://sri-cloud-storage-2026-001/uploads/test.txt
aws s3 ls s3://sri-cloud-storage-2026-001/uploads/
aws s3 cp s3://sri-cloud-storage-2026-001/uploads/test.txt downloaded.txt
cat downloaded.txt
```

To verify versioning:

``` bash
aws s3api get-bucket-versioning \
  --bucket sri-cloud-storage-2026-001
```

Expected output when versioning is enabled:

``` json
{
  "Status": "Enabled"
}
```

To confirm which IAM identity EC2 is using:

``` bash
aws sts get-caller-identity
```

The EC2 role should have only the S3 permissions required for this
bucket. An `AccessDenied` response from `aws s3 ls` with no bucket
specified can be expected if `s3:ListAllMyBuckets` is not granted. Test
the bucket directly instead.

### 5. Create the RDS DB subnet group

1.  Open Amazon RDS → **Subnet groups** → **Create DB subnet group**.
2.  Name it `CloudApp-RDS-Subnet`.
3.  Set the description to `Private subnets for CloudApp MySQL`.
4.  Select VPC `CloudApp-VPC`.
5.  Select at least two private subnets in different Availability Zones.
6.  Create the subnet group and verify it appears without an error.

### 6. Create the RDS security group

Create `CloudApp-RDS-SG` in `CloudApp-VPC`.

Inbound rule:

  Type           Protocol   Port   Source
  -------------- ---------- ------ -------------------
  MySQL/Aurora   TCP        3306   `CloudApp-EC2-SG`

Do not use `0.0.0.0/0` as the MySQL source. This rule permits
connections from network interfaces associated with the EC2 security
group.

### 7. Create the RDS MySQL instance

1.  Open RDS → Databases → **Create database**.
2.  Choose **Standard create** and engine **MySQL**.
3.  Choose an eligible small instance/template after reviewing the cost.
4.  Set the DB identifier to `cloudapp-db`.
5.  Set the master username to `admin` or another appropriate username.
6.  Create and securely store a strong password.
7.  Set the VPC to `CloudApp-VPC`.
8.  Select DB subnet group `CloudApp-RDS-Subnet`.
9.  Set **Public access** to **No**.
10. Attach `CloudApp-RDS-SG`.
11. Use port `3306`.
12. Review the estimated cost and create the database.
13. Wait for status **Available**.
14. In Connectivity & security, copy the actual endpoint hostname.

Do not use a sample or placeholder endpoint. The RDS endpoint is
assigned by AWS and must be copied from the actual database details.

### 8. Test connectivity from EC2

Install a MySQL client and a TCP testing utility:

``` bash
sudo apt update
sudo apt install mysql-client netcat-openbsd -y
```

Replace `YOUR_REAL_RDS_ENDPOINT` with the endpoint copied from the RDS
console:

``` bash
nc -zv -w 5 YOUR_REAL_RDS_ENDPOINT 3306
```

If the TCP connection succeeds, connect to MySQL:

``` bash
mysql -h YOUR_REAL_RDS_ENDPOINT -P 3306 -u admin -p
```

Enter the password only at the interactive prompt. Do not put the
password in the command or commit it to source control.

### 9. Create a sample database and table

After successfully logging in and seeing the `mysql>` prompt, run:

``` sql
CREATE DATABASE cloudapp;
USE cloudapp;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO users (name, email)
VALUES
('Sri', 'sri@gmail.com'),
('Student', 'student@gmail.com');

SELECT * FROM users;
```

Expected result: two rows are returned from the `users` table.

Exit MySQL with:

``` sql
EXIT;
```

## Troubleshooting

### MySQL connection timeout: `ERROR 2003 ... (110)`

A timeout generally indicates a network path or access-control issue.
Check these items in order:

1.  Confirm the RDS instance status is `Available`.
2.  Confirm you copied the real endpoint from the correct RDS instance
    and region.
3.  Confirm EC2 and RDS are in the same VPC (`CloudApp-VPC`).
4.  Confirm RDS has `CloudApp-RDS-SG` attached.
5.  Confirm `CloudApp-RDS-SG` inbound TCP `3306` source is
    `CloudApp-EC2-SG`.
6.  Confirm the EC2 instance actually uses `CloudApp-EC2-SG`.
7.  Check EC2 outbound rules permit the connection.
8.  Check subnet network ACLs for the required MySQL traffic and return
    ephemeral ports.
9.  Review subnet route tables and use VPC Reachability Analyzer if
    needed.
10. Retest:

``` bash
nc -zv -w 5 YOUR_REAL_RDS_ENDPOINT 3306
```

Keep RDS private during troubleshooting; do not expose MySQL to the
public internet.

### `Name or service not known`

The endpoint hostname is invalid, misspelled, or still a placeholder.
Copy the endpoint directly from RDS → Databases → your database →
Connectivity & security.

### `Access denied for user`

The network connection reached MySQL, but authentication failed. Verify
the username and password. Reset the RDS master password through the AWS
Console if necessary; do not share credentials.

### S3 `AccessDenied`

-   Check the EC2 IAM role attachment.
-   Check the role policy resource ARN and required actions.
-   Test the bucket directly:

``` bash
aws s3 ls s3://sri-cloud-storage-2026-001/uploads/
```

-   `aws s3 ls` without a bucket name requires `s3:ListAllMyBuckets`,
    which is not necessary if the role is intentionally restricted to
    one bucket.

## Security Best Practices

-   Keep RDS **Public access: No**.
-   Permit MySQL only from the EC2 security group.
-   Keep S3 Block Public Access enabled.
-   Follow least privilege for IAM policies.
-   Store database credentials securely; do not commit them to GitHub.
-   Restrict SSH to your IP where possible.
-   Avoid placing passwords, access keys, or private key files in this
    repository.
-   Review AWS billing and remove resources when the practical is
    complete.

## Validation Checklist

-   [ ] VPC and subnets created.
-   [ ] Public route table connected to the Internet Gateway.
-   [ ] RDS DB subnet group spans at least two AZs.
-   [ ] EC2 instance running in the intended public subnet.
-   [ ] EC2 IAM role can access the intended S3 bucket.
-   [ ] S3 object upload, list, and download verified.
-   [ ] S3 bucket versioning verified.
-   [ ] RDS MySQL instance status is Available.
-   [ ] RDS is not publicly accessible.
-   [ ] RDS security group permits TCP 3306 from EC2 security group
    only.
-   [ ] EC2-to-RDS TCP connection succeeds.
-   [ ] MySQL login succeeds.
-   [ ] Sample database/table created and query results verified.
-   [ ] Screenshots captured for the practical report.

## Evidence to Capture for Submission

1.  VPC details and subnet list with Availability Zones.
2.  Public route table route to the Internet Gateway.
3.  EC2 instance details and attached security group.
4.  EC2 IAM role and S3 policy.
5.  S3 bucket and uploaded object.
6.  S3 versioning status.
7.  RDS DB subnet group with subnets across two AZs.
8.  RDS security group inbound rule.
9.  RDS database status and connectivity settings.
10. Successful `nc` output and MySQL query output.
11. S3 upload/download terminal output.

## Cleanup

To reduce ongoing costs after collecting your evidence:

1.  Export or save any required database data and screenshots.
2.  Delete the RDS database if it is no longer needed. Choose a final
    snapshot only if required.
3.  Delete unused EC2 instances and associated resources if the
    practical is complete.
4.  Remove unused EBS volumes, snapshots, Elastic IP addresses, NAT
    Gateways, or other billable resources after verifying they are not
    needed.
5.  Delete the S3 bucket only if its contents are no longer needed and
    your submission is complete.
6.  Check AWS Billing and Cost Management for remaining resources.

Deletion is irreversible for some resources. Verify what you need to
retain before removing anything.

## Current Implementation Status

At the time this README was prepared:

-   EC2 was running and had an IAM role attached.
-   S3 upload/download and bucket versioning had been tested
    successfully.
-   The RDS endpoint had been copied from the console.
-   The EC2-to-RDS MySQL connection was still timing out with error 2003
    (110). Therefore, successful RDS connectivity and the sample SQL
    table should be marked complete only after retesting and receiving
    successful output.

## Conclusion

This project combines AWS networking, compute, database, storage, and
identity management services in one architecture. EC2 acts as the
application server, RDS stores relational data privately, and S3 stores
objects with IAM-controlled access. Final validation requires a
successful EC2-to-RDS connection and a successful SQL query, in addition
to the already-tested S3 operations.
