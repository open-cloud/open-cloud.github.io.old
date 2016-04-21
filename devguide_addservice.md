---
layout: page 
title: Adding Services 
permalink: /devguide/addservice/
---

{% include toc.html %}

XOS provides a collection of abstractions, interfaces and mechanisms
that enable cloud services to interoperate reliably and efficiently.
By using XOS, the service developer focuses his or her attention on
the core value provided by a service, not binding it to the underlying
mechanisms of the cloud substrate. This section describes the
framework into which services are plugged into XOS.

## Design Overview

The work of defining a new service is threefold: (1) designing
abstractions and adding them to XOS through a standard interface (the
data model, implemented in Django), (2) plumbing those abstractions
through to a Northbound Interface so users and applications can access
them, and (3) plumbing those abstractions through to the actual
mechanism that implements the service (e.g., the scalable instances
running in the underlying cloud servers and switches).

The first task is an exercise in writing Django models, which this guide
discusses in Section [Data Modeling Conventions](/devguide/datamodel/).

The second task is an exercise in writing REST API endpoints and TOSCA
specifications, which this guide discusses in Section [Programmatic
Interfaces](/devguide/interfaces/). We also recommend extending the
Django Admin API (a Graphical Interface), as explained elsewhere in
this Developer Guide.

The third task involves adding a Synchronizer, which is responsible
for enacting the state recorded in the data model (i.e., configuring
and controlling the underlying service instances).  All changes made
to the data model -- including the addition of new objects, updates to
existing objects, and the deletion of objects -- are intercepted by
the Synchronizer. Every time the Data Model changes, the Synchronizer
receives a notification, upon which it queries the Data Model to
retrieve the set of updated objects. Later subsections of this Section
describe the Synchronizer in more detail.

Another good introduction to all of these elements is to work through 
the [ExampleService Tutorial](/devguide/exampleservice/).

The following summarizes all the specific elements (files) that make
up a service:

| Component        | Source Code for Service "X"                |
|------------------|--------------------------------------------|
|Django Data Model | `xos/service/X/model.py`                   |
| Synchronizer     | `xos/synchronizers/X/steps/sync_X.py` \br
                     `xos/synchronizers/X/steps/X_playbook.yml` |
| Admin GUI        | `xos/service/X/admin.py`                   |
| REST API         | `xos/api/service/X.py` \br
                     `xos/api/tenant/Xtenant.py` \br
                     `xos/tests/api/hooks.py` \br
                     `xos/tests/api/source/service/X.md`        |
| TOSCA Spec       | `xos/tosca/custom_definitions/X.yaml` \br
                     `xos/tosca/resources/X.py`                 |

## Introduction to a Synchronizer

Although we have been describing the Synchronizer as a monolithic
entity, it is actually a modular system, consisting of a Synchronizer
Framework and a set of Synchronizer Instances. Each Synchronizer
Instance is associated with some subset of the Data Model, and acts
upon some subset of the imported service controllers. For example,
there is a Synchronizer Instance that activates User, Slice,
Instances, and Network objects by calling OpenStack controllers (Nova,
Neutron, KeyStone); another that activates CDN-related objects by
calling the HyperCache controller; and yet another that activates
subscriber bundles in the by pushing Ansible playbooks to backend
containers. In general, any service-related objects in the data model
that need to interact with a low level platform must include a
service-specific Synchronizer Instance.

This section uses the Amazon EC2 Synchronizer to illustrate how to
develop a new synchronizer that interacts with an existing service
controller (e.g., the E2 API). The ExampleService Synchronizer
described in [ExampleService Tutorial](/devguide/exampleservice/)
illustrates how to develop a new synchronizer that interacts directly
with the instances (e.g., containers) that implement the service.

*[Note: The Synchronizer Framework, which is common across all Synchronizer
 Instances, is currently embedded in the OpenStack Synchronizer Instance.
 Our plan is to lift it out of this instance and into the core of
 XOS.]*

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

## Developing a Synchronizer

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

* `dmdot` -- This is a tool that generates the dependency graph
  contained in the Data Model. You can use it with the -dot argument
  to generate a dot representation which looks like the illustration
  in the Data Model section. Or you can run it without arguments to
  generate the dependency graph used by the Synchronizer Framework to
  ensure that actions taken by the individual Synchronizer Instances are
  ordered correctly.

* `mainloop.py` -- This file contains the main run loop of the Synchronizer
  Framework. It is the mechanism that receives events, dispatches
  actions, and orchestrates the components of the Synchronizer.

* `event_manager.py` -- This file is the part of the Synchronizer Framework
  that interacts with the Feefie notification system, used to transmit
  events from the Data Model to the Synchronizer Framework.

* `toposort.py` -- This program sorts the actions in the Synchronizer
  Framework in accordance with the dependency graph generated by
  dmdot.

The remainder of the files contain peripheral support code, or base
classes to be subclassed by your Synchronizer Instance.

## Synchronizer Steps

Within the `synchronizer/` subdirectory of the XOS repository, you will
find a `steps/` subdirectory. This directory contains the actual
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

