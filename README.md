dovesnap
=================

dovesnap is a docker network provider, that works with FAUCET and OVS. This allows docker networks to make use of FAUCET's features, such as mirroring, ACLs, and Prometheus based monitoring.

Thanks to the folks who wrote the orginal [docker-ovs-plugin](https://github.com/gopher-net/docker-ovs-plugin) which is what this project was forked from.

See also https://docs.faucet.nz for FAUCET documentation, including monitoring documentation (dovesnap will supply FAUCET-based monitoring without needing configuration, when started as below).

### Requirements

* Linux host running a supported version docker (x86 and Pi are supported)
* Optionally: additional physical interfaces to connect other hosts also running dovesnap
* non-netfilter iptables. For Debian/Ubuntu, follow the legacy option at https://wiki.debian.org/iptables (`update-alternatives --set iptables /usr/sbin/iptables-legacy`). This requirement will be addressed in a future version.

### QuickStart Instructions

These instructions describe the most basic use of dovesnap - creating a docker network with Internet access, where dovesnap provides all the FAUCET infrastructure. See below for more advanced usage.

**1.** Make sure you are using Docker 1.10 or later

**2.** You need to `modprobe openvswitch` on the machine where the Docker Daemon is located. Make sure that while the module is loaded, OVS is not running on the host.

```
$ sudo modprobe openvswitch
```

**3.** Create a directory for FAUCET to store its configuration:

```
$ sudo mkdir /etc/faucet
```

**4.** Start dovesnap.

`$ docker-compose build && FAUCET_PREFIX=/etc/faucet docker-compose -f docker-compose.yml -f docker-compose-standalone.yml up -d`

**5.** Now you are ready to create a new network

```
$ docker network create mynet -d ovs --internal -o ovs.bridge.mode=nat -o ovs.bridge.dpid=0x1 -o ovs.bridge.controller=tcp:127.0.0.1:6653,tcp:127.0.0.1:6654
```

`-d ovs` tells docker to use dovesnap as a network provider.

`--internal` tells docker not to supply an additional network connection to containers on the new network for internet access. This is essential for dovesnap to complete control over connectivity.

`-o ovs.bridge.mode=nat` tells dovesnap to arrange NAT for the new network.

`-o ovs.bridge.dpid=0x1 -o ovs.bridge.controller=tcp:127.0.0.1:6653,tcp:127.0.0.1:6654` tell dovesnap which FAUCET will control this network (you can provide your own FAUCET elsewhere on the network, but in this example we are using a dovesnap-provided FAUCET instance).

**6.** Test it out!

```
$ docker run -d --net=mynet --rm --name=testcon busybox sleep 1h
$ docker exec -t testcon ping 4.2.2.2
```

### Advanced usage

There are several options available when creating a network, and when creating a container on a network, to access FAUCET features.

You can view dovesnap's OVS bridges using `ovs-vsctl` and `ovs-ofctl`, from within the dovesnap OVS container. You can even use `ovs-vsctl` to add other (for example, physical) ports to a dovesnap managed bridge and dovesnap will monitor them for you. However, it's recommended that you use dovesnap's own options (below) where possible.

#### Required options

`ovs.bridge.dpid=<dpid> -o ovs.bridge.controller=tcp:<ip>:<port>`

Every dovesnap network requires a DPID (for OVS and FAUCET), and at least one controller for OVS. dovesnap will provide FAUCET and Gauge processes to do forwarding and monitoring by default - at least one FAUCET is required, and one Gauge if monitoring is desired.

#### Network options

These options are supplied at `docker network create` time.

##### Bridge modes

There are two `ovs.bridge.mode` modes, `flat` and `nat`. The default mode is `flat`.

- `flat` causes dovesnap to provide connectivity only between containers on this docker network - not to other networks.

- `nat` causes dovesnap to provision NAT for the docker network.

If NAT is in use, you can specify `-p <outside port>:<inside port>` when starting a container. dovesnap will provision a DNAT rule, via the network's gateway from the outside port to the inside port on that container. This mapping won't show up in `docker ps`, as dovesnap is not using docker-proxy.

You can also specify an input ACL for the NAT port with `-o ovs.bridge.nat_acl=<acl>`

##### Userspace mode

`-o ovs.bridge.userspace=true`

This requests a user space ("netdev"), rather than kernel space switch from OVS. Certain OVS features such as meters, used to implement rate limiting, will only work on a user space bridge.

##### Adding a physical port/real VLAN

`-o ovs.bridge.add_ports=eno123/8`

Dovesnap will connect `eno123` to the docker network, and attempt to use OVS OFPort 8 (OVS will select another port number, if for some reason port 8 is already in use). You can specify more ports with commas. The OFPort specification is optional - if not present dovesnap will select the next free port number.

###### Enabling DHCP

`--ipam-driver null -o ovs.bridge.dhcp=true`

docker's IP management of this network will be disabled, and instead dhcp will request and maintain a DHCP lease for each container on the network, using `udhcpc`. `udhcpc` is run outside the container's PID namespace (so the container cannot see it), but within its network namespace. The container therefore does not need any special privileges and cannot change its IP address itself.

##### Mirroring

Dovesnap provides infrastructure to do centralized mirroring - you can have dovesnap mirror the traffic for any container on a network it controls, back to a single interface (virtual or physical). This allows you to (for example) run one centralized tcpdump process that can collect all mirrored traffic.

To use physical interface `eno99` for mirroring, for example:

`$ FAUCET_PREFIX=/etc/faucet MIRROR_BRIDGE_OUT=eno99 docker-compose -f docker-compose.yml -f docker-compose-standalone.yml up -d`

If you want to mirror to a virtual interface on your host, use a veth pair. For example:

```
$ sudo ip link add mirrori type veth peer name mirroro
$ sudo ip link set dev mirrori up
$ sudo ip link set dev mirroro up
$ FAUCET_PREFIX=/etc/faucet MIRROR_BRIDGE_OUT=mirrori docker-compose -f docker-compose.yml -f docker-compose-standalone.yml up -d
$ sudo tcpdump -n -e -v -i mirroro
```

From this point, any container selected for mirroring (see below) will have traffic mirrored to tcpdump running on `mirroro`

###### Mirroring across multiple hosts

You might want to run docker and dovesnap on multiple hosts, and have all the mirrored traffic from all the hosts arrive on one port on one host.

You can do this by daisy-chaining the hosts together with dedicated physical ports in a so called "mirror river". On the hosts within the chain, specify `MIRROR_BRIDGE_IN=eth77` (where `eth77` is connected to the previous host). This will cause each host to pass the mirrored traffic along to the final host.

#### Container options

These options are supplied when starting a container.

#### ACLs

`--label="dovesnap.faucet.portacl=<aclname>"`

An ACL will be applied to the port associated with the container. The ACL must already exist in FAUCET (e.g. by adding it to `faucet.yaml`).

#### Mirroring

`--label="dovesnap.faucet.mirror=true"`

The container's traffic (both sent and received) will be mirrored to a port on the bridge (see above).

#### Visualizing dovesnap networks

Dovesnap can generate a diagram of how containers and interfaces are connected together, with some information about running containers (e.g. MAC and IP addresses). This can be useful for troubleshooting or verifying configuration.

```
$ cd graph_dovesnap
$ sudo pip3 install -r requirements.txt
$ ./graph_dovesnap.py
```

A PNG file will be created that describes the networks dovesnap controls.
