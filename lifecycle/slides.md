## Caf.js APP LIFE CYCLE
<!-- .element:  style="text-transform:none" -->


![](process.env.CA_NAME/assets/gears.svg)
<!-- .element:  class="plain" width="500" heigh="450" -->

by Antonio Lain

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Three-way isomorphic apps

![](process.env.CA_NAME/assets/Iso3.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->
* CA in the Cloud executes
  * browser code
    * proactive server-side rendering
  * and device code
    * bundle introspection
    * event filtering
    * impersonation
* Package/test/deploy together
  * Consistent versioning
  * CA-as-a-Web-Server

---
<!-- .slide: class="two-floating-elements"  style="float: left" -->
### Containers everywhere

![](process.env.CA_NAME/assets/Containers.svg)
<!-- .element: class="plain" style="float: right" width="50%" -->


* Deploy in Cloud with containers
  * Mesos cluster
  * Redis backend
  * Many app instances
  * HAProxy-based http router
* Deploy in device with containers
  * Privileged daemon container
    * Builds Docker images
    * Negotiates tokens
    * Reacts to config changes
    * Manages app container
  * One app container at a time

---

## `cafjs` Command Line Tool
<!-- .element:  style="text-transform:none" -->
* **Quick prototyping mode**:
  * Build your app, run containers locally to simulate cloud
```
$ cd caf_helloiot; cafjs build; cafjs run helloiot
```
<!-- .element: class="hljs bash" spellCheck="false" -->
  * Emulate devices with `qemu-arm-static` and Docker
```
$ cafjs mkIoTImage helloiot; cafjs device foo-device1
```
<!-- .element: class="hljs bash" spellCheck="false" -->

* **Validation mode**: make container image, run app/device
```
$ cafjs mkImage . registry.cafjs.com:32000/root-helloiot
$ cafjs run --appImage registry.cafjs.com:32000/root-helloiot helloiot
$ cafjs device foo-device1
```
<!-- .element: class="hljs bash" spellCheck="false" -->


---
## Cloud Deployment

* `caf_turtles` app
  * Publish **open** apps in **your** local namespace
    * Anybody can publish
    * Everybody else can create CAs or associate devices
  * Create/flex/update/destroy
  * CA plugin wraps Mesos/Marathon APIs
  * Yes, it can deploy itself
* Private registry with authenticated local namespace


```
$ docker login registry.cafjs.com:32000 -u foo -p pleasechange
$ docker push  registry.cafjs.com:32000/foo-helloiot
```
<!-- .element: class="hljs bash" spellCheck="false" -->

---

# Demo

---
<!-- .element:  style="text-align:left" -->
## Raspberry Pi Setup
<!-- .element:  style="text-align:center; text-transform:none" -->

Start with an up to date raspbian image, install Docker with:

```
$ curl -sSL https://get.docker.com | sh
```
<!-- .element: class="hljs bash" spellCheck="false" -->

Register for a cloud account (e.g., user `bar`) in https://root-launcher.cafjs.com

Create a CA instance of the `root-gadget` app to manage the device `bar-device1`

Install master token and core images:

```
$ curl -sSL  https://raw.githubusercontent.com/cafjs/caf_rpi/master/setup.sh | bash -s -- bar-device1 your_password
$ history -c && history -w
```
<!-- .element: class="hljs bash" spellCheck="false" -->

---

# Thank You

http://www.cafjs.com

https://github.com/cafjs/caf
