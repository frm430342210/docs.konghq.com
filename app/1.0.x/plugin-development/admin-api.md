---
title: Plugin Development - Extending the Admin API
book: plugin_dev
chapter: 8
---

<div class="alert alert-warning">
  <strong>Note:</strong> This chapter assumes that you have a relative
  knowledge of <a href="http://leafo.net/lapis/">Lapis</a>.
</div>

## Introduction

Kong can be configured using a REST interface referred to as the [Admin API].
Plugins can extend it by adding their own endpoints to accommodate custom
entities or other personalized management needs. A typical example of this is
the creation, retrieval, and deletion (commonly referred to as "CRUD
operations") of API keys.

The Admin API is a [Lapis](http://leafo.net/lapis/) application, and Kong's
level of abstraction makes it easy for you to add endpoints.

## Module

```
kong.plugins.<plugin_name>.api
```

## Adding endpoints to the Admin API

Kong will detect and load your endpoints if they are defined in a module named:

```
"kong.plugins.<plugin_name>.api"
```

This module is bound to return a table with one or more entries with the following structure:

``` lua
{
  ["<path>"] = {
     schema = <schema>,
     methods = {
       <method1> = <function1>,
       <method2> = <function2>,
       ...
     }
  },
  ...
}
```

Where:

- `path` would be a string representing a route like `/users` (See [Lapis routes & URL
  Patterns](http://leafo.net/lapis/reference/actions.html#routes--url-patterns)) for details.
  Notice that the path can contain interpolation parameters, like `/users/:users/new`.
- `schema` is a schema definition. Schemas for core and custom plugin entities are available
  via `kong.db.<entity>.schema`. The schema is used to parse certain fields according to their
  types; for example if a field is marked as an integer, it will be parsed as such when it is
  passed to a function (by default form fields are all strings).
- `method1` and `method2` are strings representing the usual HTTP methods: `GET`, `POST`, etc.
   The special strings `before` and `on_error` can also be used (described below)
- `function1` and `function2` describe what to do when a particular path-method combination is matched
  (or for the special `before` function). They can be Lua functions, but often the `kong.api.endpoints` module provides some default
  implementation that is appropiate.

For example:

``` lua
local endpoints = require "kong.api.endpoints"

local credentials_schema = kong.db.keyauth_credentials.schema
local consumers_schema = kong.db.consumers.schema

return {
  ["/consumers/:consumers/key-auth"] = {
    schema = credentials_schema,
    methods = {
      GET = endpoints.get_collection_endpoint(
              credentials_schema, consumers_schema, "consumer"),

      POST = endpoints.post_collection_endpoint(
              credentials_schema, consumers_schema, "consumer"),
    },
  },
}
```

This code will create two Admin API endpoints in `/consumers/:consumers/key-auth`, to
obtain (`GET`) and create (`POST`) credentials associated to a given consumer. On this example
the functions are provided by the `kong.api.endpoints` library. If you want to see a more
complete example, with custom code in functions, see
[the real keyauth `api.lua` file](https://github.com/Kong/kong/blob/master/kong/plugins/key-auth/api.lua).

The `endpoints` module currently contains the default implementation for the most usual CRUD
operations used in Kong. This module provides you with helpers for any insert, retrieve,
update or delete operations and performs the necessary DAO operations and replies with
the appropriate HTTP status codes. It also provides you with functions to retrieve parameters from
the path, such as an API's name or id, or a Consumer's username or id.

For those cases where the provided default functions are not enough, you can write your own
handler functions in Lua as well. They take three arguments, which are, in order:

- `self`: The request object. See [Lapis request
  object](http://leafo.net/lapis/reference/actions.html#request-object)
- `db`: A shortcut to `kong.db`. It was mainly used before Kong 1.0
- `helpers`: A table containing a few helpers. Currently superseeded by the Kong PDK.

In addition to the HTTPS verbs, methods can have two special values:

- **before**: as in
  [Lapis](http://leafo.net/lapis/reference/actions.html#handling-http-verbs), a
  before_filter that runs before the executed verb action.
- **on_error**: a custom error handler function that overrides the one provided
  by Kong. See Lapis' [capturing recoverable
  errors](http://leafo.net/lapis/reference/exception_handling.html#capturing-recoverable-errors)
  documentation.

---

Next: [Write tests for your plugin]({{page.book.next}})

[Admin API]: /{{page.kong_version}}/admin-api/
