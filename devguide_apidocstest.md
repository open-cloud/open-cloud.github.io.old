---
layout: page
title: API Documentation and Testing
permalink: /devguide/apidocstest/
---

{% include toc.html %}

## API Documentation

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

## API Tests

We run automated test against the API using
[Jenkins](https://jenkins.opencord.org/) and
[dredd](https://github.com/apiaryio/dredd).

`dredd` parses the `apiary.apib` file, and performs requests against
all documented endpoints. It then compares the results against the
documented return values.

Once a new endpoint is added to the documentation, it is automatically
tested, but if your service requires new model instances, relations,
or any other kind of data, you should add them in the
`xos/tests/api/hooks.py` file. You have the option to create this data
in the `before_each` hooks, or you can define a custom hook for your
endpoint, to do this check the `@hooks.before("Example > Example
Services Collection > List all Example Services")` hooks and the
[dredd documentation](http://dredd.readthedocs.org/en/latest/)

To run API tests, use the
[test-standalone](https://github.com/open-cloud/xos/tree/master/xos/configurations/test-standalone)
configuration.