{% highlight py %}
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
{% endhighlight %}

The mandatory interface of a Step, which is used by the framework to
hook Steps into the core of the Synchronizer Framewrok, consists of the
following instance properties (three variable and three methods):

* `provides`: The models that this step provides, and the ones that
  define the position at which the step is placed when enacting the
  current data model.

* `observes`: The models that this step actually enacts new instances
  of. Usually the same as provides, but may be an auxilary model that
  holds state that complements the main model provided.

* `requested_interval`: The intervals at which this model is enacted.
  For slow executing models, set this to a high value so that it
  does not interrupt every run of the Synchronizer.

* `fetch_pending`: The method that fetches the pending set of objects
  that have to be enacted. There is a default implementation that uses
  internal timestamping and other mechanisms to automatically fetch
  the latest set of steps. That said, service implementers can define
  their own version. The variables that they get access to are
  "enacted" and "updated." Enacted is the timestamp of the last time
  an instance was instantiated.

{% highlight py %}
def fetch_pending(self):
  new_sites = GetSitesAddedSinceLastRan()
  return new_sites
{% endhighlight %}

The `fetch_pending` function can use these bookkeeping variables to
fetch the set of changed objects:

{% highlight py %}
new_objects = Instance.objects.filter(Q(enacted__lt=F('updated')) | Q(enacted=None))
{% endhighlight %}

The line above is typical, and can be copied and pasted into most
Synchronizer implementations. It retrieves the set of objects that have
never been enacted (enacted = None) or that have been updated since
they were last enacted (enacted <= updated).

* `sync_record`: The main method that invokes operations on the
  backend controller to bring it in sync with the data model.

{% highlight py %}
def sync_record(self, site):
  site.save()
{% endhighlight %}

Once the set of changed objects have been computed, the Synchronizer
invokes the appropriate set of backend operations and populates the
objects with links to the back end. This functionality is contained in
the `sync_record` function.  For example, the SyncInstances step of the
EC2 Synchronizer creates EC2 instances in response to the addition of
Instances.

{% highlight py %}
instance_type = instance.node.name.rsplit('.',1)
instance_sig = aws_run('ec2 run-instances --image-id %s --instance-type %s --count 1'%(instance.image.image_id, instance_type))
{% endhighlight %}

And then it associates the `instance_id` an identifier that uniquely
identifies the instance in the data model, with the Instance.

{% highlight py %}
    instance.instance_id = instance_sig['Instances'][0]['InstanceId']
    instance.save()
{% endhighlight %}

It is essential that models in the Data Model be linked with actual
facilities in the backend, so that they may be updated or deleted at a
later point. The `sync_record` function need not update the value of the
enacted field in the model. This operation is performed by the
Synchronizer Framework core.

## Synchronizer Steps are Idempotent

As shown in the previous sections, the bulk of the work involved in
implementing a Synchronizer Instance lies in providing a set of Synchronizer
Steps by placing them in the `synchronizer/steps/` directory within the
repository. All Steps at this location are automatically picked up by
the Synchronizer Framwork when it runs, and sewn together into the
pre-computed dependency graph generated by `dmdot`.

There is one additional requirement essential for a Synchronizer to
function correctly. The lock-free and transaction-free operation
depends on an important invariant that it is your responsibility to
maintain when implementing Steps.

Synchronizer Steps are Idempotent Operations. They may run multiple times,
and it is your responsibility to ensure that a subsequent execution
does not cause the system to go into an error state.

## Internal Steps

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

{% highlight py %}
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
{% endhighlight %}

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

{% highlight py %}
zone_ret = aws_run('ec2 describe-availability-zones')
zones = zone_ret['AvailabilityZones']
{% endhighlight %}

This exhaustive fetch makes External Steps resource-intensive and slow
to run. As a rule of thumb: Steps that do not fetch recently updated
objects using the enacted and updated variables are External Steps.

External Steps should have their requested_interval set to hours or
days, so that they do not block Internal steps. This is because the
Synchronizer is single-threaded and Internal steps are expected to be
responsive.

## Deletions

Synchronizers handle deletions in the same way as they handle
synchronization. When an object is deleted in the data model, it is
simply marked as deleted. The `fetch_pending` method in that case
fetches the set of such deleted objects and passes it on, instead of
`sync_record`, to a method called `delete_record`. It is the task of
`delete_record` to pass the deletion of the record on to the back
end. For example, if an Instance is deleted, then the method should
delete and clean up the corresponding VM, along with recovering any
other resources, such as volumes associated with that VM. If this
method returns successfully, then XOS automatically takes care of
actually deleting it from the data model.

## Django Migration Notes

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
be placed in `core/migrations/`, `hpc/migrations/`,
`syndicate_service/migrations/`, and so on. The second step
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

1. `syncdb` is now a synonym for `migrate`

2. `syncdb` (and `migrate`) no longer automatically install
`initial_data.json`. To compensate, `scripts/opencloud` has been modified
to perform a "loaddata" whenever the database is created for the very
first time.

3. `django.setup` needs to be called by external scripts before using
models. `xos-backend.py` has been modified to do this
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

