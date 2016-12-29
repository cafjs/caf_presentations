## CAF.js Programming


![](foo-programming/assets/pig.svg)
<!-- .element:  width="500" heigh="500" -->

by Antonio Lain

---

## Actors

![](foo-programming/assets/pig_actors.svg)
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

### And what about failures?
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

1. Commit `this.state`, send notification, return `good`
2. Roll back `this.state`, ignore notification, return application error
3. Ditto, but return system error that closes client session

and **then** process the next request

---
### Client Code
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
    err && console.log(myUtils.errToPrettyStr(err));//System error
    !err && console.log('Done OK');
}
```
<!-- .element: class="hljs javascript" spellCheck="false" -->

WebSocket on steroids

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
            this.$.log.debug('Minor version: transparent update');
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

![](foo-programming/assets/pig_sharing.svg)
<!-- .element:  width="500" heigh="500" -->

---
### Limitations of pure actors

Inefficient to share data that is
<!-- .element:  style="text-align:left" -->

* read by many, and rarely modified by a few

Pick your poison:
<!-- .element:  style="text-align:left" -->

* Single-writer actor serves all read requests
* Data replicated across actors, multicast updates
* Some combination of the above

Better to use distributed data structures that benefit from local shared-memory
<!-- .element:  style="text-align:left" -->

Combine DDS + Actors?

Without data races, deadlocks, ugly failure mode?

---
### DDS + Actors = Actors

![](foo-programming/assets/sharing1.svg)
<!-- .element:  width="500" heigh="500" -->

The Jocker can

* change the internal state of **any** actor,
* at **any** time,
* to, for example, implement a DDS.

---
### DDS + Actors = Actors

![](foo-programming/assets/sharing2.svg)
<!-- .element:  width="500" heigh="500" -->

Can actors emulate the Jocker's actions?

* Replace Jocker's changes by extra messages
* But end result should be observationally equivalent
    * Ignoring performance, an external entity cannot tell

---
### DDS + Actors = Actors

**Not** in the general case.

But **yes** in the following important case (Mohsen&Lain13):

1. *Single Writer*: one actor *owns* the data structure

2. *Readers Isolation*: replicas change in-between messages

3. *Fairness*: an actor cannot indefinitely block other local actors from seeing new updates

4. *Writer Atomicity*: changes are flushed, as an atomic unit, when the processing of a message finishes.

5. *Consistency*: monotonic read consistency, i.e., replicas can be stale, but they never roll back to older versions.



---



## Authorization

![](foo-programming/assets/pig_linked.svg)
<!-- .element:  width="500" heigh="500" -->

---

## Persistent Sessions

![](foo-programming/assets/pig_sessions.svg)
<!-- .element:  width="500" heigh="500" -->

---
## Components

![](foo-programming/assets/pig_components.svg)
<!-- .element:  width="500" heigh="500" -->

---

# Thank You

http://www.cafjs.com

https://github.com/cafjs/caf

Original Pig by Anniken & Andreas, NO

Creative Commons 3.0
