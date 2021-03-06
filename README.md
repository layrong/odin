Odin
====

Odin is an SDN framework programmable enterprise WLANs. It provides
a platform for developing typical enterprise WLAN services such as
mobility managers, and load balancers as "network applications".


Requirements
------------

For the agent/AP:

- Click Modular Router.
- Open vSwitch.
- An ath9k driver based WiFi card.


Building/Installation
---------------------

Running Odin implies setting up the Odin master, and Odin agents.

If you cloned Odin from the git repository, pull the individual submodules:

```
  $: git clone http://github.com/lalithsuresh/odin
  $: cd odin
  $: git submodule init
  $: git submodule update
```

To build the master (which is built as an application on top of Floodlight),
do the following:

```
  $: cd odin-master
  $: ant
```

You should find floodlight.jar inside target/

Before building the agent, apply the patch in odin-driver-patches to your
Linux kernel ath9k driver code.

To build the agent, copy the files in odin-agent/src/ to your Click source's
/elements/local folder, then build Click.

```
  $: cd odin-agent
  $: cp src/* <click-folder>/elements/local
```

Now build Click using your cross compiler. Don't forget to pass the --enable-
local flag to Click's configure script.

Generate a Click file for the agent, using your preferred values for the
options:

```
  $: python agent-click-file-gen.py <AP_CHANNEL> <QUEUE_SIZE> \
   <HW_ADDR_OF_WIFI_INTERFACE> <ODIN_MASTER_IP> <ODIN_MASTER_PORT>
     > agent.click
```

Running Odin
------------

Master
------

The master is to be run on a central server that has IP reachability to all
APs in the system. The master expects the following configuration parameter
to be set in the floodlight configuration file:

* `net.floodlightcontroller.odin.master.OdinMaster.poolFile`:

This should point to a pool file, which are essentially slices. An example
poolfile is as follows:

```
  # Pool-1
  NAME pool-1
  NODES 172.17.1.13 172.17.1.21 172.17.1.29
  NETWORKS odin
  APPLICATIONS net.floodlightcontroller.odin.applications.OdinMobilityManager

  # Pool-2
  NAME pool-2
  NODES 172.17.1.29 172.17.1.37 172.17.1.45
  NETWORKS guest-network
  APPLICATIONS net.floodlightcontroller.odin.applications.SimpleLoadBalancer
```

Each pool is defined by a name, a list of IP address of physical APs, the
list of SSIDs or NETWORKS to be announced, and a list of applications
that operate on that pool.

* `net.floodlightcontroller.odin.master.OdinMaster.clientList [optional]`:

For testing purposes, if you'd like to assign a static IP to a client
and have it connect to odin, you need to specify the client's details in a file
pointed to by this property. An example file looks as follows:

```
  00:16:7f:7e:00:00 172.17.4.2 00:1b:1b:7e:00:00 odin-ssid-1
  00:16:7f:7e:00:01 172.17.4.3 00:1b:1b:7e:00:01 odin-ssid-2
  00:16:7f:7e:00:02 172.17.4.4 00:1b:1b:7e:00:02 odin-ssid-3
```

Each row represents a client's MAC address, its static IP address, its LVAP's
BSSID, and the SSID that its LVAP will announce.

To run the master:

```
  $: java -jar floodlight.jar
```


Agents
------

To run the agents, first instantiate Open vSwitch and have it connect to the
Floodlight controller from above. Next, instantiate a monitor device:

```
  # If on OpenWRT Backfire
  $: iw phy phy0 interface add mon0 set type monitor
  $: iw dev wlan0 set channel <required-channel>
  $: ifconfig mon0 up
```

Then move the agent.click file generated through the click file generator to
the AP, and run the following:

```
  $: click-align agent.click | click &
```

(Make sure agent.click specifies the same channel as being used by the monitor
device.)

Click should have instantiated a tap device named 'ap', which should be added
to Open vSwitch' datapath:

```
  $: ovs-dpctl add-if dp0 ap
```

Wait a few seconds for the agent to successfully connect to the master.


References
----------

The system is described in the following Masters' thesis:
http://lalithsuresh.files.wordpress.com/2011/04/lalith-thesis.pdf
