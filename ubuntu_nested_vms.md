# Configure communication between Ubuntu guests within nested virtual machines

Ubuntu version 22.04.3 LTS

The topology:

![Topology](../docs/images/ubuntu_nested_vms_topology.png)

Configure 2 Host-only interfaces on the host in VirtualBox and disable DHCP:

First network is 192.168.56.254/24 (.254) is the gateway
Second network is 192.168.57.254/24

Host-only interface enables an isolated network in which guests can communicate with each other and the host,
However outside hosts cannot communicate with that network, as it doesnt have external access.

Now we need two ubuntu machines(H1 & H2), each will act as the router between our main guest, and the and internal machines H1 & H2.

### On H1:
1. Install VirtualBox
2. Add Host-only adapter with address 192.168.58.1/24 
3. Install Ubuntu machine and in its settings -> system -> proccessor,
enable **VT-x** for nested vitualization support
4. Configure Interface with the host, we will use **NetPlan** - network configuration file to take control of all networking devices.

Edit `/etc/netplan/01-network-manager-all.yaml` on **H1** for interfaces configuration, here we are going to add route to anything thats not within this internal network .254:

```yaml
network:
	version: 2
	renderer: NetworkManager
	ethernets:
		enp0s3:
			addresses:
				- 192.168.56.1/24
			routes:
				- to: default
				via: 192.168.56.254
```

Edit the same file on H1 Guest:

```yaml
network:
	version: 2
	renderer: NetworkManager
	ethernets:
		enp0s3:
			addresses:
				- 192.168.58.10/24
			routes:
				- to: default
				via: 192.168.58.254
```

### On H2:
1. Install VirtualBox
2. Add Host-only adapter with address 192.168.57.1/24 
3. Install Ubuntu machine and in its settings -> system -> proccessor,
enable VT-x for nested vitualization support
4. Configure Interface with the host, we will use NetPlan - network configuration file to take control of all networking devices.

Edit `/etc/netplan/01-network-manager-all.yaml` for interfaces configuration, here we are going to add route to anything thats not within this internal network .254:

```yaml
network:
	version: 2
	renderer: NetworkManager
	ethernets:
		enp0s3:
			addresses:
				- 192.168.57.1/24
			routes:
				- to: default
				via: 192.168.57.254
```

Edit the same file on H2 Guest:

```yaml
network:
	version: 2
	renderer: NetworkManager
	ethernets:
		enp0s3:
			addresses:
				- 192.168.59.10/24
			routes:
				- to: default
				via: 192.168.59.254
```

#### Enable IP forwarding on H1, H2, host:
Forward any packets which do not much the routes in the kernel routing table: (not permanent)
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

#### Add two routes on the host for the two isolated networks to communicate
```console
ip route add 192.168.58.0/24 via 192.168.56.1
ip route add 192.168.59.0/24 via 192.168.57.1
```

Now both guests should be able to ping each other.
