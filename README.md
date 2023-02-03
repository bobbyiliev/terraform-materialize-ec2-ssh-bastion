# Materialize Terraform Modules: EC2 SSH Bastion

A Terraform Module to set up a pre-configured EC2 SSH Bastion Host.

For the manual setup, see the [Materialize SSH Tunnel](https://materialize.com/docs/ops/network-security/ssh-tunnel) documentation.

## Overview

This module will create the following resources:

- Security group for EC2 instance: allows inbound traffic from the user's IP address and Materialize egress IP addresses
- SSH key pair: Your public key will be used to access the EC2 instance
- EC2 instance: Ubuntu 20.04 LTS

## Prerequisites

Before using this module, you must have the following:

- An AWS account
- Materialize instance
- Get your Materialize instance egress IP addresses from the `mz_egress_ips` table:

    Access the Materialize instance and run the following query:

    ```sql
    SELECT '[' || string_agg( '"' || egress_ip || '/32' || '"', ', ') || ']' as egress_ip_array
        FROM mz_egress_ips;
    ```

    The query above will return a JSON array of egress IP addresses. Define the following variable in your `terraform.tfvars` file:

    ```bash
    mz_egress_ips = ["123.456.789.0/32", "123.456.789.1/32"]
    ```
- An SSH key pair: You will need to provide the **public key** to the module.
    If you already have an SSH key pair, you can use it: `cat ~/.ssh/id_rsa.pub`.

    If you don't have one, you can generate one with the following command:

    ```bash
    ssh-keygen -t rsa -b 4096
    ```

- A VPC with public and private subnets.

## Running the module

1. Clone the repository:

    ```
    git clone https://github.com/bobbyiliev/terraform-materialize-ec2-ssh-bastion.git
    ```

2. Copy the `terraform.tfvars.example` file to `terraform.tfvars` and fill in the variables:

    ```
    cp terraform.tfvars.example terraform.tfvars
    ```

3. Define the following variables in `terraform.tfvars`:

    - `aws_region`: The AWS region, eg. `us-east-1`
    - `aws_profile`: The AWS profile, eg. `default`
    - `aws_config_file`: The AWS config file, eg. `~/.aws/config`
    - `mz_egress_ips`: Materialize instance egress IP addresses
    - `public_key`: Your public SSH key
    - `vpc_id`: VPC ID
    - `subnet_id`: Public subnet IDs

4. Apply the Terraform configuration:

    ```
    terraform apply
    ```

    Once you run the command, it might take a few minutes for the RDS instance to be created.

## Outputs

```sql
-- Your IP is: 123.456.179.1

-- On Materialize:
    -- Create a SSH connection in Materialize:
    CREATE CONNECTION ssh_connection TO SSH TUNNEL (
        HOST '123.456.179.1',
        USER 'ubuntu',
        PORT 22
    );

    -- Get the SSH private key for this connection:
    SELECT * FROM mz_ssh_tunnel_connections;

-- On the EC2 instance:
    -- From another terminal, SSH into the bastion host with:
    -- ssh ubuntu@123.456.179.1
    -- And then add the SSH key to the EC2 ~/.ssh/authorized_keys file:
    -- echo "ssh-ed25519 AAAA...76RH materialize" >> ~/.ssh/authorized_keys

-- On Materialize:
    -- Create a Materialize Kafka connection:
    CREATE CONNECTION kafka_connection TO KAFKA (
    BROKERS (
        'broker1:9092' USING SSH TUNNEL ssh_connection,
        'broker2:9092' USING SSH TUNNEL ssh_connection
        )
    );

    -- Create a Materialize Postgres connection:
    CREATE SECRET pgpass AS '<POSTGRES_PASSWORD>';

    CREATE CONNECTION pg_connection TO POSTGRES (
        HOST 'YOUR_RDS_INSTANCE.foo000.us-west-1.rds.amazonaws.com',
        PORT 5432,
        USER 'postgres',
        PASSWORD SECRET pgpass,
        SSL MODE 'require',
        DATABASE 'postgres'
        SSH TUNNEL ssh_connection
    );
```

## Helpful links

- [Materialize](https://materialize.com/)
- [Materialize SSH Tunnel](https://materialize.com/docs/ops/network-security/ssh-tunnel)
