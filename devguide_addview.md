---
layout: page
title: Adding Views
permalink: /devguide/addview/
---
{% include toc.html %}

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

## Create a Django Template

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

{% highlight html %}
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
{% endhighlight %}

It illustrates four things:

* A comment, `<!-- ... -->`

* Static text, such as "this is the hello world view"

* Django context variables, such as {{ user.firstname }} and {{ foobar }}

* An empty div that we will populate using javascript,
  "#dynamicTableOfInterestingThings"

* An input box where the user can type some stuff and a submit button
  to go with it

* Some `<script>` tags that load javascript for xoslib and its
  dependencies

* A `<script>` tag that loads javascript for the helloworld view
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

## Display Static Data Using Context Variables

XOS includes some context variables that are built in. One example of
this is "user", which represents the currently logged in user. Context
variables are useful for values that we don't expect to change while
the user is looking at the view.

[TODO: suggest the user use xoslib instead of context variables?]

The easiest place to add more Django context variables is in
/opt/xos/core/dashboard/views/view_common.py:
getDashboardContext(). This function returns the set of context
variables that are usable by the views:

{% highlight python %}
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
{% endhighlight %}

To add a new context variable, just insert a new statement near the
bottom of this function, before the return statement:

{% highlight python %}
context['foobar'] = "value_of_foobar"
{% endhighlight %}

Now we have a new context variable called "foobar", and any user view
template can display that variable by using the syntax "{{ foobar }}".
Restart the Django server, open (or refresh) the helloworld view in
your browser, and "value_of_foobar" should now be displayed in the
appropriate place.

## Create Javascript to Use with the View

Now that we've created a template, let's create some javascript to
make our view more dynamic. The javascript should be placed in
/opt/xos/core/xoslib/static/js/helloworld.js.

[Note: we could have placed the javascript in the html template using
\<script\>\</script\> tags, but placing it in a separate .js file is
cleaner for large views, and eliminates the potential of django's
template language modifying the javascript.]

## Fetching Dynamic Data with xoslib

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

{% highlight javascript %}
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
{% endhighlight %}

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

## Committing Modified Data Back to XOS

Models in xoslib have a save method that can be used to commit
modified data back to XOS. Let's modify helloworld so that when the
user presses the submit button, we change the description of the first
slice in the list:

{% highlight javascript %}
// helloworld.js
$(document).ready(function() {
    $('#submitNewDescription').bind('click', function() {
        newDescription = $("#newDescription").val();
        xos.slices.models[0].set("description", newDescription);
        xos.slices.models[0].save();
    });
});
{% endhighlight %}

The above binds a click handler to the submitNewDescription
button. The click handler fetches the new description from the input
box, and calls set() on the model. It then calls save() on the
model. The save method will cause backbone.js to issue the appropriate
REST calls to change the description of the first slice.

## Extend the XOS API with New Objects

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

## Register a View with XOS

Up to this point, the URL we've been using to display our user view
only displays the helloworld user view and no other views. The
helloworld user view hasn't been integrated with XOS yet. To do that
we need to create a data model object.

There are two different views you can register, template-based views
(like out helloworld example), and iframe-based views.

### Template-based Views

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

### iframe-based Views

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
