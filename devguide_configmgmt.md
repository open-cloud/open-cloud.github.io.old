---
layout: page
title: Configuration Management
permalink: /devguide/configmgmt/
---

{% include toc.html %}

The XOS distribution includes a collection of configurations, each of
which implies three main parameters:

 1. XOS Release
 2. Target Hardware
 3. Service Portfolio

For example, the *Devel Config* described below uses the latest stable release
(Burwell), runs on [CloudLab](https://www.cloudlab.us/), and includes
only the HelloWorld service.

Configurations are organized in the directory *xos/configurations*
within the XOS git repository. Each configuration is stored in a
single subdirectory.  For example, the Devel configuration can be
found in *xos/configurations/devel*.  At a minimum, each configuration
consists of a Makefile and a Dockerfile. Optionally, there may also be
TOSCA definitions that are used to internally configure XOS (e.g., to
bring up services and slices). A second Makefile, *Makefile.inside*, is
commonly used to execute actions that need to occur after the
container has been started.

Directory *xos/configurations/common* contains files that are useful
to multiple configurations. *Dockerfile.common* contains a baseline
Dockerfile that should be suitable for most XOS installations. By
convention, common Dockerfile actions are abstracted to
*xos/configurations/common/Dockerfile.common*, and only Dockerfile
actions unique to a particular configuration need be specified in the
individual configuration's Dockerfile.

To create a new configuration, first make a new subdirectory off of
*xos/configurations*. Then create *Dockerfile.configname* and include
any Docker actions unique to that configuration. Use one of the other
Makefiles as a template (*xos/configurations/devel/Makefile* is
generally a good starting point) and modify it as appropriate to
create the configuration's Makefile.

The final step is to optionally modify the XOS data model, for
example, to include additional services, slices, deployments, and so
on. This is done by executing one or more TOSCA files that specify the
model to be imported into XOS. More information on TOSCA can be found
elsewhere.

By convention, XOS configurations are initialized with a single
administrative user and login credentials of
*username=padmin@vicci.org* with password *letmein*.

We suggest placing a *README* file with each configuration that
documents the purpose of the configurations and any assumptions or
requirements, such as the whether the configuration must be run from
within CloudLab.

The rest of this section describes four stock configurations that are
provided with the release. It also includes a description of the
configuration we use for OpenCloud, a production system.

## Devel Config

A simple way to create an end-to-end development environment is to use
[CloudLab](https://www.cloudlab.us/) to host a basic OpenStack
cluster, and then link this cluster to XOS. To set up XOS with an OpenStack cluster hosted on CloudLab:

* If you don't already have a CloudLab account, go to `http://cloudlab.us` and join project **cord-testdrive**.
* Create your CloudLab experiment using the *OpenStack* profile.  The Juno and Kilo releases should both work.
We recommend that, under "Advanced Parameters" in the profile, you choose to "Disable Security Group Enforcement".  Instantiate it on the *CloudLab Clemson* or *CloudLab Wisconsin* clusters.
* Login to the *ctl* node of the experiment and run the following:

{% highlight sh %}
$ git clone https://github.com/open-cloud/xos.git
$ cd xos/xos/configurations/devel
$ make
{% endhighlight %}

The Makefile will build the XOS Docker image, run it in a container,
and configure XOS to talk to the OpenStack cluster on CloudLab.  You can
reach the XOS GUI on port 9999 on the *ctl* node.  XOS login credentials are
*padmin@vicci.org*/*letmein*.

Assuming everything has worked, you should be able to create a slice and
launch a VM.  You can log into the VM from the *ctl* node as the *ubuntu* user,
at the first IP address shown in the *Instances* view for the slice (typically on the
10.11.0.0/16 subnet).

## CORD Config

The CORD configuration of XOS is a variant of the Devel configuration described above.
The purpose of this configuration is to enable development of the vCPE service
and related CORD services on CloudLab.  XOS is configured with the vCPE, vOLT,
and vBNG services.  For the latest instructions on how to build this configuration,
see the [README.md](https://github.com/open-cloud/xos/blob/master/xos/configurations/cord/README.md)
file in the `xos/configurations/cord/` directory of the XOS GitHub repository.

## Test Config

The Test configuration brings XOS up on CloudLab and runs it through
a set of regression tests. All code that is to be checked into the
master branch on github should first successfully pass all the test
cases in this configuration. After the test suite is completed, the
container will automatically exit.

Generally the test suite executes in two phases. The first phase are
simple data model regression tests driven by XOS's TOSCA engine. These
test case create models with various sets of arguments, and ensure that
the XOS data model is behaving properly.

The second phase are Synchronizer-exercising tests. These test cases create
an object in the data model, and then run multiple passes of the XOS
synchronizer to instantiate the object using OpenStack.

## Bash Config

The Bash configuration may be found in *xos/configurations/bash*. Its
purpose is to serve as an interactive environment for development,
with a shell-based interface. After the Makefile has finished
executing, the user will be dropped into a bash shell inside of the
container. Postgres will be running and the XOS data model will be
populated with minimal data.

The XOS UI will not be running, but it can be started by typing

{% highlight sh %}
$ cd /opt/xos; scripts/opencloud runserver
{% endhighlight %}

## Frontend-Only Config

The Frontend-Only configuration is aimed at developers who are working
on the XOS user interface, but do not need a functioning
Synchronizer. As such, it does not require Cloudlab or any active
OpenStack deployment. While the UI is functional, this configuration
necessarily imposes the limitation that Instances will not be
instantiated.

Additionally, as the Synchronizer is not running, *model_policies*
will not be executed.

## OpenCloud Config

The preceeding configurations are primarily used for development.
This section describes the configuration used on OpenCloud, which is
an operational system that runs 24/7 and supports end-users.

Just like the other examples, the OpenCloud configuation is defined by
a Dockerfile that sets up the underlying environment. One major
difference from the development versions is that for production
environments, we recommend running XOS behind a front-end such as
nginx.

A sample configuration file for nginx is located in the nginx
subdirectory of the XOS git repository. This config fie is setup to
look for static files in */var/www/xos/static*, and that subdirectory
must be created. All static files located in the following
subdirectories must be copied to */var/www/xos/static/:*

{% highlight sh %}
/opt/xos/core/static
/opt/xos/core/xoslib/static
# note that the following two paths may vary depending on Linux distribution
/usr/local/lib/python2.7/dist-packages/Django-1.7-py2.7.egg/django/contrib/admin/static
/usr/lib/python2.7/site-packages/suit/static
{% endhighlight %}

The following commands can be used to start, stop, and restart the
uwsgi server:

{% highlight sh %}
start: cd /opt/xos; uwsgi --start-unsubscribed /opt/xos/uwsgi/xos.ini
stop: uwsgi --stop /var/run/uwsgi/uwsgi.pid
restart: uwsgi --reload /var/run/uwsgi/uwsgi.pid
{% endhighlight %}

## Debugging Configurations

There are two different kinds of configurations: terminal interactive
configurations and background configurations. Terminal interactive
configurations print output to stdout and accept input from
stdin. Examples of these configurations are Test and Bash. Debugging
these are relatively easy, as one may directly observe the output of
the container.

Background configurations do not produce output once launched.
Examples of these include the Devel and Frontend configurations. The
Makefiles for these configurations typically include a step that waits
for the XOS UI to become reachable, as it may take up to 30 seconds
for it to do so. After XOS is reachable, a background configuration's
Makefile returns to the command line, and the container continues to
execute.

Because of the nature of background configurations, failures are not
necessarily readily apparent. Docker includes a 'log' feature that may
be used to display the stdout and stderr of a background
container. This feature is exercised by first looking up the
container's ID (with `docker ps` if the container is still executing,
or `docker ps -a` if the container has exited). Then use that ID to
execute `docker logs ID`.  Several of the configurations include a
Makefile target `make showlogs` that automatically executes these
steps.

Additionally, it may be necessary for a developer to sometimes attach
a shell to a running background container to interact with it. This
may be done first looking up the container ID, and then executing
`docker exec -t -i ID bash`.

## Building XOS Without the Configuration System

A Dockerfile available at
[github.com/open-cloud/xos](https://github.com/open-cloud/xos)
can be used to build a Docker image for running XOS. The XOS
files in the Docker image are copied from the local file tree,
so it is easy to create a customized version of XOS by making
local changes to the XOS source before building the Docker image.

A minimal *initial_data.json* fixture is provided. The login
credentials are *username=padmin@vicci.org* with password
*letmein*. This *initial_data.json* doesn't contain any nodes and is
suitable for fresh installations.

To build and start the container type:

{% highlight sh %}
$ docker build -t xos .
$ docker run -t -i -p 8000:8000 xos
{% endhighlight %}

XOS will start automatically and you will see its log output in the shell window.
You can access the XOS login at *http://server:8000*, where *server*
is the name of the server hosting the Docker container.  Login credentials are
*padmin@vicci.org*/*letmein*.

Note that the above steps result in a running XOS, but without any
backend resources. This is sufficient for working on the data model
and views, but not for actually managing cloud infrastructure.

Information on bringing up a local OpenStack cluster is given in Section
[Installing OpenStack](/opsguide/#installing-openstack) of the
Operator Guide.  Information on connecting XOS to an operational
OpenStack cluster is given in Sections [Administering a
Deployment](/userguide/#administering-a-deployment) and [Administering a
Site](/userguide/#administering-a-site) of the User's Guide. These two sections
explain how to configure a Deployment to know about a set of OpenStack
clusters and how to configure a Site to know about a set of Nodes,
respectively.

