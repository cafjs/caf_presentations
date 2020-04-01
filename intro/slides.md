## Caf.js
<!-- .element:  style="text-transform:none" -->


![](process.env.CA_NAME/assets/lever.gif)
<!-- .element:  class="plain" width="750" heigh="500" -->

by Antonio Lain

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Avatars for devices and web apps

![](process.env.CA_NAME/assets/ca1.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

* A front end programmer codesigns
  * UI
  * a Cloud Assistant (CA)
  * and  device logic
  * all in JavaScript
* Cost
  * 10 cents/year per CA
* Scale
  * 1 Billion with 30K large VMs

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### WHY CAs?
<!-- .element:  style="text-transform:none" -->

![](process.env.CA_NAME/assets/Super_hero.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

* Get **superpowers**!
  * Permanent
  * Autonomous
  * Stateful
  * Reliable
  * Trusted bus

Let's see some examples

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Cloud-based multitasking
![](process.env.CA_NAME/assets/Multi.svg)
<!-- .element: class="plain" style="float: right" width="35%" -->


* Remote background processing
  * User believes it is all local
* Smooth app context switch with server-side rendering
    * proactive
    * made efficient with a CA


---

# Demo

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Dynamic device behavior
![](process.env.CA_NAME/assets/Dynamic.svg)
<!-- .element: class="plain" style="float: right" width="32%" -->

* Hot-swap code in
  * millions of devices
  * every few seconds
  * glitch free
  * and with code generated on-the-fly by CAs
* Device-as-a-Browser


---

# Demo

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Impersonation
![](process.env.CA_NAME/assets/Impersonation.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

* **Make believe** the device
  * is always ON
  * a few milliseconds away
  * ready to negotiate
  * inform
  * or react to an event
* Aggressive power saving
  * Standalone wearables
* Low-latency interactions
  * VR-to-Physical


---

# Demo

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Secure pairing
![](process.env.CA_NAME/assets/Pairing.svg)
<!-- .element: class="plain" style="float: right" width="40%" -->

* Ephemeral DH public keys
  * exchanged with trusted bus
  * no certificates
  * no long-term secrets per app
  * PFS and end-to-end security
* Social graph of devices
  * Delegation, groups, ownership
  * Pair your friend's devices
* Out-of-band Bluetooth pairing

---

# Demo

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Timely coordination at scale
![](process.env.CA_NAME/assets/Timely.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

* Soft real-time pub/sub
  * Trigger actions with UTC time
  * Delay a few seconds to
    * scale to millions of actions
    * <100 msec of each other
    * and across the globe
* Bundle commands for safety
  * Cache whole bundles in device
  * Piggyback recovery actions
  * Ignore them if next bundle OK
* CA pipelines bundles

---

# Demo

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Stateful lambda (serverless)

![](process.env.CA_NAME/assets/Lambda.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->

* Debouncing Lambda with reliable state machines
* Cloud file system of devices
  1. Sync with local directory (s3fs)
  2. Trigger Lambda with S3 event
  3. Debounce with a CA
  4. Control device
* CA-as-a-Device-Driver
  * Multi-vendor support
* Manage many devices with Git
  * Commit config changes

---

# Demo

---

### How many lines of code?


| App            |  CA  | Device |  UI  |
| -------------- | ---- | ------ | -----|
| Counter        |  48  |  _ | 249 |
| Dynamic        |  126  |  57 | 439 |
| Impersonation  |  272  |  39 | 567 |
| Pairing        |  326  |  640 | 739 |
| Timely         |  235  |  35 | 1107 |
| Lambda         |  243  |  26 | 506 |

All JavaScript...

---

# Thank You

http://www.cafjs.com

https://github.com/cafjs/caf
