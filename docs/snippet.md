## Other

### lua_module_hook

APISIX supports using the `lua_module_hook` attribute to inject codes to `apisix`. For more information, please check the [example hook](https://github.com/apache/apisix/blob/master/example/my_hook.lua):

```yaml title="conf/config.yaml"
apisix:
  ...
  lua_module_hook: "path/to/example/my_hook"
```

After restarting the APISIX instance, APISIX prints the following logs in the `path/to/logs/error.log`:

```log title="logs/error.log"
2022/07/31 21:45:49 [notice] 8714#8714: signal process started
2022/07/31 21:45:49 [emerg] 8750#8750: [lua] my_hook.lua:21: http_init(): my hook works in http
```

### Note 1

If the custom plugin needs to be initialized when Nginx start, we may need to modify the `http_init` function in the `apisix/init.lua` file and the `apisix/cli/ngx_tpl.lua` file. This action is dangerous unless you're familiar with APISIX codes.

### Note 2

To enable your plugin, copy this plugin list into `conf/config.yaml`, and add your plugin name. For instance:

```yaml
apisix:
  admin_key:
    - name: "admin"
      key: edd1c9f034335f136f87ad84b625c8f1 # using fixed API token has security risk, please update it when you deploy to production environment
      role: admin

plugins: # copied from config-default.yaml
  ...
  - your-plugin
```

If your plugin has a new code directory of its own, and you need to redistribute it with the APISIX source code, you will need to modify the `Makefile` to create directory, such as:

```
$(INSTALL) -d $(INST_LUADIR)/apisix/plugins/skywalking
$(INSTALL) apisix/plugins/skywalking/*.lua $(INST_LUADIR)/apisix/plugins/skywalking/
```

### Note 3 - Authentication

You might have noticed the key-auth plugin has `type = 'auth'` in its definition.
When we set the type of plugin to `auth`, it means that this plugin is an authentication plugin.

An authentication plugin needs to choose a consumer after execution. For example, in key-auth plugin, it calls the `consumer.attach_consumer` to attach a consumer, which is chosen via the `apikey` header.

To interact with the `consumer` resource, this type of plugin needs to provide a `consumer_schema` to check the `plugins` configuration in the `consumer`.

Here is the consumer configuration for key-auth plugin:

```json
{
  "username": "Joe",
  "plugins": {
    "key-auth": {
      "key": "Joe's key"
    }
  }
}
```

It will be used when you try to create a [Consumer](https://github.com/apache/apisix/blob/master/docs/en/latest/admin-api.md#consumer)

To validate the configuration, the plugin uses a schema like this:

```lua
local consumer_schema = {
    type = "object",
    properties = {
        key = {type = "string"},
    },
    required = {"key"},
}
```

Note the difference between key-auth's `check_schema(conf)` method to example-plugin's:

```lua
-- key-auth
function _M.check_schema(conf, schema_type)
    if schema_type == core.schema.TYPE_CONSUMER then
        return core.schema.check(consumer_schema, conf)
    else
        return core.schema.check(schema, conf)
    end
end
```

```lua
-- example-plugin
function _M.check_schema(conf, schema_type)
    return core.schema.check(schema, conf)
end
```

### Note 4 - Schema

At the same time, we need to implement the `check_schema(conf)` method to complete the specification verification.

```lua
function _M.check_schema(conf, schema_type)
    return core.schema.check(schema, conf)
end
```

Note: the project has provided the public method `core.schema.check`, which can be used directly to complete JSON
verification.

### Note 5 - Logic

Write the logic of the plugin in the corresponding phase. There are two parameters `conf` and `ctx` in the phase method, take the `limit-conn` plugin configuration as an example.

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/index.html",
    "id": 1,
    "plugins": {
        "limit-conn": {
            "conn": 1,
            "burst": 0,
            "default_conn_delay": 0.1,
            "rejected_code": 503,
            "key": "remote_addr"
        }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:1980": 1
        }
    }
}'
```

#### The `conf` parameter

The `conf` parameter is the relevant configuration information of the plugin, you can use `core.log.warn(core.json.encode(conf))` to output it to `error.log` for viewing, as shown below:

```lua
function _M.access(conf, ctx)
    core.log.warn(core.json.encode(conf))
    ......
end
```

conf:

```json
{
  "rejected_code": 503,
  "burst": 0,
  "default_conn_delay": 0.1,
  "conn": 1,
  "key": "remote_addr"
}
```

#### The `ctx` parameter

The `ctx` parameter caches data information related to the request. You can use `core.log.warn(core.json.encode(ctx, true))` to output it to `error.log` for viewing, as shown below :

```lua
function _M.access(conf, ctx)
    core.log.warn(core.json.encode(ctx, true))
    ......
end
```

### Note 6 - Testcases

For functions, write and improve the test cases of various dimensions, do a comprehensive test for your plugin! The
test cases of plugins are all in the "t/plugin" directory. You can go ahead to find out. APISIX uses
[test-nginx](https://github.com/openresty/test-nginx) as the test framework. A test case (.t file) is usually
divided into prologue and data parts by \_\_DATA\_\_. Here we will briefly introduce the data part, that is, the part
of the real test case. For example, the key-auth plugin:

```perl
=== TEST 1: sanity
--- config
    location /t {
        content_by_lua_block {
            local plugin = require("apisix.plugins.key-auth")
            local ok, err = plugin.check_schema({key = 'test-key'}, core.schema.TYPE_CONSUMER)
            if not ok then
                ngx.say(err)
            end

            ngx.say("done")
        }
    }
--- request
GET /t
--- response_body
done
--- no_error_log
[error]
```

A test case consists of three parts :

- `Program code`: configuration content of Nginx location
- `Input`: http request information
- `Output check`: status, header, body, error log check

When we request `/t`, which config in the configuration file, the Nginx will call `content_by_lua_block` instruction to
complete the Lua script, and finally return. The assertion of the use case is response_body return "done",
`no_error_log` means to check the `error.log` of Nginx. There must be no ERROR level record. The log files for the unit test
are located in the following folder: 't/servroot/logs'.

The above test case represents a simple scenario. Most scenarios will require multiple steps to validate. To do this, create multiple tests `=== TEST 1`, `=== TEST 2`, and so on. These tests will be executed sequentially, allowing you to break down scenarios into a sequence of atomic steps.

Additionally, there are some convenience testing endpoints which can be found [here](https://github.com/apache/apisix/blob/master/t/lib/server.lua#L36). For example, see [proxy-rewrite](https://github.com/apache/apisix/blob/master/t/plugin/proxy-rewrite.lua). In test 42, the upstream `uri` is made to redirect `/test?new_uri=hello` to `/hello` (which always returns `hello world`). In test 43, the response body is confirmed to equal `hello world`, meaning the proxy-rewrite configuration added with test 42 worked correctly.

Refer the following [document](building-apisix.md) to setup the testing framework.

#### attach the test-nginx execution process:

According to the path we configured in the makefile and some configuration items at the front of each `.t` file, the
framework will assemble into a complete nginx.conf file. `t/servroot` is the working directory of Nginx and start the
Nginx instance. according to the information provided by the test case, initiate the http request and check that the
return items of HTTP include HTTP status, HTTP response header, HTTP response body and so on.
