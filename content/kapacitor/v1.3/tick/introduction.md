---
title: TICKscript Language Introduction

menu:
  kapacitor_1_3:
    name: Introduction
    identifier: tick_intro
    parent: tick
    weight: 1
---
# Contents
* [Overview](#overview)
* [Nodes](#nodes)
* [Pipelines](#pipelines)
* [Basic Examples](#basic-examples)
* [TICKscript in Kapacitor](#tickscript-in-kapacitor)
* [TICKscript in Chronograf](#tickscript-in-chronograf)

# Overview

Kapacitor uses a Domain Specific Language(DSL) named **TICKscript** to define **tasks** involving the extraction, transformation and loading of data.  TICKscript is used in `.tick` files to define **pipelines** for processing data.  The TICKscript language is designed to chain together the invocation of data processing operations defined in **nodes**.  The Kapacitor [Getting Started](/kapacitor/v1.3/introduction/getting_started/) guide introduces TICKscript basics in the context of that product.  For a better understanding of what follows, it is recommended that the reader review that document first.

Each script has a flat scope and each variable in the scope can reference a literal value, such as a string, an integer or a float value, or a node instance with methods that can then be called.

These methods come in two forms.

* **Property methods** &ndash; A property method modifies the internal properties of a node and returns a reference to the same node.  Property methods are called using dot ('.') notation.
* **Chaining methods** &ndash; A chaining method creates a new child node and returns a reference to it.  Chaining methods are called using pipe ('|') notation.

# Nodes

In TICKscript the fundamental type is the **node**.  A node has **properties** and, as mentioned, chaining methods.  A new node can be instantiated from a parent or sibling node using a chaining method of that parent or sibling node.  For each **node type** the signature of this method will be the same, regardless of the parent or sibling node type.  The chaining method can accept zero or more arguments used to initialize internal properties of the new node instance.  Common node types are `batch`, `query`, `stream`, `from`, `eval` and `alert`, though there are dozens of others.  The most common argument types used during instantiation are:

   * Strings representing usually a query expression, but can also represent some other property of the node, for example an endpoint context.
      * example: `query('SELECT sum(value) FROM "pages"."default".errors')`
      * example: `httpOut('top10')`
   * **lambda expressions** representing small function blocks to be applied to the data.
      * example: `eval(lambda: "lows.count"/("norms.count" + "lows.count" ))`
   * Duration literals.
      * example: `shift(6h)`
   * Other nodes and their pipelines captured in variables within the script.

**Example 1 &ndash; chaining method takes a node variable**
```javascript
var errors = batch
    |query('SELECT sum(value) FROM "pages"."default".errors')
...
// Get views batch data
var views = batch
    |query('SELECT sum(value) FROM "pages"."default".views')
...
// Join errors and views
errors
    |join(views)
        .as('errors', 'views')
```

Example 1 shows how the variable `views` is used in the call to the chaining method that instantiates a new `join` node.

The top level nodes that establish the processing style, `stream` and `batch`, are simply declared and take no arguments.  Nodes with more complex sets of properties rely on **Property methods** for their internal configuration.  

Each node type **wants** data in either batch or stream mode.  Some can handle both. Each node type also **provides** data in batch or stream mode.  Some can provide both.  This _wants/provides_ pattern is key to understanding how nodes work together.  Taking into consideration the _wants/provides_ pattern, four general node use cases can be defined:

   * _want_ a batch and _provide_ a stream - for example, when computing an average or a minimum or a maximum.
   * _want_ a batch and _provide_ a batch - for example, when identifying outliers in a batch of data.
   * _want_ a stream and _provide_ a batch - for example, when grouping together similar data points.
   * _want_ a stream and _provide_ a stream - for example, when applying a mathematical function like a logarithm to a value in a point.

The [node reference documentation](/kapacitor/v1.3/nodes/) lists the property and chaining methods of each node along with examples and descriptions.

# Pipelines

Every TICKscript is broken into one or more **pipelines**.  The nodes within a pipeline can be assigned to variables allowing the results of different pipelines to be combined or for sections of the pipeline to broken into reasonably understandable functional units.  Alternately a simple TICKscript can use only one anonymous pipeline.  The pipeline processing style is initially set as either `stream` or `batch`.  These two types of pipelines cannot be combined.  

### Stream or Batch?

With `stream` processing, datapoints are read, as in a classic data stream, point by point as they arrive.  With `stream` Kapacitor subscribes to all writes of interest in InfluxDB.  With `batch` processing a frame of 'historic' data is read from the database and then processed.  With `stream` processing data can be transformed before being written to InfluxDB.  With `batch` processing, the data should already be stored in InfluxDB.  After processing, it can also be written back to it.  

Which to use depends upon system resources and the kind of computation being undertaken.  When working with a large set of data over a long time frame `batch` is preferred.  It leaves data stored on the disk until it is required, though the query, when triggered, will result in a sudden high load on the database.  Processing a large set of data over a long time frame with `stream` means needlessly holding potentially billions of data points in memory.  When working with smaller time frames  `stream` is preferred.  It lowers the query load on InfluxDB.  

### Pipelines as graphs

Pipelines in Kapacitor are directed acyclic graphs ([DAGs](https://en.wikipedia.org/wiki/Directed_acyclic_graph)).  This means that
each edge has a direction down which data flows, and that there cannot be any cycles in the pipeline.  An edge can also be thought of as the data-flow relationship that exists between a parent node and its child or between a child and its sibling.  

At the start of any pipeline will be declared one of two fundamental edges.  This first edge establishes the style of processing for all edges that follow.

* `stream`&rarr;`from()`&ndash; an edge that transfers data a single data point at a time.
* `batch`&rarr;`query()`&ndash; an edge that transfers data in chunks instead of one point at a time.  

When connecting nodes and then creating a new Kapacitor task, Kapacitor will check whether or not the TICKscript syntax is well formed, and if the new edges are applicable to the most recent node.  However full functionality of the pipeline will not be validated until runtime, when error messages can appear in the Kapacitor log.

# Basic Examples

**Example 2 &ndash; An elementary stream &rarr; from() pipeline**
```javascript
stream
    |from()
        .measurement('cpu')
    |httpOut('dump')
```

The simple script in Example 2 can be used to create a task with the default Telegraf database.

```
$ kapacitor define sf_task -type stream -tick sf.tick -dbrp telegraf.autogen
```

The task, `sf_task`, will simply dump the latest cpu datapoint as JSON to the HTTP REST endpoint(e.g http<span>://localhost:</span><span>9092/kapacitor/v1/tasks/sf_task/dump</span>).  

This example contains three nodes:

   * The base `stream` node.
   * The requisite `from()` node, that defines the stream of data points.
   * The processing node `httpOut()`, that publishes the data it receives to the REST service of Kapacitor.  

It contains two edges.

   * `stream`&rarr;`from()`&ndash; sets the processing style and the data stream.
   * `from()`&rarr;`httpOut()`&ndash; passes the data stream to the HTTP output processing node.

It contains one property method, which is the call on the `from()` node to `.measurement('cpu')` defining the measurement to be used for further processing.  

**Example 3 &ndash; An elementary batch &rarr; query() pipeline**

```javascript
batch
    |query('SELECT * FROM "telegraf"."autogen".cpu WHERE time > now() - 10s')
        .period(10s)
        .every(10s)
    |httpOut('dump')
```

When used to create a task called `bq_task` with the default Telegraf database, the TICKscript in Example 3 will simply dump the last cpu datapoint of the batch of measurements representing the last 10 seconds of activity to the HTTP REST endpoint(e.g. http://localhost:9092/kapacitor/v1/tasks/bq_task/dump).

This example contains three nodes:

   * The base `batch` node.
   * The requisite `query()` node, that defines the data set.
   * The processing node `httpOut()`, that defines the one step in processing the data set.  In this case it is to publish it to the REST service of  Kapacitor.

It contains two edges.

   * `batch`&rarr;`query()`&ndash; sets the processing style and data set.
   * `query()`&rarr;`httpOut()`&ndash; passes the data set to the HTTP output processing node.

It contains two property methods, which are called from the `query()` node.    

   * `period()`&ndash; sets the period in time which the batch of data will cover.
   * `every()`&ndash; sets the frequency at which the batch of data will be processed.

# TICKscript in Kapacitor

In Kapacitor TICKscript is essential for creating and updating tasks.  On the command line, three key commands provide the basics:

   * `define`&ndash; along with a TICKscript, creates or updates a task.
      * To create a new task: `$ kapacitor define <TASKNAME> -tick <PATH_TO_TICKSCRIPT> -type [stream OR batch] -dbrp <DB_NAME>.<RETENTION_POLICY>`
      * To update a task. `$ kapacitor define <TASKNAME> -tick <PATH_TO_TICKSCRIPT>`
   * `list`&ndash; shows existing tasks, their status and the InfluxDB databases, with which they are used. `$ kapacitor list tasks`
   * `show`&ndash; retrieves the TICKscript that defines the task, along with metadata and statistics. `$ kapacitor show <TASKNAME>`

If a task needs to be deleted:
```
$ kapacitor delete tasks <TASKNAME>
```
<!-- FIXME: uncomment this line once the Working with Kapacitor document is ready.
For more information see the section [Working with Kapacitor](tbd/)   
-->

# TICKscript in Chronograf

When Chronograf is properly integrated with Kapacitor, its Alert rules feature can be used to generate a Kapacitor task.  The generated TICKscripts written by Chronograf into Kapacitor can offer insight into how the user can develop TICKscripts.  See [Create a Kapacitor Alert](/chronograf/v1.3/guides/create-a-kapacitor-alert/) in the Chronograf documentation for a step by step guide in how to setup a Kapacitor Alert from Chronograf.  

If a Chronograf alert has been successfully created, it can be listed and shown from the command line like any other task.

**Example 4 &ndash; Listing with task generated by Chronograf**

```
$ kapacitor list tasks
ID                                                 Type      Status    Executing Databases and Retention Policies
bq_task                                            batch     enabled   true      ["telegraf"."autogen"]
chronograf-v1-af6eeb07-f3ca-4ac8-a533-df2ca50f619d stream    enabled   true      ["website"."autogen"]
cpu_alert                                          stream    enabled   true      ["telegraf"."autogen"]
...
```
The task ID, which is formed from the "chronograf" token, a version tag and a UUID, stands out as being generated in Chronograf.

To inspect the generated TICKscript use the show command, which will list the generated script in the section TICKscript.

**Example 5 &ndash; A TICKscript generated by Chronograf***

```
$ kapacitor show chronograf-v1-af6eeb07-f3ca-4ac8-a533-df2ca50f619d
ID: chronograf-v1-af6eeb07-f3ca-4ac8-a533-df2ca50f619d
...
TICKscript:
var db = 'website'

var rp = 'autogen'

var measurement = 'responses'

var groupBy = []

var whereFilter = lambda: ("lb" == '17.99.99.71')

var name = 'test rule'

var idVar = name + ':{{.Group}}'

var message = ' {{.ID}} {{.Group}} {{.Level}} {{ index .Fields "value" }} {{.Time}}'

var idTag = 'alertID'

var levelTag = 'level'

var messageField = 'message'

var durationField = 'duration'

var outputDB = 'chronograf'

var outputRP = 'autogen'

var outputMeasurement = 'alerts'

var triggerType = 'threshold'

var crit = 100

var data = stream
    |from()
        .database(db)
        .retentionPolicy(rp)
        .measurement(measurement)
        .groupBy(groupBy)
        .where(whereFilter)
    |eval(lambda: "r500")
        .as('value')

var trigger = data
    |alert()
        .crit(lambda: "value" > crit)
        .stateChangesOnly()
        .message(message)
        .id(idVar)
        .idTag(idTag)
        .levelTag(levelTag)
        .messageField(messageField)
        .durationField(durationField)
        .log('/tmp/ws-alerts.log')

trigger
    |influxDBOut()
        .create()
        .database(outputDB)
        .retentionPolicy(outputRP)
        .measurement(outputMeasurement)
        .tag('alertName', name)
        .tag('triggerType', triggerType)

trigger
    |httpOut('output')
...
```
Example 5 shows how Chronograf has generated user friendly variable identifiers for the database name, the measurement name, fields, tags and literals to be used in the pipeline.  The pipeline is then broken up using additional variables to make its functional units more easily understandable.  

<!-- TODO - get correct link -->
The next section covers [TICKscript syntax](/kapacitor/v1.3/tick/syntax/) in more detail. [Continue...](/kapacitor/v1.3/tick/syntax/)