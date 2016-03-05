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

 - `tenant_message`, a string that contains the message to display on a
   per-Tenant basis

When a user creates a new Slice and Tenant in the Admin website (constructed
using Django), the data describing the service is stored in a database.

The Synchronizer runs on a recurring basis, obtains the `tenant_message` (and
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
   and run `make`.  This will create an XOS installation using the `devel`
   configuration on the OpenStack host by installing prerequisites and creating
   and starting the XOS Docker containers.

 - Navigate to `http://<ip_or_dns_name_of_host>:9999` and verify that you can
   login with the default XOS credentials, which are username:
   `padmin@vicci.org` and password: `letmein`.

 - **Optional:** Check to make sure you can run the provided `exampleservice`
   code.  This is not enabled by default, and you'll need to follow the steps
   in [Install the Service in Django](#install-the-service-in-django) and
   [Create a Docker container to run the
   synchronizer](#create-a-docker-container-to-run-the-synchronizer) to enable
   it.


### The development loop

Once you've prepared your development environment as described above, the
change/build/test development loop for service development on XOS is as follows:

 1. Make changes to your copy of XOS and propagate them to your OpenStack host.

 2. On your OpenStack host in XOS's `xos/configurations/devel` directory, run:
    `make rm`, which will stop XOS and delete the Docker containers.

 3. Run `make containers`.  This will rebuild all the Docker containers to
    include your changes.  Initially this will take a long time as everything
    is rebuilt, but will take less time during subsequent runs as Docker saves
    intermediate state.

 3. Run: `make` in the same directory to start XOS running with the newly built
    containers.

 4. Test and verify your changes.

 5. Once you're done testing, go back to step #1.

The process to do the above can be done with this command:

{% highlight shell %}
git pull && make rm && make containers && make
{% endhighlight %}

Assuming you're pulling your changes from a development git repo. 

## Create the Django components of a Service

XOS services are located in the `xos/services` directory in the XOS source
tree.  Create a new directory for your service.

In this example, the service will be named `exampleservice` and the example
code is in the XOS repo at
[xos/services/exampleservice](https://github.com/open-cloud/xos/tree/master/xos/services/exampleservice).

In your service directory, create an empty `__init.py__` file.  This is
required for Python to recognize your service directory [as a
package](https://docs.python.org/2/tutorial/modules.html#packages), in order to
avoid namespace conflicts.

### Create a Django Model

Create a file named
[models.py](https://github.com/open-cloud/xos/tree/master/xos/services/exampleservice/models.py)
in your service directory. The start of this file:

{% highlight python %}
from core.models import Service, TenantWithContainer
from django.db import models, transaction
{% endhighlight %}

This brings in the two XOS Core classes that will be extended by our Service,
define the model and lets us atomically update the database on instance
creation/removal.

{% highlight python %}
SERVICE_NAME = 'exampleservice'
SERVICE_NAME_VERBOSE = 'Example Service'
SERVICE_NAME_VERBOSE_PLURAL = 'Example Services'
TENANT_NAME_VERBOSE = 'Example Tenant'
TENANT_NAME_VERBOSE_PLURAL = 'Example Tenants'
{% endhighlight %}

These are used to uniquely identify and provide human readable names for the
service in the admin web UI.  For your own service, change these.

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

XOS uses `KIND` variable to uniquely identify each service.

The [Meta options](https://docs.djangoproject.com/es/1.9/ref/models/options/)
for `app_label` and `verbose_name` are used on the admin web UI.

`proxy` is used as there isn't any new fields in this model, thus it can use it's
super's data model, per [Django's
documentation](https://docs.djangoproject.com/en/1.9/topics/db/models/#proxy-models).

You wouldn't use `proxy` if your Service is going to add additional fields to
the model.

#### Extending TenantWithContainer

We extend [XOS Core's
TenantWithContainer](https://github.com/open-cloud/xos/blob/master/xos/core/models/service.py)
class, which is a Tenant that creates a Container (virtual machine) instance.

{% highlight python %}
class ExampleTenant(TenantWithContainer):

    KIND = SERVICE_NAME

    class Meta:
        verbose_name = TENANT_NAME_VERBOSE
{% endhighlight %}

As in [Extending Service](#extending-service).

{% highlight python %}
    tenant_message = models.CharField(max_length=254, help_text="Tenant Message to Display")
{% endhighlight %}

Using [Django's
Models](https://docs.djangoproject.com/en/1.9/topics/db/models/), create a
`CharField` in the data model to store the message this Tenant will display.

{% highlight python %}
    def __init__(self, *args, **kwargs):
        exampleservice = ExampleService.get_service_objects().all()
        if exampleservice:
            self._meta.get_field('provider_service').default = exampleservice[0].id
        super(ExampleTenant, self).__init__(*args, **kwargs)
{% endhighlight %}

When creating the Tenant, provide a default value of the first service
available in the UI.

{% highlight python %}
    def save(self, *args, **kwargs):
        super(ExampleTenant, self).save(*args, **kwargs)
        model_policy_exampletenant(self.pk)
{% endhighlight %}

On save, you may need to create an Instance, which is done by calling the
`model_policy` function (see below).

{% highlight python %}
    def delete(self, *args, **kwargs):
        self.cleanup_container()
        super(ExampleTenant, self).delete(*args, **kwargs)
{% endhighlight %}

On delete, need to delete the instance created by this Tenant, which is done by
`cleanup_container()`.

{% highlight python %}
def model_policy_exampletenant(pk):
    with transaction.atomic():
        tenant = ExampleTenant.objects.select_for_update().filter(pk=pk)
        if not tenant:
            return
        tenant = tenant[0]
        tenant.manage_container()
{% endhighlight %}

If a TenantWithContainer is updated, call `manage_container()`, which
creates/destroys the appropriate Instance VMs.

### Create a Django Admin

Create a file named
[admin.py](https://github.com/open-cloud/xos/tree/master/xos/services/exampleservice/admin.py)
in your service directory. The start of this file:

{% highlight python %}
from core.admin import ReadOnlyAwareAdmin, SliceInline
from core.middleware import get_request
from core.models import User

from django import forms
from django.contrib import admin

from services.exampleservice.models import *
{% endhighlight %}

Import the classes to extend, as well as other needed functions.

Also import the model we created, and the *_NAME_* variables.


#### Extend admin classes for the service and tenant classes

{% highlight python %}
class ExampleServiceAdmin(ReadOnlyAwareAdmin):

    model = ExampleService
    verbose_name = SERVICE_NAME_VERBOSE
    verbose_name_plural = SERVICE_NAME_VERBOSE_PLURAL
{% endhighlight %}

As in [Extending Service](#extending-service).

{% highlight python %}
    list_display = ('backend_status_icon', 'name', 'enabled',)
    list_display_links = ('backend_status_icon', 'name', )
{% endhighlight %}

Columns to display for the list of ExampleService objects, in the Admin web UI at `/admin/exampleservice/exampleservice/`. 

{% highlight python %}
    fieldsets = [(None, {
        'fields': ['backend_status_text', 'name', 'enabled', 'versionNumber', 'description',],
        'classes':['suit-tab suit-tab-general',],
        })]

    readonly_fields = ('backend_status_text', )
    user_readonly_fields = ['name', 'enabled', 'versionNumber', 'description',]
{% endhighlight %}

Rows displayed when you're looking at an of ExampleService at `/admin/exampleservice/exampleservice/<service id>/` with field privileges.

{% highlight python %}
    inlines = [SliceInline]
{% endhighlight %}

Display the [Slice tab]((https://github.com/open-cloud/xos/tree/master/xos/core/admin.py).
). 

{% highlight python %}
    extracontext_registered_admins = True
{% endhighlight %}

[Render this page for Service admin users](https://github.com/open-cloud/xos/tree/master/xos/core/admin.py).

{% highlight python %}
    suit_form_tabs = (
        ('general', 'Example Service Details', ),
        ('slices', 'Slices',),
        )

    suit_form_includes = ((
        'top',
        'administration'),
        )
{% endhighlight %}

Order of the tabs, and additional [Suit form includes](http://django-suit.readthedocs.org/en/develop/form_includes.html).

{% highlight python %}
    suit_form_tabs = (
    def queryset(self, request):
        return ExampleService.get_service_objects_by_user(request.user)
{% endhighlight %}

List only a user's service objects in the [Suit form_tabs](http://django-suit.readthedocs.org/en/develop/form_tabs.html)

{% highlight python %}
admin.site.register(ExampleService, ExampleServiceAdmin)
{% endhighlight %}

[Register](https://docs.djangoproject.com/en/1.9/ref/contrib/admin/#modeladmin-objects) the `ExampleServiceAdmin` with Django.

{% highlight python %}
class ExampleTenantForm(forms.ModelForm):

    class Meta:
        model = ExampleTenant
{% endhighlight %}

Specify that this Form will use the [model](#extending-tenantwithcontainer) we defined before.

{% highlight python %}
    creator = forms.ModelChoiceField(queryset=User.objects.all())
{% endhighlight %}

Create a field later used to assign a user to this Tenant. 

{% highlight python %}
    def __init__(self, *args, **kwargs):
        super(ExampleTenantForm, self).__init__(*args, **kwargs)

        self.fields['kind'].widget.attrs['readonly'] = True
        self.fields['kind'].initial = SERVICE_NAME

        self.fields['provider_service'].queryset = ExampleService.get_service_objects().all()

        if self.instance:
            self.fields['creator'].initial = self.instance.creator
            self.fields['tenant_message'].initial = self.instance.tenant_message

        if (not self.instance) or (not self.instance.pk):
            self.fields['creator'].initial = get_request().user
            if ExampleService.get_service_objects().exists():
                self.fields['provider_service'].initial = ExampleService.get_service_objects().all()[0]
{% endhighlight %}

When creating the Form, set initial values for the fields.

{% highlight python %}
    def save(self, commit=True):
        self.instance.creator = self.cleaned_data.get('creator')
        self.instance.tenant_message = self.cleaned_data.get('tenant_message')
        return super(ExampleTenantForm, self).save(commit=commit)
{% endhighlight %}

Save the [validated data](https://docs.djangoproject.com/en/1.9/topics/forms/#field-data), for who created this Tenant and the message. 

{% highlight python %}
class ExampleTenantAdmin(ReadOnlyAwareAdmin):

    verbose_name = TENANT_NAME_VERBOSE
    verbose_name_plural = TENANT_NAME_VERBOSE_PLURAL

    list_display = ('id', 'backend_status_icon', 'instance', 'tenant_message')
    list_display_links = ('backend_status_icon', 'instance', 'tenant_message', 'id')

    fieldsets = [(None, {
        'fields': ['backend_status_text', 'kind', 'provider_service', 'instance', 'creator', 'tenant_message'],
        'classes': ['suit-tab suit-tab-general'],
        })]

    readonly_fields = ('backend_status_text', 'instance',)

    form = ExampleTenantForm

    suit_form_tabs = (('general', 'Details'),)

    def queryset(self, request):
        return ExampleTenant.get_tenant_objects_by_user(request.user)

admin.site.register(ExampleTenant, ExampleTenantAdmin)
{% endhighlight %}

See notes above on `ExampleServiceAdmin` - this configures the fields for the Tenant admin UI.

### Install the Service in Django

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

Go through [the development loop](#the-development-loop) to include your
service in XOS.  During the final `make` step, you may want to run `docker logs
-f devel_xos_1` and look out for any errors which may occur when you first run
the code.   If so, fix them and restart the loop. 

Once XOS is up, go to
`http://<ip_or_dns_name_of_host>:9999/admin/exampleservice`, and you should see
the admin interface: 

{% include figure.html url="/figures/devguide_exampleservice_fig01_adminpage.png" caption="ExampleService Administration" %}

Select "Change" next to "Example Services", and you'll see list of Example
Services (empty for now):

{% include figure.html url="/figures/devguide_exampleservice_fig02_addservice.png" caption="" %}

Click on "Add Example Service", and you'll see options for configuring a
service.

{% include figure.html url="/figures/devguide_exampleservice_fig03_configservice.png" caption="" %}

Fill in the "Name:" and "VersionNumber:" fields, then click the "Slices" tab at
top, then "Add another slice".

{% include figure.html url="/figures/devguide_exampleservice_fig04_configslice.png" caption="" %}

Fill in the slice name, then select "mysite" in the Site popdown, then click
"Save".

{% include figure.html url="/figures/devguide_exampleservice_fig05_servicesuccess.png" caption="" %}

You should see a message similar to this saying that adding the service was
successful. 

Go back to the main ExampleService admin page at `/admin/exampleservice` and
next to "ExampleTenants" click "Add".

{% include figure.html url="/figures/devguide_exampleservice_fig06_createtenant.png" caption="" %}

Fill in a "Tenant Message", then click Save.  You should then see a message
that "Success! The Example Tenant "exampleservice-tenant-1" was added
successfully.", and a list of Tenants with your message listed. 

## Create a Synchronizer

Synchronizers are scripts that run in a loop and check for changes to the
Tenant model and apply them to the running Instances. In this case, we're using
TenantWithContainer, which will create a Virtual Machine Instance. 

XOS Synchronizers are located in the
[xos/synchronizers](https://github.com/open-cloud/xos/tree/master/xos/synchronizers/)
directory in the XOS source tree.  It's customary to name the synchronizer
directory with the same name as your service.  The example code given below is
in the XOS repo at
[xos/synchronizers/exampleservice](https://github.com/open-cloud/xos/tree/master/xos/synchronizers/exampleservice).

Create a file named `model-deps` with the contents: `{}`. 

*NOTE: This is used to track model dependencies using `tools/dmdot`, but that tool currently isn't working.*

Create a file named `exampleservice-synchronizer.py`:

{% highlight python %}
#!/usr/bin/env python

# Runs the standard XOS synchronizer

import importlib
import os
import sys

synchronizer_path = os.path.join(os.path.dirname(
    os.path.realpath(__file__)), "../../synchronizers/base")
sys.path.append(synchronizer_path)
mod = importlib.import_module("xos-synchronizer")
mod.main()
{% endhighlight %}

This is boilerplate which loads and runs the [default xos-synchronizer module](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/base/xos-synchronizer.py) in [it's own Docker container](#create-a-docker-container-to-run-the-synchronizer).

To configure this module, create a file named `exampleservice_config`, which
specifies various configuration and logging options: 

{% highlight ini %}
# Required by XOS
[db]
name=xos
user=postgres
password=password
host=localhost
port=5432

# Required by XOS
[api]
nova_enabled=True

# Sets options for the synchronizer
[observer]
name=exampleservice
dependency_graph=/opt/xos/synchronizers/exampleservice/model-deps
steps_dir=/opt/xos/synchronizers/exampleservice/steps
sys_dir=/opt/xos/synchronizers/exampleservice/sys
logfile=/var/log/xos_backend.log
pretend=False
backoff_disabled=True
save_ansible_output=True
proxy_ssh=False
{% endhighlight %}

*NOTE: Historically, synchronizers were named "observers", so `s/observer/synchronizer/` when you come upon this term in the XOS code/docs.*

Create a directory within your synchronizer directory named `steps`.  In `steps`, create a file named `sync_exampletenant.py`: 


{% highlight python %}
import os
import sys
from django.db.models import Q, F
from services.exampleservice.models import ExampleService, ExampleTenant
from synchronizers.base.SyncInstanceUsingAnsible import SyncInstanceUsingAnsible

parentdir = os.path.join(os.path.dirname(__file__), "..")
sys.path.insert(0, parentdir)
{% endhighlight %}

Bring in some basic prerequities, [Q](https://docs.djangoproject.com/es/1.9/topics/db/queries/#complex-lookups-with-q-objects) to perform complex queries, and  [F](https://docs.djangoproject.com/es/1.9/ref/models/expressions/#f-expressions) to get the value of the model field. Also include the models created earlier, and [SyncInstanceUsingAnsible](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/base/SyncInstanceUsingAnsible.py) which will run the Ansible playbook in the Instance VM. 

{% highlight python %}
class SyncExampleTenant(SyncInstanceUsingAnsible):

    provides = [ExampleTenant]
{% endhighlight %}

Used by [XOSObserver : sync_steps](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/base/event_loop.py) to determine dependencies.

{% highlight python %}
    observes = ExampleTenant
{% endhighlight %}

The Tenant that is synchronized.

{% highlight python %}
    requested_interval = 0

    template_name = "exampletenant_playbook.yaml"
{% endhighlight %}

Name of the ansible playbook to run. 

{% highlight python %}
    service_key_name = "/opt/xos/synchronizers/exampleservice/exampleservice_private_key"
{% endhighlight %}

Path to the SSH key used by Ansible.

{% highlight python %}
    def __init__(self, *args, **kwargs):
        super(SyncExampleTenant, self).__init__(*args, **kwargs)

    def fetch_pending(self, deleted):

        if (not deleted):
            objs = ExampleTenant.get_tenant_objects().filter(
                Q(enacted__lt=F('updated')) | Q(enacted=None), Q(lazy_blocked=False))
        else:
            # If this is a deletion we get all of the deleted tenants..
            objs = ExampleTenant.get_deleted_tenant_objects()

        return objs
{% endhighlight %}

Determine if there are Tenants that need to be updated by running the Ansible playbook.

{% highlight python %}
    # Gets the attributes that are used by the Ansible template but are not
    # part of the set of default attributes.
    def get_extra_attributes(self, o):
        return {"tenant_message": o.tenant_message}
{% endhighlight %}

Pass the `tenant_message` variable to the Ansible playbook. 

Next, create an [Ansible playbook](http://docs.ansible.com/ansible/playbooks.html) named `exampletenant_playbook.yml`:

{% highlight yaml %}
---
# exampletenant_playbook

- hosts: "{{ "{{ instance_name " }}}}"
  connection: ssh
  user: ubuntu
  sudo: yes
  gather_facts: no

  tasks:
  - name: install apache
    apt:
      name=apache2
      update_cache=yes

  - name: write message
    shell: echo "{{ "{{ tenant_message " }}}}" > /var/www/html/index.html
{% endhighlight %}

This performs two steps:

 - Installs Apache using `apt`
 - Writes a message to the `index.html` file in Apache's document root

It's a good idea to check this file with [ansible-lint](https://github.com/willthames/ansible-lint) if you have it available. 

### Create a Docker container to run the Synchronizer

Synchronizers run in their own Docker containers, and these containers are
defined in the [Docker Compose
files](https://docs.docker.com/compose/compose-file/) in each configuration.
For the devel configuration, we'll need to modify
[xos/configurations/devel/docker-compose.yml](https://github.com/open-cloud/xos/blob/master/xos/configurations/devel/docker-compose.yml).

Using the `xos_synchronizer_openstack` as an example, create a new section as
follows:

{% highlight yaml %}
...
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
...
{% endhighlight %}

We'll use the same synchronizer image, `xosproject/xos-synchronizer-openstack`,
as it is suitable in most cases.  The `command` is a path to the Synchronizer
and it's config file.  The `org.xosproject.target` label should be updated as
well.

For Ansible to communicate with the VM, it requires an SSH key in order to communicate with the 
the Instance VM. This is added read-only as a Docker volume:
`- ../setup/id_rsa:/opt/xos/synchronizers/exampleservice/exampleservice_private_key:ro`

Remember to rebuild your docker containers (`make rm && make containers &&
make`) after making these changes.  Then verify that your new container is
running with `sudo docker ps`, in addition to the other 3 devel configuration
containers.

In the Admin web UI, navigate to the Slice -> `<slicename>` -> Instances, and
find an IP address starting with `10.11.X.X` in the Addresses column (this
address is the "nat" network for the slice, the other address is for the
"private" network). 

Run `curl <10.11.X.X address>`, and you should see the display message you
entered when creating the ExampleTenant.

{% highlight shell %}
user@ctl:~/xos/xos/configurations/devel$ curl 10.11.10.7
Example Tenant example text
{% endhighlight %}

After verifing that the text is shown, change the message in the "Example
Tenant" section of the Admin UI, wait a bit for the Synchronizer to run, and
then the message that `curl` returns should be changed.

## Debugging

### XOS isn't coming up after making changes

Verify that the docker containers for XOS are running with:

`sudo docker ps`

If you need to see log messages for a container:

`sudo docker logs <docker_container>`

There's also a shortcut in the makefile to view logs for all the containers: `make showlogs`

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

