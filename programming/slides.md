## CAF.js PROGRAMMING
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
### A CAF.js ACTOR HAS
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

What's wrong with this code?
<!-- .element: class="fragment" data-fragment-index="0"-->

```
exports.methods = {
    increment: function(cb) {
        var self = this;
        var oldValue = this.state.counter;
        this.state.counter = this.state.counter + 1;
        setTimeout(function() {
            assert(self.state.counter === (oldValue + 1));
            cb(null, self.state.counter);
        }, 1000);
    }
}
```
<!-- .element: class="hljs javascript fragment" data-fragment-index="0" spellCheck="false" -->

Races, races, races...
<!-- .element: class="fragment" data-fragment-index="1"-->

---

### And this one?

```
exports.methods = {
    hello: function(cb) {
        if (this.isMonday()) {
            cb(null, 'hello');
        } else {
            setTimeout(function() {
                cb(null, 'hello');
            }, 1000);
        }
    }
}
```
<!-- .element: class="hljs javascript fragment" data-fragment-index="0" spellCheck="false" -->

Releasing Zalgo in async APIs (see Isaac Schlueter's blog).

<!-- .element: class="fragment" data-fragment-index="1"-->

---

### What about failures?
```
exports.methods = {
    transfer: function(sender, receiver, money, cb) {
        if (this.state[sender] > money) {
            this.state[sender] = this.state[sender] - money;
            // Crashes here...
            this.state[receiver] = this.state[receiver] + money;
            cb(null, 'done');
        } else {
            cb(new Error('Not enough funds!'));
        }
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

Keeping a consistent internal state

* for several years
* with a billion CAs
* programmed by back-end newbies

---
### CAF.js ACTORS TO THE RESCUE
<!-- .element:  style="text-align:center; text-transform:none" -->

![](process.env.CA_NAME/assets/Rescue.svg)

* No races
  * Serialized message processing
* Zalgo locked up
  * Decoupling queue
* Recover from failures gracefully
  * Request processed in a transaction
  * Checkpoint state before externalize

---
### Client Code: WebSocket on steroids
```
var URL = 'http://root-hello.vcap.me/#from=foo-ca1&ca=foo-ca1';
var s = new caf_cli.Session(URL);
s.onopen = function() {
    s.foo('good', function(err, good) {
        if (err) { // Application Error
            console.log(myUtils.errToPrettyStr(err));
        } else {
            console.log('Should be ' + good);
            s.close();
        }
    });
}
s.onclose = function(err) {
    err && console.log(myUtils.errToPrettyStr(err));//System Err
    !err && console.log('Done OK');
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* `s` dynamically populated with method `foo`
* Ordered requests with transparent retries
* Token-based authentication
<!-- * Support for multi-method transactions -->

---
### Implementing a transactional actor
```
exports.methods = {
    foo: function(something, cb) {
        this.state.something = something;
        this.$.session.notify("Got 'good'");
        setTimeout(function() {
            switch (something) {
            case 'good':
                cb(null, something); break; // Case 1
            case 'bad':
                cb(new Error('bad')); break; // Case 2
            default:  // ugly
                throw new Error('Oops'); // Case 3
            }
        }, 1000);
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

1. Commit `this.state`, send notif, return `good`
2. Roll back `this.state`, ignore notif, return app error
3. Ditto, but return system error that closes client session

---


### Managing internal state

```
exports.methods = {
    __ca_init__: function(cb) {
// Initialize this.state at creation time, just once
    },
    __ca_resume__: function(cp, cb) {
// Reload this.state from checkpoint cp, many times
    },
   foo: function(something, cb) {
...
```
<!-- .element: class="hljs javascript" spellCheck="false" -->
* Optional ` __ca_init__` and `__ca_resume__` to initialize or recover `this.state`
* `this.scratch` for state not checkpointed or rolled-back

---
<!-- .element:  style="text-align:left" -->
### Transactional plugins
<!-- .element:  style="text-align:center" -->

For each message, the plugins in `this.$` join the application in a local 2PC protocol:
* Delay external actions with a log
* If anybody aborts
  * Ignore non-committed actions, reset state
* Otherwise
  * Checkpoint commitments with a remote service
  * Execute pending actions
* Recovery after a failure retries actions in checkpoint
    * Assumed idempotent


---
### Autonomous behavior

```
exports.methods = {
    __ca_pulse__: function(cb) {
        this.$.session.notify("I'm up");
        cb(null);
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

`__ca_pulse__` called periodically by the framework

* Get things done regardless of external interactions

---
### Versioning

```
exports.methods = {
   __ca_upgrade__: function(newVer, cb) {
        var oldVer = this.state.__ca_version__;
        if (semver.valid(oldVer) && semver.valid(newVer) &&
            semver.satisfies(newVer, '^' + oldVer)) {
            this.$.log.debug('Minor version:transparent update');
        } else {
            this.$.log.debug('Major version change:' + newVer);
            // do some magic to this.state
        }
        this.state.__ca_version__ = newVer;
        cb(null);
    }
 }
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

Compare checkpointed state *semver* version vs code
* *Minor change*: transparent upgrade
* *Major change*: custom upgrade code called before processing a new message


---
## Sharing Actors

![](process.env.CA_NAME/assets/pig_sharing.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .element:  style="text-align:left" -->
### Actors Limitations
<!-- .element:  style="text-align:center" -->

* Inefficient to share data that is
  * read by many
  * and rarely modified by a few
* Pick your poison
  * One writer actor serves readers
  * or replicate across actors, and multicast updates
* Faster to use Distributed Data Structures (DDS)
  * Local shared-memory
* Combine DDS + Actors
  * **Without data races, deadlocks, or ugly failure mode!**

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
* **Writer Atomicity** use a CAF.js transactional plugin
* **Monotonic Read Consistency** track version numbers



---
<!-- .element:  style="text-align:left" -->
### SharedMap Example
<!-- .element:  style="text-align:center; text-transform:none" -->

```
exports.methods = {
    __ca_init__: function(cb) {
        isAdm(this) && this.$.sharing.addWritableMap('ms', ADM);
        this.$.sharing.addReadOnlyMap('slave', masterMap(this));
        cb(null);
    },
    increment: function(cb) { // Only called by `admin`
        var $$ = this.$.sharing.$;
        var counter = $$.ms.get('counter') || 0;
        $$.ms.set('counter', counter + 1);
        cb(null, counter);
    },
    getCounter: function(cb) {
        cb(null, this.$.sharing.$.slave.get('counter'));
    }
};
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* Users have an `admin` CA with a counter in a *SharedMap*
* CAs use `masterMap()` and `isAdm()` to find the map

---
<!-- .element:  style="text-align:left" -->
### Sharing Code with a SharedMap
<!-- .element:  style="text-align:center; text-transform:none" -->
```
var body = "return prefix + (this.get('base') + Math.random());";
$$.master.setFun('computeLabel', ['prefix'], body);
...
$$.slave.applyMethod('computeLabel', ['myPrefix']));
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* `setFun` stores a serialized **pure** function
* `applyMethod` invokes the function
  * with `this` bound to the *SharedMap*
* Glitch free behavior updates
* Example uses
  * Getters/setters to hide schema versioning
  * Event filtering
  * Policy enforcement

---
## Authorization

![](process.env.CA_NAME/assets/pig_linked.svg)
<!-- .element:  width="500" heigh="500" -->

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
<!-- .slide: class="two-floating-elements"  style="float: left" style="text-align:left"  -->
### Security Mechanisms
<!-- .element:  style="text-align:center" -->
![](process.env.CA_NAME/assets/Mechanisms.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

* Authenticated client sessions with Single Sign-On
  * signed JSON Web Tokens
    * no CA-in-the-middle
    * token semilattice
  * SRP+accounts service
* Application Trusted Bus
  * safe interactions between CAs
    * source anti-spoofing
    * single-writer guarantees
* Policy Engine (**focus of this talk**)
  * linked local namespaces

---
<!-- .element:  style="text-align:left" -->
### It's all in the name
<!-- .element:  style="text-align:center" -->

* Core ideas from SDSI (Rivest&Lampson'96)
* A principal is globally identified by his/her public key, or
  * username + public key of the `accounts` service.
* A principal has a **local namespace** with names bound to
  * **owned resources**: images, apps, CAs, *SharedMaps*,...
  * other **principals**: usernames or pub keys
  * **links** to other names, possibly in other namespaces
  * **groups** of the above
* Linking is sound
  * *Principal + local name* is globally unique
* Query by transitive closure implements ACLs
  * start with matching local names, keep following links

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
    * resources, e.g., *all my apps*, or
    * principals, e.g., *all my friends*
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
  * revocation is fast (milliseconds)
  * discovery is efficient, and managed by the framework
  * properties of *Sharing Actors* lead to **safe policy changes**

---

<!-- .element:  style="text-align:left" -->
### AggregateMap Example
<!-- .element:  style="text-align:center; text-transform:none" -->

```
exports.methods = {
    __ca_init__: function(cb) {
        var shar = this.$.sharing; var sec = this.$.security;
        isAdm(this) && shar.addWritableMap('acl', 'autho');
        shar.addReadOnlyMap('aclAgg', authoMap(this),
                            {isAggregate: true, linkKey: LINK});
        var rule = sec.newAggregateRule('getCounter','aclAgg');
        sec.addRule(rule);
        cb(null);
    },
    changeACL: function(principal, delegate, cb) { //only admin
        var $$ =  this.$.sharing.$;
        !delegate && $$.acl.set(principal, true);
        delegate && $$.acl.set(LINK, addLink(authoMap(principal),
                                             $$.acl.get(LINK)));
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* *Joe's* CA `admin` manages permissions for his CAs
* *Susan* can access `getCounter` in **any**  of *Joe*'s CAs if
  * *Joe's* CA `admin` writes to *SharedMap* `acl`
  * or *Joe* delegates by linking to her `admin` *SharedMap*

---

## Persistent Sessions

![](process.env.CA_NAME/assets/pig_sessions.svg)
<!-- .element:  width="500" heigh="500" -->
---

<!-- .element:  style="text-align:left" -->
### Cookies...
<!-- .element:  style="text-align:center" -->

What's wrong with cookies for session management?

* Chosen by the server, not by the client
* Cannot have sensible, human-friendly, values
* Do not move between devices
* Browser-based, non-portable to gadgets

---
<!-- .slide: class="two-floating-elements-70"  style="float: left" style="text-align:left"  -->
### A better cookie
<!-- .element:  style="text-align:center" -->

![](process.env.CA_NAME/assets/Queue.svg)
<!-- .element: class="plain" style="float: right" width="30%" -->

* Add notification queues to a CA
  * handle off-line clients
* Queue name identify **session**
  * simple name, relative to its CA
  * chosen by the client
* Queue contents visible to the CA
  * apps manage undelivered notifications
* Best effort delivery
  * concurrent dequeue, non-transactional
  * not checkpointed
* Support in our **universal** client library
  * browser, script, Cloud, and gadget

---
<!-- .element:  style="text-align:left" -->
### Notifications Example
<!-- .element:  style="text-align:center" -->
```
// CA
exports.methods = {
    notify: function(to, msg, cb) {
        var from = this.$.session.getSessionId()
        this.$.session.notify([from, msg], to);
        cb(null);
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->
```
// Tablet notifying watch, the watch code is symmetric
var URL = 'http://root-hello.vcap.me/#from=foo-ca1&ca=foo-ca1';
var s = new caf_cli.Session(URL);
s.changeSessionId('tablet');
s.onopen = function() {
    s.notify('watch', 'Hello', function(err) {...});
}
s.onmessage = function(msg) {
    var notif = caf_cli.getMethodArgs(msg);
    console.log('from:'+ notif[0] + ' msg:' + notif[1]);
};
...
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

---
### Reliable stateless clients
![](process.env.CA_NAME/assets/Toaster.svg)
<!-- .element:  width="500" heigh="500" -->

* Stateless client crashes, or unplanned change of device
  * Guarantee that certain actions are only done once
  * You didn't want **two** toasters, did you?

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
## Components

![](process.env.CA_NAME/assets/pig_components.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .element:  style="text-align:left" -->
### CAF.js COMPONENTS
<!-- .element:  style="text-align:center; text-transform:none" -->
* Core ideas from the SmartFrog framework and Erlang/OTP
* Describe a **hierarchy** of components with JSON
  * templates, linking, and environment properties
* Asynchronous constructors, **deterministic** creation order
  * access services without blocking main loop
  * dependency injection
* Supervision tree, where a parent component
    * monitors children
    * triggers local recovery if needed
    * bubbles up failure if local recovery fails

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
        "msg": "Hello World!"
    }
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* `module` implementation of this component
* `name` key to register the new component in local context
* `env` set of properties to configure the component

---
<!-- .element:  style="text-align:left" -->
### hello.js
<!-- .element:  style="text-align:center; text-transform:none" -->
```
exports.newInstance = function($, spec, cb) {
    cb(null, {
        hello: function() {
            console.log(spec.name + ':' + spec.env.msg);
        },
        __ca_checkup__: function(data, cb0) {
            cb0(null);
        },
        __ca_shutdown__: function(data, cb0) {
            cb0(null);
        }
    });
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* `newInstance` is, by convention, a factory method
    * `$` is the local context provided by its parent
    * `spec` a parsed description from `hello.json`
* `__ca_checkup__` health check method
* `__ca_shutdown__` disable the component forever

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

* `load`
  * reads and parses description
  * creates hierarchy
  * starts monitoring
* `[module]` specify path(s) to load resources
* `$.foo.hello` top level context with new component

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

---

<!-- .element:  style="text-align:left" -->
### Transforming Descriptions
<!-- .element:  style="text-align:center" -->

![](process.env.CA_NAME/assets/Transforms.svg)
<!-- .element:  width="900" heigh="350" -->

* Typical Cloud deployment workflow
  * Customize base template
  * Create instances with different arguments
  * Propagate arguments to inner components
  * Fill the rest from the environment

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
### Wrapping up

![](process.env.CA_NAME/assets/pig.svg)
<!-- .element:  width="500" heigh="500" -->

---

# Thank You

http://www.cafjs.com

https://github.com/cafjs/caf

Original Pig by Anniken & Andreas, NO

Creative Commons 3.0
