---
layout: default
version: 3.0.12
title: VOMS System Administrator Guide}
---

# VOMS System Administrator guide

{% include sysadmin-guide-version.liquid %}

#### Table of contents

* [Introduction](#Intro)
* [Prerequisites and recommendations](#Prereq)
* [Upgrade instructions](#Upgrade)
* [Installation instructions](#Installation)
* [Configuration instructions](#Configuration)
* [Service operation](#Operation)
* [Service migration](#Migration)
* [Troubleshooting](#Troubleshooting)

#### Other guides
{% assign ref = site.data.docs.sysadmin-guide.versions[page.version] %}

- [VOMS Services configuration reference](configuration.html)
- [VOMS Admin guides]({{site.baseurl}}/documentation/voms-admin-guide/{{ref.admin_server_version}}/index.html)

## Introduction <a name="Intro">&nbsp;</a>

The Virtual Organization Membership Service (VOMS) is an attribute authority
which serves as central repository for VO user authorization information,
providing support for sorting users into group hierarchies, keeping track of
their roles and other attributes in order to issue trusted attribute
certificates and SAML assertions used in the Grid environment for authorization
purposes.

This guide is targeted at VOMS service administrators, i.e. people installing
and running the VOMS server.

## Prerequisites and recommendations <a name="Prereq">&nbsp;</a>

### Hardware

* CPU: No specific requirements
* Memory: 2GB if serving <= 15 VOs, more otherwise
* Disk: 10GB free space (besides OS and EMI packages)

### Operating system

* NTP Time synchronization: required.
* Host certificates: required
* Networking: for the service ports see the [Service Reference Card]({{site.baseurl}}/documentation/sysadmin-guide/{{page.version}}/service-ref-card.html)

### Installed software

Besides the usual OS packages you will need the UMD package repository configured.

All the other dependencies are resolved by the installation of the VOMS metapackages, **emi-voms-mysql**.

### Recommended deployment scenarios

A single-node installation, with the hardware recommendations given above should serve well most scenarios.
Serving a large number of VOs (> 15) will require more memory and disk space.

## Upgrade instructions <a name="Upgrade">&nbsp;</a>

### Upgrade preparation

It is always a good idea to dump the contents of the VOMS database. For MySQL-based installation follow
the instructions in the [database migration section](#Migration).

Also archive the configuration files for VOMS and VOMS-Admin, which live in the following directories:
```
/etc/voms
/etc/voms-admin
```

### UMD repository configuration <a name="Repository">&nbsp;</a>

Follow the [UMD installation instructions][umd] to install basic UMD
repositories.

If you want to install packages from the VOMS repository, follow the
instructions given [here]({{site.baseurl}}/releases.html).

### Upgrading to the latest VOMS release <a name="Upgrading">&nbsp;</a>

After having properly configured the repositories as explained in the previous section, just run:

```bash
yum update
```
to get the latest versions of the VOMS packages.

If the release notes indicate that a reconfiguration of the services is required, run `voms-configure` with the same parameters
that you used the first time you configured the VO. See the [Configuration section](#Configuration) for more information on how
to install and reconfigure the VOMS services.

If the release notes indicate that restarting the VOMS services is required, run

```bash
service voms restart
service voms-admin restart
```

#### <span class="label label-important">db upgrade</span> Upgrading the VOMS database <a name="db-upgrade">&nbsp;</a> 
If the release notes of the version that you are installing indicate that an upgrade of the VOMS database is required, follow 
the procedure described below:

1. Stop the services.
2. Backup the contents of the VOMS database following the instructions in the [database migration section](#Migration).
3. Run the upgrade script for each configured vo as follows:

   ```bash
   voms-db-util upgrade --vo <vo_name>
   ```

4. Restart the services

### Configuring the VOMS Admin container <a name="ContainerConf">&nbsp;</a>

Since version 3.0.1 VOMS Admin does not depend anymore on Tomcat but uses an
embedded Jetty container for running the VO web applications. Please set the
host, port and ssl information by editing the

```
/etc/voms-admin/voms-admin-server.properties
```

before reconfiguring the VOs (as explained in the following sections) or start the voms-admin server.
See the [VOMS configuration reference][voms-conf-ref] for a detailed reference of configuration parameters.

#### Configuring file limits for the VOMS Admin container

It is safe to configure the VOMS Admin container to have a reasonable limit for
the number of open files that can be opened by the voms-admin process (which
runs as user `voms`). The default file limit can be modified by editing the
`/etc/security/limits.conf` file:

```
voms          soft    nofile  63536
voms          hard    nofile  63536
```

#### Configuring memory for the VOMS Admin container <a name="voms-admin-mem-conf"></a>

The default Java VM memory configuration for the VOMS Admin container is
suitable for deployments which have at max 10 VOs configured, and is set in the
voms-admin init script:

```bash
VOMS_JAVA_OPTS="-Xms256m -Xmx512m -XX:MaxPermSize=512m"
```

In case your server will host more VOs, you should adapt the memory configuration for the container accordingly.
This can be done by setting the `VOMS_JAVA_OPTS` variable in the `/etc/sysconfig/voms-admin` file.
We recommend to allocate roughly 50m of heap space and 75m of permanent space per VO.
For example, for 15 VOs, the memory should be configured as follows:

```bash
VOMS_JAVA_OPTS="-Xms375m -Xmx750m -XX:MaxPermSize=1125m"
```

### <span class="label label-info">reconfiguration</span> Reconfiguring the VOs <a name="reconf"></a>

Sometimes a VOMS Admin upgrade requires a reconfiguration.

The VOs can be reconfigured using the `voms-configure` configuration tool (YAIM
is no longer supported), or your favourite configuration management tool (e.g.,
puppet).

The `voms-configure` tool is the evolution of the `voms-admin-configure`
script, and provides access to most VOMS and VOMS-Admin service configuration
parameters. For more detailed information about the `voms-configure` tool, see
the [configuration section](#Configuration).

The following command shows a basic reconfiguration of the VO:

```bash
voms-configure install \
--vo <vo_name> \
--hostname <hostname> \
--dbname <dbname> \
--dbusername <dbusername> \
--dbpassword <dbpassword> \
--core-port 15000 \
--mail-from <mail-from> \
--smtp-host <smtp-host>
```

The above command will migrate the configuration to the latest supported version.

<div class="alert alert-info">
	<i class="icon-eye-open"></i> Save the command you use to configure your VOs in a script for future reference. 
</div>

Once the configuration is over, you will need to upgrade the database as
explained in the [database upgrade section](#db-upgrade), i.e. running:

```bash
voms-configure upgrade --vo <vo_name>
```

for each configured VO and then restart the services with the following
commands:

```bash
service voms start
service voms-admin start
```

### Reconfiguring the information system

Configuration files for voms information providers are now provided by voms. To
configure the information providers for VOMS follow the instructions
[here](#bdii).

To configure EMIR follow the instructions [here](#emir).

## Clean installation instructions<a name="Installation">&nbsp;</a>

These are the full instructions for a clean installation. If you are upgrading,
see the [upgrade instructions section](#Upgrade).

### Repositories

See the instructions [above](#Repository).

### Certificate revocation lists

To enable periodic fetching of certificate revocation lists, run the fetch-crl script

```bash
/usr/sbin/fetch-crl
```
and enable a cron job that periodically refresh CRLs on the filesystem as follows:

```bash
/sbin/chkconfig fetch-crl-cron on
/sbin/service fetch-crl-cron start
```
### Clean installation

Install the `emi-voms-mysql` metapackage.

```bash
yum install emi-voms-mysql
```

## Configuration instructions <a name="Configuration">&nbsp;</a>

This section provides information on how to configure the VOMS services and the services VOMS depends on (e.g., mysql).
VOMS is now configured only using its own configuration utility, *voms-configure*. YAIM configuration is **no longer supported**.

A reference of VOMS services configuration files can be found in the [VOMS
Services Configuration reference][voms-conf-ref].

### Database backend configuration

#### MySQL

Make sure that the MySQL administrator password that you specify when running `voms-configure` matches the password that is set for the root MySQL account, as `voms-configure` will not set it for you. 

Ensure that MySQL is running. If not running, start it (as root) using the following command:
```
service mysqld start
```

The following commands change the password for the MySQL root account:

```bash
/usr/bin/mysqladmin -u root password <adminPassword>
/usr/bin/mysqladmin -u root -h <hostname> password <adminPassword>
```

### Configuring the VOMS Admin container

See the instructions [above](#ContainerConf).

### VOMS services configuration

Run `voms-configure` to configure VOs for both voms-admin and voms. The general syntax of the command is

```bash
voms-configure COMMAND [OPTIONS]
```

Available commands are:

* `install` is used to configure a VO
* `remove`: is used to remove a VO configuration
* `upgrade`: is used to upgrade the configuration of a VO installed with an older version of voms-admin.

Usually, you do not have a dedicated MySQL administrator working for you, so you will use voms-admin tools to create the database schema, configure the accounts and deploy the voms database. If this is the case, you need to run the following command:


```bash
voms-configure install --vo <vo name> \
--dbtype mysql \
--createdb \
-–deploy-database \
--dbauser <mysql root admin  username> \
--dbapwd <mysql root admin  password> \
--dbusername <mysql voms username> \
--dbpassword  <mysql voms password> \
--core-port <voms core service port> \
--smtp-host <STMP relay host> \
--mail-from <Sender address for service-generated emails>
```

Note that the above command is entered as a single command; it has been broken up into multiple lines for clarity.
The command creates and initializes a VOMS database, and configures the VOMS core and admin services that use such database. 
For more information about `voms-configure` options, see the man page.

An example MySQL VO installation command is shown below:

```bash
voms-configure install --vo test.vo \
--dbtype mysql --createdb --deploy-database \
--dbauser root --dbapwd pwd \
--dbusername voms --dbpassword pwd \ 
--core-port 15000 \
--mail-from ciccio@cnaf.infn.it \ 
--smtp-host iris.cnaf.infn.it
```
`voms-configure` is used also for removing already configured vos

```bash
voms-configure remove --vo VONAME
```

Available options are:

* undeploy-database:	 Undeploys the VOMS database. By default when removing a VO the database is left untouched. All the database content is lost.
* dropdb:	 This flag is used to drop the mysql database schema created for MySQL installations using the --createdb option

See the `voms-configure` help for a list of all the supported options and their meaning.

### Other configuration utilities

#### voms-mysql-util

The `voms-mysql-util` command is used for the creation or removal of the
database that will host the VOMS services tables on a MySQL database backend.
This command does not create the VOMS tables. This is done by the
`voms-db-util` command, which is described below. As `voms-mysql-util` is
invoked internally by `voms-configure` normally system administrators do not
use it, but it can sometimes be of help.

The general invocation is

```bash
voms-mysql-util COMMAND OPTIONS
```

Available options are:

* `create_db`  creates a MySQL database and grants read and write access
* `drop_db`  drops a MySQL database
* `grant_rw_access`  grants read and write access to a user
* `grant_ro_access`  grants read-only access to a user

The options are described in the following table

| Option | Description | Default value |
|:--------:|:-----------:|:-------------:|
| `--dbhost HOST` | Uses HOST when connecting to MySQL. | localhost |
| `--dbport PORT` | Uses PORT when connecting to MySQL. | 3306 |
| `--mysql-command CMD` | Uses CMD ad mysql command. | mysql |
| `--dbauser USER` | Uses USER when connecting to MySQL. USER must have the rights to create database and grant access to them. | root |
| `--dbapwd PWD` | Uses PWD when connecting to MySQL. This is the password of the user specified using the --dbauser option.|
| `--dbapwdfile FILE` | Reads the password to connect to the database from FILE. |
| `--dbusername USER` | Sets the database username to USER. |
| `--dbpassword PWD` | Sets the database password to PWD |
| `--vomshost HOST` | Sets the HOST where VOMS is running. This is the host from which MySQL will receive connections for the database. |
| `--dbname DBNAME` | Sets the VOMS database name to DBNAME. |

#### voms-db-util

The `voms-db-util` command is used to manage the deployment and upgrade of the VOMS database tables, and to add/remove administrators without requiring VOMS Admin VOs to be active. As `voms-db-util` is invoked internally by `voms-configure` normally system administrators do not use it, but it can sometimes be of help.

The general invocation is

```bash
voms-db-util COMMAND [OPTIONS]
```

The commands for installing or removing the database are

* `check-connectivity` check whether the database can be contacted
* `deploy` deploys the database for a given VO
* `undeploy` undeploys the database for a given VO
* `upgrade` upgrades the database for a given VO

The commands for adding or removing administrators are

* `add-admin` creates an administrator with full privileges for a given VO
* `remove-admin` removes an administrator from a given VO
* `grant-read-only-access`  creates ACLs so that VO structure is readable for any authenticated user

The options are described in the following table

| Option | Description |
|:--------:|:-----------:|
| `--vo` | The VO for which database operations are performed |
| `--dn` | The DN of the administrator certificate |
| `--ca` | The DN of the CA that issued the administrator certificate |
| `--email` | The EMAIL address of the administrator |
| `--cert` | The x.509 CERTIFICATE of the administrator being created |
| `--ignore-cert-email` | Ignores the email address in the certificate passed in with the --cert option |

### Configuration files

The VOMS server configuration lives in the `/etc/voms/<VO_NAME>` directory and is composed of two files:

- *voms.conf*, which contains the configuration for the server
- *voms.pass*, which contains the password to access the database

The VOMS Admin container configuration lives in the `/etc/voms-admin` directory and consists of the following files:

- *voms-admin-server.properties*, which contains the main service configuration (host, port, certificates)
- *voms-admin-server.logback*, which contains the logging configuration for the server

The VOMS Admin VO configuration lives in the `/etc/voms-admin/<VO_NAME>` direcotory and is composed
of the following files:

- *service.properties*, which contains the main VO configuration
- *database.properties*, which contains database access and connection pool configuration for the VO
- *logback.xml*, which controls logging of the VO application

Detailed information on configuration files can be found in the [VOMS Services Configuration reference][voms-conf-ref].

### Information system <a name="information-system">&nbsp;</a>

#### BDII <a name="bdii">&nbsp;</a>

##### Configure the info providers

The script `voms-config-info-providers` configures the providers for the resource bdii. Run

```bash
voms-config-info-providers -s SITENAME -e
```

giving the site name (which in the past went into the sitedef configuration file). If not deploying the administration service, skip the -e option.

Start the bdii service and check services are published. The query

```bash
ldapsearch -x -h localhost -p 2170 -b 'GLUE2GroupID=resource,o=glue' objectCLass=GLUE2Service
```

should return a service for each virtual organization.

#### EMIR <a name="emir">&nbsp;</a>

You can use [EMIR-SERP](https://twiki.cern.ch/twiki/bin/view/EMI/SERP) to publish VOMS information to EMIR. EMIR-SERP uses the information already available in the resource bdii and publish it to an EMIR DSR endpoint. You have to know the EMIR endpoint to do this, in the following example the EMI testbed EMIR
endpoint is used.

Install emir-serp

```bash
yum install emir-serp
```

and edit the configuration file `/etc/emi/emir-serp/emir-serp.ini`, providing the url for the EMIR DSR and the url for the resource bdii

```bash
...
url = http://emitbdsr1.cern.ch:9126
...
[servicesFromResourceBDII]
resource_bdii_url = ldap://localhost:2170/GLUE2GroupID=resource,o=glue
...
```

See the configuration file documentation for other options. You for sure will want to change the validity (the time EMIR DSR is told to consider the information valid) and period (the interval at which emir-serp will check for change in the bdii and refresh the publishing) attributes

```bash
# Period of registration/update messages
# Mandatory configuration parameter
# Value is given in hours
period = 1

# Time of registration entry validity
# Mandatory configuration parameter
# Value is given in hours
validity = 1
```

Start emir-serp with

```bash
service emir-serp start
```

and check your EMIR deployment to make sure the endpoints are published. You can spot problems increasing the verbosity of the emir-serp logging by editing the configuration file 

```bash
verbosity = debug
```

## Service operation <a name="Operation">&nbsp;</a>

VOMS is made of two daemons, VOMS Core and VOMS Admin. 

To start and stop all VOs on the machine, use the following commands:

```bash
service voms start
service voms-admin start
```

To start or stop a specific VO, use the following commands:

```bash
service voms start <vo>
service voms-admin start <vo> 
```

### Log files locations

|Service| Directory| Filename |
|:------|:---------|:---------|
| VOMS core | `/var/log/voms` | voms.VO_NAME |
| VOMS admin | `/var/log/voms-admin` | voms-admin-VO_NAME.log |
| VOMS admin | `/var/log/voms-admin` | server.log |

### Logging verbosity configuration

#### VOMS core

The VOMS core service logging verbosity is set with the `--loglevel`
option in the:
```
/etc/voms/VO_NAME/voms.conf
```

Log levels are numeric values which have the meaning defined in the following table:

| Value | Level name | Meaning |
|:---|:----------|:------------|
| 1 | LEV_NONE | Do not log |
| 2 | LEV_ERROR | Log only error messages |
| 3 | LEV_WARN | Log warn error messages and above |
| 4 | LEV_INFO | Log info messages and above |
| 5 | LEV_DEBUG | Log debug messages and above |

&nbsp;

The `--logtype` flag controls which type of information is logged by the voms server.
The default value for this option is `7` and should be left configured so.

#### VOMS admin

The VOMS admin service uses logback for logging configuration. 

The container level logging configuration is maintained in the file:

```
/etc/voms-admin/voms-admin-server.logback
```

while for a given VO is maintained in the file:

```
/etc/voms-admin/VO_NAME/logback.xml
```

Instructions for configuring the logging can be found directly in the configuration files.

##### voms-db-util logging

To change the verbosity of the voms-db-util command, refer to the 
following logback configuration file:
```
/var/lib/voms-admin/tools/logback.xml
```

## Migration <a name="Migration">&nbsp;</a>

In order to migrate VOMS to a different machine, the following items will need to be migrated:

1. The configuration
1. The database content. This holds only if VOMS was configured to access a local database instance. If a remote database is used for VOMS only the configuration will need to be migrated to the new installation.

### Configuration  migration
To migrate VOMS configuration, archive the contents of the following directories and move the archive to the new installation:

```
/etc/voms/*
/etc/voms-admin/*
```

### Database migration

In order to dump the contents of the VOMS datbase issue the following command on the original VOMS installation machine:

```
mysqldump -uroot -p<MYSQL_ROOT_PASSWORD> --all-databases --flush-privileges > voms_database_dump.sql
```

This database dump contains all the VOMS data and can be moved to the new VOMS installation machine.

To restore the database contents on the new VOMS installation machine, ensure that:

1. mysql-server is up & running
1. the password for the MySQL root account is properly configured (see the [configuration section](#Configuration) for more details)

The database content can then be restored using the following command:
```
mysql -uroot -p<PASSWORD> < voms_database_dump.sql
```

## Troubleshooting <a name="Troubleshooting">&nbsp;</a>

See the [known issues page]({{ site.baseurl }}/documentation/known-issues)

[voms-conf-ref]: {{site.baseurl}}/documentation/sysadmin-guide/{{page.version}}/configuration.html
[umd]: http://repository.egi.eu/category/umd_releases/distribution/umd-3/
