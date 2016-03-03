---
layout: page
title: ExampleService Tutorial
permalink: /devguide/exampleservice/
---

{% include toc.html %}

## Summary

This tutorial shows the steps to [Add a Service](/devguide/addservice/) that
uses [Ansible](https://www.ansible.com/) to XOS.

The ExampleService can be used to create a Tenant, which is a blueprint used to
deploy a virtual machine Instance, which runs the Apache web server. In this
example, the web server is configured to serve a message that can be changed
from the XOS administrative web interface.

Services are made up of two major code components:

 - A [Django](https://www.djangoproject.com) data model and administrative
   interface that runs in the XOS core Docker container.

 - A Synchronizer that is used to instantiate and configure the Instance from
   the Tenant blueprint, and is run in it's own Docker container.

For this example, there is just one unique addition to the Service data model:

 - `display_message`, a string that contains the message to display

When a user creates a new Slice and Tenant in the Admin website (constructed
using Django), the data describing the service is stored in a database.

The Synchronizer runs on a recurring basis, obtains the `display_message` (and
additional needed information) from the database, and uses it to run an Ansible
[playbook](http://docs.ansible.com/ansible/playbooks.html) that applies the
configuration to the Instance.

## Prepare your development environment

In order to get started, you need a OpenStack host to deploy XOS on.

This can be [CloudLab](https://cloudlab.us) or a local
[DevStack](http://docs.openstack.org/developer/devstack/) installation, or one
of the other [Configurations](/devguide/configmgmt/) that XOS supports.

Once you have that set up, do the following:

 - Create a working copy of the [XOS
   repository](https://github.com/open-cloud/xos).  You can do this locally, or
   by forking the repo.  You will also need a copy of this on your OpenStack
   host, so either do development on that host or on another system and then
   `git pull` your changes onto the OpenStack host.

 - On the OpenStack host, in your working copy of XOS, go to
   [xos/configurations/devel](https://github.com/open-cloud/xos/tree/master/xos/configurations/devel)
   and run `make`.  This will create an XOS installation on that OpenStack host
   by installing prerequistes and creating and starting the XOS Docker
   containers.

 - Navigate to `http://<ip_or_dns_name_of_host>:9999` and verify that you can
   login with the default XOS credentials, which are username:
   `padmin@vicci.org` and password: `letmein`.

 - *Optional:* Check to make sure you can run the provided `exampleservice`
   code.  This is not enabled by default, and you'll need to follow the steps
   in [Create a Docker container to run your
   synchronizer](#create-a-docker-container-to-run-your-synchronizer) to enable
   it.


### The development loop

Once you've prepared your development environment as described above, the
edit/build/test development loop for service development on XOS is as follows:

 1. Make changes to your copy of XOS and propagate them to your OpenStack host.

 2. On your OpenStack host in XOS's `xos/configurations/devel` directory, run:
    `make rm`, which will stop XOS and delete the containers, which may not
    have your updated changes.

 3. Run `make containers`.  This will rebuild all the Docker containers to
    include your changes.  Initially this will take a long time as everything
    is rebuilt, but will take less time during subsequent runs as Docker saves
    intermediate state.

 3. Run: `make` in the same directory to start XOS running with the newly built
    containers.

 4. Test and verify your changes.

 5. Once you're done testing, go back to step #1.


## Create the Django components of a Service

XOS services are located in the `xos/services` directory in the XOS source
tree.  Create a new directory for your service.

In this example, the service will be named `exampleservice` and the example
code is in the XOS repo at
[xos/services/exampleservice](https://github.com/open-cloud/xos/tree/master/xos/services/exampleservice).

### Create a Django Model

In your service directory, create an empty `__init.py__` file.  This is
required for Python to recognize your service directory as a package in order
to avoid namespace conflicts.

Create a file named
[models.py](https://github.com/open-cloud/xos/tree/master/xos/services/exampleservice/models.py)
in your service directory. The start of this file:

{% highlight python %}
from core.models import Service, TenantWithContainer
from django.db import transaction
{% endhighlight %}

This brings in the two XOS Core Django models that will be extended by our
Service and lets us atomically update the database.

{% highlight python %}
SERVICE_NAME = "exampleservice"
SERVICE_NAME_VERBOSE = "Example Service"
SERVICE_NAME_VERBOSE_PLURAL = "Example Services"
{% endhighlight %}

These are used to uniquely identify and provide human readable names for the
service.  They're used both in this file and when this file is included in
`admin.py`.  For your own service, change these.

#### Extending Service

We extend [XOS Core's
Service](https://github.com/open-cloud/xos/blob/master/xos/core/models/service.py)
class. 

{% highlight python %}
class ExampleService(Service):

    KIND = SERVICE_NAME

    class Meta:
        app_label = SERVICE_NAME
        verbose_name = SERVICE_NAME_VERBOSE
        proxy = True
{% endhighlight %}

XOS uses `KIND` variable to uniquely identify each service, and the `app_label`
and `verbose_name` on the admin website.

`proxy` is used as there isn't any new data in this model, thus it can use it's
super's data model, per the [Django
documentation](https://docs.djangoproject.com/en/1.9/topics/db/models/#proxy-models).  You shouldn't use `proxy` if your Service's model contains

#### Extending TenantWithContainer

We extend [XOS Core's
TenantWithContainer](https://github.com/open-cloud/xos/blob/master/xos/core/models/service.py)
class, which is a Tenant that creates a Container instance, in this case a
virtual machine.

{% highlight python %}
class ExampleServiceTenant(TenantWithContainer):

    KIND = SERVICE_NAME

    class Meta:
        verbose_name = SERVICE_NAME_VERBOSE + " Tenant"
        proxy = True
{% endhighlight %}

Same as in [Extending Service](#extending-service).

{% highlight python %}
    sync_attributes = ("nat_ip", "nat_mac",)
{% endhighlight %}

The Synchronizer requires the IP address the Instance created by the Tenant in
order to run Ansible on the Instance.

{% highlight python %}
    default_attributes = {
            'display_message': "Example Service Text",
            }
{% endhighlight %}

This sets the default text for `display_message`. If you were to add other
variables to the data model, there default values would be set here.

{% highlight python %}
    def __init__(self, *args, **kwargs):
        exampleservice = ExampleService.get_service_objects().all()
        if exampleservice:
            self._meta.get_field('provider_service').default = exampleservice[0].id
        super(ExampleServiceTenant, self).__init__(*args, **kwargs)
{% endhighlight %}


### Create a Django Admin

#### Extend `core.admin.ReadOnlyAwareAdmin` for the service and tenant classes

#### Extends `django.forms.ModelForm` for the service and tenant classes

### Create an HTML template

### Install Service in Django

In order for Django to load your Service, you need to add it to the list of
[INSTALLED_APPS](https://docs.djangoproject.com/en/1.9/ref/settings/#std:setting-INSTALLED_APPS).

This is set in [xos/xos/settings.py](https://github.com/zdw/xos/blob/master/xos/xos/settings.py):

{% highlight python %}
INSTALLED_APPS = (
    ...
    'services.exampleservice',
    ...
)
{% endhighlight %}

Next, so any data migrations get run if the data model of your service changes, you need to tell XOS to run that migration when it comes up.  This is done by adding a line to [xos/tools/xos-manage](https://github.com/open-cloud/xos/blob/master/xos/tools/xos-manage).

{% highlight shell %}
function makemigrations {
    ...
    python ./manage.py makemigrations exampleservice
    ...
}
{% endhighlight %}


### Test your Admin interface



## Create a Synchronizer

Synchronizers are scripts, run in a loop, that check for changes to the Tenant
model and apply them to the running Instances.

### Test your synchronizer

After creating a Service and Tenant in the admin interface, log into the existing `xos_synchronizer_openstack`, by running `make enter-synchronizer`, then navigate to `/opt/xos/synchronizers/exampleservice/`. 

Run `exampleservice-synchronizer.py -C exampleservice_config`

This will perform one run of the synchronizer. 

### Create a Docker container to run your Synchronizer

Synchronizers run in their own Docker containers, and these containers are
defined on a per-configuration basis. For the devel configuration, the Docker
containers are defined in
[xos/configurations/devel/docker-compose.yml](https://github.com/open-cloud/xos/blob/master/xos/configurations/devel/docker-compose.yml).

Using the `xos_synchronizer_openstack` as an example, add the following to that
file:

{% highlight yaml %}
xos_synchronizer_exampleservice:
    image: xosproject/xos-synchronizer-openstack
    command: bash -c "sleep 120; python /opt/xos/synchronizers/exampleservice/exampleservice-synchronizer.py -C /opt/xos/synchronizers/exampleservice/exampleservice_config"
    labels:
        org.xosproject.kind: synchronizer
        org.xosproject.target: exampleservice
    links:
        - xos_db
    extra_hosts:
        - ctl:${MYIP}
    volumes:
        - ../common/xos_common_config:/opt/xos/xos_configuration/xos_common_config:ro
        - ../setup/id_rsa:/opt/xos/synchronizers/exampleservice/exampleservice_private_key:ro
{% endhighlight %}

We'll use the same synchronizer image, `xosproject/xos-synchronizer-openstack`,
as it is suitable in most cases.  The command that is being run is specific to
our exampleservice.

For Ansible to communicate with the VM, it requires an SSH key that to access
the Tenant Instance. This is added with the line under volumes: `  -
../setup/id_rsa:/opt/xos/synchronizers/exampleservice/exampleservice_private_key:ro`

Remember to rebuild your docker containers (`make rm && make containers &&
make`) after making these changes.  Then verify that your new container is
running with `sudo docker ps`.

## Debugging

### XOS isn't coming up after making changes

Verify that the docker containers for XOS are running with:

`sudo docker ps`

If you need to see log messages for a container:

`sudo docker logs <docker_container>`

If you want to delete the containers, including the database, and start over, run:

`make rm`

Which will delete the containers.

### "500 Internal Server Error" when navigating the admin webpages

This is most likely Django reporting a problem in `admin.py` or `model.py`.

Django's debug log is located in in `/var/log/django_debug.log` on the
xos container, so run `make enter-xos` and then look at the end of that logfile.

### "Ansible playbook failed" messages

The logs messages for when the Synchronizer runs Ansible are in
`/opt/openstack/*` in your service's synchronizer Docker container.  You can
access the docker container by running:

`sudo docker exec -it bash <synchronizer_container>`

