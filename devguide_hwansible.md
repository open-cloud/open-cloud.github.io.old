---
layout: page
title: Step-by-Step Tutorial
permalink: /devguide/hwansible/
---

*Author: Jeremy Mowery, The University of Arizona*

{% include toc.html %}

## Summary

This document details all of the steps to create a Hello World service that uses Ansible to create a file containing a custom message on an apache web server. This is meant to be a tutorial document that will go through each step in detail and explain the importance of every step. This codelab will be useful to you if:

* You want to create a service with tenants using the existing service abstractions

* You want to use Ansible to configure instances on cloudlab

## Quick Reference

This section lists each change you need to make without explaining them. Use this only if you know what you are doing and don’t need full explanations or guided help:

### Add your Models

* Create a folder for your models in xos/xos

    * Create `models.py`, have your service extend Service and your tenant extend TenantWithContainer

    * Create `admin.py`, create admin classes for the service and tenant that extend `core.admin.ReadOnlyAwareAdmin`, create a form for the service and the tenant by extending `django.forms.ModelForm`

    * Create an empty `__init__.py`

    * Create a templates directory

        * Create an admin html that contains a link to your tenants page `/admin/<folder name from above>/<tenant class name>`

### Install Your Model

* Add the name of the folder containing your model to `INSTALLED_APPS` in `xos/xos/xos/settings.py`

* If you want to use this service on CloudLab, add a line to make migrations for your service to `xos/xos/scripts/opencloud`

### Add Your Synchronizer

* Create a directory for your synchronizer in `xos/xos/synchronizers`

    * Create `<service name>_config` with a config for your service

    * Create `<service name>-synchronizer.py` to run the xos-synchronizer module

    * Create model-deps with the dependencies for your service

    * Add `run.sh` to run your `<service name>-sychronizer.py` with your `<service name>_config`

    * Create a steps directory for the steps to sync your service in the data model with instances

        * Create `sync_<class name>.py` with a class that extends `observers.base.SyncInstanceUsingAnsible` that handles syncing your instance

        * Repeat the above for each model class you need to sync

        * Create a `sync_<class name>.yaml` to define to configuration you want to send to instances when the synchronizer runs for class name

        * Repeat the above for each model class you need to sync with Ansible

* Add a line to the appropriate configuration’s Dockerfile in `xos/xos/configurations` to copy `xos/configurations/common/id_rsa` to `/opt/xos/synchronizers/<service name>/<service name>_private_key`.

Note that earlier versions of XOS referred to the "Synchronizer" as the "Observer". File and directory names now consistently call it the "Synchronizer" but there are still references to "observer" in the code itself.

### Complete Version

