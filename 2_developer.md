---
layout: page
title: Developer Guide
---

This guide describes how to contribute to the XOS core, as well as to
extend XOS with new services and views. It also documents modeling and
naming conventions, and describes the development and testing
environments we use.

When necessary for clarity of exposition, this guide uses examples
from OpenCloud for illustrative purposes. For example, on OpenCloud
the XOS server process is started in directory
/opt/xos/scripts/opencloud. Substitute local specifics for an
alternative XOS installation.

##<a name="config-mgmt">Configuration Management</a>

The XOS distribution includes a collection of configurations, each of
which implies three main parameters: (1) the XOS release, (2) the
target hardware, and (3) the service portfolio. For example, the
*Devel Config* described below uses the latest stable release
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

The rest of this section describes four stock configuations that are
provided with the release. It also includes a description of the
configuration we use for OpenCloud, a production system.

###Devel Config

A simple way to create an end-to-end development environment is to use
[CloudLab](https://www.cloudlab.us/) to host a basic OpenStack
cluster, and then link this cluster to the running XOS you just
installed. To set up XOS with an OpenStack cluster hosted on CloudLab:

* If you don't already have a CloudLab account, you can go to `http://cloudlab.us` and join project **xos**.
* Create your CloudLab experiment using the *OpenStack* profile.  The Juno and Kilo releases should both work.  
We recommend that, under "Advanced Parameters" in the profile, you choose to "Disable Security Group Enforcement".  Instantiate it on the *CloudLab Clemson* or *CloudLab Wisconsin* clusters.
* Login to the *ctl* node of the experiment and run the following:

```
$ git clone https://github.com/open-cloud/xos.git
$ cd xos/xos/configurations/devel
$ make
```

The Makefile will build the XOS Docker image, run it in a container, 
and configure XOS to talk to the OpenStack cluster on CloudLab.  You can 
reach the XOS GUI on port 9999 on the *ctl* node.

Assuming everything has worked, you should be able to create a slice and 
launch a VM.  You can log into the VM from the *ctl* node as the *ubuntu* user,
at the first IP address shown in the *Instances* view for the slice (typically on the
10.11.0.0/16 subnet).

###Test Config

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

###Bash Config

The Bash configuration may be found in *xos/configurations/bash*. Its
purpose is to serve as an interactive environment for development,
with a shell-based interface. After the Makefile has finished
executing, the user will be dropped into a bash shell inside of the
container. Postgres will be running and the XOS data model will be
populated with minimal data.

The XOS UI will not be running, but it can be started by typing 

```
$cd /opt/xos; scripts/opencloud runserver
```

###Frontend-Only Config

The Frontend-Only configuration is aimed at developers who are working
on the XOS user interface, but do not need a functioning
Synchronizer. As such, it does not require Cloudlab or any active
OpenStack deployment. While the UI is functional, this configuration
necessarily imposes the limitation that Instances will not be
instantiated.

Additionally, as the Synchronizer is not running, *model_policies*
will not be executed.

###OpenCloud Config

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

```
/opt/xos/core/static
/opt/xos/core/xoslib/static
# note that the following two paths may vary depending on Linux distribution
/usr/local/lib/python2.7/dist-packages/Django-1.7-py2.7.egg/django/contrib/admin/static
/usr/lib/python2.7/site-packages/suit/static
```

The following commands can be used to start, stop, and restart the
uwsgi server:

````
start: cd /opt/xos; uwsgi --start-unsubscribed /opt/xos/uwsgi/xos.ini
stop: uwsgi --stop /var/run/uwsgi/uwsgi.pid
restart: uwsgi --reload /var/run/uwsgi/uwsgi.pid
```

###Debugging Configurations

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
container's ID (with "docker ps" if the container is still executing,
or "docker ps -a" if the container has exited). Then use that ID to
execute "docker logs ID".  Several of the configurations include a
Makefile target "make showlogs" that automatically executes these
steps.

Additionally, it may be necessary for a developer to sometimes attach
a shell to a running background container to interact with it. This
may be done first looking up the container ID, and then executing
"docker exec -t -i ID bash".

###Building XOS Without the Configuration System

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

```
$ docker build -t xos .
$ docker run -t -i -p 8000:8000 xos
```

XOS will start automatically and you will see its log output in the shell window. 
You can access the XOS login at *http://server:8000*, where *server*
is the name of the server hosting the Docker container.  Login credentials are 
*padmin@vicci.org*/*letmein*.

Note that the above steps result in a running XOS, but without any
backend resources. This is sufficient for working on the data model
and views, but not for actually managing cloud infrastructure.

Information on bringing up a local OpenStack cluster is given in Section
[Installing OpenStack](../3_operator/#install-openstack) of the
Operator Guide.  Information on connecting XOS to an operational
OpenStack cluster is given in Sections [Administering a
Deployment](../1_user/#admin-deployment) and [Administering a
Site](../1_user/#admin-site) of the User's Guide. These two sections
explain how to configure a Deployment to know about a set of OpenStack
clusters and how to configure a Site to know about a set of Nodes,
respectively.

##REST API

A REST API and associated
[documentation](http://portal.opencloud.us/docs/) is auto-generated
from the data model.

The REST API may be used via a number of programming languages. Below are a few examples using common tools and languages:

###command line via curl

     # use your XOS username and password for my_email and my_password
     curl -H "Accept: application/json; indent=4" -u my_email:my_password http://portal.opencloud.us/xos/users/

###python

    import requests
    admin_auth=("my_email", "my_password")   # use your XOS username and password
    users = requests.get("http://portal.opencloud.us/xos/users", auth=admin_auth).json()
    for user in users:
         print user["email"]

##xoslib

xoslib is a client/server library for extending XOS. The server side
of the library defines a [REST API](http://portal.opencloud.us/docs/)
that extends the XOS core with new xoslib-defined objects. This REST
API uses HTTP as a transport mechanism and may be used from a variety
of clients and languages.

To facilitate development in Javascript, xoslib also includes an
extensive client library based on Backbone.js and Marionette.js.
Backbone.js provides an efficient event-driven interface, where the
xoslib's client-side library fetches models from the server-side, and
notifies client programs when data has been fetched for display to the
user. Portions of the XOS user interface (specifically, [User
Views](../1_user/#user-views)) are implemented on top of this library.

##<a name="adding-views">Adding Views to XOS</a>

XOS is designed to be extensible -- to provide an explicit means to
incorporate services deployed on XOS back into XOS so they are readily
available to other users. Our goal is to build a diverse collection of
services that can be accessed through the XOS programmatic
interface. This is done by extending the XOS data model with objects
that represent the available services.

This collection of services, in turn, defines a range of functionality
that can be combined in different ways to provide customized Views
targeted at different user communities and usage scenarios. These
views are similar to scripts -- small programs that leverage the
underlying set of commands to create a new function for a particular
purpose, where in XOS's case, the "underlying set of commands"
corresponds to operations on XOS's data model. The data model includes
both core objects (e.g., Users, Slices, Instances, Roles, Sites,
Deployments, Services) and objects that represent various services
(e.g., Syndicate Volumes, HPC OriginServers, RequestRouter
ServiceMaps). A given View can operate on any combination of these
objects.

We refer to these scripts as Views because they represent a particular
perspective of XOS, but they also happen to be implemented as a
tailored view in Django, extending Django's standard template view.
When we need to disambiguate between the XOS abstraction and the
Django implementation, we refer to the former as a User View and the
latter as a Django View.

There are four aspects to creating a typical user view:

1. Create a Django template that contains the HTML that will be
displayed in the user's browser.

2. Display static data using context variables

3. Create a javascript file that contains the code that will execute
the client side of the view
  * fetch dynamic data from xoslib
  * commit modified data back to xoslib

4. (optional) extend the XOS api with new objects

5. Register your view with XOS so that users can access it on their
homepage.

The rest of this document walks through these steps, creating a "Hello
World" view as an illustration. It uses
http://devel.opencloud.us:8000 as the example server running
Django/XOS, and assumes the software at
http://github.com/open-cloud/xos.git has been installed on that
node.

###Create a Django Template

A template contains the HTML and javascript that presents the view to
the user. A view can be almost entirely HTML or almost entirely
javascript. Most of our example templates are a combination of the
two. In the git repository, templates for views are stored in
xoslib/dashboards/. In a given XOS installation, you'll find these
views in /opt/xos/xoslib/dashboards/.

There is no need to worry about building a full HTML page with \<head\>
and \<body\> and all the usual boilerplate. XOS automatically handles
those parts, as well as including the navigation panel, title bar, and
so on. The view need only include HTML that contains what needs to be
shown to the user. For example, file helloworld.html contains:

```
<!-- /opt/xos/templates/admin/dashboard/helloworld.html -->
<div>Hello, {{ user.firstname }} {{ user.lastname }}.</div>
<div>This is the hello world view. The value of foobar is {{ foobar }}.</div>
<div id="dynamicTableOfInterestingThings"></div>
<p>Type a new description for the the first slice in the collection:</p>
<input type="text" name="newDescription" id="newDescription">
<input type="button" name="submitNewDescription" value="submit new description"  id="submitNewDescription">

<script src="{{ STATIC_URL }}/js/vendor/underscore-min.js"></script>
<script src="{{ STATIC_URL }}/js/vendor/backbone.js"></script>
<script src="{{ STATIC_URL }}/js/xoslib/xos-backbone.js"></script>

<script src="{{ STATIC_URL }}/helloworld.js"></script>  
```

It illustrates four things:

* A comment, "<!-- ... -->"

* Static text, such as "this is the hello world view"

* Django context variables, such as {{ user.firstname }} and {{ foobar }}

* An empty div that we will populate using javascript,
  "#dynamicTableOfInterestingThings"

* An input box where the user can type some stuff and a submit button
  to go with it

* Some \<script\> tags that load javascript for xoslib and its
  dependencies

* A \<script\> tag that loads javascript for the helloworld view
  (helloworld.js).

At this point, we are able to display the user view in the browser. To
confirm that it's working, navigate to
http://devel.opencloud.us:8000/admin/dashboard/helloworld/

This will cause your user view to be displayed by itself, without
invoking the home page customization machinery. This is useful for
debugging and testing your view in isolation before publishing it to
other users. Your browser should display:

```
This is the hello world view. The value of fubar is .
```

###Display Static Data Using Context Variables

XOS includes some context variables that are built in. One example of
this is "user", which represents the currently logged in user. Context
variables are useful for values that we don't expect to change while
the user is looking at the view.

[TODO: suggest the user use xoslib instead of context variables?]

The easiest place to add more Django context variables is in
/opt/xos/core/dashboard/views/view_common.py:
getDashboardContext(). This function returns the set of context
variables that are usable by the views:

```
def getDashboardContext(user, context={}, tableFormat = False):
   context = {}

   userSliceData = getSliceInfo(user)
   if (tableFormat):
       context['userSliceInfo'] = userSliceTableFormatter(userSliceData)
   else:
       context['userSliceInfo'] = userSliceData

   context['cdnData'] = getCDNOperatorData(wait=False)
   context['cdnContentProviders'] = getCDNContentProviderData()

   (dashboards, unusedDashboards)= getDashboards(user)
   unusedDashboards=[x for x in unusedDashboards if x!="Customize"]
   context['dashboards'] = dashboards
   context['unusedDashboards'] = unusedDashboards

   return context
```

To add a new context variable, just insert a new statement near the
bottom of this function, before the return statement:

```
context['foobar'] = "value_of_foobar"
```

Now we have a new context variable called "foobar", and any user view
template can display that variable by using the syntax "{{ foobar }}".
Restart the Django server, open (or refresh) the helloworld view in
your browser, and "value_of_foobar" should now be displayed in the
appropriate place.

###Create Javascript to Use with the View

Now that we've created a template, let's create some javascript to
make our view more dynamic. The javascript should be placed in
/opt/xos/core/xoslib/static/js/helloworld.js.

[Note: we could have placed the javascript in the html template using
\<script\>\</script\> tags, but placing it in a separate .js file is
cleaner for large views, and eliminates the potential of django's
template language modifying the javascript.]

###Fetching Dynamic Data with xoslib

A javascript library, xoslib, has been developed to facilitate
fetching dynamic data using for views. xoslib is based on backbone.js
and uses XOS's REST API as a transport mechanism. In addition to stock
objects (Slice, Node, ...), xoslib is also designed to make it
relatively easy for developers to extend the API with custom objects
that implement increased functionality.

backbone.js is an asynchronous, event-driven library. A view will
typically request some data, and an event will show up at a later time
indicating that the data has been received. Let's update helloworld to
display a list of slice names inside of #tableOfItnerestingThings:

```
// helloworld.js
function updateHelloWorldData() {
    var html = "<table table table-bordered table-striped>";
    for (var slicekey in xos.slices.models) {
        slice = xos.slices.models[slicekey]
        html = html + "<tr><td>" + slice.get("name") + "</td><td>" + slice.get("description") + "</td></tr>";
    }
    html = html + "</table>";
    $('#dynamicTableOfInterestingThings').html(html);
}

$(document).ready(function(){
    xos.slices.on("change", function() { updateHelloWorldData(); });
    xos.slices.on("remove", function() { updateHelloWorldData(); });
    xos.slices.on("sort", function() { updateHelloWorldData();  });

    xos.slices.startPolling();
});
```

The above defines a callback function, updateHelloWorldData(), that
displays slice information in a table. We could have chosen to use
datatables or marionette or underscore templates to render this
information, but we chose to do it by manual HTML demonstration to
demonstrate the basic technique.

xos.slices is a backbone.js collection. It contains an array of models
that can be accessed by enumerating xos.slices.models. Each model is a
slice object and contains a name, description, etc.

We register three event handlers that watch for events on xos.slices
and then call our updateHelloWorldData() function to update the
table. These are change, remove, and sort.

* *change* -- responds to changes in the models of the collection, such
  as the name of a slice changing

* *remove* -- responds to items being removed

* *sort* -- responds to the sort that happens on a collection after it
  has been downloaded from the server. Also implicitly responds when
  items are added to the collection (because this triggers a new sort)

Finally, we need to tell backbone to fetch the collection. This could
be done with xos.slices.fetch(), but xoslib implements a helper method
called 'startPolling()' which will automatically cause fetches to
occur in 10-second intervals.

###Committing Modified Data Back to XOS

Models in xoslib have a save method that can be used to commit
modified data back to XOS. Let's modify helloworld so that when the
user presses the submit button, we change the description of the first
slice in the list:

```
// helloworld.js
$(document).ready(function() {
    $('#submitNewDescription').bind('click', function() {
        newDescription = $("#newDescription").val();
        xos.slices.models[0].set("description", newDescription);
        xos.slices.models[0].save();
    });
});
```

The above binds a click handler to the submitNewDescription
button. The click handler fetches the new description from the input
box, and calls set() on the model. It then calls save() on the
model. The save method will cause backbone.js to issue the appropriate
REST calls to change the description of the first slice.

###Extend the XOS API with New Objects

Sometimes we feel the need to add new objects to the XOS API. While
the REST API does a pretty good job of exposing the data model,
fetching each individual bit of it separately may not be the most
efficient means of displaying a page.

xoslib is designed with the ability to easily add new REST API
functions. These are done in two steps:

1. Create an "object" that contains the new files. These go in
/opt/xos/core/xoslib/objects/.

2. Create some REST API views that expose the new object via
django. These go in /opt/xos/core/xoslib/methods/

How to do this is best demonstrated by taking a look at an
example. SlicePlus is an object that was created to use with the
developer view, and looks like a slice object but with some additional
fields that tabulate the number of instances and sites used by the
slice, as well as the role of the current user. Take a look at
sliceplus.py in the objects/ and methods/ directory and it should be
relatively straightforward how this was done.

Although SlicePlus extends an existing object (Slice) with new
methods, it's also possible to create entirely new objects that aren't
necessarily related to existing objects, or that relate to a
collection of existing objects.

###Register a View with XOS

Up to this point, the URL we've been using to display our user view
only displays the helloworld user view and no other views. The
helloworld user view hasn't been integrated with XOS yet. To do that
we need to create a data model object.

There are two different views you can register, template-based views
(like out helloworld example), and iframe-based views.

####Template-based Views

1. Open your browser to
http://devel.opencloud.us:8000/admin/core/dashboardview/

2. You should see a list of existing dashboard views (Slice
Interactions, Tenant, Developer, ...)

3. Press the green <Add Dashboard View> button

4. In the new screen that pops up, enter the following:
   * Name: "hello world"
   * Url: "template:helloworld"

5. Press the <Save> button.

Note that you can't put a comma in a view name. That's why we made the
name "hello world" instead of "hello, world".

If you now go back to the home page, and use the <Customize> tab,
"hello world" should now be listed in the available dashboard
views. You can add it to your selected dashboard views, and it will be
available as part of your home page.

####iframe-based Views

iframe-based views allow you to insert and arbitrary web page into the
XOS dashboard. This is done by specifying a "http://" url
instead of a "template:" url. For example,

1. Open your browser to
http://devel.opencloud.us:8000/admin/core/dashboardview/

2. You should see a list of existing dashboard views (Slice
Interactions, Tenant, Developer, ...)

3. Press the green <Add Dashboard View> button

4. In the new screen that pops up, enter the following:
   * Name: "ON.LAB"
   * Url: "http://onlab.us/"

5. Press the <Save> button.

Use the <customize> tab as before, add the "ON.LAB" view, and you'll
now have a tab that displays the ON.LAB home page.

##<a name="adding-services">Adding Services to XOS</a>

XOS provides a collection of abstractions, interfaces and mechanisms
that enable Cloud services to interoperate reliably and efficiently.
By using XOS, the service developer focuses his or her attention on
the core value provided by a service, not binding it to the underlying
mechanisms of the Cloud substrate. This section describes the
framework into which services are plugged into XOS.

### Design Overview

The work of defining a new service is twofold: (1) designing
abstractions and adding them to XOS through a standard interface (the
data model, implemented in Django), and (2) plumbing those
abstractions through to the actual mechanism that implements the
service (the service controller). For example, if the value provided
is that of a storage service, then the first task is to capture the
user-facing configuration of the storage service as a set of data
values that represent it, and the second task is to link this
representation to the actual implementation of the service.

The first task is an exercise in writing Django models, which this
guide discusses in Section [Data Modeling
Conventions](#model-conventions).

The second task involves extending the XOS Synchronizer, which is
responsible for enacting the state recorded in the XOS data model
(i.e., configuring and controlling the underlying service by invoking
operations on its controller). All changes made to the data model --
including the addition of new objects, updates to existing objects,
and the deletion of objects -- are intercepted by the Synchronizer. The
Synchronizer is an event-driven program written in Python. Every time the
Data Model changes, the Synchronizer receives a notification, upon which
it queries the Data Model to retrieve the set of updated objects.

Although we have been describing the Synchronizer as a monolithic entity,
it is actually a modular system, consisting of a Synchronizer Framework
and a set of Synchronizer Instances. Each Synchronizer Instance is associated
with some subset of the Data Model, and acts upon some subset of the
imported service controllers. For example, there is a Synchronizer
Instance that activates User, Slice, Instances, and Network objects by
calling OpenStack controllers (Nova, Neutron, KeyStone); another that
activates CDN-related objects by calling the HyperCache controller;
yet another that activate file system volumes by calling the Syndicate
controller; and so on. In general, any service-related objects in the
data model that need to interact with a low level platform must
include a service-specific Synchronizer Instance. 

The XOS Data Model consists of the set of configuration values that
need to be set for the system to be operational. This configuration is
organized as a graph of interconnected Object-Oriented Classes (called
Models). The graph of the core models is illustrated below.

In this graph, arrows indicate relations between models. The Instance
model is related to and depends on the Slice model. Concretely, it
means that an Instance (VM on a node) is associated with a Slice (name
for a global resource allocation), or even more specifically, that
Instance objects have a field of type Slice.  The core data model should
satisfy the needs of most services, but if not, then it can be
extended.

XOS currently has five Synchronizer Instances: (1) an OpenStack Synchronizer,
(2) an Amazon EC2 Synchronizer, (3) a Syndicate Storage Synchronizer, (4) a
High-Performance Cache (HPC) Synchronizer, and (5) a Request Router
Synchronizer.  Each of these Synchronizer reads from the same data model, but
administers a different set of backend resources. The OpenStack
Synchronizer uses OpenStack to create, manage and tear down VMs. The EC2
Synchronizer helps manage instances on Amazon EC2. Syndicate, HPC, and
RequestRouter are service-specific Synchronizer.  In this document, we
will use the Amazon EC2 Synchronizer to illustrate how to develop a new
instance.

*[Note: The Synchronizer Framework, which is common across all Synchronizer
 Instances, is currently embedded in the OpenStack Synchronizer Instance.
 Our plan is to lift it out of this instance and into the core of
 XOS.]*

### Introduction to a Synchronizer

For simplicity, we sometimes say Synchronizer in lieu of Synchronizer
Instance.

Much like a declarative programing language, Synchronizers are
goal-oriented (as opposed to task-oriented.) They consider the
contents of the data model to be the declaration of the "goal state"
of the system, and then take the necessary steps to steer the system
into that target state.

The implementation of each Synchronizer is lock-free, transaction-free,
and therefore stateless. Every time a Synchronizer runs, it calculates
the delta between the state of the backend system and the state of the
Data Model. It then applies this delta to the backend system, bringing
its state up to date with the state of the data model.

Each Synchronizer is an Event-driven application that monitors the Data
Model for changes. When a change is detected in the data model, XOS
notifies the Synchronizer using an XMPP-based notification system, called
Feefie. The Synchronizer then queries the data model to calculate the
delta on the basis of which it updates the state of the system.

The actions of a Synchronizer are a direct consequence of the state of
the data model. The data model has several properties that make such
actions execute properly. As mentioned in the previous section, one of
the properties is a dependency structure implied by the relationships
between various models. Each Synchronizer ensures that actions are
executed in an order that is consistent with these dependencies. For
example, since the Instance model depends on the Slice model, the
Synchronizer guarantees that an Instance is only added to the System if a
corresponding Slice already exists.

### Developing a Synchronizer

We have developed the Synchronizer Framework in a way that relieves the
developer of each Synchronizer Instance of having to re-implement certain
core functionality. By using this framework, the following benefits
are automatically realized:

* Robust error-correcting synchronization that is lock-free and
  transaction-free

* XMPP-based event notifications

* Automatic enforcement of Data Model invariants (e.g., model
  dependencies)

* Optimized delta computation based on timestamps

* Error reporting -- errors encountered by a Synchronizer are reported
  to the user via the web interface

* Concurrent execution of independent steps

* Automatically inferred object-level dependencies 

The remainder of this section uses the Amazon EC2 Synchronizer as an
example to illustrate how a Synchronizer Instance works. First, a look at
the core. You can skip the next few paragraphs if you are not
interested in finding out about the inner details of the framework.

Let's start with the main source files:

* dmdot -- This is a tool that generates the dependency graph
  contained in the Data Model. You can use it with the -dot argument
  to generate a dot representation which looks like the illustration
  in the Data Model section. Or you can run it without arguments to
  generate the dependency graph used by the Synchronizer Framework to
  ensure that actions taken by the individual Synchronizer Instances are
  ordered correctly.

* mainloop.py -- This file contains the main run loop of the Synchronizer
  Framework. It is the mechanism that receives events, dispatches
  actions, and orchestrates the components of the Synchronizer.

* event_manager.py -- This file is the part of the Synchronizer Framework
  that interacts with the Feefie notification system, used to transmit
  events from the Data Model to the Synchronizer Framework.

* toposort.py -- This program sorts the actions in the Synchronizer
  Framework in accordance with the dependency graph generated by
  dmdot.

The remainder of the files contain peripheral support code, or base
classes to be subclassed by your Synchronizer Instance.

### Synchronizer Steps

Within the *Synchronizer/* subdirectory of the XOS repository, you will
find a *steps/* subdirectory. This directory contains the actual
"payload" of that Synchronizer Instance -- the actions that read state
from the XOS data model and apply them to the backend system, to the
point that the backend system has been brought up-to-date with that
state.

Thus, your task in implementing your specific Synchronizer Instance boils
down to providing a set of sync steps, each step corresponding to a
set of models in the Data Model.

A Synchronizer Step is a Python class that subclasses the SyncStep class
in the main Synchronizer directory, and exports a documented interface.
Here is an example of a Synchronizer Step:

```
    class SyncInstance(SyncStep):
        provides=[Instance]
        Synchronizer=[Instance]
        requested_interval=3600

        def fetch_pending(self):
            current_sites = Site.objects.all()
            zones = aws_run('ec2 describe-availability-zones')
            available_sites = [zone['ZoneName'] for zone in zones]

            new_site_names = list(set(available_sites) - set(zones))

            new_sites = []
            for s in new_site_names:
                site = Site(name=s,
                        login_base=s,
                        site_url="www.amazon.com",
                        enabled=True,
                        is_public=True,
                        abbreviated_name=s)
                new_sites.append(site)

        return new_sites
   
     def sync_record(self, site):
        site.save()
```

The mandatory interface of a Step, which is used by the framework to
hook Steps into the core of the Synchronizer Framewrok, consists of the
following instance properties (three variable and three methods):

* **provides**: The models that this step provides, and the ones that
  define the position at which the step is placed when enacting the
  current data model.

* **observes**: The models that this step actually enacts new instances
  of. Usually the same as provides, but may be an auxilary model that
  holds state that complements the main model provided.

* **requested_interval**: The intervals at which this model is enacted. 
  For slow executing models, set this to a high value so that it 
  does not interrupt every run of the Synchronizer.

* **fetch_pending**: The method that fetches the pending set of objects
  that have to be enacted. There is a default implementation that uses
  internal timestamping and other mechanisms to automatically fetch
  the latest set of steps. That said, service implementers can define
  their own version. The variables that they get access to are
  "enacted" and "updated." Enacted is the timestamp of the last time
  an instance was instantiated.

```
        def fetch_pending(self):
            new_sites = GetSitesAddedSinceLastRan()
            return new_sites
```

The fetch_pending function can use these bookkeeping variables to
fetch the set of changed objects:

```
        new_objects = Instance.objects.filter(Q(enacted__lt=F('updated')) | Q(enacted=None))
```

The line above is typical, and can be copied and pasted into most
Synchronizer implementations. It retrieves the set of objects that have
never been enacted (enacted = None) or that have been updated since
they were last enacted (enacted <= updated).

* **sync_record**: The main method that invokes operations on the
  backend controller to bring it in sync with the data model.

```
         def sync_record(self, site):
             site.save()
```

Once the set of changed objects have been computed, the Synchronizer
invokes the appropriate set of backend operations and populates the
objects with links to the back end. This functionality is contained in
the sync_record function.  For example, the SyncInstances step of the
EC2 Synchronizer creates EC2 instances in response to the addition of
Instances.

```
instance_type = instance.node.name.rsplit('.',1)
instance_sig = aws_run('ec2 run-instances --image-id %s --instance-type %s --count 1'%(instance.image.image_id, instance_type))
```

And then it associates the ‘instance_id’ an identifier that uniquely
identifies the instance in the data model, with the Instance.

```
    instance.instance_id = instance_sig['Instances'][0]['InstanceId']
    instance.save()
```

It is essential that models in the Data Model be linked with actual
facilities in the backend, so that they may be updated or deleted at a
later point. The sync_record function need not update the value of the
enacted field in the model. This operation is performed by the
Synchronizer Framework core.

###Synchronizer Steps are Idempotent

As shown in the previous sections, the bulk of the work involved in
implementing a Synchronizer Instance lies in providing a set of Synchronizer
Steps by placing them in the *Synchronizer/steps/* directory within the
repository. All Steps at this location are automatically picked up by
the Synchronizer Framwork when it runs, and sewn together into the
pre-computed dependency graph generated by dmdot.

There is one additional requirement essential for a Synchronizer to
function correctly. The lock-free and transaction-free operation
depends on an important invariant that it is your responsibility to
maintain when implementing Steps.

Synchronizer Steps are Idempotent Operations. They may run multiple times,
and it is your responsibility to ensure that a subsequent execution
does not cause the system to go into an error state.

### Internal Steps

A Synchronizer has two kinds of steps: Internal Steps and External
Steps. An Internal Step is one that responds to specific changes in
the Data Model, and can make use of the updated and enacted variables
to discover the set of changed objects in the Model.

Internal steps are typically fast since they operate on the set of
object that have been added or updated since the last execution of the
step. When there are no such objects, the step is a no-op and simply
yields execution back to the Synchronizer, so that the next Step can run.

The SyncInstance step illustrated in the previous section is an example
of an Internal step.

Whenever possible, implement Steps as Internal steps.

```
    class SyncSites(SyncStep):
        provides=[Site]
        requested_interval=3600

        def fetch_pending(self, deletion):
            deployment = Deployment.objects.filter(Q(name="Amazon EC2"))[0]
            current_site_deployments = SiteDeployments.objects.filter(Q(deployment=deployment))

            zone_ret = aws_run('ec2 describe-availability-zones')
            zones = zone_ret['AvailabilityZones']

            available_sites = [zone['ZoneName'] for zone in zones]
            site_names = [sd.site.name for sd in current_site_deployments]

            new_site_names = list(set(available_sites) - set(site_names))

            new_sites = []
            for s in new_site_names:
                site = Site(name=s,
                            login_base=s,
                            site_url="www.amazon.com",
                            enabled=True,
                            is_public=True,
                            abbreviated_name=s)
                new_sites.append(site)

            return new_sites

        def sync_record(self, site):
            site.save()
```

Sometimes, the authoritative state for a given model is not entered by
users, but rather, is sourced from the back end. Usually, this happens
when a model represents the state of actual physical resources. In the
example above, we illustrate the SyncSites Step in the Amazon EC2
Synchronizer.

Rather than bringing up Sites in Amazon EC2 when a new Site is created
in the XOS data model, this Step populates the XOS data model with
Sites corresponding to Availability Zones in EC2. Unfortunately, there
is no way to query Amazon Web Services for "the set of Availability
Zones added since last time." So this Step needs to fetch all of the
Availability Zones every single time that it runs.

```
    zone_ret = aws_run('ec2 describe-availability-zones')
    zones = zone_ret['AvailabilityZones']
```

This exhaustive fetch makes External Steps resource-intensive and slow
to run. As a rule of thumb: Steps that do not fetch recently updated
objects using the enacted and updated variables are External Steps.

External Steps should have their requested_interval set to hours or
days, so that they do not block Internal steps. This is because the
Synchronizer is single-threaded and Internal steps are expected to be
responsive.

### Deletions

Synchronizers handle deletions in the same way as they handle
synchronization. When an object is deleted in the data model, it is
simply marked as deleted. The fetch_pending method in that case
fetches the set of such deleted objects and passes it on, instead of
sync_record, to a method called delete_record. It is the task of
delete_record to pass the deletion of the record on to the back
end. For example, if an Instance is deleted, then the method should
delete and clean up the corresponding VM, along with recovering any
other resources, such as volumes associated with that VM. If this
method returns successfully, then XOS automatically takes care of
actually deleting it from the data model.

##Django Migration Notes

This section discusses Django database migration. For a general
overview of migration considerations, see
https://docs.djangoproject.com/en/1.7/topics/migrations/

When adding a new field to the data model, execute the following two
steps:

```
python manage.py makemigrations
python manage.py migrate
```

The first step creates the migration scripts automatically. These will
be placed in core/migrations/, hpc/migrations/,
syndicate_service/migrations/, and so on. The second step
automatically runs all migration scripts as necessary to bring the
data model from any previous state up to the current state.

Migration scripts are numbered, and they have dependency information
encoded into them. For example, migration script 0003 usually depends
on script 0002, which depends on 0001, and so on. For this reason it
is important that you check your migration scripts into git, and that
you pull others' migration scripts before running makemigrations
(otherwise multiple developers will end up creating conflciting or
redundant scripts). Note that these migration scripts work much like
the SQL migrations used for MyPLC.

Notable changes in Django 1.7:

1. "syncdb" is now a synonym for "migrate"

2. "syncdb" (and migrate) no longer automatically install
initial_data.json. To compensate, scripts/opencloud has been modified
to perform a "loaddata" whenever the database is created for the very
first time.

3. django.setup needs to be called by external scripts before using
models. xos-backend.py has been modified to do this
automatically.

Upgrading from Django 1.5 to Django 1.7 was messy, so it could be
messy inside of peoples' private XOS installations as well. It may be
necessary to fully uninstall Django and it's related packages

```
pip uninstall django djangorestframework django-filter django-timezones 
django-crispy-forms django-geoposition django-extensions django-suit 
django-evolution django-bitfield django-ipware django-encrypted-fields
```

and then reinstall them

```
pip install django==1.7 djangorestframework django-filter django-timezones 
django-crispy-forms django-geoposition django-extensions django-suit 
django-evolution django-bitfield django-ipware django-encrypted-fields
```

Django's migration will get upset if you attempt to have an unmigrated
app that depends on a migrated app, for example, in Foreign Key
relationships or model inheritance. Most of our apps have such a
dependency between the 'Service' object in the app and
core.Service. When creating new apps, run 

```
python manage.py makemigrations <appname>
```
to create the initial migration for the app.


##<a name="model-conventions">Data Model Conventions</a>

This section outlines our modeling conventions and how they map onto
Django.

###Modeling Basics

We first establish Django terminology, and how it relates to common
object-related terminology. An Object Class is called a Model in
Django. A Django Model consists of a set of Fields (equivalent to
saying an Object Class is defined by a set of Attributes). Each Django
Field has a Type and each Type has a set of Attributes. Some of these
Attributes are core (common across all Types) and some are
Type-specific. Relationships between Models are expressed by Fields
with one of a set of distinguished relationship-oriented Types (e.g,
OneToOneField). Finally, an Object is an instance (instantiation of) a
Model, where each Object has a unique primary key (or more precisely,
a primary index into the table that implements the Model). By
convention, that index/key should be the Django AutoField id which is
automatically created for any Model that has not identified a separate
unique primary key. The Django default primary key is always "id" for
system level tables, and "pk" for model tables.

####Naming Conventions

Model names should use CamelCase without underscore. Model names should always
be singular, never plural. For example: Slice, Network, NetworkTemplate.

Sometimes a model is used as a through relation to relate two other models, and
should be named after the two models that it relates. For example, a model that
relates the Controller and User models should be called ControllerUser.

Field names use lower case with underscores separating names. Examples of
valid field names are: name, disk_format, controller_format.

####Field Types

There are various built in Fields that may be specified at Class
declaration. The basic field types commonly in use with XOS:

| FieldType          | Why to Use It?     |
|--------------------|--------------------|
| BooleanField       | Displays as a checkbox in GUI; limits input to valid possibilities. If None is a valid option, please use *NullBooleanField* which uses default widget of choice: None, Yes, No.|
| CharField          | Allows for String representation, additional required attributes are "max_length". This should be used for small to large size strings. For mulitiple sentences, use TextField.|
| DateTimeField      | Special field that maps to python DateTime. May set optional attributes of *auto_now* (update field each time it is saved), *auto_add_now* (update field when created only) |
| EmailField         | Automatically checks for well formed Emails based on RFC3696/5321. Note that for full compliance we need to set the length to 254; default is 75. [**This is a bug in our model.**] |
| FloatField         | Used to represent a real number; default widget is TextInput.|
| IntegerField       | Used to represent Integer values; default widget is TextInput. [**We should review whether the IntegerFields should be changed to be PositiveIntegers, or PositiveSmalls accordingly.**]|
| GenericIPAddressField | Able to validate IPv4 and IPv6 flavors of addresses; default widget is TextInput.|
| URLField           | Verifies well formed URL, max_length is defaulted to 200.  GUI convenience is that the value of the field is displayed as a clickable link above the input widget.|
| SlugField          | Used to represent a short label for something containing only letters, numbers, underscores or hyphens. This should be used for the name of Tags so that there is a better chance of promoting the Tag to be an actual field attribute once the model has been solidified.|
	
####Core Field Attributes

List of Field level optional attributes currently in use:

| Attribute          | Effect             |
|--------------------|--------------------|
| verbose_name="..." | Provides a label to be used in the GUI display for this field. Differs from the field name itself, which is used to create the column in the database table.|
| verbose_name_plural="..." | Way to override the verbose name for this field.|
| help_text="..." | Provides some context-based help for the field; will show up in the GUI display.|
| default=... | Allows a predefined default value to be specified.|
| choices=CHOICE_LIST | An interable (list or tuple). Allows the field to be filled in from an enumerated choice. For example, *ROLE_CHOICES = (('admin', 'Admin'), ('pi', 'Principle Investigator'), ('user','User'))*|
| unique=True |	Requires that the field be unique across all entries.|
| blank=True | Allows the field to be present but empty.|
| null=True | Allows the field to have a value of null if the field is blank.|
| editable=False | If you would like to make this a readOnly field to the user.|
	 
List of Field level optional attributes we should not use, or use judiciously:

| Attribute          | Why                |
|--------------------|--------------------|
| primary_key        | Some of the plugins we use, particularly in the REST area, do not do well with CharField's as the primary key. In general, it is best to use the system primary key instead, and put a *db_index=True, unique=True* on the CharField you would have used.|
| db_column, db_tablespace | Convention is to use the Field name as the db column, and use verbose_name if you want to change the display. For tablespace, all models should be defined within the application they are specified in -- overwriting the tablespace will make it more challenging for the next developer to find and fix any issues that might arise.|

List of Field level optional attributes we will likely need/want to
use:

| Attribute          | Effect             |
|--------------------|--------------------|
| db_index=True      | Allows for quicker queries if you are going to order by, or filter by, the specified field.|
| error_messages={...} | Pass in a dictionary of error keys, and the message you want to display if the error occurs.  For example: null, blank, invalid, invalid_choice, and unique. Other error messages may be present per FieldType.|
| validators=[...]      | Allows a list of validators to be run before committing the value of this field to the model (https://docs.djangoproject.com/en/dev/ref/validators/).|

####Expressing Relationships

There are a few different types of Relationship based fields offered
in Django.

| FieldType          | When to Use it     |
|--------------------|--------------------|
| ForeignKeyField    | Used to represent a 1-to-Many relationship. For example: Instance's may have 1 and only 1 Node; Node's may have 0 or more Instances. Can also be used to represent recursive relationships for the same object types by providing "self" as the relationship (first position) parameter.|
| ManyToManyField    | Used to represent an N-to-N relationship. For example: Deployments may have 0 or more Sites; Sites may have 0 or more Deployments.|
| OneToOneField      | Not currently in use, but would be useful for applications that wanted to augment a core class with their own additional settings. This has the same affect as a ForeignKey with unique=True.  The difference is that the reverse side of the relationship will always be 1 object (not a list).|
| GenericForeignKey | Not currently in use, but can be used to specify a non specific relation to "another object." Meaning object A relates to any other object. This relationship requires a reverse attribute in the "other" object to see the relationship -- but would primarily be accessed through the GenericForeignKey owner Model. For example, https://docs.djangoproject.com/en/dev/ref/contrib/contenttypes/#id1. The nuances of these relationships is brought about by the additional optional attributes that can be ascribed to each Field.|

Note that we should likely convert our Tags to use GenericForeignKey
so that all objects can be extensible during development, but then
converted/promoted to attributes once the model has stabilized.

####Optional Attribute Side Effects

| Attribute          | Key Type           | Why                |
|--------------------|--------------------|--------------------|
| limit_choices_to   | ForeignKeyField, ManyToManyField | Not currently in use. Good to limit the possible selection choices for a remote relationship by some constraint.  Constraints may either be expressed by a dictionary of lookups, or by a Q object. (See https://docs.djangoproject.com/en/dev/topics/db/queries/#complex-lookups-with-q.) Example of data limit for another object: *limit_choices_to = {'pub_date__lte': datetime.date.today}*|
| relate_name        | ForeignKeyField, ManyToManyField | The name to use for the relation back to this one.  This is extremely useful both to provide additional context in the way the objects are related but also when you have the same objects and object types related in different contexts to the same instance. It allows for lookups and incorporation into Admin Forms based on the relationship. For example, *SliceMembership: user = models.ForeignKey('User', related_name='slice_memberships') slice = models.ForeignKey('Slice', related_name='slice_memberships')*. "+" is a special meaning in this field. As the only character (or the trailing character of the field) it has the effect of eclipsing the backward relationship so that it is unseen by the related object.|
| on_delete          | ForeignKeyField | Default behavior is to behave like "ON DELETE CASCADE." This may be turned of so long as the *null=True, blank=True* options are also set. For example, *user = models.ForeignKey(User, blank=True, null=True, on_delete=models.SET_NULL)*|
| symmetrical        | ManyToManyField | Only used when ManyToManyField definition is against "self." For example, *class Person: friends = models.ManyToManyField("self")* would mean that if Person Alice is friended to Person Brian, then Brian is also Friends with Alice.|
| through            | ManyToManyField | Django automatically creates a mapping table for ManyToMany relationships. By using this field, you can specify the exact mapping table you would like to use, therefore allowing you to augment/extend that table with additional data. For example, https://docs.djangoproject.com/en/dev/topics/db/models/#intermediary-manytomany.|
| parent_link        | OneToOneField | Used only for multiple inheritance (specifically from a concrete class) to avoid additional/extra OneToOneFields being created via subclassing. Django documentation for Related Fields can be found at: https://docs.djangoproject.com/en/dev/ref/models/fields/#module-django.db.models.fields.related.|

####Avoid List

Avoid using the following optional attributes as they can have adverse
effects on data integrity and REST relationships:

| Attribute          | Effect             |
|--------------------|--------------------|
| to_field           | Would allow the relationship to exist against a field other than the primary key in the remote object.|
| db_constraint      | Controls whether or not a constraint should be created in the database for the foreignKey relationship. It defaults to True. If you set it to False you could cause database pointers to non-existent data. We should avoid this.|

###Model-Specific MetaData

| Model Setting      | When to use it     |
|--------------------|--------------------|
| abstract           | Used to specify that the defined model is not intended/able to be instantiated directly. For example, PlCoreBase, which is used to ensure that created, updated, and enacted fields will be provided for all XOS participating objects.|
| app_label          | Necessary if models are defined in modules other than models.py  In our core application we split out the model definitions into their own modules for clarity -- each of the models not derived from the PlCoreBase needs to explicitly state the "core" as the application this object is associated to. For example, PlCoreBase and User.|
| order_with_respect_to | |
| ordering | Defines the default column to order lists of the object type by. For example, Users => email.|

###Model CheckList

After creating a new model, here are the additional changes required
for your new model to fully take effect and be visible in XOS:

| Where to go        | Why                |
|--------------------|--------------------|
| /opt/xos/core/models/__init__.py | Add your new model into the python package.|
| /opt/xos/core/admin.py | Register your model with the admin interface, until this is complete your model will not be visible in the Admin interface. For example, *admin.site.register(Deployment)*. You may also want a custom Admin Page rather than the auto-generated page so that you can tailor the view to just the fields and display widgets you prefer. To do that use *admin.site.register(Deployment, DeploymentAdmin)* and see the Tips and Tricks for GUI Views for more detailed instructions.|
| /opt/xos/xos/urls.py | REST related. Add in the url paths to tie in to callbacks for CRUD. Typically there are 2 different urls to expose: (1) Creation at the directory/container level to handle POST commands; and (2) RetrieveUpdateDestroy path based on GET|PUT|DELETE commands. For example, *url(r'^xos/sites/$', SiteListCreate.as_view(), name='site-list')*, *url(r'^xos/sites/(?P<pk>[a-zA-Z0-9_\-]+)/$', SiteRetrieveUpdateDestroy.as_view(), name='site-detail')*.|
| /opt/xos/core/views | REST related. Add classes to support the Create, Read, Update and Destroy needs for your new model when it is accessed via REST.|
| /opt/xos/core/serializers.py | REST related. Add a class to specify the fields that are exposed via the REST api for your model.  In particular, the serialization preferences if other than default for the field type would also be specified here.|

###Modeling and Filter impacts

###Tips and Tricks for GUI Views

###REST Support

Currently we are using an external plugin for Django to provide the
hooks for our REST framework: http://django-rest-framework.org/

####Interacting with the REST Framework

Once the rest framework has been installed on the base django system,
the system is now capable of registering and responding to direct HTTP
requests.

In order to declare certain models, and certain attributes within
those models as participating in the REST interface, the developer
needs to:

1. Create a serializer for the model
2. Create views for the model
3. Register URLs/hierarchies to be part of the REST api

####About Serializers

Serializers are required in order to properly pack and unpack data
that is being exchanged via the HTTP requests.  Go to the
/opt/xos/core/serializers.py

####About REST Views

Our convention is to make user of GenericAPI views for each of the
objects. This will automatically map GET, POST, PUT, DELETE -
including patch behavior (partial updates).

####About URL registrations

There is a good quick start guide available on the
django-rest-framework site for further details:
http://django-rest-framework.org/#quickstart

###Our conventions for the REST API

##Participaing in XOS Development

Bugs reports and feature requests can be filed using the 
[GitHub Issue Tracker](https://github.com/open-cloud/xos/issues).

To participate in a discussion about XOS development, join
[devel@opencloud.us](https://groups.google.com/a/opencloud.us/forum/#!forum/devel).
