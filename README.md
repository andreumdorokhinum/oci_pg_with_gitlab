# GitLab & Standalone Managed PostgreSQL on OCI

This tutorial shows how to configure a standalone managed PostgreSQL
database on OCI as the backend for a gitlab solution deployed on OCI Compute.

### First, create a managed PostgreSQL database
> OCI > Databases > PostgreSQL > DB Systems

Select appropriate PostgreSQL version according to your GitLab version:
https://docs.gitlab.com/administration/package_information/postgresql_versions/

In PostgreSQL, create a gitlab user, database and assign permissions:

    CREATE ROLE gitlab WITH LOGIN PASSWORD 'your_very_secure_password';
    CREATE DATABASE gitlabhq_production;
    GRANT ALL privileges ON DATABASE gitlabhq_production TO gitlab;

### Create an OCI Compute instance in the same private subnet where PostgreSQL was deployed and assign a public IP address to it:
> OCI > Compute > Instances > Create instance

SSH to the instance forwarding port 80:

    ssh -p 22 -L 8080:localhost:80 -o ServerAliveInterval=60 -o ServerAliveCountMax=10 -o ExitOnForwardFailure=yes opc@X.X.X.X'
where X.X.X.X is the public IP address of the instance and opc is the username

### Install GitLab community edition
(The script below is for Oracle Linux 8, adapt it depending on your
distribution of choice by selecting an appropriate package from packages.gitlab.com)

#### add GitLab CE repository
    wget https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh

#### make the script executable
    chmod +x script.rpm.sh

#### run the script
    sudo os=el dist=8 ./script.rpm.sh

#### install gitlab-ce
    sudo dnf install gitlab-ce -y

#### edit gitlab preferences
    sudo vim /etc/gitlab/gitlab.rb
        # Disable the bundled PostgreSQL in the gitlab.rb file:
        postgresql['enable'] = false
        # Enable Standalone PG
        gitlab_rails['db_adapter'] = 'postgresql'
        gitlab_rails['db_encoding'] = 'unicode'
        gitlab_rails['db_host'] = 'db_fqdn'
        gitlab_rails['db_port'] = '5432'
        gitlab_rails['db_username'] = 'gitlab'
        gitlab_rails['db_password'] = 'your_very_secure_password'
        gitlab_rails['db_database'] = 'gitlabhq_production'
where db_fqdn is the PostgreSQL primary endpoint (like 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-primary.postgresql.bb-ccccccccc-1.oc1.oraclecloud.com')

### Reconfigure GitLab
    sudo gitlab-ctl reconfigure

### Check the initial password
    sudo cat /etc/gitlab/initial_root_password

### In your browser, log into localhost:8080
and provide username = root, and the password taken from initial_root_password file

### For troubleshooting use:

#### gitlab logs
    sudo gitlab-ctl tail

#### status of gitlab
    sudo gitlab-ctl status
    sudo systemctl status gitlab-runsvdir.service

#### reset gitlab password
    sudo gitlab-rake "gitlab:password:reset"
