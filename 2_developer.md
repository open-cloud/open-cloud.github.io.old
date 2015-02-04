---
layout: page
title: Developer Guide
---

This guide describes how to contribute to the XOS core, as well as to
extend XOS with new services and views. It also documents modeling and
naming conventions, and describes the development and testing
environments we use.

##Development Environment

##Testing Framework

##REST API

A REST API and associated
[documentation](http://portal.opencloud.us/docs/) is auto-generated
from the data model.

##xoslib

xoslib is a client/server library for extending XOS. The server side
of the library defines a REST API that is used to create, read,
update, and delete XOS objects (e.g., deployments, slices, users),
along with new xoslib-defined objects that extend XOS. This REST API
uses HTTP as a transport mechanism and may be used from a variety of
clients and languages.

To facilitate development in Javascript, xoslib also includes an
extensive client library based on Backbone.js and Marionette.js.
Portions of the XOS user interface are implemented on top of this
library. Backbone.js provides an efficient event-driven interface,
where the xoslib's client-side library fetches models from the
server-side, and notifies client programs when data has been fetched
for display to the user.

##<a name="adding-views">Adding Views to XOS</a>

XOS is designed to be extensible -- to provide an explicit means to
incorporate services deployed on OpenCloud back into OpenCloud so they
are readily available to other users. Our goal is to build a diverse
collection of services that can be accessed through the OpenCloud
programmatic interface. This is done by extending the XOS data model
with objects that represent the available services.

This collection of services, in turn, defines a range of functionality
that can be combined in different ways to provide customized Views
targeted at different user communities and usage scenarios. These
views are similar to scripts -- small programs that leverage the
underlying set of commands to create a new function for a particular
purpose, where in OpenCloud's case, the "underlying set of commands"
corresponds to operations on XOS's data model. The data model includes
both core objects (e.g., Users, Slices, Slivers, Roles, Sites,
Deployments, Services) and objects that represent various services
(e.g., Syndicate Volumes, HPC OriginServers, RequestRouter
ServiceMaps). A given View can operate on any combination of these
objects.

We refer to these scripts as Views because they represent a particular
perspective of OpenCloud, but they also happen to be implemented as a
tailored view in Django, extending Django's standard template view. 
When we need to disambiguate between the OpenCloud abstraction and the
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
xoslib/dashboards/. In an OpenCloud installation, you'll find these
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

Opencloud includes some context variables that are built in. One
example of this is "user", which represents the currently logged in
user. Context variables are useful for values that we don't expect to
change while the user is looking at the view.

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
fields that tabulate the number of slivers and sites used by the
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
opencloud dashboard. This is done by specifying a "http://" url
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

##Data Model Conventions

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
declaration. The basic field types commonly in use with OpenCloud:

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
| ForeignKeyField    | Used to represent a 1-to-Many relationship. For example: Sliver's may have 1 and only 1 Node; Node's may have 0 or more Slivers. Can also be used to represent recursive relationships for the same object types by providing "self" as the relationship (first position) parameter.|
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
| abstract           | Used to specify that the defined model is not intended/able to be instantiated directly. For example, PlCoreBase, which is used to ensure that created, updated, and enacted fields will be provided for all OpenCloud participating objects.|
| app_label          | Necessary if models are defined in modules other than models.py  In our core application we split out the model definitions into their own modules for clarity -- each of the models not derived from the PlCoreBase needs to explicitly state the "core" as the application this object is associated to. For example, PlCoreBase and User.|
| order_with_respect_to | |
| ordering | Defines the default column to order lists of the object type by. For example, Users => email.|

###Model CheckList

After creating a new model, here are the additional changes required
for your new model to fully take effect and be visible in OpenCloud:

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

##Reporting Bugs

Bugs reports and feature requests can be filed using the 
[GitHub Issue Tracker](https://github.com/open-cloud/xos/issues).



