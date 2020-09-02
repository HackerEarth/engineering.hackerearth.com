---
author: Vishal Gowda
layout: post
title: "Streaming Android applications via the browser"
description: "Comprehensive guide to setting up remote emulators and interacting with them from the browser"
category:
tags: [Android, VNC, RFB, Server Client Architecture, Distributed Systems, RPC, Python, X Window System, WebSockets, Virtualisation]
---
{% include JB/setup %}

HackerEarth prides itself in its scalable & automated evaluation system. What was initially designed keeping standard programming problems in mind (check this [post](http://engineering.hackerearth.com/2013/03/12/100000-strong/) out), gradually evolved to accommodate a plethora of problem types across various tech domains.

| **Currently supported Problem Types**   | 
| 				:---:   			  |
| _Programming_   					  |
| _Frontend_      					  |
| _Objective_   					  |
| _Android_   						  |
| _Subjective_   					  |
| _File based_  					  |
| _Multiplayer_ 	    			  |
| _Approximate_ 	    			  |
| _Golf_ 	    					  |
| _Machine Learning_	    		  |
| _SQL_ 	    					  |
| _Regex_ 	    					  |
| _File eval_	    				  |
| _Map Reduce_	    				  |


_**Note:** Not all of the problem types are accessible by end users publicly. Some are reserved for HackerEarth's [Recruit](https://www.hackerearth.com/recruit/) product._

While most of them have their own _evaluation stack_ and are automated in the complete sense of the word, evaluating submissions for some of these problem types requires partial manual intervention. Evaluation of Android submissions for instance, is not automated.

### Why though?

An android submission is basically an [apk](https://en.wikipedia.org/wiki/Android_application_package). Given a requirement, the user has the freedom of designing and implementing an app _in whatever way he/she deems fit_. Given the nature of such submissions, it would be ill conceived to design an evaluation system in a _one-size-fits-all_ manner. Hence the need for manual intervention.

### Evaluating Android Submissions

Each submission is rated based on the following parameters:

1. Requirement compliance
2. Bugs
3. Performance benchmarks
4. Look and feel of the application
5. User experience

For a given hiring challenge, a dashboard containing all the submissions along with the candidate details is provided to the recruiter. The recruiter then follows each of these steps:

1. Download a candidate's apk onto his/her local machine
2. Install the apk onto a connected android device or emulator
3. Test and interact with the app on a device or emulator
4. Update score for the candidate in the dashboard

Straight off, you can identify serious cons to this approach.

1. [Android Studio](https://developer.android.com/studio/index.html) should be set up on your local machine(for a non-technical guy, this can be a daunting task in itself)
2. Manually install & uninstall apps from an emulator/device
3. Error-prone bookkeeping while updating scores(more pronounced when evaluating 100s of submissions)
4. No means of running automated tests

This was _tedious_ and **not** _scalable_.

## The Fix

We set out to solve these problems and defined the following objectives:

1. A mechanism to provision and allot emulators on the fly
2. A means to interact with the emulator from the browser
3. Programmatically run certain defined operations on the emulator(like _install apk, start package, unlock screen,_ etc)
4. Integration in the recruiter dashboard for android submissions

_**Note**: The technique we apply for streaming applications is not tailor made for the emulator, it applies to **any** kind of GUI application._

Before I present to you a system-level overview, lets run through the core components. We will introduce each component and it's functionality via usecases.

---
### Running GUI apps on a Headless Server

Any GUI application requires a graphical system that provides a basic framework, or primitives, for building GUI environments. Basic primitives like:

- Drawing and moving windows on the display
- Interacting with a mouse, keyboard or touchscreen.

All [*nix based systems](https://en.wikipedia.org/wiki/Unix-like) provide the [X Window System](https://en.wikipedia.org/wiki/X_Window_System) for the same. Every GUI application that runs on a Linux machine interacts with what is called an **X Server**.

<img style="display: block; margin: 0 auto;" src="/images/X_client_server_example.png" alt="X client-server example" title="Depiction of the client server model in the X Window System"/>

As depicted above, the _X Server_ relies on input and output devices like the Monitor, Keyboard and Mouse.

_**Note**: [DISPLAY](http://gerardnico.com/wiki/linux/display) is an environment variable that instructs an X client(read GUI application) which X server it is to connect to by default._


#### Headless Server

A [Headless Server](https://en.wikipedia.org/wiki/Headless_computer), does not have a screen, keyboard or mouse attached. Most servers are configured to be headless so as to reduce operational cost. Since the server has no associated display, there is no _X Server_ running on such machines. 

Which is why, you get something like this when you try running a GUI app on a headless server.

```
$ glxgears
Error: couldn't open display (null)
```

_**Note**: glxgears is just a GUI app that we'll utilise for the purpose of this demo._

So how _does_ one go about running GUI apps on a headless server? Enter **Xvfb**.

#### Xvfb

Xvfb or _X virtual framebuffer_ is a display server that performs all graphical operations in memory without showing any screen output. This virtual server does not require the computer it is running on to have a screen or any input device. Only a network layer is necessary.
    
Here's how you'd run glxgears, with Xvfb set up.

```
$ Xvfb :1 -screen 0 1024x768x24 > /dev/null 2>&1 &
$ DISPLAY=:1 glxgears
11749 frames in 5.0 seconds = 2349.722 FPS
12433 frames in 5.0 seconds = 2486.563 FPS
12668 frames in 5.0 seconds = 2533.599 FPS
...(truncated)
```
    
This time, the app was able to successfuly run. But you won't see anything, because this machine is by definition _headless_(i.e without a display). In order to actually interact with it, we need to setup a _VNC server_.

> So how _does_ one go about running GUI apps on a headless server? Enter **Xvfb**.

---
### Virtual Network Computing Primer

#### RFB protocol

RFB (_remote framebuffer_) is an open simple protocol for remote access to graphical user interfaces. It enables one to transmit GUI _frames_ and input events across a server and a client, over the network.

####  VNC

VNC (_Virtual Network Computing_) is a graphical desktop sharing system that uses the Remote Frame Buffer protocol (RFB) to remotely control another computer.

A VNC implementation comprises of:

1. **Server**: The program on the machine that shares its screen. It passively allows the client to take control of it.
2. **Client(or viewer)**: The program that watches, controls, and interacts with the server. The client controls the server.
3. **Protocol**: The Server and Client talk using the RFB protocol.

Various implementations for VNC Servers and clients exist. Most VNC Servers create their own virtual X Display. But we already had that covered by using Xvfb. We simply needed to export an existing X Server, which is why we opted for the [x11vnc](https://en.wikipedia.org/wiki/X11vnc) server.

Here's how you'd start an x11vnc server pointing to a [DISPLAY](http://gerardnico.com/wiki/linux/display) at :1.

```
$ x11vnc -display :1 -quiet -nopw

The VNC desktop is:      deathstar:0
PORT=5900
```

_Note: Make note of the port 5900, which is the port that the server is listening on_

Connecting to this VNC session can be done using a _VNC viewer_(read client) or via [_SSH X11 forwarding_](http://en.tldp.org/HOWTO/XDMCP-HOWTO/ssh.html). 

Here we connect to a VNC server running on localhost at port 5900 using a _vncviewer_.

```
$ vncviewer localhost:0
```

which then allows us to interact with the DISPLAY :1, which is what the VNC Server was connected to.

---
### Connecting to a VNC Server from the browser

In order to interact with a VNC Server, one would need to install a VNC viewer. This is a no go, if we wish to eliminate dependencies at the recruiter's end.
Ideally, we want to establish a VNC session from within the browser. Basically _we needed a VNC client built for the browser_.

#### NoVNC

NoVNC is a VNC client using HTML5 (Web Sockets, Canvas). The project is available on [GitHub](https://github.com/novnc/noVNC). Particularly check out the [Integration](https://github.com/novnc/noVNC/wiki/Integration) section.

Connecting to a VNC server from javascript is as simple as:

{% highlight javascript %}
// Initialise rfb object(refer to the modules documentation in the project)
var rfb = new RFB({'target': document.getElementById('noVNC_canvas'});
// Initialise host and port of the VNC server
var host = 'localhost';
var port = 5900;
// Establish session. On success, this starts drawing on #noVNC_canvas.
rfb.connect(host, port);
{% endhighlight %}

With the VNC Server process running, if you try establishing a connection from the noVNC client, it will fail with the following error.

`Skipping unsupported WebSocket binary sub-protocol`

The reason this happens is because the noVNC client communicates using WebSockets, which is different from the raw TCP protocol that the VNC server uses. 
To bride this gap, the folks at noVNC built a websocket proxy called [websockify](https://github.com/novnc/websockify), that translates low-level TCP traffic into WebSocket traffic and vice-versa.

> Basically _we needed a VNC client built for the browser_.

The following command listens for WebSocket traffic at port 6080, translates and forwards the same onto port 5900(which is where the VNC server is listening). Communication is bi-directional.

```
$ websockify :6080 :5900
WebSocket server settings:
  - Listen on :6080
  - Flash security policy server
  - No SSL/TLS support (no cert file)
  - proxying from :6080 to :5900
```

Consequently, we update our noVNC client to instead connect to the websocket port.

{% highlight javascript %}
// port 5900 is no longer relevant to the client
var port = 6080; 
{% endhighlight %}

The noVNC client should now be able to establish a connection.

---
### Setting up the Android Emulator

The Android Emulator supports several hardware acceleration features to improve performance, sometimes drastically. An Android Developer usually concerns himself/herself with these details only once. One can find a comprehensive guide in the [Android Docs](https://developer.android.com/studio/run/emulator-acceleration.html).

#### Configuring the emulator VM for acceleration

The emulator runs inside a Virtual Machine. In order to configure the VM for acceleration, the underlying hardware needs to expose what is called a [hypervisor](https://en.wikipedia.org/wiki/Hypervisor). On a Linux machine this comes in the form of Kernel-based Virtual Machine(KVM).
    
In order to get access to the hypervisor, we needed to spin a [bare metal machine](https://en.wikipedia.org/wiki/Bare_machine)(as opposed to running the emulator on a machine provisioned by a cloud provider, which are VMs themselves).
    
You can refer to the [docs](https://developer.android.com/studio/run/emulator-acceleration.html#accel-vm) for how to go about exposing the hypervisor.

#### Custom AVDs

An Android Virtual Device (AVD) definition lets you define the characteristics of an Android phone, tablet, Android Wear, or Android TV device that you want to simulate in the Android Emulator.
    
We pretty much stuck to the procedure defined in the [docs](https://developer.android.com/studio/run/managing-avds.html) and defined a bunch of AVDs, one for each emulator that we were planning to run. Few things worth a mention here are:
    
- An AVD can be utilized by any one emulator at most
- Virtual Memory and Heap Size can be defined in the AVD
- Internal Storage and SD card size can be defined as well
- Set the GPU option to auto to dynamically choose between a hardware/software renderer

#### Starting the emulator

In order to start an emulator, you need to first define at least one AVD. While there are multiple parameters that can be specified, we will concern ourselves with only 2 of them.

1. [gpu](https://developer.android.com/studio/run/emulator-acceleration.html#command-gpu) - We use a software renderer called _swiftshader_. We use this because the host machine does not have a dedicated GPU.

2. qemu - We configure the emulator to utilise _kvm_ which results in a great performance boost.

{% highlight bash %}
# Point to the running Xvfb process
DISPLAY=:1
# Start emulator
emulator -avd <name-of-predefined-avd> -no-boot-anim -nojni -netfast -gpu swiftshader -qemu -enable-kvm
{% endhighlight %}

_**Note**: As we saw in the Running GUI apps on Headless Server section, we need to point the **DISPLAY** to the running Xvfb process first._

---
### Sizing up the machine

Let's break down the requirements for running a single emulator:

- Memory : **2 GiB** of RAM (defined by the AVD)
- Compute : **1 CPU** dedicated for a VM
- Storage : **1 GiB** of disk space(Internal Storage + SD Card; defined by AVD)
- Rendering : We utilised an OpenGL Software Renderer - [Swiftshader](https://blog.chromium.org/2016/06/universal-rendering-with-swiftshader.html), which implied CPU overhead.
- Hypervisor : Underlying platform needs to be expose the hypervisor for performant emulators(this constraint applies to all emulators)

_**Note**: In software renderers, graphical computations are performed on the CPU itself. One would opt for a software renderer if there is no GPU available, since it introduces quite a bit of computational overhead._

Factoring in these requirements. We came up with the following machine specifications to parallely run **6 emulators**:

- Memory : Min. **16 GiB** RAM
- Compute : **8 CPU**s (Factoring in GPU Rendering overhead)
- Storage : Min. **256 GiB** of disk space(factoring in dependencies, kernel, sdk, etc)
- [Bare Metal](https://en.wikipedia.org/wiki/Bare_machine)ness == True (for the hypervisor)

We purchased a dedicated server meeting these specifications from this vendor called [Hetzner](https://www.hetzner.de/en/). Their pricing is surprisingly reasonable and their servers have consistently performed well.

---
### Interacting with the Emulator

While there was now a way to facilitate live interaction with an emulator, we needed certain primitives to be exposed on the emulator so that we may run them programmatically.

For this, we leveraged the [ADB](https://developer.android.com/studio/command-line/adb.html) (Android Debug Bridge), which let us send commands, among other things, into a running emulator by exposing a port(ADB PORT) on the host machine.

_**Note**: When running more than one emulator on a machine, one needs to explicitly specify the ADB PORT for the consecutive emulators to avoid port clashes._

We were now able to perform operations like:
    
- Installing/uninstalling apks

`$ adb install /path/to/apk`

- Get/Set various states in the emulator

`$ adb shell getprop sys.boot_completed`

- Starting packages

`$ adb shell am start -n com.package.name/com.package.name.ActivityName`

- Running instrumented tests

`$ adb shell am instrument -w <test_package_name>/<runner_class>`

- Sending intents to activities, clearing the activity stack, etc.
    
---
### Putting it all together

<img style="display: block; margin: 0 auto;" src="/images/app-stream-components.png" alt="Component Overview" title="App Stream Component Overview"/>

And here's a simple bash script that incorporates all of the components, to finally expose a websocket that a noVNC client can connect to.

{% highlight bash %}
# Start Xvfb at DISPLAY :1
Xvfb :1 -screen 0 1024x768x24 > xvfb.log 2>&1 &
# Point DISPLAY to virtual X Server
export DISPLAY=:1
# Start emulator for a pre-defined avd
emulator -avd Nexus_5X_API_24 -gpu swiftshader -no-boot-anim -nojni -netfast -qemu -enable-kvm > emulator.log 2>&1 &
# Start a VNC server and point it to the same display
x11vnc -display :1 -quiet -nopw -rfbport 5900 -bg -o vnc.log
# Proxy websocket traffic to raw tcp traffic
websockify -D :6080 :5900 > websockify.log 2>&1
{% endhighlight %}

_**Note**:This is a grossly oversimplified version. Because there are so many moving parts, we need to ensure each of these processes have been initialised and are in running state before starting the next one._

The entire setup can be considered as one unit, which we will refer to as an _endpoint_ hereon. We built a [Thrift](https://thrift.apache.org/) service that provisions such _endpoints_, along with exposing certain other primitives like - Installing/uninstalling of apks, Flushing of the _Activity Stack_, etc.

<img style="display: block; margin: 0 auto;" src="/images/app-stream-worker-service.png" alt="App Stream Thrift Service" title="App Stream Thrift Service Overview"/>

And here's the thrift definition for _Droid Service_.

{% highlight python %}
enum ErrCode {
  DEFAULT = 0,
  NO_DROIDS_AVAILABLE = 1,
}

struct ConnParams {
  1: string host,
  2: string port,
  3: optional string password,
}

exception ApplicationException {
  1: string msg,
  2: optional i32 code = ErrCode.DEFAULT,
}

service DroidService {
   void ping(),

   string get_package_name(1: string apk_url) throws (
          1: ApplicationException ae),

   ConnParams get_endpoint(1: string endpoint_id),

   bool run_operation(1: string endpoint_id, 2: string operation, 3: string apk_url) throws (
          1: ApplicationException ae),
}
{% endhighlight %}

#### Scaling out

In _Sizing up the machine_ we saw that we could run at most **6 emulators** on a machine with those specs. We had to be able to accommodate more _endpoints_ by [horizontally scaling](http://searchcio.techtarget.com/definition/horizontal-scalability) up.

We employed the [Apache Zookeeper](https://zookeeper.apache.org/) project to this effect. One of it's usecases lies in centrallized configuration management, which provides recipes for designing [distributed systems](https://en.wikipedia.org/wiki/Distributed_computing).

##### Concepts

1. Each _Droid Service_ is configured to attach itself to ZooKeeper on startup
2. Scaling up is simply a matter of starting another _Droid Service_
3. A _Master Droid Service_ simply directs RPC calls to the relevant _Droid Service_ and relays the response back to external clients(for eg. a Django app).

<img style="display: block; margin: 0 auto;" src="/images/app-stream-droid-master.png" alt="Droid Master Overview" title="App Stream Cluster"/>

And here's the thrift definition for the _Master Droid Service_

{% highlight python %}
enum ErrCode {
  DEFAULT = 0,
  NO_DROIDS_AVAILABLE = 1,
}

struct DroidRequest {
  1: string user,
  2: optional string apk_url,
  3: optional string op,
}

struct ConnParams {
  1: string host,
  2: string port,
  3: optional string password,
}

exception ApplicationException {
  1: string msg,
  2: optional i32 code = ErrCode.DEFAULT,
}

service DroidKeeper {
   void ping(),

   string get_package_name(1: string apk_url) throws (
          1: ApplicationException ae),

   ConnParams get_endpoint_for_user(1: string user) throws (
          1: ApplicationException ae),

   bool interact_with_endpoint(1: DroidRequest dr) throws (
          1: ApplicationException ae),

   oneway void release_endpoint_for_user(1: string user),
}
{% endhighlight %}

### What's next?

- Leveraging [VirtualGL](http://www.virtualgl.org/) and exploring the [TurboVNC](http://www.turbovnc.org/) project to spin emulators that use host GPUs.
- Parameterize AVD creation so as to specify [API Levels](https://developer.android.com/guide/topics/manifest/uses-sdk-element.html#ApiLevels). 

*Posted by [Vishal Gowda](https://www.hackerearth.com/@cartmanboy1991)* &middot; *[@VishalGowdaN](https://twitter.com/VishalGowdaN)* &middot; *[GitHub](https://github.com/farthVader91)*
