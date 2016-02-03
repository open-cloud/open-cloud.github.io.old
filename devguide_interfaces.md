---
layout: page
title: Programmatic Interfaces
permalink: /devguide/interfaces/
---

{% include toc.html %}

XOS provides three programmatic interfaces. The first -- a RESTful
interface to the underlying data model -- is XOS's "system call"
interface. The other two -- xoslib and TOSCA -- are built on top of
the RESTful API.

## REST API

A REST API and associated
[documentation](http://guide.xosproject.org/restapi/) is auto-generated
from the data model. In addition to the statically-generated documentation linked here,
XOS also supports Swagger, and an interactive Swagger page is available at
`/docs/` on any XOS development server.

The REST API may be used via a number of programming languages. Below are a few examples using common tools and languages:

### command line via curl

{% highlight sh %}
# use your XOS username and password for my_email and my_password
curl -H "Accept: application/json; indent=4" -u my_email:my_password http://portal.opencloud.us/xos/users/
{% endhighlight %}

### python

{% highlight python %}
import requests
admin_auth=("my_email", "my_password")   # use your XOS username and password
users = requests.get("http://portal.opencloud.us/xos/users", auth=admin_auth).json()
for user in users:
  print user["email"]
{% endhighlight %}

## XOSlib

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
Views](/userguide/#user-views)) are implemented on top of this library.

## TOSCA

XOS supports use of
[TOSCA](http://www.oasis-open.org/committees/tosca/) as a mechanism to
configure and provision XOS services. There are two ways to use TOSCA
in XOS.

The first is by loading and running a TOSCA program via the XOS GUI.
This is done as follows:

  1. Navigate to Home>Core>Programs in the GUI, and select the 'Add Program' button.
  2. Enter a name and a description for your program.
  3. Click the 'Program Source' tab. Here you may either type in your TOSCA specification directly, or use your browser to upload a file
  4. Click the 'Save and Continue' button.
  5. Go back to the 'Program Details' tab
  6. Select 'Run' in the Command dropdown.
  7. Click the 'Save and Continue' button.
  8. XOS will now run your program in the background. Check back later (i.e. refresh the page in your browser) and the result of the program will be displayed in the Output box.

The second is by running a TOSCA program using command line tools. To
do this from inside the XOS container, you don't have to add the
specification to the data model, and you don't have to wait for XOS to
queue and execute the specification. The command-line tool returns
output on completion. To execute a TOSCA specification, use the
following command:

{% highlight sh %}
$ /opt/xos/tosca/run.py <email-address> <filename>
{% endhighlight %}

For example,

{% highlight sh %}
$ /opt/xos/tosca/run.py padmin@vicci.org /opt/xos/tosca/samples/new_site_deploy_slice.yaml
{% endhighlight %}

For a reference guide to XOS-specific TOSCA extensions, see
[http://guide.xosproject.org/tosca_reference.html](http://guide.xosproject.org/tosca_reference.html)

For samples of XOS TOSCA specifications, consult the
[xos/tosca/samples](https://github.com/open-cloud/xos/tree/master/xos/tosca/samples)
section of the XOS git repository.
