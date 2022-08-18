---
title: Plugin Development
description: Cloud Native API Gateway Apache APISIX supports using Lua, Rust, WASM, Golang, Python, JavaScript to develop custom plugins.
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Apache APISIX supports using Lua or [other languages](./external-plugin.md) to develop custom plugins, and you can find all built-in plugins [here](https://github.com/apache/apisix/tree/master/apisix/plugins).

Let's use the [`example-plugin`](https://github.com/apache/apisix/blob/master/apisix/plugins/example-plugin.lua#L45-L51) plugin to explain the plugin development processes.

## Prerequisite

Before developing a custom plugin, please check if the custom logic has dependencies.

For example, the [`openid-connect`](https://github.com/apache/apisix/blob/master/apisix/plugins/openid-connect.lua) plugin needs `Nginx shared memory` functionality, we have to inject Nginx snippet like the following to make it work as expected:

```yaml title="conf/config.yaml"
nginx_config:
  http_configuration_snippet: |
    lua_shared_dict discovery             1m; # cache for discovery metadata documents
    lua_shared_dict jwks                  1m; # cache for JWKs
    lua_shared_dict introspection        10m; # cache for JWT verification results
```

## Structure

In APISIX, we use a standalone `.lua` file to contain all custom logics usually.

```lua title="apisix/plugins/example-plugin.lua"
local schema = {}

local metadata_schema = {}

local plugin_name = "example-plugin"

local _M = {
  version = 0.1,
  priority = 0,
  name = plugin_name,
  schema = schema,
  metadata_schema = metadata_schema,
}


function _M.check_schema(conf, schema_type)
  ...
end


function _M.init()
  ...
end


function _M.destroy()
  ...
end


function _M.rewrite(conf, ctx)
  ...
end


function _M.access(conf, ctx)
  ...
end


function _M.body_filter(conf, ctx)
  ...
end


function _M.delayed_body_filter(conf, ctx)
  ...
end


function _M.control_api()
  ...
end


return _M
```

## Attributes

### `_M.name`

All plugins should use a unique name.

### `_M.version`

TBD

### `_M.priority`

All plugins should use a unique priority, please check `conf/config-default.yaml` to get all plugins' priority.

As shown in the example code above in this guide, a plugin can perform different actions at different phases of the request/response. When different plugins perform actions in the same phase, the plugin with higher priority will be executed first.

When setting the priority, in order to avoid unexpected cases caused by prioritizing the execution of built-in plugins, it is recommended to set the priority range from `1 to 99`.

### `_M.schema`

TBD

### `_M.metadata_schema`

TBD

### `_M.type`

This field is optional, please omit this field when there is no need.

:::note

<!-- TODO: depends on what? -->

Not all Authentication plugins must have the `_M.type = "auth"` attribute, e.g., [authz-keycloak](https://github.com/apache/apisix/blob/master/apisix/plugins/authz-keycloak.lua).

:::

If the plugin needs to work with `Consumer`, then set the `_M.type` field to `auth`. Please check the built-in `Authentication` plugins for reference, e.g., [`basic-auth`](https://github.com/apache/apisix/blob/master/apisix/plugins/basic-auth.lua#L56), [`key-auth`](https://github.com/apache/apisix/blob/master/apisix/plugins/key-auth.lua#L57), [`jwt-auth`](https://github.com/apache/apisix/blob/master/apisix/plugins/jwt-auth.lua#L125).

### `_M.run_policy`

This field is optional, please omit this field when there is no need.

If set the `_M.run_policy` field to `prefer_route`, then when we enable the same plugin both in the [`Global`](https://apisix.apache.org/docs/apisix/admin-api/#global-rule) level and the `Route` level, only the `Route` level will work.

## Execution Phases

Determine which phase to run, generally access or rewrite. If you don't know the [OpenResty lifecycle](https://github.com/openresty/lua-nginx-module/blob/master/README.markdown#directives), it's
recommended to know it in advance. For example key-auth is an authentication plugin, thus the authentication should be completed
before forwarding the request to any upstream service. Therefore, the plugin must be executed in the rewrite phases.
In APISIX, only the authentication logic can be run in the rewrite phase. Other logic needs to run before proxy should be in access phase.

The following code snippet shows how to implement any logic relevant to the plugin in the OpenResty log phase.

```lua
function _M.log(conf, ctx)
-- Implement logic here
end
```

:::note
We can't invoke `ngx.exit`, `ngx.redirect` or `core.respond.exit` in rewrite phase and access phase. if need to exit, just return the status and body, the plugin engine will make the exit happen with the returned status and body. [example](https://github.com/apache/apisix/blob/35269581e21473e1a27b11cceca6f773cad0192a/apisix/plugins/limit-count.lua#L177)
:::

### `_M.rewrite`

TBD

### `_M.access`

TBD

### `_M.body_filter`

TBD

### `_M.delayed_body_filter`

Besides OpenResty's phases, we also provide extra phases to satisfy specific purpose:

- `delayed_body_filter`

```lua
function _M.delayed_body_filter(conf, ctx)
    -- delayed_body_filter is called after body_filter
    -- it is used by the tracing plugins to end the span right after body_filter
end
```

## Schema

### `schema`

TBD

### `metadata_schema`

TBD

### `_M.check_schema()`

TBD

## Logic

TBD

### Public API

A plugin can register API which exposes to the public. Take jwt-auth plugin as an example, this plugin registers `GET /apisix/plugin/jwt/sign` to allow client to sign its key:

```lua
local function gen_token()
    --...
end

function _M.api()
    return {
        {
            methods = {"GET"},
            uri = "/apisix/plugin/jwt/sign",
            handler = gen_token,
        }
    }
end
```

Note that the public API will not be exposed by default, you will need to use the [public-api plugin](plugins/public-api.md) to expose it.

### Control API

If you only want to expose the API to the localhost or intranet, you can expose it via [Control API](./control-api.md).

Take a look at example-plugin plugin:

```lua
local function hello()
    local args = ngx.req.get_uri_args()
    if args["json"] then
        return 200, {msg = "world"}
    else
        return 200, "world\n"
    end
end


function _M.control_api()
    return {
        {
            methods = {"GET"},
            uris = {"/v1/plugin/example-plugin/hello"},
            handler = hello,
        }
    }
end
```

If you don't change the default control API configuration, the plugin will be expose `GET /v1/plugin/example-plugin/hello` which can only be accessed via `127.0.0.1`. Test with the following command:

```shell
curl -i -X GET "http://127.0.0.1:9090/v1/plugin/example-plugin/hello"
```

[Read more about control API introduction](./control-api.md)

### Custom variable

We can use variables in many places of APISIX. For example, customizing log format in http-logger, using it as the key of `limit-*` plugins. In some situations, the builtin variables are not enough. Therefore, APISIX allows developers to register their variables globally, and use them as normal builtin variables.

For instance, let's register a variable called `a6_labels_zone` to fetch the value of the `zone` label in a route:

```lua
local core = require "apisix.core"

core.ctx.register_var("a6_labels_zone", function(ctx)
    local route = ctx.matched_route and ctx.matched_route.value
    if route and route.labels then
        return route.labels.zone
    end
    return nil
end)
```

After that, any get operation to `$a6_labels_zone` will call the registered getter to fetch the value.

Note that the custom variables can't be used in features that depend on the Nginx directive, like `access_log_format`.

## Testcases

TBD

## Load plugins

Apache APISIX's built-in plugins are under the `path/to/apisix/plugins` directory, and it supports two ways to load custom plugins.

:::tip

1. If the custom plugin has the same name (`_M.name`) as a built-in plugin, the custom plugin will override the built-in plugin.
<!-- TODO: Does Tip 2 work for "extra_lua_path"? -->
2. We could use `require "apisix.plugins.<_M.name>"` in Lua to require the custom plugin.

:::

<!-- Why recommended? -->
<Tabs>
  <TabItem value="config.yaml" label="Update config.yaml (recommended)" default>

1. Add the following snippet in the `config.yaml` file to load custom Lua files. For example:

```yaml title="conf/config.yaml"
apisix:
  extra_lua_path: "/path/to/?.lua"
  extra_lua_cpath: "/path/to/?.lua"
```

2. Restart APISIX instance.

</TabItem>
<TabItem value="update-source-codes" label="Update source codes">

1. Open the [`path/to/apisix/plugins`](https://github.com/apache/apisix/blob/master/apisix/plugins) directory, edit a built-in plugin or add a new plugin.
2. For example, add the `3rd-party` plugin:

```
├── apisix
│   └── apisix
│       ├── plugins
│       │   └── 3rd-party.lua
│       └── stream
│           └── plugins
│               └── 3rd-party.lua
```

3. [Rebuild APISIX](./building-apisix.md) or distribute it in different formats.

</TabItem>
</Tabs>

## Enable plugins

<!-- TODO: check all reload/restart APISIX text's description; How? -->

After loading the custom plugins, we need to explicitly add them to the `config.yaml` file and restart APISIX.

:::tip

APISIX can't merge the `plugins` attribute from `config.yaml` to `config-default.yaml` file, please copy **all plugin names** to the `config.yaml` file.

:::

```yaml title="config.yaml"
plugins:                           # plugin list (sorted by priority)
  - real-ip
  - ...
  - 3rd-party                      # Custom plugin names
```

## Unload plugins

TBD
