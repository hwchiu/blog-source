---
title: LXR Server With Multiple Projects
tags:
  - System
  - LXR
  - Ubuntu
  - DRBD
  - Ceph
  - SourceCode
keywords: 'LXR,lxr,Ubuntu,Linux,System,SourceCode'
abbrlink: 53585
date: 2017-05-08 17:38:04
---

Install LXR on Ubuntu 16.04
===========================
In this article, I will write down a example to descrip how to setup the **LXR**(Linux Cross Reference) with multiple projects support. In this configuration, your can view multiple projects' refernce in one LXR service.
For this article, I use the **Ceph** and **DRBD** for my LXR projtects.

<!--more-->

Environment
-----------
- Ubuntu 16.04
- Lxr 2.2.1
    1. You can download the tarball from [here](https://sourceforge.net/projects/lxr/files/stable/)
- Ceph
    - [Github Page](https://github.com/ceph/ceph)
- DRBD-9.0
    - [Git Page](http://git.drbd.org/drbd-9.0.git)


Prepare LXN Environment
-----------------------
1. Install the required softwared
    - perl (5.10 or above version)
    - ctags (5.8 or above version)

``` bash
apt-get install perl
apt-get install exuberant-ctags
```

2. Install the Datbase, I choosed the MySQL for my lxr env.

```
apt-get install mysql-server
```

3. Install Perl DBI
    - Install the CPAN for Perl.
    - Type the following command to install DBI

```
cpan DBI
```

4. Install the Web Server, I choose the Apacher for my lxr env.
    - Also install perl module.

```
apt-get install apache2
apt-get install libapache2-mod-perl2
```

5. Install free-text searching engine, I choose Glimpse.
    - Download from the github page and follow the README.install to install.

```
git clone https://github.com/gvelez17/glimpse
cd glimpse
./configure
make
make install
```

6. Install Perl Module
    - File::MMagic
    - DBD::mysql

```
apt-get install libmysqlclient-dev
cpan File::MMagic
cpan DBD::mysql
```

Install LXR
-----------
1. Download the LXR-2.2.1 and expand the tarball
    - Download from [lxr-2.2.1.tgz](https://sourceforge.net/projects/lxr/files/stable/lxr-2.2.1.tgz/download)
    - Decompress into the /opt direcotory (choose the path you like) and rename to lxr
2. Check the environment

```bash
cd /opt/lxr
./genxref --checkonly
```

The result will like below.

```
ERROR: could not open configuration file lxr.conf
[  OK  ]     Perl     version ... 5.22.1
Parameter 'ectagsbin' not defined - trying to find ctags
ctags found at /usr/bin/ctags
[  OK  ]     ctags    version ... 5.9
Parameter 'glimpsebin' not defined - trying to find glimpse
glimpse found at /usr/local/bin/glimpse
Checked:    glimpse   version ... 4.18.7
Parameter 'glimpseindex' not defined - trying to find glimpseindex
glimpseindex found at /usr/local/bin/glimpseindex
Checked: glimpseindex version ... 4.18.7
Parameter 'swishbin' not defined - trying to find swish-e
swish-e not found, `command -v swish-e` returned a null string
genxref stopped without indexing by --checkonly option
```

Since we have not config the LXR, we won't have the lxr.conf.
We choose the glimpse as our search engine and we can ignore the warning of swish-e.

Configure LXR
-------------
1. Since the **GIT** type of source project doesn't support the submodule reference in LXR, we use **FILE** instead of. Before we generate the code reference, we should **update** code by ourself.
2. Prepare the source fo DRBD and CEPH. I put them in **/opt/lxr/source_code**
    - We refer to the master branch of ceph, for rdbe, is version 9.0.

```
mkdir -p /opt/lxr/source_code/drbd
mkdir -p /opt/lxr/source_code/ceph
git clone --recursive http://git.drbd.org/drbd-9.0.git /opt/lxr/source_code/drbd/9.0
git clone --recursive http://git.drbd.org/drbd-9.0.git/tags  /opt/lxr/source_code/ceph/master
```

3. Create both LXR and database configuration via tool `configure-lxr.pl`

```
cd /opt/lxr
./scripts/configure-lxr.pl -vv
```

```
*** LXR configurator (version: 2.2) ***

LXR root directory is /opt/lxr
Configuration will be stored in custom.d/

Configure for single/multiple trees? [S/m] > m

*** LXR web server configuration ***

Many different configurations are possible, they are related to the way
LXR service is accessed, i.e. to the structure of the URL.
Refer to the User's Manual for a description of the variants.

LXR can be located at the server-root (so called dedicated)
or lower in the server hierarchy (shared because there are
usually other pages or sections).
Server type? [dedicated/SHARED] > SHARED

Selecting which tree to display can be done in various ways:
  1. from the host name (all names are different),
  2. from a prefix to a common host name (similar to previous)
  3. from the site section name (all different)
  4. from interpretation of a section name part (similar to previous)
  5. from the head of script arguments
Method 5 is highly recommended because it has no impact on webserver
  configuration.
Method 3 is second choice but involves manually setting up many
  symbolic links (one per source-tree).
Method 1 & 2 do not involve symbolic links but need populating webserver
  configuration with virtual hosts.
  Note that method 2 does not work well on //localhost.
Method 4 is deprecated because it has proved not easily portable
  under alternate webservers (other than Apache).

Tree designation?:
   ARGUMENT
   section name
   prefix in hos
   hostname
   embedded in section
 > ARGUMENT

The computer hosting the server is described by an URL.
The form is scheme://host_name:port
where:
  - scheme is either http or https (http: can be omitted),
  - host_name can be given as an IP address such as 123.45.67.89
              or a domain name like localhost or lxr.url.example,
  - port may be omitted if standard for the scheme.
--- Host name or IP? [//localhost] > //127.0.0.1
--- Alias name or IP? >
URL section name for LXR in your server? [/lxr] > /lxr
Will it be shared by all trees? [YES/no] >

*** LXR database configuration ***


The choice of the database engine can make a difference in indexing performance,
but resource consumption is also an important factor.
  * For a small personal project, try SQLite which do not
    need a server and is free from configuration burden.
  * For medium to large projects, choice is between MySQL,
    PostgreSQL and Oracle.
    Oracle is not a free software, its interface has not been
    tested for a long time.
  * PostgreSQL databases are smaller than MySQL's
    and performance is roughly equivalent.
  * MySQL is at its best with large-sized projects
    (such as kernel cross-referencing) where it is fastest at the cost
    of bigger databases.
  * Take also in consideration the number of connected users.
Database engine? [MYSQL/oracle/postgres/sqlite] >
The safest option is to create one database per tree.
You can however create a single database for all your trees with a specific set of
tables for each tree (though this is not recommended).
How do you setup the databases? [PER TREE/global] >
All databases can be accessed with the same username and
can also be described under the same names.
Will you share database characteristics? [YES/no] >
Will you use the same username and password for all DBs? [YES/no] >
--- DB user name? [lxr] > lxr
--- DB password ? [lxrpw] > lxrpw
Will you give the same prefix to all tables? [YES/no] >
--- Common table prefix? [lxr_] >
--- Directory for glimpse databases? > /opt/lxr/glimpse_db

file .htaccess written into LXR root directory
file apache2-require.pl written into configuration directory
file apache-lxrserver.conf written into configuration directory
file lighttpd-lxrserver.conf written into configuration directory
file nginx-lxrserver.conf written into configuration directory
file thttpd-lxrserver.conf written into configuration directory
Mercurial support files written into configuration directory

*** LXR master configuration file setup ***
    Global section part

*** Configuring auxiliary tool paths
*** Host name previously defined as http://104.154.246.9
*** Configuring HTML parameters
*** 'Buttons-and-menus' interface is recommended for the kernel
*** to avoid screen cluttering.
--- Use 'buttons-and-menus' instead of 'link' interface? [YES/no] >
*** Configuring file subsection
*** Configuring "common factors"
*** Marking tree section

*** LXR master configuration file setup ***
    Tree section part
    SQL script for database initialisation

*** Configuring LXR server parameters
*** The virtual root is the fixed URL part after the hostname.
*** You previously defined the virtual root as /lxr
--- Caption in page header? (e.g. Project XYZZY displayed by LXR) > drbd
Do you want a speed switch button for this tree ? [YES/no] >
--- Short title for button? (e.g. XYZZY) > drbd
--- Tree identification in URL? (e.g. the-tree) > drbd
Do you need a specific encoding for this tree ? [yes/NO] >
*** Describing tree location
How is your tree stored? [FILES/cvs/git/svn/hg/bk] >
*** A source directory contains one sub-directory for every version.
--- Source directory? (e.g. /home/myself/project-tree) > /opt/lxr/source_code/drbd
Name to display for the path root? (e.g. Project or $v for version) [$v] >
*** Enumerating versions
Label for version selection menu?  [Version] >
*** Versions can be explicitly enumerated, be read from a file or computed
*** by a function. The latter case is recommended for VCS-stored trees.
Version enumeration method? [LIST/file/function] >
--- Version name?  >
No default choice, try again...
--- Version name?  > 0.9
--- Version name? (hit return to stop) >
*** By default, first version in list is displayed. You may also indicate
*** a prefered version.
--- Default displayed version is first in 'range'? [YES/no] >
*** Setting directory lists
*** Some directories may contain non-public project data (binaries,
*** compilers caches, SCM control data, ...). They can be hidden from LXR.
--- Directory to ignore, e.g. CVSROOT or CVS? (hit return to stop) >
*** If your source code uses "include" statements (#include, require, ...)
*** LXR needs hints to resolve the destination file.
--- Include directory, e.g. /include? (hit return to stop) >
*** Configuring data storage
--- Database name? > drbd
Do you want to override the global 'lxr' user name? [yes/NO] >
Do you want to override the global 'lxr_' table prefix? [yes/NO] >

*** Configure another tree? [YES/no] >
        , 'shortcaption' => 'drbd'
*** Configuring LXR server parameters
*** The virtual root is the fixed URL part after the hostname.
*** You previously defined the virtual root as /lxr
--- Caption in page header? (e.g. Project XYZZY displayed by LXR) > Ceph
Do you want a speed switch button for this tree ? [YES/no] >
--- Short title for button? (e.g. XYZZY) > Ceph
--- Tree identification in URL? (e.g. the-tree) > Ceph
Do you need a specific encoding for this tree ? [yes/NO] >
*** Describing tree location
How is your tree stored? [FILES/cvs/git/svn/hg/bk] >
*** A source directory contains one sub-directory for every version.
--- Source directory? (e.g. /home/myself/project-tree) > /opt/lxr/source_code/ceph/
Name to display for the path root? (e.g. Project or $v for version) [$v] >
*** Enumerating versions
Label for version selection menu?  [Version] >
*** Versions can be explicitly enumerated, be read from a file or computed
*** by a function. The latter case is recommended for VCS-stored trees.
Version enumeration method? [LIST/file/function] >
--- Version name?  > master
--- Version name? (hit return to stop) >
*** By default, first version in list is displayed. You may also indicate
*** a prefered version.
--- Default displayed version is first in 'range'? [YES/no] >
*** Setting directory lists
*** Some directories may contain non-public project data (binaries,
*** compilers caches, SCM control data, ...). They can be hidden from LXR.
--- Directory to ignore, e.g. CVSROOT or CVS? (hit return to stop) >
*** If your source code uses "include" statements (#include, require, ...)
*** LXR needs hints to resolve the destination file.
--- Include directory, e.g. /include? (hit return to stop) >
*** Configuring data storage
--- Database name? > ceph
Do you want to override the global 'lxr' user name? [yes/NO] >
Do you want to override the global 'lxr_' table prefix? [yes/NO] >

*** Configure another tree? [YES/no] > no
```

4. Initail DB via tool initdb.sh

```
./custom.d/initdb.sh
```

5. Copy the lxr.conf from custom.d dir to root dir.

```
cp ./custom.d/lxr.conf .
```

Generate Reference
==================
- Generate the reference of project `ceph`

```
./genxref --url=http://localhost/lxr --tree=Ceph --version=master
```

- Generate the reference of project `drbd`

```
./genxref --url=http://localhost/lxr --tree=drbd --version=9.0
```

Setup WEB Server
================
1. Copy the server config to apache2 configuration dir.

```
cp apache-lxrserver.conf  /etc/apache2/conf-available
a2enconf apache-lxrserver.conf
```

2. Start the apache2

```
service apaceh2 start
```

Test
====
1. Go to **http://localhost/lxr** and your see there're two options there, ceph and drbd.
2. choose any one of them and you can use that to help you trace the code now.

Reference
=========
1. [LXR] (http://lxr.sourceforge.net/en/1-0-InstallSteps/1-0-install1tools.php)
