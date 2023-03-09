# Container Options

# Docker

## icommands (with a note on running on a mac)

Taken from the [iRODS-Chat](https://groups.google.com/d/msgid/irod-chat/ca981c1b-a09c-43a2-b654-6aab3abd8218n%40googlegroups.com?utm_medium=email&utm_source=footer) mailing list, authored by Terrell Russell from RENCI.

### start a container

note the `platform` flag if you are on a Mac!

```
docker run -d --platform linux/amd64 --name ub20icommands ubuntu:20.04 tail -f /dev/null
docker exec -it ub20icommands /bin/bash
```

then within the container, install the iCommands

```
apt-get update
apt-get install -y wget sudo lsb-release gnupg
wget -qO - https://packages.irods.org/irods-signing-key.asc | sudo apt-key add -
echo "deb [arch=amd64] https://packages.irods.org/apt/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/renci-irods.list
apt-get update
apt-get install -y irods-icommands
```

then back on the host

```
docker exec -it ub20icommands iinit
```

then the rest of the iCommands are available

```
$ docker exec -it ub20icommands ils
/tempZone/home/rods:
```

Note... putting and getting files are done from 'within' the container... so if you need to move data, you'll probably want to use a volume mount or `docker cp` to share between host and container.

```
$ docker cp filetoput.txt ub20icommands:/
$ docker exec -it ub20icommands iput filetoput.txt foo
$ docker exec -it ub20icommands ils
/tempZone/home/rods:
  foo
```




# Apptainer

## icommands

```
# icommands.def
# https://github.com/ll4strw
# Modify appropriately and
# Build with
# sudo singularity build  icommands.sif icommands.def

Bootstrap: docker
From: centos:centos7

%post
    yum upgrade -y
    yum install -y wget epel-release
    yum install -y python36-jsonschema python36-psutil
    rpm --import https://packages.irods.org/irods-signing-key.asc
    wget -qO - https://packages.irods.org/renci-irods.yum.repo | tee /etc/yum.repos.d/renci-irods.yum.repo
    yum install -y irods-icommands.x86_64

%environment
    export LC_ALL=C


%runscript
    exec "$@"

```

To deploy the icommands container to an HPC environment that uses [EasyBuild](https://easybuild.io/) to deliver software modules create the following module template in an appropriate location on your system

```
-- cat icommands/4.3.0.lua

help([==[

Description
===========
iRODS icommands v4.3.0.


More information
================
 - Homepage: https://github.com/irods/irods_client_icommands
]==])


conflict("icommands")

```

then complete the template executing

```
/path/to/icommands.sif ihelp | awk '$2=="-"{print "set_alias(\""$1"\",\"/path/to/icommands.sif "$1"\")"}' >> icommands/4.3.0.lua
```




### Useful links

* [easybuild](https://easybuild.io/)
    EasyBuild is a software build and installation framework that allows you to manage (scientific) software on High Performance Computing (HPC) systems in an efficient way.
* [Lmod](https://lmod.readthedocs.io/en/latest/190_Integration_of_EasyBuild_and_Lmod.html);
    Lmod is a Lua based module system that easily handles the MODULEPATH Hierarchical problem. Environment Modules provide a convenient way to dynamically change the users’ environment through modulefiles. This includes easily adding or removing directories to the PATH environment variable. Modulefiles for Library packages provide environment variables that specify where the library and header files can be found.
