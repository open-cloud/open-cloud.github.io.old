---
layout: page
title: API Documentation and testing
permalink: /devguide/apidocstest/
---

{% include toc.html %}

## API documentation

The API documentation is available at [docs.xos.apiary.io](http://docs.xos.apiary.io)

We are using [API Blueprint](https://apiblueprint.org/) description language for web APIs to document our APIs in XOS and we are publishing the docs on [Apiary](https://apiary.io/).

Since Apiary require the documentation to be in a single file, we provide a tool to combine small documentation files into one, to use it navigate to `xos/tests/api` and run the command `npm start` (require `node js`). This command will watch all `.md` files in the `xos/tests/api/source/` folder and upon change combine them in a `apiary.apib` file.

## API tests

We are running automated test against the API using [Jenkins](https://jenkins.opencord.org/) and [dredd](https://github.com/apiaryio/dredd).

`dredd` parse our `apiary.apib` file, and perform request against XOS for any documented ebdpoint. It then compare the request result with the documented one.

Once that a new endpoint is added to the documentation it get automatically tested, but if your service require model instances, relation or any other kind of data you should add them in the `xos/tests/api/hooks.py` file. You have the option to create this data in the `before_each` hooks, or you can define a custom hook for your endpoint, to do this check the `@hooks.before("Example > Example Services Collection > List all Example Services")` hooks and the [dredd documentation](http://dredd.readthedocs.org/en/latest/)

To run your test you can use the [test-standalone](https://github.com/open-cloud/xos/tree/master/xos/configurations/test-standalone) configuration.

