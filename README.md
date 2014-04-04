# shib

* http://github.com/tagomoris/shib

## DESCRIPTION

Shib is web client application for SQL-like query engines, written in Node.js, supporting
 * Hive (hiveserver, hiveserver2)
 * Facebook Presto

Once configured, we can switch query engines per executions.

Some extension features are supported:

* Setup queries: options to specify queries executed before main query, like 'create functions ...'
* Default Database: option to specify default database for Hive 0.6 or later
* Huahin-Manager (Job Controller Proxy with HTTP API) support: Kill hive mapreduce job correctly from shib, with Huahin-Manager
  * see: http://huahin.github.com/huahin-manager/

### Versions

Latest version of 'shib' is v0.3.2.

'shib' versions are:

* v0.3 series
  * multi engines/databases support
  * presto support
  * storages of v0.3.x are compatible with v0.2
* v0.2 series
  * current status of master branch
  * uses local filesystem instead of KT, depends on latest node (v0.8.x, v0.10.x)
  * higher performance, more safe Web UI and updated features
  * storages of v0.2 are **NOT complatible with v0.1**
* v0.1 series
  * uses KT, depends on node v0.6.x
  * see `v0.1` tag

## INSTALL

### Hive/Presto

For Hive queries, shib requires HiveServer or HiveServer2. Setup and run these.

* For HiveServer2
  * Configure `hive.server2.authentication` as `NOSASL`
    * Strongly recommended to configure `hive.support.concurrency` as `false`
  * Database selection is not supported now

For Presto, shib is tested with Presto version 0.57.

### Node.js

To run shib, you must install node.js (v0.10.x recommended), and export PATH for installed node.

### shib

Clone shib code.

    $ git clone git://github.com/tagomoris/shib.git

Install libraries, configure addresses of HiveServer (and other specifications).

    $ npm install
    $ vi config.js

And run.

    $ npm start

Shib listens on port 3000. see http://localhost:3000/

You can also run shib with command below for 'production' environment, with production configuration file 'production.js':

    $ NODE_ENV=production NODE_PATH=lib node app.js

## Configuration

Shib can have 2 or more query executor engines.

### HiveServer2

Basic configuration with HiveServer2 in config.js (or productions.js):

```js
var servers = exports.servers = {
  listen: 3000,
  fetch_lines: 1000,   // lines per fetch in shib
  query_timeout: null, // shib waits queries forever
  setup_queries: [],
  storage: {
    datadir: './var'
  },
  engines: [
    { label: 'mycluster1',
      executer: {
        name: 'hiveserver2',
        host: 'hs2.mycluster1.local',
        port: 10000,
        usename: 'hive',
        support_database: false
      },
      monitor: null
    },
  ],
};
```

`username` should be same as user name that hive job will be executed on. (`password` is not required for NOSASL transport.)

For UDFs, you can specify statements before query executions in `setup_queries`.

```js
var servers = exports.servers = {
  listen: 3000,
  fetch_lines: 1000,
  query_timeout: null,
  setup_queries: [
    "add jar /path/to/jarfile/foo.jar",
    "create temporary function foofunc as 'package.of.udf.FooFunc'",
    "create temporary function barfunc as 'package.of.udf.BarFunc'"
  ],
  storage: {
    datadir: './var'
  },
  engines: [
    { label: 'mycluster1',
      executer: {
        name: 'hiveserver2',
        host: 'hs2.mycluster1.local',
        port: 10000,
        support_database: false
      },
      monitor: null
    },
  ],
};
```

### HiveServer

Classic HiveServer is available if you want database supports instead of HiveServer2.

```js
var servers = exports.servers = {
  listen: 3000,
  fetch_lines: 1000,
  query_timeout: null,
  setup_queries: [],
  storage: {
    datadir: './var'
  },
  engines: [
    { label: 'mycluster1',
      executer: {
        name: 'hiveserver',  // HiveServer(1)
        host: 'hs1.mycluster1.local',
        port: 10000,
        support_database: true,
        default_database: 'mylogs1'
      },
      monitor: null
    },
  ],
};
```

