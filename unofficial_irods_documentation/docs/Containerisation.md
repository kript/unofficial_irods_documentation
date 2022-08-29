# Container Options

# Singularity

## icommands

Contributed by [leonardo](https://github.com/ll4strw);

```
# The simplest singularity def file to get you started. Customise appropriately
# cat icommands.def
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

# Build with
# sudo singularity build  icommands.sif icommands.def
```

An example to deploy to an HPC environment that uses [Modules](http://modules.sourceforge.net/) with;

```
help([==[

Description
===========
iRODS icommands v4.3.0.


More information
================
 - Homepage: https://github.com/irods/irods_client_icommands
]==])


conflict("icommands")
set_alias("iinit","/path/to/icommands.sif iinit")
set_alias("ils","/path/to/icommands.sif ils")
# add other icommands in a similar fashion. This can be done automatically via
ihelp | awk '$2=="-"{print "set_alias(\""$1"\",\"/path/to/icommands.sif "$1"\")"}' >> icommands/4.3.0.lua

# Place icommands/4.3.0.lua in an appropriate location on your system
```

Some links fro the above;

* [easybuild](https://easybuild.io/)
    EasyBuild is a software build and installation framework that allows you to manage (scientific) software on High Performance Computing (HPC) systems in an efficient way.
* [Lmod](https://lmod.readthedocs.io/en/latest/190_Integration_of_EasyBuild_and_Lmod.html);
    Lmod is a Lua based module system that easily handles the MODULEPATH Hierarchical problem. Environment Modules provide a convenient way to dynamically change the users’ environment through modulefiles. This includes easily adding or removing directories to the PATH environment variable. Modulefiles for Library packages provide environment variables that specify where the library and header files can be found.
