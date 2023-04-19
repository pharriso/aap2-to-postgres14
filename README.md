Migrate AAP2 DB to postgres14
=========

This repo contains some steps to move AAP from a "managed" postgres13 database to an external "self-managed" postrges14. Please be aware of [scope of support for self-managed databases in AAP2](https://access.redhat.com/articles/4010491).

Environment overview
------------

For this example, I'm migrating from an all-in-one AAP2 controller instance to a seperate postgresql 14 server. This is the starting point:

1 x Red Hat Enterprise Linux release 8.7 virtual machine
AAP installed with 2.3-2
Ansible Automation Platform Controller 4.3.8
postgresql-server-13.10-1

We are going to move to an external postgres server with the following details:

1 x Red Hat Enterprise Linux release 8.7 virtual machine
postgresql14-server-14.7-1


Installing postgresql server
------------

```bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql14-server
/usr/pgsql-14/bin/postgresql-14-setup initdb
systemctl enable postgresql-14
```

Allow external connections. Edit **/var/lib/pgsql/14/data/pg_hba.conf** and add:

```
host    all         all         0.0.0.0/0 scram-sha-256
```

Edit **/var/lib/pgsql/14/data/postgresql.conf** and add:

```
listen_addresses = '*'
```

Start postgres server

```bash
systemctl restart postgresql-14
```

Validate it is listening on port 5432:

```bash
ss -tulnp | grep 5432
tcp   LISTEN 0      128          0.0.0.0:5432      0.0.0.0:*    users:(("postmaster",pid=23716,fd=6))
tcp   LISTEN 0      128             [::]:5432         [::]:*    users:(("postmaster",pid=23716,fd=7))
```

Prepare the postgres14 database
------------

```
# su - postgres
$ psql
postgres=# CREATE ROLE awx;
CREATE ROLE
postgres=# ALTER ROLE awx WITH PASSWORD 'Redhat123' LOGIN;
ALTER ROLE
postgres=# CREATE DATABASE awx OWNER awx;
CREATE DATABASE
postgres=# ALTER ROLE awx WITH CREATEDB;
```

Configure remote connectivity to controller database
------------

This step is needed if using a co-located postgres database with controller. 

Allow external connections. Edit **/var/lib/pgsql/14/data/pg_hba.conf** and add:

```
host    all         all         0.0.0.0/0 scram-sha-256
```

Stop controller services ready for backup
------------

Stop automation controller services. For a clustered install, do this on all nodes.

```bash
automation-controller-service stop
```

If postgres is installed on the same server as controller we need to start postgresql:

```bash
systemctl start postgresql
```

Backup the database
------------

We need to backup the database using the client tools on the later version of postgres. I'm going to run the backup from the new postgres server:


```bash
pg_dump awx --host=controller.pharriso.demolab.local --port=5432 --username=awx --clean --create --exclude-table=main_instance --exclude-table=main_instancegroup --exclude-table=main_instancegroup_instances > backups/tower.db
Password: 
```

Backup the instance topology:

```bash
pg_dump awx --host=controller.pharriso.demolab.local --port=5432 --username=awx --clean --create --table=main_instance --table=main_instancegroup --table=main_instancegroup_instances  > backups/new_instance_topology.db
```

Restore the database
------------

Restore the controller database:

```bash
psql --dbname=awx --host=pg14.pharriso.demolab.local --port=5432 --username=awx < backups/tower.db
```

Restore the instance topology:

```bash
psql --dbname=awx --host=pg14.pharriso.demolab.local --port=5432 --username=awx < backups/new_instance_topology.db
```

Configure controller to use the new database
------------

On the controller node edit **/etc/tower/conf.d/postgres.py** and update the postgres server. Repeat this on all nodes in a cluster:

```
DATABASES = {
   'default': {
       'ATOMIC_REQUESTS': True,
       'ENGINE': 'awx.main.db.profiled_pg',
       'NAME': 'awx',
       'USER': 'awx',
       'PASSWORD': """Redhat123""",
       'HOST': 'pg14.pharriso.demolab.local',
       'PORT': '5432',
       'OPTIONS': { 'sslmode': 'prefer',
                    'sslrootcert': '/etc/pki/tls/certs/ca-bundle.crt',
       },
   }
}
```

Start the automation controller services:

```bash
automation-controller-service start
```

If this is an embedded postgres server then we can stop the service on the controller node:

```bash
systemctl stop postgresql
```

Testing
------------

Login to the controller UI. Attempt to run project sync, inventory source sync and run a job template.

![](/static/pg_migrate_jobs.png)

We can query the new database to make sure we are connecting to it. On the new postgres14 server we can query job id 699:

```bash
su - postgres
postgres=# \c awx
You are now connected to database "awx" as user "postgres".
awx=# select * from main_jobevent where job_id = 699;
```

Run the AAP installer
------------

Re-run the AAP setup script to ensure we can still manage the environment with Red Hat provided installer for further day2 operations like upgrades.

Update the installer **inventory** with the new DB connection details:


```
[all:vars]
admin_password='Redhat123'

pg_host='pg14.pharriso.demolab.local'
pg_port=5432

pg_database='awx'
pg_username='awx'
pg_password='Redhat123'
pg_sslmode='prefer'  # set to 'verify-full' for client-side enforced SSL
```

Run the setup script and ensure we get a clean run:

```bash
./setup.sh


PLAY RECAP *********************************************************************
controller.pharriso.demolab.local : ok=284  changed=49   unreachable=0    failed=0    skipped=196  rescued=0    ignored=0   
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   

The setup process completed successfully.
Setup log saved to /var/log/tower/setup-2023-04-19-09:59:59.log.
```


