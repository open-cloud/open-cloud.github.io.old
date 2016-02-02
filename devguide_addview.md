---
layout: page
title: Adding Views
permalink: /devguide/addview/
---
{% include toc.html %}

XOS is designed to be extensible, with new services incorporated
into XOS so they are readily available to other users. Our goal is to
build a diverse collection of services that can be accessed through
the XOS programmatic interface. This is done by extending the XOS
data model with objects that represent the available services, as
described in the [Adding Services](/devguide/addservice/) section.

This collection of services, in turn, defines a range of functionality
that can be combined in different ways to provide customized *views*
(portals) targeted at different user communities and usage scenarios.
A view might provide a graphical interface to just one  service, it
might present a subset of a service customized for a particular user
community, it might serve as a dashboard for one or more services, or
it might create a new capability by combining features from multiple
services.

XOS provides a default view, which we sometimes call the *Developer
View*. Confusingly, the Developer View is implemented by Django's
"standard template view" (also called the *Admin View*) that Django
auto-generates from its data model. Each service developer is expected
to provide any service-specific customizations to the Admin View as
part of the service implementation. (This corresponds to the
`admin.py` file in the service implementation.)

In addition to this default view, XOS provides an AngularJS-based
mechanism for creating new views. The mechanism builds on top of
*xoslib* and includes the ability to link the view into the XOS
portal. This section describes how to create a view using this
mechanism.

Users are aways free to implement stand-alone portals on top of XOS's
RESTful API using whatever GUI framework they want. In the source
tree, example stand-alone applications (GUI-based and otherwise)
are given in the `applications` directory. Example views built using
the mechanism described in this section can be found in the `views`
directory.

Note that the source tree contains legacy views created as extensions
to Django's standard template view, but XOS no longer supports these
Django-based views.

## Preliminaries

Directory `views/ngXosLib` contains a collection of tools for
implementing an XOS View as an Angular Single Page Application
(SPA). In the following description, we assume XOS is running on
your development system and respondin at `localhost:9999`. The
`xos/configurations/frontend` is normally sufficient for GUI
development. 

### Apigen

Usage: `npm run apigen`

This tool generates an angular resource file for each endpoint available in Swagger.

>You can generate api related documentation with: `npm run apidoc`. The output is locate in `api/docs`. You can also see a list of available methods through Swagger at `http://localhost:9999/docs/`

### Vendors

XOS comes with a set of common libraries, as listed in `bower.json`

 * angular
 * angular-route
 * angular-resource
 * angular-cookie
 * ng-lodash

These libraries are served through Django, so they are not included in
your minified vendor file. To add a library and generate a new file
(that will override the old one):

 * enter `ngXosLib` folder
 * run `bower install [myPackage] --save`
 * rebuild the file with `gulp vendor`

>_NOTE before adding libraries please discuss it on the devel list to avoid this file becoming too big_

### Helpers

XOS comes with a helper library that is automatically loaded in the
Django template.

To use it, add `xos.helpers` to your required modules:

```
angular.module('xos.myView', [
  'xos.helpers'
])
```

Doing so will automatically add a `token` to all your requests.
Eventually you can take advantage of some other services:

- **NoHyperlinks Interceptor**: will add a `?no_hyperlinks=1` to your request, to tell Django to return ids instead of links.
- **XosApi** wrapper for `/xos` endpoints.
- **XoslibApi** wrapper for `/xoslib` endpoints.
- **HpcApi** wrapper for `/hpcapi` endpoints.

>_NOTE: for the API related service, check the documentation in Section [Apigen](#apigen)._

### Scaffolding

We have created a [yeoman](http://yeoman.io/) generator to help scaffold views.

>As it is in an early stage of development, you should manually link it to your system. To do this enter `/views/ngXosLib/generator-xos` and run `npm link`.

## Generating a New View

By convention, customized views are located in `views/ngXosViews`.
From `/views` run `yo xos`. This command creates a new folder with
the provided name in `/views/ngXosViews` that contains your application.

>If you left View name empty it should be `/views/ngXosViews/sampleView`

### Run a Development Server

In your `view` folder run `npm start`.

This will install the required dependencies and start a local server
with [BrowserSync](http://www.browsersync.io/).

### Publish Your View

Once your view is done, from your view root folder, run: `npm run build`.
This will build your application and copy files in the appropriate
directories for use by Django.

At this point you can enter
`http://localhost:9999/admin/core/dashboardview/add/` and add your
custom view.

>_NOTE: url field should be `template:xosSampleView`_


You can easily set this as a default view in a configuration by
editing the `{config}.yml` file for that configuration. Add these
lines:

```
{TabName}:                                    
  type: tosca.nodes.DashboardView              
  properties:                                  
      url: template:{viewName}     
```

Then edit the _User_ section (normally it starts with
`padmin@vicci.org`) as follows:


```
padmin@vicci.org:                                          
  type: tosca.nodes.User                                   
  properties:                                              
      firstname: XOS                                       
      lastname: admin                                      
      is_admin: true                                       
  requirements:                                            
      - tenant_dashboard:                                  
          node: Tenant                                     
          relationship: tosca.relationships.UsesDashboard  
      - {custom_dashboard}:                              
          node: {TabName}                                 
          relationship: tosca.relationships.UsesDashboard  
```

### Install Dependencies

To install a local dependency use bower with `--save`. Common modules
are saved in `devDependencies` as they already loaded in the Django
template.

The `npm start` command watches your dependencies and will automatically inject it in your `index.html`.

### Linting

A styleguide is enforced through [EsLint](http://eslint.org/) and is
checked during the build process. We **highly** recommend installing
the linter in your editor to have realtime hints.

### Test

The generator sets up a test environment with a default test.
To run it, execute: `npm test`
