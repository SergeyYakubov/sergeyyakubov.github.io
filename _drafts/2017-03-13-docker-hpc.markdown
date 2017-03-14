---
layout: post
title:  "Using Docker containers in HPC environments"
date:   2017-03-14 10:34:58 +0100
categories: HPC Docker
style: "text-align:center"
---

Using Docker in HPC systems is an appealing idea, at least from the user perspective. No more need to spend hours trying to compile your application and all third-party libraries it needs (you cannot use packages since you have no admin rights there) on a new HPC cluster you've got an access to. No worries that they'll update something and your application suddenly stops working. And what about remembering to sync your source changes on all systems you are working with? Docker helps to solve all these problems at a probable price of slight performance loss (dockerized mpi library, system specific compilers, etc.).

From sysadmin's point of view it requires some efforts while there is npo out-of-the box solution for Docker on HPC yet, although there is some work in this [direction][shifter].

 I'll describe how to set-up Docker for HPC. The idea is to create for each job a virtual HPC cluster of Docker containers. It has to be secure (no root access), utilize high-speed network and parallel file system. The job should be submittted using existing resource management system and the process of job submission should not require too much input from user.


Actually, the most difficult part is to make sure that users cannot escalate their privileges when running Docker containers. Since I've described it in a previous [post][docker-noroot] we assume here that Docker Engine and the authorization plugin are running on each node of HPC system. After this, we need the following for our virtual cluster:


1. A Docker image with ssh daemon
2. Start an sshd daemon in Docker container on each of the allocated nodes
3. Prepare a hostfile for MPI library
4. Start an HPC application

### Make Docker communicate via an HTTP socket


This is relatively easy. One have to switch Docker from unix socket to a tcp socket and create TLS certificates for every user. This allows us to indentificate user how executes Docker commands (which is impossible when using a unix socket)


You can read how to do it in [Docker documentation] [ssl_certificate] or have a look at [my scripts][docker_scripts] that automate the work to create [CA][CA], [server certificate][server_cert] and [user certificate][user_cert]. After this you have to make sure Docker daemon is started with `-H=0.0.0.0:2376 --tlsverify --tlscacert=<PATH TO>/ca.pem --tlscert=<PATH TO>/server-cert.pem --tlskey=<PATH TO>/server-key.pem` [custom options][docker_custom_options] and set `DOCKER_TLS_VERIFY=1, DOCKER_HOST=localhost:2376, DOCKER_CERT_PATH=<PATH TO USER CERTIFICATES>` environment variables for every user that will run Docker commands. The last one can be omited if you place certificates in ~/.docker (I do it via a [script][profiled] in /etc/profile.d).

### An authorization plugin to control user input

An [authorization plugin][auth_plugin] allows to check any command to Docker daemon before its execution. One cannot change the command, just allow or deny it. When Docker uses HTTP socket for communications, authorization plugin knows also the name of the user who executes the command.

Some Go programming is necessary for the plugin. The one I've developed does the following:

* User root can execute any commands
* If user namespace is not disabled, any command can be executed
* If user namespace is disabled and user is not root:
    * If it is not run/exec/create command allow it (pull,push, ...)
    * For run command
        * make sure user and group arguments are there and they correspond to our user
        * make sure no new privileges flag is there (--security-opt=no-new-privileges)
    * For exec command
        * make sure user argument correspond to our user and contaner was started by this user

You can have a look at the plugin [here][myauth_plugin]. Note that this was my first Docker program, so it make look a bit messy :).

### Propagate user name and groups into a Docker container

So what we have up to now is that any user is allowed to execute Docker commands. The authorization plugin checks each user command and makes sure that whether user namespaces are used for this command or no-new-privileges flag is there and user in Docker command corresponds to the user in the TLS certificate. Which means that we force user to use Docker run like

{% highlight bash %}
docker run -u `id -u` -g `id -g` --userns=host --security-opt no-new-privileges --group-add <extra groups> ...
{% endhighlight %}

With the above command all processes running inside a Docker container will have correct owner and no privilege escalation is allowed. Note that there is actually no new user created in a container. In some cases this is not enough and application wants to see the real user (eg. sshd will not run). In this case one can pass user information from host to the container. The simplest way is to mount /etc/passwd and /etc/group from host and then replace or merge with corresponding files inside a container. A home directory can then be created via entrypoint script.

I've written a small [script][dockerrun] that hides all this stuff from user. So a user can just type
{% highlight bash %}
dockerrun centos:7 id
{% endhighlight %}

and feel himself as on being the host :). The script does a bit more to prepare for running job on HPC, which I'll describe soon in other post.


[dockerrun]:https://github.com/SergeyYakubov/docker/tree/master/scripts
[docker-noroot]:https://sergeyyakubov.github.io/hpc/docker/2017/03/13/docker-noroot.html
[shifter]:http://www.nersc.gov/research-and-development/user-defined-images/
