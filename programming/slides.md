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

### A CAF.js actor has:

* a queue that serializes message processing,
* a location-independent name,
* some private state,
* the ability to change behavior,
* or interact with other actors.

Initial design based on Erlang/OTP *gen_server*.

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

* for several years,
* with a billion CAs,
* programmed by back-end newbies.

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

and **then** process the next request

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
* Token based authentication
* Support for multi-method transactions

---


### Implementing a transactional actor (2)

**Behind the scenes**

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

* Checkpoint `this.state` before externalization
* Customizable hooks to initialize or recover `this.state`
* `this.scratch` not checkpointed or rolled back

---
### Implementing a transactional actor (3)

**Transactional plugins in `this.$`**

```
        this.$.session.notify("Got 'good'");
```
<!-- .element: class="hljs javascript" spellCheck="false" -->


Participate in a local 2PC protocol with other plugins:

* Scoped by the processing of a request
* Log actions to delay external interactions
* Ignore non-committed actions after any plugin aborts
* Checkpoint commitments with a remote service
* Retry committed (idempotent) actions after a failure

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

Compare checkpointed state schema vs code schema:

* Using *semver* versioning
* Major version changes require custom upgrade code
  * Called after resume but before message processing


---
## Sharing Actors

![](process.env.CA_NAME/assets/pig_sharing.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .element:  style="text-align:left" -->
### Limitations of pure actors
<!-- .element:  style="text-align:center" -->

* Inefficient to share data that is
  * read by many, and rarely modified by a few
* Pick your poison:
  * Single-writer actor serves all read requests
  * Data replicated across actors, multicast updates
  * Some combination of the above
* Faster to use distributed data structures that benefit from local shared-memory
* Combine DDS + Actors
  * **Without data races, deadlocks, ugly failure mode!**

---
### DDS + Actors = Actors

![](process.env.CA_NAME/assets/sharing1.svg)
<!-- .element:  width="500" heigh="500" -->

The Jocker can

* change the internal state of **any** actor,
* at **any** time,
* to, for example, implement a DDS.

---
### DDS + Actors = Actors

![](process.env.CA_NAME/assets/sharing2.svg)
<!-- .element:  width="500" heigh="500" -->

Can actors emulate the Jocker's actions?

* Replace Jocker's changes by extra messages
* But end result should be observationally equivalent
    * Ignoring performance, an external entity cannot tell

---
### DDS + Actors = Actors

**Not** in the general case

But **yes** in the following **important** case (Lesani&Lain13):

1. *Single Writer*: one actor *owns* the data structure
2. *Readers Isolation*: replicas change in-between messages
3. *Fairness*: an actor cannot indefinitely block other local actors from seeing new updates
4. *Writer Atomicity*: changes are flushed, as an atomic unit, when the processing of a message finishes.
5. *Consistency*: monotonic read consistency, i.e., replicas can be stale, but never roll back to older versions.

---
<!-- .element:  style="text-align:left" -->
### SharedMap
<!-- .element:  style="text-align:center; text-transform:none" -->

*Readers Isolation* + *Fairness* requires multiple local versions

* Use a persistent data structure (*Immutable.js*) for efficient versioning
* For each message, pick the most recent version that is locally available
* For a CA, keep version until the next message

*Writer Atomicity*: Use a CAF.js transactional plugin

*Consistency*: Keep track of version numbers

*Single Writer*: a *SharedMap* name is relative to its CA

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

* Each user has an `admin` CA that keeps a counter in a *SharedMap*
* CAs can use `masterMap()` and `isAdm()` to find it

---
<!-- .element:  style="text-align:left" -->
### Share code, not only data
<!-- .element:  style="text-align:center" -->
Pure functions serialized and stored in a *SharedMap*:

    setFun(methodName, argsNameArray, bodyString)
Invoked as a method of the *SharedMap*:

    applyMethod(methodName, argsArray)
Example:
```
var body = "return prefix + (this.get('base') + Math.random());";
$$.master.setFun('computeLabel', ['prefix'], body);
...
$$.slave.applyMethod('computeLabel', ['myPrefix']));
```
<!-- .element: class="hljs javascript" spellCheck="false" -->
Uses:
* Getters/setters hide schema versioning, event filtering, policy enforcement,...

---
## Authorization

![](process.env.CA_NAME/assets/pig_linked.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .element:  style="text-align:left" -->
### Security Assumptions
<!-- .element:  style="text-align:center" -->
* Goal
  * Write apps with safe **collaborative** multi-tenancy
* Hosting framework controlled by the application writer
    * Design framework and application together
    * A node.js framework process runs only one app
    * The service provider isolates apps using IaaS/PaaS
* Apps never trust each other
  * Cross-app interactions similar to the external world
* Apps are not expected to run untrusted code but,
  * for extra protection, frameworks do not fully trust apps
  * Think about it as a parachute for programming mistakes

---
<!-- .element:  style="text-align:left" -->
### Security Mechanisms
<!-- .element:  style="text-align:center" -->

* Authenticated Client Sessions
  * Single sign-on without CA-in-the-middle
  * Signed JSON Web Tokens, SRP, *accounts* service
  * Semilattice to attenuate tokens
* Application Trusted Bus
  * For all interactions between CAs
  * Source anti-spoofing of requests with security proxies
  * Single-writer guarantees
* Policy Engine
  * Authorization based on linked local namespaces
  * Focus of this talk

---
<!-- .element:  style="text-align:left" -->
### It's all in the name
<!-- .element:  style="text-align:center" -->

* Core ideas from SDSI (Rivest&Lampson'96)
* A principal is globally identified by his/her public key, or
  * username + public key of the `accounts` service.
* A principal has a local namespace with names bound to:
  * Owned resources: images, apps, CAs, *SharedMaps*,...
  * Other principals: usernames or pub keys
  * Links to other names, possibly in other namespaces
  * Groups of the above
* Linking is sound
  * *Principal + local name* is globally unique
* Query by transitive closure to implement ACLs
  * Start with matching local names, keep following links

---
<!-- .element:  style="text-align:left" -->
### Advantages
<!-- .element:  style="text-align:center" -->

* Decentralized authorization
  * Easy to federate `accounts` services
  * or sign your own tokens
* Monotonicity
  * Partial information leads to conservative decisions
* Expressivity
  * Concise ownership-based policies
  * Built-in support for groups of
    * resources, e.g., *all my apps*, or
    * principals, e.g., *all my friends*
* Delegation
  * Create groups by linking to other groups

---
<!-- .element:  style="text-align:left" -->
### Novel implementation
<!-- .element:  style="text-align:center" -->

* Represent namespaces with *SharedMaps* not certificates
  * Only within an app (see previous assumptions)
  * Single-writer, i.e., by the owner, is a **strong** property
  * Wrapper *AggregateMap* transparently manages linking
* Users do not need to manage private keys
  * Federated `accounts` services
  * Authenticated client sessions
* Advantages
  * Revocation is fast (milliseconds)
  * Discovery is efficient, and managed by the framework
  * Properties of *Sharing Actors* lead to safe policy changes

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
        ...
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

* *Joe's* *admin* CA manages permissions for his CAs
* *Susan* gets access to method `getCounter` in any of *Joe*'s CAs, when the *admin* CA writes to *SharedMap* `acl`,
* or *Joe* can delegate by linking to her *admin* *SharedMap*.

---

## Persistent Sessions

![](process.env.CA_NAME/assets/pig_sessions.svg)
<!-- .element:  width="500" heigh="500" -->
---

<!-- .element:  style="text-align:left" -->
### Cookies...
<!-- .element:  style="text-align:center" -->

What's wrong with cookies for managing notifications?

* Chosen by the server, not by the client
* Cannot have sensible, human-friendly, values
* Do not move between devices
* Browser-based, non-portable to basic gadgets

---
<!-- .element:  style="text-align:left" -->
### A better cookie
<!-- .element:  style="text-align:center" -->

Add notification queues to an actor (CA)

* Simple queue names that identify sessions
  * Relative to its CA's name
  * Chosen by the client
* Best effort delivery
  * Concurrent dequeue, non-transactional
  * Not checkpointed
  * Supports off-line clients
* Contents visible to the CA
  * Application manages undelivered notifications
* Built-in support in our **universal** client lib

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
<!-- .element:  style="text-align:left" -->
#### Recoverable requests from stateless clients
<!-- .element:  style="text-align:center" -->

* Core ideas from Bernstein&Hsu&Mann'90
* Stateless client crashes, or unplanned change of device
  * Guarantee that certain actions are only done once
  * You didn't want two toasters, did you?
* Client application
  * Piggyback client memento to requests to identify last committed action
  * Explicitly start and end a persistent session
    * Client crashed if not properly ended
    * When session re-opens, return memento if crashed
* Leverage CA's transactions and message serialization
  * Simulate a reliable queue per session for mementos

---
## Components

![](process.env.CA_NAME/assets/pig_components.svg)
<!-- .element:  width="500" heigh="500" -->

---
<!-- .element:  style="text-align:left" -->
### Components Everywhere
<!-- .element:  style="text-align:center" -->
* Core ideas from
  * The SmartFrog Framework and Erlang/OTP
* Describe a hierarchy of components with JSON
  * Description templates, linking, and env properties
* Asynchronous constructors, deterministic creation order
  * Access *Redis* without blocking main loop
  * Dependency injection
  * CommonJS modules
* Supervision tree
  * Parent component monitors children,
  * triggers local recovery,
  * failure bubbles up if recovery fails.

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

* `module`: implementation of this component
* `name`: key to register the new component in local context
* `env`: set of properties to configure the component

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

* `newInstance`: factory method
    * `$`: local context provided by its parent
    * `spec`: parsed description from `hello.json`
* `__ca_checkup__`: health check method
* `__ca_shutdown__`: disable the component forever

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

* `load`:
  * reads and parses description
  * creates hierarchy
  * starts monitoring
* `[module]`: specify path(s) to load resources
* `$.foo.hello`: top level context with new component

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

Merge base configuration with `load` argument:

* `{name: 'bar', env: {msg: 'Bye!'}}`

---

<!-- .element:  style="text-align:left" -->
### Hierarchy: helloTree.json
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

* Construction order: children in array-order first, then parent
* Destruction reverses this order
* `caf_components#supervisor` similar to  `module.require("caf_components").supervisor`
---

<!-- .element:  style="text-align:left" -->
### Hierarchy: hello.js
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
  * Order components in description to satisfy dependencies
  * Already created siblings in `$`
  * Top level component is `$._` at any depth
    * `$._.$` is the top level context
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
### Transforms: Templates (helloTree++.json)
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
### Transforms: Linking (helloTree++.json)
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

* `$._.env`: links to properties in the top level component
* Hides internal details for `load` arguments
   * Merged top values propagate to inner components

---

<!-- .element:  style="text-align:left" -->
### Transforms: Env Props (helloTree++.json)
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

* `process.env` : properties from the environment
   * Default values with separator `||`
     * Assumed JSON, if parsing fails defaults to string

---
### Wrapping up

![](process.env.CA_NAME/assets/pig.svg)
<!-- .element:  width="300" heigh="300" -->


* Build three-way isomorphic apps with CAF.js
  * Web app with React/Redux + our client library
  * A CA in the Cloud
  * A device app using `caf_iot`
* Deploy in Cloud **and device** with Docker containers


---

# Thank You

http://www.cafjs.com

https://github.com/cafjs/caf

Original Pig by Anniken & Andreas, NO

Creative Commons 3.0
