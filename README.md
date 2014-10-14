TIBCO ActiveSpaces(tm) Binding for Node.js.
===

## Summary:

This is a native Node.js add-on for the TIBCO in-memory data grid TIBCO ActiveSpaces.

TIBCO ActiveSpaces provides support for large scale, distributed processing and storing of structured and unstructured information.

## Basic Concepts:

At the highest level AS has the concept of a Metaspace. Each Metaspace contains multiple Spaces. Each space contains tuples. Each tuple contains 1 or more elements.

A space is roughly analogous to a Table. A Tuple is roughly analogous to a row in a traditional RDBMS.

## Pre-requisites:

Make sure your TIBCO ActiveSpaces is installed.
AS_HOME environment variable should be properly set, so are the PATH and variants of LD_LIBARY_PATH on UNIX.  For example on MacOS:

    export DYLD_LIBRARY_PATH=$AS_HOME/lib:$DYLD_LIBRARY_PATH

## PRE-RELEASE STEPS
Before this repository made public, we need to use a private registry. Follow instructions in http://strongloop.com/strongblog/deploy-a-private-npm-registry-without-couchdb-or-redis/

Install the private registry:

    $ npm install -g reggie

Start the private registry (on a separate window):

    $ reggie-server -d ~/.reggie

Publish to the private registry:

    $ cd as-node
    $ reggie -u http://127.0.0.1:8080 publish

Install package from the private registry:

    npm install http://127.0.0.1:8080/package/activespaces/0.1.1

## Install

Install and include AS from NPM:

    npm install activespaces

Install from binary:

    npm install

Install from source:

    npm install --build-from-source

## Developing

The [node-pre-gyp](https://github.com/mapbox/node-pre-gyp#usage) tool is used to handle building from source and packaging. You need to set credential variables with AWS user who can write to s3 buckets:

    export node_pre_gyp_accessKeyId=xxx
    export node_pre_gyp_secretAccessKey=xxx

To recompile only the C++ files that have changed, first put `node-gyp` and `node-pre-gyp` on your PATH:

    export PATH=`npm explore npm -g -- pwd`/bin/node-gyp-bin:./node_modules/.bin:${PATH}

Then simply run:

    node-pre-gyp build

Similarly you can package and publish binary to AWS S3. The following command can do all in one command:
    node-pre-gyp clean build unpublish publish info

## API Example
Here is a basic description of the API. For more examples please refer to /test

### Connect and Basic Usage
```javascript
var ActiveSpaces = require('activespaces');
var as = new ActiveSpaces.ActiveSpaces;
as.Connect("ms",{discovery:"tcp://localhost:50000", listen:"tcp://localhost:50000",remote:"tcp://localhost:50002"});
```
This creates a basic ActiveSpaces object and connects to a local or remote space. The Connect supports the options: discovery, listen and remote

```javascript
var as = ActiveSpaces.Connect();
```
This creates the ActiveSpaces object and connects to the space using the parameters specified in the command line (process.argv)

```javascript
as.DefineSpace("test",
    {
        "field": {type: ActiveSpaces.AS_TYPE.TIBAS_INTEGER, nullable: false, encrypted: false},
        "field2": {type: ActiveSpaces.AS_TYPE.TIBAS_STRING, nullable: true},
        "field3": {type: ActiveSpaces.AS_TYPE.TIBAS_DOUBLE},
        "field4": {type: ActiveSpaces.AS_TYPE.TIBAS_DATETIME}
    },
    {fields: "field", key: ActiveSpaces.INDEXTYPE.HASH});
```
**DefineSpace(name,{field_name: {type:field_type,nullable:true/false,encrypted:true/false} },{fields:keys,type:index_type})**
Supports the majority of ActiveSpaces field Types

* Make sure that the keys are defined
* Ensure that there are no overlaps in terms of names
* Ensure that there are no overlaps in terms of names


```javascript
var testSpace = new ActiveSpaces.Space(as,"test",{role:ActiveSpaces.DISTRIBUTIONROLE.TIBAS_DISTRIBUTION_ROLE_SEEDER});
```
**new Space(ActiveSpaces, space_name, options)**
This 'gets the space' for further processing
* `ActiveSpaces`: The ActiveSpaces Object
* `space_name`: String, name of the space you want to connect to
* `options`: Object, defines the various options for the space connector. Currently only ROLE_SEEDER and ROLE_LEECH are used

```javascript
var testObj = "Hello";
testSpace.Listen(testObj,function(obj,ev){
    log.debug("Listen: " + JSON.stringify(ev));
    testSpace1.Invoke("TEST_INVOKE",{"field":3});
})
```
**space.Listen(context,callback(object,event))**
Set the listener callback for a space
* `context`: object passed in as closure for a callback
* `callback(obj,ev)`: callback which receieves a context object and an event. The event is an object with name/value pairs

```javascript
testSpace.PutTuple({
    "field":i,
    "field2":"String"+ i.toString(),
    "field3":i+1.8765,
    "field4":new Date()
});
```
**space.PutTuple(tuple)**
Puts a tuple if it doesn't exist into the space
* `tuple`: Name value pair object

```javascript
var result = testSpace.GetTuple({"field":3});
```
or
```javascript
testSpace.GetTuple({"field":3},function(result){
    //code
});
```
**result = space.GetTuple(tuple_keys)**
or
**space.GetTuple(tuple_keys,function(result))**
* `result`: result of the get, NULL if none
* `tuple_keys`: object containing the tuple keys to search for


```javascript
var testBrowser = new ActiveSpaces.Browser(testSpace,"",{});
testBrowser.Browse(function(obj){
    log.debug("Tuple from Browse: " + JSON.stringify(obj));
})
```
**browser = ActiveSpaces.Browser(space_name,filter,options)**
Creates the browser to be used
* `browser`: browser object that is the result of the browse setup
* `space_name: string of the space itself
* `filter`: refer to the ActiveSpaces documentation for the filter options
* `options`: not used

**browser.Browse(function(next_tuple){})**
This is the callback version of browse. You can also use **next_tuple = browse.Next()**
* `browser`: Browser object
* `next_tuple`: tuple passed in to browse


```javascript
var context = {"one":1};
testSpace1.SetInvoke("TEST_INVOKE",context,function(obj,ev){
    log.debug("Invoked "+ context["one"]);
    log.debug("input: " + JSON.stringify(ev));
});

testSpace.Invoke("TEST_INVOKE",{"field":3});
```
**space.SetInvoke(invoke_name,context,callback(obj,ev))**
Creates an invokable function on a space
* `space`: Space original retrieved
* `invoke_name`: name of the invoke function
* `context`: object passed into the callback as closure
* `callback(obj,ev): obj is the closure, ev is the tuple to be used as context

**space.Invoke(invoke_name,tuple_context)**
Invokes the space with the named function
* `invoke_name`: name set up by `SetInvoke`
* `tuple_context`: tuple key on space that is used to identify the seeder, pass in as `ev` in callback
*


```javascript
testSpace.SetSeederInvoke("TEST_SEEDER_INVOKE",context,function(obj,ev){
    log.debug("Invoked Seeder "+ context["one"]);
    log.debug("input: " + JSON.stringify(ev));
})

testSpace.InvokeSeeders("TEST_SEEDER_INVOKE",{"field":3});
```
