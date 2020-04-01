## Caf.js PROGRAMMING
<!-- .element:  style="text-align:center; text-transform:none" -->


![](process.env.CA_NAME/assets/pig.svg)
<!-- .element:  width="500" heigh="500" -->

by Antonio Lain

---

## Actors

![](process.env.CA_NAME/assets/pig_actors.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .slide: class="two-floating-elements-70"  style="float: left" -->
<!-- .element:  style="text-align:left" -->
### A Caf.js ACTOR HAS
<!-- .element:  style="text-align:center; text-transform:none" -->

![](process.env.CA_NAME/assets/Actor.svg)
<!-- .element: class="plain" style="float: right" width="30%" -->
* a queue that serializes message processing
* a location-independent name
* some private state
* the ability to change behavior
* or interact with other actors

Initial design based on Erlang/OTP *gen_server*

---

## Why Actors?

```
var setTimeoutPromise = util.promisify(setTimeout);
exports.methods = {
    async __ca_init__() {
        this.state.counter = 0;
        return [];
    }
    async increment() {
        var oldValue = this.state.counter;
        this.state.counter = this.state.counter + 1;
        await setTimeoutPromise(1000);
        assert(this.state.counter === (oldValue + 1));
        return [null, this.state.counter];
    }
}

```
<!-- .element: class="hljs javascript" spellCheck="false" -->

Message serialization avoids races

Assertion always true!


---

### Client Library
```
var URL = 'http://root-hello.vcap.me/#from=foo-ca1&ca=foo-ca1';
var s = new caf_cli.Session(URL);
s.onopen = async function() {
    try {
        var cnt = await s.increment().increment().getPromise();
        console.log('Got ' + cnt);
        s.close();
    } catch (err) {
        s.close(err);
    }
}
s.onclose = function(err) {
    if (err) {
       console.log(myUtils.errToPrettyStr(err));
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Method `increment()` dynamically added to `s`
* Multiple `increment()` calls in a single request
* Common library for browser, cloud, script and IoT

---

### Transactional Actors
<!-- .element:  style="text-align:center" -->

![](process.env.CA_NAME/assets/Rescue.svg)

* Each request processed in a transaction
* Delay external actions with transactional plugins
* On error, rollback state and ignore actions
* Checkpoint `this.state` and actions before externalize
* On commit, repeat actions until succeed

---

### Much more...
<!-- .element:  style="text-align:center" -->

* Autonomous behavior with `__ca_pulse__`
* Custom recovery with `__ca_resume__`
* Non-transactional state with `this.scratch`
* Hot state versioning with `__ca_upgrade__`
* Support for IoT devices with `caf_iot`

See https://cafjs.github.io/api/caf_ca/

---
## Sharing Actors


![](process.env.CA_NAME/assets/pig_sharing.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .element:  style="text-align:left" -->
### Distributed Data Structures + Actors
<!-- .element:  style="text-align:center" -->

* Actors cannot efficiently share data that is
  * read by many
  * and modified by a few
* DDS + Actors = Actors
  * Add efficient sharing with Actor semantics
  * Use local shared-memory, skip message processing
  * **No data races, deadlocks, or an ugly failure mode!**
  * Feasible under some assumptions (Lesani&Lain'13)
* Implemented with `immutable.js` collections
  * Supports cloud, browser, and device
* See  https://cafjs.github.io/api/caf_sharing/ for details

---
<!-- .element:  style="text-align:left" -->
### SharedMap Example (Writer)
<!-- .element:  style="text-align:center; text-transform:none" -->

```
exports.methods = {
    async __ca_init__() {
        this.$.sharing.addWritableMap('writer', 'cnt');
        return [];
    },
    async increment() {
        var $$ = this.$.sharing.$;
        var counter = $$.writer.get('counter') || 0;
        $$.writer.set('counter', counter + 1);
        return [null, counter];
    }
};
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Single writer enforced by naming
  * Map name `cnt` relative to writer Actor's name
* Update map with transactional plugin
  * Rollback or commit with message

---
<!-- .element:  style="text-align:left" -->
### SharedMap Example (Reader)
<!-- .element:  style="text-align:center; text-transform:none" -->

```
exports.methods = {
    async __ca_init__() {
        this.$.sharing.addReadOnlyMap('rep', mapOf(this, 'cnt'));
        return [];
    },
    async getCounter() {
        var $$ = this.$.sharing.$;
        var counter1 = $$.rep.get('counter');
        await setTimeoutPromise(1000);
        var counter2 = $$.rep.get('counter');
        assert (counter1 === counter2);
        return [null, counter1];
    }
};
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Custom function `mapOf()` finds source map full name
* *Reader Isolation:* maps only change between messages
  * Assertion is always `true`

---
<!-- .element:  style="text-align:left" -->
### Sharing Code with a SharedMap
<!-- .element:  style="text-align:center; text-transform:none" -->

```
async installCode() { // Writer
    var $$ = this.$.sharing.$;
    var b = "return prefix + this.get('counter') + Math.random();";
    $$.writer.setFun('computeLabel', ['prefix'], b);
    return [];
}
async computeLabel(prefix) { // Reader
    var $$ = this.$.sharing.$;
    var res = $$.rep.applyMethod('computeLabel', [prefix]));
    return [null, res];
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Glitch-free code update in cloud, browser, and device
* `setFun` stores a serialized **pure** function
* `applyMethod` binds `this` to the map and calls function
* Many uses: getters/setters, filtering, policy enforcement

---
## Authorization

![](process.env.CA_NAME/assets/pig_linked.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .slide: class="two-floating-elements"  style="float: left" style="text-align:left"  -->
### Security in Caf.js
<!-- .element:  style="text-align:center; text-transform:none" -->

![](process.env.CA_NAME/assets/Mechanisms.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

* Safe **collaborative** multi-tenancy
  * Help application programmers
* Authenticate with Single Sign-On
  * SRP + JSON Web Token
  * Signed tokens form semilattice
    * No *CA-in-the-middle* attacks
* Application Trusted Bus for Actors
  * Source anti-spoofing
  * Single-writer guarantees
* Policy engine for authorization
  * Linked local namespaces
  * See next slides!

---
<!-- .element:  style="text-align:left" -->
### It's all in the name
<!-- .element:  style="text-align:center" -->

* Main ideas from SDSI (Rivest&Lampson'96)
* A principal is globally identified by its public key or
  * `username` + public key of an `accounts` service
* A principal has a **local namespace** with names bound to:
  * **owned resources**: images, apps, Actors, *SharedMaps*,...
  * other **principals**
  * **links** to names in other namespaces
  * **groups** of the above
* Linking is sound
  * *Principal + local name* is globally unique
* ACL queries implemented with transitive closure
  * First match local names, then keep following links

---
<!-- .element:  style="text-align:left" -->
### Advantages
<!-- .element:  style="text-align:center" -->

* Decentralized authorization
  * easy to federate `accounts` services
  * or sign your own tokens
* Monotonicity
  * partial information leads to conservative decisions
* Expressivity
  * concise ownership-based policies
  * built-in support for groups of
    * resources, e.g., *all my apps*,
    * or principals, e.g., *all my friends*
* Delegation
  * create groups by linking to other groups

---
<!-- .element:  style="text-align:left" -->
### Novel implementation
<!-- .element:  style="text-align:center" -->

* *SharedMaps* represent namespaces
  * single-writer is a **strong** property **within an app**
  * wrapper *AggregateMap* transparently manages linking
* Features
  * no certificates or user private keys
  * revocation is really fast
  * discovery is efficient, and managed by the framework
  * properties of *Sharing Actors* lead to **safe policy changes**
* More info
  * https://cafjs.github.io/api/caf_security

---
## Persistent Sessions

![](process.env.CA_NAME/assets/pig_sessions.svg)
<!-- .element:  width="500" heigh="500" -->
---

<!-- .element:  style="text-align:left" -->
### Cookies...
<!-- .element:  style="text-align:center" -->

What's **wrong** with cookies for session management?

* Chosen by the server, not by the client
* Cannot have sensible, human-friendly, values
* Do not move between devices
* Browser-based, non-portable to basic devices

---
<!-- .slide: class="two-floating-elements-70"  style="float: left" style="text-align:left"  -->
### Managing sessions in Caf.js
<!-- .element:  style="text-align:center; text-transform:none" -->

![](process.env.CA_NAME/assets/Queue.svg)
<!-- .element: class="plain" style="float: right" width="30%" -->

* Add notification queues to an Actor
  * support off-line clients
* Queue name identify **session**
  * simple name, chosen by the client
  * made unique by the Actor's name
* Contents of the queue visible to the app
  * custom handling of undelivered notifications
* Same client on browser, Cloud, and gadget


---
<!-- .element:  style="text-align:left" -->
### Notifications Example
<!-- .element:  style="text-align:center" -->
```
exports.methods = {
    async notify(to, msg) {
        var from = this.$.session.getSessionId()
        this.$.session.notify([from, msg], to);
        return [];
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->
```
var URL = 'http://root-hello.vcap.me/#from=foo-ca1&ca=foo-ca1';
var s = new caf_cli.Session(URL);
s.changeSessionId('tv');
s.onopen = async function() {
    try { await s.notify('car', 'Hello');} catch (err) { ... };
}
s.onmessage = function(msg) { //similar for 'car'
    var notif = caf_cli.getMethodArgs(msg);
    console.log('from:'+ notif[0] + ' msg:' + notif[1]);
};
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* A `tv` client notifies a `car` client, even if `car` is off-line
* Switch cars, select queue `car`, and read notification

---
### Reliable stateless clients
![](process.env.CA_NAME/assets/Toaster.svg)
<!-- .element:  width="400" heigh="400" -->

* Stateless client crashes, or unplanned change of device
  * Guarantee that certain actions are only done once
  * You didn't want **two** toasters, did you?
* Core ideas from Bernstein&Hsu&Mann'90
* See https://cafjs.github.io/api/caf_session/

---
## Components

![](process.env.CA_NAME/assets/pig_components.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .element:  style="text-align:left" -->
### Caf.js COMPONENTS
<!-- .element:  style="text-align:center; text-transform:none" -->

* Describe a **hierarchy** of components with JSON
  * templates, links, and environment properties
* **Asynchronous** constructors, **deterministic** creation order
  * access remote services without blocking main loop
  * dependency injection
* Supervision tree, where a parent component
    * monitors children,
    * triggers local recovery if needed,
    * and bubbles up failure if local recovery fails.
* Based on the SmartFrog framework and Erlang/OTP
* In Caf.js **everything** is built with components!

---

<!-- .element:  style="text-align:left" -->
### Transforming Descriptions
<!-- .element:  style="text-align:center" -->

![](process.env.CA_NAME/assets/Transforms.svg)
<!-- .element:  width="900" heigh="350" -->

* Common deployment workflow
  * Customize base template
  * Create instances with different arguments
  * Propagate arguments to inner components
  * Fill the rest from the environment

---
<!-- .element:  style="text-align:left" -->
### hello.json
<!-- .element:  style="text-align:center; text-transform:none" -->
```
{
    "module": "./hello",
    "description": "Hello World Component",
    "name": "foo",
    "env": {
        "maxRetries": 10,
        "retryDelay": 1000,
        "msg": "Hello World!"
    },
    "components": [
       ...
    ]
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* `module` implementation of this component
* `name` key to register the new component in local context
* `env` set of properties to configure the component
* `components` optional children components

---
<!-- .element:  style="text-align:left" -->
### hello.js
<!-- .element:  style="text-align:center; text-transform:none" -->
```
var caf_comp = require('caf_components');
var genContainer = caf_comp.gen_container;
// factory method
exports.newInstance = async function($, spec) {
    try {
        var that = genContainer.constructor($, spec);
        that.hello =  function() {
            console.log(spec.name + ':' + spec.env.msg);
        }
        return [null, that];
    } catch (err) {
        return [err]
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Functions called `newInstance` are factory methods
    * `$` is the local context provided by its parent
    * `spec` a parsed description from `hello.json`

---
<!-- .element:  style="text-align:left" -->
### main.js
<!-- .element:  style="text-align:center; text-transform:none" -->
```
var main = require('caf_components');
main.load(null, null, 'hello.json', [module], function(err, $) {
    if (err) {
        console.log(main.myUtils.errToPrettyStr(err));
    } else {
        $.foo.hello();
    }
});
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Putting it all together
  * `load` parses description, creates hierarchy, and starts monitoring
  * `[module]` specify path(s) to load resources
  * `$.foo` new component in top level context `$`
* See https://cafjs.github.io/api/caf_components/

---
### Wrapping up

![](process.env.CA_NAME/assets/pig.svg)
<!-- .element:  width="500" heigh="500" -->

---

# Thank You

http://www.cafjs.com

https://github.com/cafjs/caf

Original Pig by Anniken & Andreas, NO

Creative Commons 3.0

---
# Extra

---
<!-- .slide: class="two-floating-elements"  style="float: left" style="text-align:left"  -->
### DDS + Actors = Actors
<!-- .element:  style="text-align:center" -->

![](process.env.CA_NAME/assets/sharing1.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

If the *Jocker* can
* modify the state of **any** actor
* at **any** time

it is trivial to implement a DDS

**But**...

---
<!-- .slide: class="two-floating-elements"  style="float: left" style="text-align:left"  -->
### DDS + Actors = Actors
<!-- .element:  style="text-align:center" -->

![](process.env.CA_NAME/assets/sharing2.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->


* Can actors emulate the *Jocker*'s actions?
  * Replace changes with messages
  * End result *observationally equivalent*
    * external entity cannot tell
* Emulation guarantees
  * No data races
  * No deadlocks
  * Better failure mode

---

### DDS + Actors = Actors

**Not** in the general case

But **yes** in the following **important** case (Lesani&Lain'13):

1. **Single Writer**: one actor *owns* a DDS instance
2. **Readers Isolation**: replicas change between messages
3. **Fairness**: an actor cannot **indefinitely** prevent other local actors from seeing new updates
4. **Writer Atomicity**: bundle changes after each message
5. **Monotonic Read Consistency**: replicas can be stale, but never roll back to older versions

---

<!-- .element:  style="text-align:left" -->
### SharedMap Implementation
<!-- .element:  style="text-align:center; text-transform:none" -->

* **Single Writer** *SharedMap* name relative to its CA
* **Readers Isolation** + **Fairness** with multiple local versions
  * Persistent data structure for efficient versioning
     * *Immutable.js*
  * For each message
     * pick the most recent version that is locally available
     * keep version until the next message
* **Writer Atomicity** use a Caf.js transactional plugin
* **Monotonic Read Consistency** track version numbers

---

<!-- .element:  style="text-align:left" -->
### Security Assumptions and Goals
<!-- .element:  style="text-align:center" -->
* Write apps with safe **collaborative** multi-tenancy
* Application writer co-designs framework+app
    * one app per *node.js* process(es)
* Apps expected to run trusted code but
  * for extra protection, frameworks do not fully trust apps
* Apps never trust each other
  * *app-to-app* similar to *app-to-external world*
* Service provider never trusts apps or framework
  * Isolates apps using IaaS/PaaS

---

### Recovery with a Persistent Session
![](process.env.CA_NAME/assets/Recover.svg)
<!-- .element:  width="500" heigh="500" -->

* Core ideas from Bernstein&Hsu&Mann'90
* Piggyback memento to identify last committed action
* Explicitly start and end a persistent session
    * If client crashed return memento
* Implemented with CA transactions+message serialization

---

<!-- .element:  style="text-align:left" -->
### Template Transform (helloTree++.json)
<!-- .element:  style="text-align:center; text-transform:none" -->
```
{
    "name" : "top",
     "components": [
         {
             "module": null,
             "name" : "foo"
         },
         {
             "module": "./hello",
             "name" : "bar",
             "env": {"msg" : "Bye!"}
         }
     ]
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Merge `xx.json` with `xx++.json` recursively
    * `name` identifies matching components
    * Assign `null` to `module` to delete a component
    * New components inserted after the last one touched

---

<!-- .element:  style="text-align:left" -->
### Linking Transform (helloTree++.json)
<!-- .element:  style="text-align:center; text-transform:none" -->

```
{
    "name" : "top",
    "env": {
        "myLogLevel": "DEBUG"
    },
    "components": [
         {
             "name" : "log",
             "env" : {
                 "logLevel": "$._.env.myLogLevel"
             }
         }
     ]
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* `$._.env` links to properties in the top level component
* Hides internal details from `load` arguments
   * merged top values propagate to inner components

---

<!-- .element:  style="text-align:left" -->
### Env Props Transform (helloTree++.json)
<!-- .element:  style="text-align:center; text-transform:none" -->

```
{
    "name" : "top",
    "env": {
        "myLogLevel": "process.env.MY_LOG_LEVEL||DEBUG",
        "somethingElse": "process.env.SOMETHING_ELSE||{\"goo\":2}"
    },
    "components": [
         {
             "name" : "log",
             "env" : {
                 "logLevel" : "$._.env.myLogLevel"
             }
         }
     ]
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* `process.env` reads properties from the environment
   * default values added with separator `||`
     * assumes JSON, if parsing fails defaults to string
---
<!-- .element:  style="text-align:left" -->
### Another main.js
<!-- .element:  style="text-align:center; text-transform:none" -->
```
var main = require('caf_components');
main.load(null, {name: 'bar', env: {msg: 'Bye!'}}, 'hello.json',
          [module], function(err, $) {
    if (err) {
        console.log(main.myUtils.errToPrettyStr(err));
    } else {
        $.bar.hello();
    }
});
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

Merge base configuration with

* `{name: 'bar', env: {msg: 'Bye!'}}`

---

<!-- .element:  style="text-align:left" -->
### Hierarchy with helloTree.json
<!-- .element:  style="text-align:center; text-transform:none" -->
```
{
    "module": "caf_components#supervisor",
    "name": "top", "env": {"maxRetries": 10, "retryDelay": 1000,
                           "dieDelay": 100, "maxHangRetries": 1,
                           "interval": 1000 },
    "components": [
        {
            "module": "caf_components#plug_log",
            "name": "log", "env": { "logLevel" : "DEBUG" }
        },
        {
            "module": "./hello",
            "name": "foo", "env": { "msg" : "Hello World!"}
        }
    ]
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Construction order: first, children in array-order, then parent
* Destruction reverses this order
* `caf_components#supervisor` similar to  `module.require("caf_components").supervisor`

---

<!-- .element:  style="text-align:left" -->
### Hierarchy with hello.js
<!-- .element:  style="text-align:center; text-transform:none" -->
```
exports.newInstance = function($, spec, cb) {
    $.log.debug('Initializing hello');
    // or $._.$.log.debug('Initializing hello');
    cb(null, {
    ...
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Dependency injection
  * description component order satisfies dependencies
    * safe to call the logger plugin
  * created sibling components in `$`
  * top level component is `$._`
    * and `$._.$` is the top level context