A complete version of this codelab is available in the repo and is the [helloworldservice_complete](https://github.com/open-cloud/xos/tree/master/xos/synchronizers/helloworldservice_complete) service.

## Detailed Guide

This section is intended as a detailed guide of all of the steps needed to create a new service in XOS using the existing Service model and Ansible. If you are already familiar with this process and just need a quick reference please see the quick reference section of this document. This is guided documentation: you should follow along and write the code given here. After asking you to write code this guide explains what each step does and why it is necessary. This section assumes you are creating a service named helloworldservice. It also assumes you have created a folder named xos to store the contents of xos in.

### Prerequisites

This guide assumes:

* [You have read the XOS developer documentation](/devguide/)

* [You are familiar with Django](https://docs.djangoproject.com/en/1.8/)

* [You are familiar with Python](https://docs.python.org/3/)

### Overview

In this code lab we are going to create a Hello World Service that will allow users to enter  a message on a form on an admin page in XOS. When a message is entered or changed, an instance (VM) of XOS will be created and an Apache web server will be started with an index.html file at the root containing the message the user entered. We will be using the Service and Tenant abstractions (Django models) that already exist in XOS. The Service part will handle slices and creating VMs for Tenants. Tenants will contain the message that user wants to display. We will be syncing our logical data model with the VMs using Ansible. First we will work on the code and then observe the changes in the XOS administration. The flow at the end of this codelab will be like the diagram on the right:

To accomplish this we will need to do the following:

* Add HelloWorldService to the Django data model

* Add HelloWorldTenant to the Django data model

* Add a `synchronizer` to synchronize the data model with XOS using Ansible

* Configure XOS to use the HelloWorldService

* Demonstrate the full working service

### Add the Model

XOS uses the Django web framework as a database. Django is an object-relational database that uses a data model that XOS defines. The first step in creating a new service is to add our service to the data model by adding classes that represent our service. An XOS service is represented by a Service class and a Tenant class. The Service contains information for the Service as a whole (metadata, the slices allocated for the service, privileges, service specific attributes, and the tenants that subscribe the service). The Tenant is a subscriber to the service, when a tenant is created it’s state is synced with the underlying instances (a later section). The Tenant contains tenant specific attribute, in our case a display message that the user wants to show on a web server.

#### Create HelloWorldService and HelloWorldTenant in the Model

First we need to alter the data model so that our service is logically reflected in XOS. Navigate to xos/xos and create a directory named helloworldservice. This directory will contain all of the code needed to add helloworldservice classes to the data model. Create a file named [models.py](https://github.com/open-cloud/xos/blob/master/xos/services/helloworldservice_complete/models.py)

For the most part the comments in the file explain what every line is doing. The following is what we do in this file:

* We create a HelloWorldService class that extends Service, this will be used to represent the data for our service in the data model. HelloWorldService contains no fields specific to any instance of the service so we don’t have much to do here.

    * We do however add some important things to the Meta inner class

        * Here we indicate that this class is a Proxy of its super class, this means that HelloWorldService itself doesn’t have its own space in the Django database, it is treated the same as a Service. In this case this is appropriate because there isn’t anything different per instance of HelloWorldService.

        * We also set the app_label in Meta, we need to do this so that when we start the SQL server used by Django we can migrate these data models.

        * `verbose_name` is used on web forms for instances of this service.

* We create a HelloWorldTenant class that extends TenantWithContainer, this will be used to present the data for each tenant of the data model. For this class we need to define some extra fields for the message contained in each tenant and because we are going to use Ansible fields for the NAT IP and NAT MAC of the VM for our tenant.

    * For the most part this class is simple, we add some getters and setters for the display_message, nat_ip, and nat_mac field. The sync_attributes attribute is used later by the synchronizer and minimally must have nat_ip and nat_mac. It is important to note in the save function that we atomically manage the container (sets up the instance and the VM) to avoid race conditions.

    * We also make HelloWorldTenant a proxy of TenantWithContainer because we want XOS to treat this as a Tenant so that we can leverage the existing framework.

#### Create Forms to Create and Edit the Service and Tenants

Now that we have changed the data model to support the HelloWorldService we need to add some admin forms so that we can create and edit HelloWorldServices and HelloWorldTenants. We are going to be leveraging the forms from Django and some of the custom fields from XOS to do this. In xos/xos/helloworldservice create a file named [admin.py](https://github.com/open-cloud/xos/blob/master/xos/services/helloworldservice_complete/admin.py).

For a line-by-line breakdown please see the comments in the file. As a summary here is what this file is doing:

* We are creating forms that Django will use to create HTML forms that add and edit instances of HelloWorldService and HelloWorldTenant that we created in [Add the Model](#add-the-model).

* The form for HelloWorldService doesn’t require any custom logic because we didn’t have any additional fields in the data model, so we simply supply the configuration that Django needs. It is important to note that we are using our own template, helloworldserviceadmin.html for the administration tab, we will create this file in the next step.

* The form for HelloWorldTenant requires some additional logic because we added a field for the display_message and because as a tenant we need to associate the creator of the tenant with the object in the data model.

    * To do this we configure the HelloWorldTenantAdmin like the HelloWorldServiceAdmin but set the form attribute to HelloWorldTenantForm, this is what will be used for our custom logic.

        * The most important parts of the HelloWorldTenantForm are the init function and the save function. In the init function we provide the logic to set up the form when it first created depending on if we are adding a new HelloWorldTenant or we are editing an existing one. In the save function we set attributes in the HelloWorldTenant attribute for the creator and the display_message so that these will be passed along when we enact the instance in a VM and when we want to edit the instance again.

#### Create the Custom Template for the Admin View

As is noted above we need to create a custom template for the administration tab of HelloWorldServiceAdmin. This can be very complex if needed but for our purposes we are just going to create a button that links to the HelloWorldTenant form. Create a directory named templates in xos/xos/helloworldservice. Create a file named [helloworldserviceadmin.html](https://github.com/open-cloud/xos/blob/master/xos/services/helloworldservice_complete/templates/helloworldserviceadmin.html).

This makes use of the "left-nav" class to style the button, it leads to the form for the HelloWorldTenant.

## Install the Service

Now that we have the model complete we can need to install our service as an app and configure the cloudlab script to add the HelloWorldService to the Django database when we start XOS on cloudlab.

### Register HelloWorldService with XOS

Navigate to `xos/xos/xos/settings.py` and add `'helloworldservice',` to the INSTALLED_APPS list. This will install helloworldservice in XOS. Note that this is the name given in the `app_label` field of the HelloWorldService.

### Add HelloWorldService to the DB on CloudLab

Navigate to `xos/xos/scripts/opencloud`. Add the line `python ./manage.py makemigrations helloworldservice` to the makemigrations function. This will add the helloworldservice model to the Django database when XOS is started on opencloud.

## Verify the Model on CloudLab

In this section you will start XOS on CloudLab and verify the following:

* There is a form for creating instances of your HelloWorldService class

* There is a form for creating instances of your HelloWorldTenant class

* You can create a HelloWorldService instance

* You can create a HelloWorldTenant instance

* The state of your HelloWorldTenant is saved between instances

### Start XOS

This part assumes that you have an account on CloudLab. First we need to start the experiment on CloudLab to do this do the following:

1. Navigate to [https://www.cloudlab.us/instantiate.php](https://www.cloudlab.us/instantiate.php)

2. Change the profile to "OpenStack"

3. Click next

4. Expand "Advanced Parameters"

5. Check the box for the option "Disable Security Group Enforcement"

6. Click next

7. In the "Cluster" dropdown select “Cloudlab Clemson”

8. (optional) Give your experiment a name in the name field

9. Click finish

After a long time the experiment will start up with three nodes, each one of these nodes is a machine in the Clemson cluster, under list view you should see something like this:

{% include figure.html url="/figures/devguide_hwansible-fig01_start_cluster.png" caption="" %}

Each SSH command takes you to a specific machine in indicated by the ID. Copy and paste the SSH command for the ctl node into a terminal and press enter (or if your browser supports you can click on the command to immediately SSH).

To start XOS on cloudlab do the following:

1. Copy over the files you have been working on to the node (forking the XOS repo and using Git is often easiest).

2. SSH to the ctl node if you haven’t already and navigate to the folder containing the root folder for xos (assumed as xos as it is in the previous steps).

3. Navigate to `xos/xos/configurations/devel`.

4. Run make, this will start XOS as a VM in docker after a few minutes

### Create a HelloWorldService Instance

Now XOS will be running and will be available at the address of your ctrl node on port 9999, for example with the above list view it would be at:

`http://clnode086.clemson.cloudlab.us:9999/`

Navigate to this address in your web browser and log in using the following credentials:

 - Username: `padmin@vicci.org`
 - Password: `letmein`

Navigate to your service at `http://<node name>:9999/admin/helloworldservice_complete`, for example using the address above you would have: `http://clnode086.clemson.cloudlab.us:9999/admin/helloworldservice_complete`. You will see a page like this:

{% include figure.html url="/figures/devguide_hwansible-fig02_create_instance.png" caption="" %}

Click Add in the Hello World Services row to create a Hello World Service.

Now you will see a form like this:

{% include figure.html url="/figures/devguide_hwansible-fig03_add_service_form.png" caption="" %}

This is a form we will use to fill in information for the HelloWorldService.

* Fill in the name field with "Hello World Service"

* Fill in the VersionNumber field with "1"

* Click the Slices tab

Now you will see a form like this:

{% include figure.html url="/figures/devguide_hwansible-fig04_sliceform_png" caption="" %}

Using this form we will create a Slice that will contain the VMs for the instances hosting our tenants, at the moment this part won't be implemented, but it is useful to think about this.

* Click "Add another slice"

* In the Name field enter "mysite_helloworldservice"

* In the Site field select "mysite"

* In the ServiceClass field select "Best Effort"

* Click Save

This will save both HelloWorldService and the Slice.

You should now see a page like this:

{% include figure.html url="/figures/devguide_hwansible-fig05_sliceadded.png" caption="" %}

### Create a HelloWorldTenant Instance

From the page above, click the "Hello World Service" link in the first row.

This will give you a form to edit the HelloWorldService you created, click the Tenants tab, you will see a page like this:

{% include figure.html url="/figures/devguide_hwansible-fig06_addinstance.png" caption="" %}

Click the "Hello World Tenants" button, you will see a form like this:

{% include figure.html url="/figures/devguide_hwansible-fig07_addtenant.png" caption="" %}

Click the "Add hello world tenant" button, you will see a form like this:

{% include figure.html url="/figures/devguide_hwansible-fig08_addtenant2.png" caption="" %}

Here we can verify that the Display message has the initial value of "Hello world!" like we set in getter for display_message in models.py and init in admin.py.

Change this value and click save.

You will now see a form like this:

{% include figure.html url="/figures/devguide_hwansible-fig09_tenantadded.png" caption="" %}

Click anyone of the links, verify that the display message is the same as when you submitted the form, try changing it, saving, and then coming back.

If you can change the message and that message is restored when you edit the HelloWorldTenant then you have successfully completed the model.

## Add the Synchronizer

To enact the state of the data model on the XOS instances we need to create an synchronizer program. This program will wait for changes to the data model and then complete a series of steps to synchronize the model with the instances. Most of this is boilerplate, but it is a good idea to become familiar with the setup. In our case we will be waiting for changes to the HelloWorldTenant (since HelloWorldService doesn’t add anything on top of Service we can rely on the synchronizer already built for synchronizing Services), when there is a change if we are creating or editing the display_message of the tenant we will pass on the new display_message to an Ansible template which will start up a web server displaying the message.

### Set up the Boilerplate

To set up the boilerplate code do the following:

* Create the directory `xos/xos/synchronizers/helloworldservice_complete/`

* Create the directory `xos/xos/synchronizers/helloworldservice_complete/steps`

* Create the file [xos/xos/synchronizers/helloworldservice_complete/helloworldservice-synchronizer.py](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/helloworldservice_complete/helloworldservice-synchronizer.py) which will run the existing XOS synchronizer. We will create a script in a later step that will pass a particular configuration into this program that will have the XOS synchronizer execute the steps for the HelloWorldTenant.

* Create the file [xos/xos/synchronizers/helloworldservice/model-deps](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/helloworldservice_complete/model-deps). This file is used to denote the dependencies of the synchronizer that are needed, in our case we don’t have any dependencies so we indicate this with the empty set.

**Note: **For a real model with dependencies you can generate the dependency graph within a VM from /opt/xos: ./dmdot helloworldservice.

* Next create the file [xos/xos/synchronizers/helloworldservice_complete/helloworldservice_config](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/helloworldservice_complete/helloworldservice_config). This file is a config file that is used by the XOS synchronizer to run the necessary steps for synchronizing the HelloWorldTenant. Most of this is explained in the comments but the following deserve some special attention:

    * Make sure that backoff_disabled is set to True, this ensures that when the synchronizer fails it tries again immediate instead of waiting increasingly longer periods of time before retrying. Since we will need to wait for an instance to come online before we can display the message the synchronizer will fail at first but quickly succeed.

    * The proxy_ssh field must be set to False so that we can connect to clients over SSH on CloudLab, if this is not set to False the synchronizer will always fail on CloudLab.

    * All of the *_dir fields need to be set as they are above, they will be used by the synchronizer to find our steps and as a working directories for created files.

* Create the file [xos/xos/synchronizers/helloworldservice_complete/run.sh](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/helloworldservice_complete/run.sh). This script will run the xos synchronizer with the config we just created to run your synchronizer.

* Create the file [xos/xos/synchronizers/helloworldservice_complete/stop.sh](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/helloworldservice_complete/stop.sh). This is a simple script used to kill your synchronizer.

### Create the Sync Steps

Now that we have all of the necessary boilerplate we can define what should happen when a HelloWorldTenant is altered in the data model. In particular we want to start a web server to display the message that was entered using ansible. To do this do the following:

* Create the file [xos/xos/synchronizers/helloworldservice_complete/steps/sync_helloworldtenant.yaml](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/helloworldservice_complete/steps/sync_helloworldtenant.yaml).  This file is the ansible template that will be used to configure instances that are created for the HelloWorldTenants. The top part (before the tasks section) gives certain configuration attributes, namely the name of the instance that we are connecting to, the connection type, the user, and if we need admin privileges. In the tasks section we enumerate in order the tasks that need to be performed when this configuration is used. Each task as a name and what it does. There are some shorthands like apt and service, but you can always use shell to do anything you need. In this case we install apache, write out the display_message that is passed to template, and then stop and start the apache service.

* Lastly we need to define what to do when the synchronizer runs for a HelloWorldTenant. We define this in [xos/xos/synchronizers/helloworldservice_complete/steps/sync_helloworldtenant.py](https://github.com/open-cloud/xos/blob/master/xos/synchronizers/helloworldservice_complete/steps/sync_helloworldtenant.py). In this file we using the SyncInstanceUsingAnsible class to handle most of the ansible configuration for us. All we need to do is indicate what is being observed, how often the synchronizer runs, the template name, the ssh key location (in a later step), what instances of HelloWorldTenant need to be enacted, and what additional attributes are necessary in the ansible template. This is all explained in the comments of the file, but special attention should be given to the get_extra_attributes function. This function is used in SyncInstanceUsingAnsible to get the extra attributes needed in the ansible template. Here we return the display_message of  particular data model instance of a HelloWorldTenant so that we can make use of it like we do in the template.  

### Add the SSH Key

As we saw in sync_helloworldtenant.py above, we need to have an SSH key to use ansible. To add this we simply change the Dockerfile to copy over the appropriate key when starting up XOS. To do this open the file [configurations/devel/Dockerfile.devel](https://github.com/open-cloud/xos/blob/master/containers/xos/Dockerfile.devel) and add these lines after `Install XOS`:

```
# Added for hellowordservice_complete's synchronizer can run
ADD xos/configurations/setup/id_rsa /opt/xos/synchronizers/helloworldservice_complete/helloworldservice_private_key
```

Note that this is one line, not two. This key is used by ansible when using an SSH connection to avoid using passwords.

## Verify the Synchronizer on CloudLab

Now that we have the synchronizer made we can verify that it is working on CloudLab. To do this we do the following:

* Copy your changes over to the ctl node on CloudLab as you did in [Start XOS on Cloudlab](#start-xos-on-opencloud).

* On the ctl node navigate to `xos/xos/configurations/devel`

* Stop your previous instance of XOS (if it is still running) by running `make stop`

* Run `make` to start up XOS again with your changes (note: if you don’t see anything change or files are missing in the steps below run `make rebuild_xos` and `make rebuild_synchronizer` to recreate the Docker containers for each, then `make` again to run them)

* Once XOS has finished starting up, enter the VM by running `make enter-synchronizer`

* Navigate to `/opt/xos/synchronizers/helloworldservice_complete`

* Run `./run.sh` (keep the window open with the output from this to observe the synchronizer in action)

You should see some output constantly streaming from the synchronizer:

```
root@4c7c64485c10:/opt/xos/synchronizers/helloworldservice_complete# ./run.sh 
Skipping model policies thread for service observer.
INFO:xos.log:Deletion=False...
INFO:xos.log:Deletion=False...
INFO:xos.log:Starting to work on step SyncHelloWorldTenantComplete, deletion=False
INFO:xos.log:Executing step SyncHelloWorldTenantComplete, deletion=False
Executing step SyncHelloWorldTenantComplete
INFO:xos.log:Step 'SyncHelloWorldTenantComplete' succeeded
Step 'SyncHelloWorldTenantComplete' succeeded
INFO:xos.log:Step <class 'sync_helloworldtenant.SyncHelloWorldTenantComplete'> is a leaf
INFO:xos.log:Deletion=True...
INFO:xos.log:Deletion=True...
INFO:xos.log:Starting to work on step SyncHelloWorldTenantComplete, deletion=True
INFO:xos.log:Executing step SyncHelloWorldTenantComplete, deletion=True
Executing step SyncHelloWorldTenantComplete
INFO:xos.log:Step 'SyncHelloWorldTenantComplete' succeeded
Step 'SyncHelloWorldTenantComplete' succeeded
INFO:xos.log:Step <class 'sync_helloworldtenant.SyncHelloWorldTenantComplete'> is a leaf
INFO:xos.log:Waiting for event
INFO:xos.log:Observer woke up
INFO:xos.log:Deletion=False...
```

* Repeat steps [Create a HelloWorldService Instance](#create-a-helloworldservice-instance) and [Create a HelloWorldTenant Instance](#create-a-helloworldtenant-instance) until you complete creating one HelloWorldTenant instance.

* Pay close attention to the window with the output from the synchronizer, initially since you just created a new HelloWorldTenant, the instance will be still be starting up so you will see an error like:

```
Exception: defer object helloworldservice-tenant-3 due to waiting on instance.instance_name
```

or:

```
Exception: Unreachable results in ansible recipe
```

These are normal while the instance is starting because the VM is not yet initialized, eventually you will see output that includes the steps given in the `sync_helloworldtenant.yaml` file and a success message. This means that a web server is up and running with the message.

* The last thing we need to do is view the page that we just created. To do this not the instance name in XOS for the instance you just created from the Hello world tenants page (you may need to refresh, if this is your first instance it will be called `mysite_helloworldservice-1`). Then click the Slices button on the left. Then click the slice you created, click instances, and note the IP addresses of the instance with the name from earlier. Record the 10.11.X.X address as we will need this.

* Finally, from the ctl node, run `curl <10.11.X.X address>`, and you should see the display message you entered when creating the Hello World Tenant.

```
user@ctl:~/xos/xos/configurations/devel$ curl 10.11.10.7
Hello World!
```

## Common Problems

* After running make to start XOS, the XOS does not incorporate my changes

    * This can happen because docker uses a cache, to resolve this edit `xos/xos/configurations/devel/Makefile` so that the docker build command uses the `--no-cache` option.

* No 10.11.X.X address is given on an instance

    * On the control node in the `xos/xos/configurations/devel` directory run make stop

    * In the control node not as root run: `/proj/xos-PG0/acb/cleanup.sh`

* I have run out of disk space on the control node

    * As root run the following:

{% highlight sh %}
docker stop $(docker ps -a -q)

docker rm $(docker ps -a -q)

docker rmi $(docker images -f "dangling=true" -q)
{% endhighlight %}

## Using HelloWorldService as a Template

The HelloWorldService you build in this codelab is suitable as a template for creating another service. Most of this is boilerplate, so the only changes that need to be made at the following:

* Replace all of the names are refer to Hello World with the name of your service

* Alter `models.py` to add fields you need

* Alter `admin.py` to add the fields you need to the form

* Change the `sync_*.py` files to return attributes you need in ansible in the `get_extra_attributes` function

* Change the ansible template to do what you need it to do

## Conclusion

Congratulations you have completed the Hello World in XOS using Ansible CodeLab! You now know how to make a Service in XOS with Tenants that use Ansible to configure instances. Most of this can be used to create new services as a lot of it is boilerplate.

