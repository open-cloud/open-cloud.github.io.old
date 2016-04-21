---
layout: page
title: Programmatic Interfaces
permalink: /devguide/interfaces/
---

{% include toc.html %}

XOS provides two programmatic interfaces: a RESTful API and
TOSCA-based API. Earlier versions of XOS also includes xoslib,
but it has since been deprecated.

## REST API

Source for the XOS REST API lives in directory `xos/api`. An importer
tool, `import_methods.py`, auto-generates the REST API by searching
this directory (and sub-directories) for valid API methods. These
methods are descendents of the Django View class. This should include
django_rest_framework based Views and Viewsets.

We establish a convention for locating API methods within the XOS
hierarchy. The root of the api is automatically `/api/`. Under that
are the following paths:

* `/api/service` ... API endpoints that are service-wide
* `/api/tenant` ... API endpoints that are relative to a tenant within a service

For example, `/api/tenant/cord/subscriber/` contains the Subscriber
API for the CORD service.

The API importer automatically constructs REST paths based on
where files are placed within the directory hierarchy. For example,
the files in `xos/api/tenant/cord/` will automatically appear at the
API endpoint `http://server_name/api/tenant/cord/`.  The directory
`examples` contains examples that demonstrate using the API from the
Linux command line.

### Documenting the API

Documentation for the REST API is available at
[docs.xos.apiary.io](http://docs.xos.apiary.io)

We are using the [API Blueprint](https://apiblueprint.org/)
description language to document our APIs, and we are publishing the
docs on [Apiary](https://apiary.io/).

Since Apiary requires documentation to be in a single file, XOS
provides a tool to combine small documentation files into one file, to
use it navigate to `xos/tests/api` and run the command `npm start`
(require `node js`). This command watches all `.md` files in the
`xos/tests/api/source/` directory, and upon detecting a change,
combines them in a single `apiary.apib` file.

### Testing the API

We run automated test against the API using
[Jenkins](https://jenkins.opencord.org/) and
[dredd](https://github.com/apiaryio/dredd).

`dredd` parses the `apiary.apib` file, and performs requests against
all documented endpoints. It then compares the results against the
documented return values.

Once a new endpoint is added to the documentation, it is automatically
tested. If your service requires new model instances, relations,
or any other kind of data, you should add them in the
`xos/tests/api/hooks.py` file. You have the option to create this data
in the `before_each` hooks, or you can define a custom hook for your
endpoint, to do this check the `@hooks.before("Example > Example
Services Collection > List all Example Services")` hooks and the
[dredd documentation](http://dredd.readthedocs.org/en/latest/)

To run API tests, use the
[test-standalone](https://github.com/open-cloud/xos/tree/master/xos/configurations/test-standalone)
configuration.

### Using the API

The REST API may be used via a number of programming languages. Below
are a few examples using common tools and languages:

#### command line via curl

{% highlight sh %}
# use your XOS username and password for my_email and my_password
curl -H "Accept: application/json; indent=4" -u my_email:my_password http://portal.opencloud.us/xos/users/
{% endhighlight %}

#### python

{% highlight python %}
import requests
admin_auth=("my_email", "my_password")   # use your XOS username and password
users = requests.get("http://portal.opencloud.us/xos/users", auth=admin_auth).json()
for user in users:
  print user["email"]
{% endhighlight %}

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
