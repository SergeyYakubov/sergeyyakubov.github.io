---
layout: post
title:  "How to allow non-privileged users run Docker containers"
date:   2017-03-13 10:34:58 +0100
categories: HPC Docker
style: "text-align:center"
comments: true
---

Docker containers are nowadays used for many different tasks like software development and testing, service orchestration, software deployment. The idea that one can prepare an image with everything he needs on his own laptop and then make it run everywhere else in a matter of seconds is very attractive.

There is one problem with this approach - how to allow users run their own container in some infrastructure where they have no root rights? As we [know][docker_root], as soon as you allow a user to run Docker commands he can do bad things to your system. Since Docker has been developed with microservices in mind, it is supposed that only a trusted user (read root) should execute Docker commands. For example, you  as admin start a service which is running inside a Docker container and user (or other microservices) access this container via e.g. REST API.

What I want to do is different - I want that a user could bring an image with his scientific (or whatever else) application, mount data he needs from the host filesystem and run it. I also do not want to write any additional software around Docker but be able to use native Docker commands (well, I've still created a couple of helper scripts). The activation of user namespaces could partially solve the problem, but one can always disable it by *--userns=host* and using mounted host data with user namespaces may be problematic due to file permissions.


So, what I do is:

1. Make Docker communicate via an HTTP socket
2. An authorization plugin to control user input
3. A helper to propagate user and his groups into Docker container

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

You can have a look at the plugin [here][myauth_plugin]. Note that this was my first Go program, so it make look a bit messy :).

### Propagate user name and groups into a Docker container

So what we have up to now is that any user is allowed to execute Docker commands. The authorization plugin checks each user command and makes sure that whether user namespaces are used for this command or no-new-privileges flag is there and user in Docker command corresponds to the user in the TLS certificate. Which means that we force user to use Docker run like

{% highlight bash %}
docker run -u `id -u` -g `id -g` --userns=host --security-opt no-new-privileges --group-add <extra groups> ...
{% endhighlight %}

With the above command all processes running inside a Docker container will have correct owner and no privilege escalation is allowed. Note that there is actually no new user created in a container. In some cases this is not enough and application wants to see the real user (eg. sshd will not run). In this case one can pass user information from host to the container. The simplest way is to mount /etc/passwd and /etc/group from host and then replace or merge with corresponding files inside a container. A home directory can then be created via entrypoint script or mounted during the Docker run command.

I've written a small [script][dockerrun] that hides all this stuff from user. So a user can just type
{% highlight bash %}
dockerrun centos:7 id
{% endhighlight %}

and feel himself as on being the host :). The script does a bit more to prepare for running job on HPC, which I'll describe soon in other post.


[dockerrun]:https://github.com/SergeyYakubov/docker/tree/master/scripts/wrappers/dockerrun
[myauth_plugin]:https://github.com/SergeyYakubov/docker/tree/master/plugins/docker-auth-plugin
[docker_root]:https://reventlov.com/advisories/using-the-docker-command-to-root-the-host
[profiled]:https://raw.githubusercontent.com/SergeyYakubov/docker/master/config/etc/profile.d/docker.sh
[docker_scripts]:https://github.com/SergeyYakubov/docker/tree/master/scripts/certs
[docker_custom_options]: https://docs.docker.com/engine/admin/systemd/#/custom-docker-daemon-options
[CA]: https://raw.githubusercontent.com/SergeyYakubov/docker/master/scripts/certs/create_ca.sh
[server_cert]: https://raw.githubusercontent.com/SergeyYakubov/docker/master/scripts/certs/create_server_cert.sh
[user_cert]: https://raw.githubusercontent.com/SergeyYakubov/docker/master/scripts/certs/create_client_cert.sh
[auth_plugin]: https://docs.docker.com/engine/extend/plugins_authorization/
[ssl_certificate]: https://docs.docker.com/engine/security/https/
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

<!---
{% highlight go %}
func print(){
fmt.Println("Hi")
}
// prints 'Hi, Tom' to STDOUT.
{% endhighlight %}
-->