### Presto

For Presto, use `presto` executer.

```js
var servers = exports.servers = {
  listen: 3000,
  fetch_lines: 1000,
  query_timeout: 30, // 30 seconds for Presto query timeouts (it will fail)
  setup_queries: [],
  storage: {
    datadir: './var'
  },
  engines: [
    { label: 'prestocluster1',
      executer: {
        name: 'presto',
        host: 'coordinator.mycluster2.local',
        port: 8080,
        catalog: 'hive',  // required configuration argument
        support_database: true,
        default_database: 'mylogs1'
      },
      monitor: null
    },
  ],
};
```

### Multi clusters and engines

Shib supports 2 or more engines for a cluster, and 2 or more clusters with same engines. These patterns are available.

* HiveServer1, HiveServer2 and Presto for same data source
* 2 or more catalogs for same Presto cluster
* Many clusters which has one of HiveServer, HiveServer2 or Presto

You should write configurations in `engines` how you wants. `fetch_lines`, `query_timeout` and `setup_queries` in each engines overwrites global default of these configurations.

For example, This is configuration example.
 * ClusterA has HiveServer2
   * listenes port 10000
   * customized udfs in `foo.jar` are availabe
 * ClusterB has HiveServer
   * listenes port 10001
   * customized udfs in `foo.jar` are available
 * Presto cluster is configured with `hive` catalog and `hive2` catalog
   * udfs are not available

```js
var servers = exports.servers = {
  listen: 3000,
  fetch_lines: 1000,
  query_timeout: null,
  setup_queries: [
    "add jar /path/to/jarfile/foo.jar",
    "create temporary function foofunc as 'package.of.udf.FooFunc'",
    "create temporary function barfunc as 'package.of.udf.BarFunc'"
  ],
  storage: {
    datadir: './var'
  },
  engines: [
    { label: 'myclusterA',
      executer: {
        name: 'hiveserver2',
        host: 'master.a.cluster.local',
        port: 10000,
        support_database: false
      },
      monitor: null
    },
    { label: 'myclusterB',
      executer: {
        name: 'hiveserver',
        host: 'master.b.cluster.local',
        port: 10001,
        support_database: true,
        default_database: 'mylogs1'
      },
      monitor: null
    },
    { label: 'prestocluster1',
      executer: {
        name: 'presto',
        host: 'coordinator.p.cluster.local',
        port: 8080,
        catalog: 'hive',
        support_database: true,
        default_database: 'mylogs1',
        query_timeout: 30,  // overwrite global config
        setup_queries: []   // overwrite global config
      },
      monitor: null
    },
    { label: 'prestocluster2',
      executer: {
        name: 'presto',
        host: 'coordinator.p.cluster.local',
        port: 8080,
        catalog: 'hive2',  // one engine config per catalogs
        support_database: true,
        default_database: 'default',
        query_timeout: 30,  // overwrite global config
        setup_queries: []   // overwrite global config
      },
      monitor: null
    }
  ],
};
```

### Access Control

Shib have access control list for Databases/Tables. Default is 'allow' for all databases/tables.

Shib's access control rules are:
  * configure per executer
  * database level default ('allow' or 'deny') without any optional rules makes that database unvisible
  * database level default + allow/deny table list makes its tables visible/unvisible
    * in this case, this 'database' is visible
  * default 'allow' or 'deny' decides visibilities of databases without any optional rules

'unvisible' databases and tables:
 * are not shown in tables/partitions list and schema list
 * cannot be queried by users (these queries always fails)

Access control options are written in 'executer' like this:

```js
executer: {
  name: 'presto',
  host: 'coordinator.p.cluster.local',
  port: 8080,
  catalog: 'hive',
  support_database: true,
  default_database: 'default',
  query_timeout: 30,
  setup_queries: [],
  access_control: {
    databases: {
      secret: { default: "deny" },
      member: { default: "deny", allow: ["users"] },
      test:   { default: "allow", deny: ["secretData", "userMaster"] },
    },
    default: "allow"
  }
},
```

