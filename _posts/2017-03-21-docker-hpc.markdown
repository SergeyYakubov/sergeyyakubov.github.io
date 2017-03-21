---
layout: post
title:  "Using Docker containers in HPC environments"
date:   2017-03-21 10:34:58 +0100
categories: HPC Docker
style: "text-align:center"
---

Using Docker in HPC systems is an appealing idea, at least from the user perspective. No more need to spend hours trying to compile your application and all third-party libraries it needs (you cannot use packages since you have no admin rights there) on a new HPC cluster you've got an access to. No worries that they'll update something and your application suddenly stops working. And what about remembering to sync your source changes on all systems you are working with? Docker helps to solve all these problems at a probable price of slight performance loss (due to dockerized MPI library, system specific compilers, etc.).

From sysadmin's point of view it requires some efforts while there is no out-of-the box solution for Docker on HPC yet, although there is some work in this direction ([shifter][shifter], [singularity][singularity]).

 I'll describe how to set-up native Docker for HPC. The idea is to create for each job a virtual HPC cluster of Docker containers. It has to be secure (no root access), utilize high-speed network and parallel file system. The job should be submittted using existing resource management system and the process of job submission should not require too much input from user.


Actually, the most difficult part is to make sure that users cannot escalate their privileges when running Docker containers. Since I've described it in a previous [post][docker-noroot], we assume here that Docker Engine and the authorization plugin are running on each node of HPC system. After this, we need the following for our virtual cluster:


1. A Docker image with ssh daemon and HPC stuff installed
2. Start an sshd daemon in Docker container on each of the allocated nodes
3. Start an HPC application

### A Docker image with ssh daemon and HPC stuff installed

Apart from your HPC application, Docker image should have the ssh server and client installed, necessary communication drivers (infiniband in our case) and an MPI library.

{% highlight docker %}

FROM centos:7

# install ssh
# Docker needs pam_loginuid to be optional
RUN yum install -y openssh-clients openssh-server && ssh-keygen -A && \
        sed -i 's/required\(.*pam_loginuid\)/optional\1/' /etc/pam.d/sshd


# install infiniband
RUN yum install -y ibibverbs-utils libibverbs-devel libibverbs-devel-static libmlx4 \
        libmlx5 ibutils libibcm libibcommon libibmad libibumad rdma  librdmacm-utils \
        librdmacm-devel librdmacm libibumad-devel perftest

# install mpi
RUN yum install -y openmpi openmpi-devel make

# add mpi to path
ENV PATH=/usr/lib64/openmpi/bin:$PATH
ENV LD_LIBRARY_PATH=/usr/lib64/openmpi/lib:$LD_LIBRARY_PATH
RUN echo "export PATH=/usr/lib64/openmpi/bin:${PATH}" > /etc/profile.d/scripts-path.sh && \
echo "export LD_LIBRARY_PATH=/usr/lib64/openmpi/lib:$LD_LIBRARY_PATH" >> /etc/profile.d/scripts-path.sh
 && \
chmod 755 /etc/profile.d/scripts-path.sh

#....

{% endhighlight %}

Alternatively, you can just start yout image from `yakser/centos_mpi`.


### Start an sshd daemon in Docker container on each of the allocated nodes

Now when we have a Docker image with ssh, we need to start it on every allocated compute node.

At first, how to start ssh without root privileges in Docker container. This can be done in [entrypoint script][sshentry]. Important is to create key files on host so that all containers in virtual cluster get the same key files and could communicate with each other. Entrypoint script assumes that keys are mounted under */ssh_keys*. Then we create home directory, copy key files there, create config file for sshd and start it with this config file. `$DOCKER_SSHPORT` sets the port number to listen.


With this entrypoint Docker containers have to be started on the nodes. For SLURM I've created a [script][dockercluster] doing this. It reads user arguments, creates key files, calls another [script][hostfile] to create hostfile for MPI calls inside Docker and then uses mpirun on host to start Docker containers. The mpirun calls a script from [previous post][docker-noroot] with additional parameter to mount directory with ssh keys, hostfile and set port for sshd daemon. Note that the script will overwrite the original entrypoint stored in image file. It it is not desirable, you should change this behaviour (e.g. put ssh stuff inside an image, or add your entrypoint stuff into ssh_entrypoint script).

### Start an HPC application

Now we have virtual cluster running and awaiting MPI commands. By default container gets name `docker_$SLURM_JOB_ID`. So we can now just call docker exec with for this name and correct user id (otherwise authorization plugin will complain). A simple [helper script][dockerexec] is available.

### Example workflow

With everything set-up, it is now quite easy to use. Below is an example script for a SLURM batch job.

{% highlight bash %}

#!/bin/bash
#SBATCH --ntasks=64
#SBATCH --cpus-per-task=1
#SBATCH --time=00:01:00                  # Maximum time request
#SBATCH --partition=test


# start-up docker cluster, we use -u to pull new image from repository if necessary
dockercluster -u yakser/centos_mpi


# run an mpi command in docker cluster
dockerexec mpirun -np 64 hostname

{% endhighlight %}

[dockerrun]:https://github.com/SergeyYakubov/docker/tree/master/scripts
[docker-noroot]:https://sergeyyakubov.github.io/hpc/docker/2017/03/13/docker-noroot.html
[shifter]:http://www.nersc.gov/research-and-development/user-defined-images/
[dockercluster]:https://raw.githubusercontent.com/SergeyYakubov/docker/master/scripts/wrappers/dockercluster
[sshentry]:https://raw.githubusercontent.com/SergeyYakubov/docker/master/config/etc/docker/entrypoints/startsshd.sh
[sshnoroot]:https://cygwin.com/ml/cygwin/2008-04/msg00363.html
[hostfile]:https://raw.githubusercontent.com/SergeyYakubov/docker/master/scripts/slurm/slurm_make_hostfile
[dockerexec]:https://raw.githubusercontent.com/SergeyYakubov/docker/master/scripts/wrappers/dockerexec
[singularity]:http://singularity.lbl.gov/