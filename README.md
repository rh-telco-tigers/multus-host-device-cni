# multus-host-device-cni for OCP on OSP 13
example for host-device CNI

# About
These are examples for enabling multus using host-device CNI plugin.

# Prerequisites
1. Provider network shared with project in which OCP is running
2. This is tested with OCP 4.7.x
3. OCP is running on OSP 13

# Step 1
Add second interface to worker nodes. Use the following command to add additional interface to worker instance
```
$ openstack port create <port name> --network <provider network name>
$ openstack server add port <server> <port>
```
Repeat this for all the worker nodes

# Step 2
Create a project in OCP
```
$ oc new-project networking-demos
```
Create Network Attach Definition in networking-demos namespace
```
$ cat <<EOF | oc apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: multus2
  namespace: networking-demos
spec:
  config: '{
        "cniVersion": "0.3.1",
        "type": "host-device",
        "device": "ens6",
        "ipam": {
            "type": "dhcp"
            }
          }'
EOF
```
List Network Attach Definition we just created
```
$ oc get network-attachment-definitions -n networking-demos
NAME      AGE
multus2   10s
```
Now create the pod
```
$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  namespace: network-demos
  annotations:
    k8s.v1.cni.cncf.io/networks: |-
      [
        {
          "name": "multus2",
          "namespace": "networking-demos",
          "default-route": ["192.168.30.1"]
        }
      ]
spec:
  containers:
  - name: example-pod
    command: ["/bin/bash", "-c", "sleep 2000000000000"]
    image: centos/tools
```
Check events to see if pod gets additional interface
```
$ oc get events
......
49m         Normal    Scheduled                pod/example-pod2                       Successfully assigned networking-demos/example-pod2 to pluto-vb6sx-worker-0-gpx29
49m         Normal    AddedInterface           pod/example-pod2                       Add eth0 [10.129.3.71/23]
49m         Normal    AddedInterface           pod/example-pod2                       Add net1 [192.168.30.210/24] from networking-demos/multus2
49m         Normal    Pulling                  pod/example-pod2                       Pulling image "centos/tools"
49m         Normal    Pulled                   pod/example-pod2                       Successfully pulled image "centos/tools" in 1.365335692s
49m         Normal    Created                  pod/example-pod2                       Created container example-pod
......
```
Login to pod to check
```
$ oc rsh example-pod2
sh-4.2# ifconfig
...
net1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.30.210  netmask 255.255.255.0  broadcast 192.168.30.255
        inet6 fe80::f816:3eff:fef7:1ffc  prefixlen 64  scopeid 0x20<link>
...
sh-4.2# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.30.1    0.0.0.0         UG    0      0        0 net1
10.128.0.0      0.0.0.0         255.252.0.0     U     0      0        0 eth0
10.129.2.0      0.0.0.0         255.255.254.0   U     0      0        0 eth0
...
```
Check connectivity
```
[root@bastion ~]# ping 192.168.30.210
PING 192.168.30.210 (192.168.30.210) 56(84) bytes of data.
64 bytes from 192.168.30.210: icmp_seq=1 ttl=63 time=41.8 ms
64 bytes from 192.168.30.210: icmp_seq=2 ttl=63 time=7.76 ms
^C
--- 192.168.30.210 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 7.757/24.770/41.784/17.013 ms
```