## Monitors

`monitor` configurations are used to get query status and to kill queries.
  * `hiveserver` has no monitoring features
  * `hiveserver2` monitor is under development
  * `presto` monitor is under development

### Huahin Manager

For monitors in CDH4 + MRv1 environment, Huahin manager is available.

To show map/reduce status, and/or to kill actual map/reduce jobs behind hive query, shib can use Huahin Manager. Current version supports only 'Huahin Manager CDH4 + MRv1' only.

http://huahinframework.org/huahin-manager/

Configure `monitor` argument like below instead of `null`.

```js
monitor: {
  name : 'huahin_mrv1',
  host: 'localhost',
  port: 9010
}
```

## As HTTP Proxy for query engines

POST query string into `/execute` with some parameters.

```
curl -s -X POST -F 'querystring=SELECT COUNT(*) AS cnt FROM yourtable WHERE field="value"' http://shib.server.local:3000/execute | jq .
{
  "queryid": "69927e67c5b1d5f665697943cc4867ec",
  "results": [],
  "dbname": "default",
  "engine": "hiveserver",
  "querystring": "SELECT COUNT(*) AS cnt FROM yourtable WHERE field=\"value\""
}
```

Specify `engineLabel` and `dbname` for non-default query engines and databases:

```
curl -s -X POST -F "engineLabel=presto" -F "dbname=testing" -F "querystring=SELECT COUNT(*) AS cnt FROM yourtable WHERE field='value'" http://shib.server.local:3000/execute
```

Then, fetch query's status whenever you want.

```
curl -s http://shib.server.local:3000/status/69927e67c5b1d5f665697943cc4867ec 
executed
```

Or get whole query object.

```
curl -s http://shib.server.local:3000/query/69927e67c5b1d5f665697943cc4867ec | jq .
{
  "queryid": "69927e67c5b1d5f665697943cc4867ec",
  "results": [
    {
      "resultid": "969629614dff69411a2f4f1733c9616a",
      "executed_at": "Wed Feb 26 2014 16:02:00 GMT+0900 (JST)"
    }
  ],
  "dbname": "default",
  "engine": "hiveserver",
  "querystring": "SELECT COUNT(*) AS cnt FROM yourtable WHERE field=\"value\""
}
```

If this query object has `executed` status, or a member of `results`, you can fetch its result by `resultid`.

```
# if you want elasped time or bytes or lines or ....
curl -s http://shib.server.local:3000/result/969629614dff69411a2f4f1733c9616a | jq .
{
  "schema": [
    {
      "type": "bigint",
      "name": "cnt"
    }
  ],
  "completed_msec": 1393398893759,
  "completed_at": "Wed Feb 26 2014 16:14:53 GMT+0900 (JST)",
  "completed_time": null,
  "bytes": 6,
  "queryid": "69927e67c5b1d5f665697943cc4867ec",
  "executed_time": null,
  "executed_at": "Wed Feb 26 2014 16:14:52 GMT+0900 (JST)",
  "executed_msec": 1393398892752,
  "resultid": "969629614dff69411a2f4f1733c9616a",
  "state": "done",
  "error": "",
  "lines": 2
}
# raw result data as TSV (fast)
curl -s http://shib.server.local:3000/download/tsv/969629614dff69411a2f4f1733c9616a | jq .
CNT
1234567
# or CSV (slow)
curl -s http://shib.server.local:3000/download/csv/969629614dff69411a2f4f1733c9616a | jq .
"CNT"
"1234567"
```

These HTTP requests/response are same with that javascript on browser does.

* * * * *

## TODO

* Monitor support of `hiveserver2` and `presto`
* 'hiveserver2' database support (for Hive 0.13 or later?)

## License

Copyright 2011- TAGOMORI Satoshi (tagomoris)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
