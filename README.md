
# ssh-over-ssm
Configure SSH and use AWS SSM to connect to instances. This allows Covalent employees with appropriate AWS IAM access to SSH into "bastion" linux EC2 instances in AWS and create a tunnel to a given AWS RDS database for secure access. With the tunnel in place the user can connect to the database via `localhost`.

## Info and requirements

### Requirements
- Install and configure AWS CLI https://covalentnetworks.atlassian.net/wiki/x/DQBiI
- Install `session-manager-plugin` locally - https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html#install-plugin-macos

## How it works
You configure each bastion host in a given AWS environment in your SSH config and specify `ssh-ssm.sh` to be executed as a `ProxyCommand` with your `AWS_PROFILE` environment variable set.
If your key is available via `ssh-agent` it will be used by the script, otherwise a temporary key will be created, used and destroyed on termination of the script. The public key is copied across to the instance using `aws ssm send-command` and then the SSH session is initiated through SSM using `aws ssm start-session` (with document `AWS-StartSSHSession`) after which the SSH connection is made. The public key copied to the server is removed after 15 seconds and provides enough time for SSH authentication. With the suggested configuration below a temporary key will always be generated and used. 

## Installation and Usage
This tool is intended to be used in conjunction with `ssh`. It requires that you've configured your awscli (`~/.aws/{config,credentials}`) and modified your ssh config. Place `ssh-ssm.sh` in `/usr/local/bin` and make it executable with `chmod +x /usr/local/bin/ssh-ssm.sh`.

### SSH config

Now we need to update our SSH config (`~/.ssh/config`).

#### Suggested Covalent configuration
```
host *
    TCPKeepAlive yes
    ServerAliveInterval 30
    ConnectTimeout 10
# SSH over Session Manager
host testRDS
    Hostname i-02a13b3faadfcf5b9
    User ec2-user
    ExitOnForwardFailure yes
    LocalForward 5432 test-postgres.caz4vt9gl7kc.us-east-1.rds.amazonaws.com:5432
    ProxyCommand bash -c "AWS_PROFILE=default ssh-ssm.sh %h %r"
host stageRDS
    Hostname i-0ab5c0d8f3a91a40c
    User ec2-user
    ExitOnForwardFailure yes
    LocalForward 5432 staging-postgres-blue.caz4vt9gl7kc.us-east-1.rds.amazonaws.com:5432
    ProxyCommand bash -c "AWS_PROFILE=default ssh-ssm.sh %h %r"
host prodRDS
    Hostname i-00b905674144d563a
    User ec2-user
    ExitOnForwardFailure yes
    LocalForward 5432 production-postgres-blue.caz4vt9gl7kc.us-east-1.rds.amazonaws.com:5432
    ProxyCommand bash -c "AWS_PROFILE=default ssh-ssm.sh %h %r"
host geUATRDS
    Hostname i-05e3bbea38a5ff85d
    User ec2-user
    ExitOnForwardFailure yes
    LocalForward 5432 ge-uat-postgres.caz4vt9gl7kc.us-east-1.rds.amazonaws.com:5432
    ProxyCommand bash -c "AWS_PROFILE=default ssh-ssm.sh %h %r"
host govRDS
    Hostname i-0c245fe21b57c9801
    User ec2-user
    ExitOnForwardFailure yes
    LocalForward 5432 ge-production-cluster.cluster-cfyo9nkme8sx.us-gov-east-1.rds.amazonaws.com:5432
    ProxyCommand bash -c "AWS_PROFILE=gov ssh-ssm.sh %h %r"
Match host i-*
    IdentityFile ~/.ssh/ssm-ssh-tmp
    PasswordAuthentication no
    GSSAPIAuthentication no
```
Above we've configured the bastion hosts in each environment for SSH access over SSM, specifying the username, instance ID and host to use for local commands i.e. `ssh {host}`. We've added the `AWS_PROFILE` that corresponds to each environment to the `ProxyCommand` so we don't need to manually provide credentials. 

`LocalForward` handles setting up the SSH tunnel to each environment's database endpoint automatically when connecting to the host. Occasionally the endpoints for a database may change. While an attempt will be made to communicate these changes to the team and keep this example configuration up to date, upon connection failures you can look for the RDS endpoint you are trying to reach in the AWS RDS console and ensure that your configuration is up to date. The first port number after `LocalForward` is the local port, this can be changed if your local `5432` is being used or you want to use some other port for your local connection.  

Finally `ExitOnForwardFailure` ends the SSH connection automatically if the port forwarding fails.

### How to use

To access a given database you `ssh {host}` where `{host}` is a host defined in your `~/.ssh/config` file. With the ssh tunnel created, you can connect to the database locally with the username `localhost` and the port `5432`. When you are finished working with the database return to your terminal and `logout` of the bastion host. This will tear down your session cleanly. 

Using the suggested config file above, to access the test database you would `ssh testRDS`

### Testing/debugging SSH connections

Show which config file and `Host` you match against and the final command executed by SSH:
```
ssh -G testRDS 
```

Debug connection issues:
```
ssh -vvv testRDS
```

For further informaton consider enabling debug for `aws` (edit ssh-ssm.sh):
```
aws ssm --debug command
```