---
layout: post
title:  "How to safely let \"normal user\" to run Docker containers"
date:   2017-02-02 15:34:58 +0100
categories: HPC Docker
style: "text-align:center"
---

Docker containers are nowadays used for many different tasks like software development and testing, service orchestration, software deployment. The idea that one can prepare an image with everything he needs on his own laptop and then make it run everywhere else in a matter of seconds is very attractive.

There is one problem with this approach - how to allow users run their own container in some infrastructure where they have no root rights? As we [know][docker_root], as soon as you allow a user to run Docker commands he can do bad things to your system. Since Docker has been developed with microservices in mind, it is supposed that only a trusted user (read root) should execute Docker commands. For example, you  as admin start a service which is running inside a Docker container and user (or other microservices) access this container via e.g. REST API.

What we want to do is different - we want that a user could bring an image with his scientific (or whatever else) application, mount data he needs from the host filesystem and run it. We also do not want to write any additional software around Docker but be able to use native Docker commands (well, we've still created a couple of helper scripts). The activation of user namespaces could partially solve the problem, but one can always disable it by *--userns=host* and using mounted host data with user namespaces may be problematic due to file permissions.


So, we what we do is:

1. Make Docker communicate via an HTTP socket
2. An authorization plugin to control user input
3. A helper to propagate user and his groups into Docker container

### Make Docker communicate via an HTTP socket


This is relatively easy. One have to switch Docker from unix socket to a tcp socket and create TLS certificates for every user. This allows us to indentificate user how executes Docker commands (which is impossible when using a unix socket)


You can read how to do it in [Docker documentation] [ssl_certificate] or have a look at [my scripts][docker_scripts] that automate the work to create [CA][CA], [server certificate][server_cert] and [user certificate][user_cert]. After this you have to make sure Docker daemon is started with `-H=0.0.0.0:2376 --tlsverify --tlscacert=<PATH TO>/ca.pem --tlscert=<PATH TO>/server-cert.pem --tlskey=<PATH TO>/server-key.pem` [custom options][docker_custom_options] and set `DOCKER_TLS_VERIFY=1, DOCKER_HOST=localhost:2376, DOCKER_CERT_PATH=<PATH TO USER CERTIFICATES>` environment variables for every user that will run Docker commands. The last one can be omited if you place certificates in ~/.docker (we do it via a [script][profiled] in /etc/profile.d).

### An authorization plugin to control user input

An [authorization plugin][auth_plugin] allows to check any command to Docker daemon before its execution. One cannot change the command, just allow or deny it. When Docker uses HTTP socket for communications, authorization plugin knows also the name of the user who executes the command.

Some Go programming is necessary for the plugin. The one we've developed does the following:

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

[myauth_plugin]:https://github.com/SergeyYakubov/docker/tree/master/plugins/docker-auth-plugin
[docker_root]:https://reventlov.com/advisories/using-the-docker-command-to-root-the-host
[profiled]:https://raw.githubusercontent.com/SergeyYakubov/docker/master/scripts/config/etc/profile.d/docker.sh
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
