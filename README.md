# Service Function Chaining (SFC) with Neutron, CNI and CNM using OpenContrail    
## (One SDN to rule them all)

CNI (Container Network Interface) and CNM (Container Network Model) are two competing    
container networking frameworks. CNI can be used for k8s, OpenShift and Mesos,    
whereas CNM is the default for Dockers libnetwork.    
Both provide well documented APIs and support a driver plugin model.    
The driver plugin model allows to use a SDN backend providing the overlay network between    
containers located on the same or different hosts.    
A SDN backend providing drivers for both can abstract the differences in the networking    
models and provide a common networking infrastructure.    
Because of being an OpenSource product OpenContrail is the perfect candidate.    
It also provides very rich APIs for different programming languages which makes it easy    
to implement new drivers. The CNM driver is written in Python and the CNI in Go:    

https://github.com/michaelhenkel/opencontrail-cni-plugin    
https://github.com/michaelhenkel/opencontrail-docker-libnetwork    

In order to show the full potential of the abstraction this example uses one CNI    
and one CNM based container as Service Instances (SI) in a Service Chain between    
two other containers. To complete the setup an OpenStack based VM is added as a third SI.    
The OpenStack SI uses the OpenContrail neutron plugin.        
Two additional containers will be connected to virtual networks (VNs) net1 and net2.    
A network policy will force traffic between the containers through the OpenStack, CNM and CNI SI.    
Therefore two VNs (net1 and net2) will be created. The SIs will have connections into    
net1 and net2. The other two containers are single legged and connected to net1 and net2.    
The VNs must be created using the CNM command line as CNM doesn't support discovery    
VNs not being created by CNM.

# Logical Network Setup
```
                            +-------------+
                            |    cnmSI    |
                            |   Docker    |
                        +---+.3         .3+---+
                        |   +-------------+   |
                        |                     |
                        |   +-------------+   |
             +----------++  |  neutronSI  |  ++----------+
+---------+  |VN net1:   +--+.252 KVM .252+--+VN net2:   |  +---------+
|Docker .2+--+10.1.1.0/24|  |             |  |10.2.2.0/24+--+.2 Docker|
+---------+  +----------++  +-------------+  +-+---------+  +---------+
                        |                      |
                        |   +-------------+    |
                        +---+.254     .254+----+
                            |  Namespace  |
                            |    cniSI    |
                            +-------------+

```

# Communication Flow
```

             +-------------+  +-------------+  +-------------+
             |    cnmSI    |  |    cniSI    |  |  neutronSI  |
+---------+  |   Docker    |  |  Namespace  |  |    KVM      |  +---------+
|Docker .2+-->.3         .3+-->.254     .254+-->.252     .252+-->.2 Docker|
+---------+  +-------------+  +-------------+  +-------------+  +---------+

```
#Setup Instructions:
Prerequisites:    
* A running OpenContrail backend    
* A running OpenStack backend    
* docker-engine and docker-compose installed on the host    
      (for multi-host networking the docker-engine must be    
       connected to a key/value store)    

The CNM and CNI drivers (and vrouter agent) are run as containers so    
that the host must not install any depending package besides kernel    
headers which are needed for the vrouter kernel module.    

Install Kernel headers:    
```
apt-get install -y linux-headers-`uname -r` \
                   linux-image-`uname -r` \
                   linux-headers-`uname -r`-generic \
                   linux-image-`uname -r`-generic
```

Setup Environment Variables (adjust as needed):    
```
FULL_KERNEL=`uname -r`
KERNEL=`echo $FULL_KERNEL |awk -F"-" '{print $1 "-" $2}'`
CONTRAIL_IP=10.87.64.34
CONTRAIL_USER=admin
CONTRAIL_PWD=contrail123
TOKEN=contrail123
INTERFACE=eth0
IP_PREF=`ip addr sh dev $INTERFACE |grep "inet " |awk '{print $2}'`
PREFIX=`echo $IP_PREF|awk -F"/" '{print $1}'`
PREFIX_LEN=`echo $IP_PREF|awk -F"/" '{print $2}'`
GATEWAY=`ip r sh |grep default |awk '{print $3'}`
```

Create environment file for CNM container:    
```
mkdir contrail
cd contrail
cat << EOF > common.env
SERVICE_NAME=vrouter
CREATE_MODULE=true
ADMIN_PASSWORD=$CONTRAIL_PWD
ADMIN_TENANT=$CONTRAIL_USER
ADMIN_TOKEN=$TOKEN
ADMIN_USER=$CONTRAIL_USER
CIDR=$PREFIX_LEN
DISCOVERY_SERVER=$CONTRAIL_IP
CONFIG_API_SERVER=$CONTRAIL_IP
GATEWAY_IP=$GATEWAY
KEYSTONE_SERVER=$CONTRAIL_IP
MEMCACHED_SERVER=$CONTRAIL_IP
HOST_IP=$PREFIX
MYSQL_ROOT_PASSWORD=contrail123
MYSQL_SERVER=$CONTRAIL_IP
NEUTRON_SERVER=$CONTRAIL_IP
NOVA_API_SERVER=$CONTRAIL_IP
INTERFACE=$INTERFACE
PHYSICAL_INTERFACE=$INTERFACE
RABBIT_SERVER=$CONTRAIL_IP
REDIS_SERVER=$CONTRAIL_IP
VNC_PROXY=$CONTRAIL_IP
OS_TOKEN=$TOKEN
OS_URL=http://$CONTRAIL_IP:35357/v2.0
SOCKET_PATH=/run/docker/plugins
SCOPE=global
DEBUG=true
EOF
```

Create environment for CNI container:        
```
cat << EOF > cni.env
API_SERVER_PORT=8082
API_SERVER_IP=$CONTRAIL_IP
OS_TENANT_NAME=$CONTRAIL_USER
OS_AUTH_URL=http://$CONTRAIL_IP:35357/v2.0
OS_USERNAME=$CONTRAIL_USER
OS_PASSWORD=$CONTRAIL_PWD
LOG_LEVEL=debug
CNI_PATH=~/cni/bin
CNI_IFNAME=eth0
EOF
```

Create docker-compose file:    
```
cat << EOF > docker-compose.yml
version: '2'
services:
  cnm:
    image: michaelhenkel/opencontrail-docker-libnetwork:3.0.2-09e5d48
    network_mode: "host"
    depends_on:
      - vrouter-agent
    env_file: common.env
    volumes:
      - /run/docker/plugins:/run/docker/plugins
    cap_add:
      - NET_ADMIN
  cni:
    image: michaelhenkel/opencontrail-cni-plugin:3.0.2-09e5d48
    network_mode: "host"
    depends_on:
      - vrouter-agent
    env_file: cni.env
    privileged: true
    volumes:
      - /etc/cni/net.d:/etc/cni/net.d
      - /var/run/netns:/var/run/netns
    volumes_from:
      - vrouter-agent
    cap_add:
      - NET_ADMIN
  vrouter-agent:
    image: michaelhenkel/vrouter-agent:3.0.2-09e5d48
    privileged: true
    network_mode: "host"
    env_file: common.env
    environment:
      - SERVICE_NAME=vrouter
    ports:
      - 8097:8097
      - 9090:9090
      - 9091:9091
      - 8085:8085
    volumes:
      - /etc/redhat-release:/etc/redhat-release
      - /usr/src/linux-headers-$FULL_KERNEL:/usr/src/linux-headers-$FULL_KERNEL
      - /usr/src/linux-headers-$KERNEL:/usr/src/linux-headers-$KERNEL
      - /lib/modules:/lib/modules
      - /var/lib/contrail/ports
EOF
```

Create network definition for CNI based namespaces:    
```
mkdir -p /etc/cni/net.d
cat << EOF > /etc/cni/net.d/10-opencontrail-multi.conf
{
    "networks": [
        {
            "name": "net1",
            "type": "opencontrail",
            "mtu": 1492,
            "ipam": {
                "subnet": "10.1.1.0/24",
                "routes": [
                    { "dst": "0.0.0.0/0" }
                ]
            }
        },
        {
            "name": "net2",
            "type": "opencontrail",
            "mtu": 1492,
            "ipam": {
                "subnet": "10.1.2.0/24",
                "routes": [
                    { "dst": "0.0.0.0/0" }
                ]
            }
        }
    ]
}
EOF
```

Create a CNI source file and source it:    
```
cat << EOF > ~/.cnirc
export API_SERVER_PORT=8082
export API_SERVER_IP=$CONTRAIL_IP
export OS_TENANT_NAME=$CONTRAIL_USER
export OS_AUTH_URL=http://$CONTRAIL_IP:35357/v2.0
export OS_USERNAME=$CONTRAIL_USER
export OS_PASSWORD=$CONTRAIL_PWD
export LOG_LEVEL=debug
export CNI_PATH=~/cni/bin
export CNI_IFNAME=eth0
export PATH=$PATH:$CNI_PATH
EOF
source ~/.cnirc
```

Bring up vrouter agent:    
```
docker-compose up -d vrouter-agent
docker-compose logs -f
ip addr sh
```

Bring up CNM driver container:    
```
docker-compose up -d cnm
docker-compose logs -f cnm
```

Create VNs using CNM CLI:    
```
docker network create -d opencontrail -o rt=64512:1 -o name=net1 --subnet 10.1.1.0/24 net1
docker network create -d opencontrail -o rt=64512:2 -o name=net2 --subnet 10.1.2.0/24 net2
```

Create CNM containers:    
```
docker create --name cnmSI --net net1 alpine:3.1 tail -f /dev/null && docker network connect net2 cnmSI && docker start  cnmSI
docker run -tid --name alp1 --net net1 alpine:3.1 /bin/sh
docker run -tid --name alp2 --net net2 alpine:3.1 /bin/sh
```

Bring up CNI driver container:    
```
docker-compose up -d cni
```

Create CNI based namespace:    
```
ip netns add cniSI
docker-compose run -e CNI_COMMAND=ADD -e CNI_NETNS=/var/run/netns/cniSI -e CNI_CONTAINERID=cniSI --rm cni  < /etc/cni/net.d/10-opencontrail-multi.conf
```

Check vrouter interfaces:    
```
docker exec -it contrail_vrouter-agent_1 vif --list
```

Check assigned ip addresses:    
```
docker exec -it alp1 ip addr sh
docker exec -it alp2 ip addr sh
docker exec -it cnmSI ip addr sh
ip netns exec cniSI ip addr sh
```

Get the neutron net ids for net1 and net2:    
```
nova net-list
+--------------------------------------+-------------------------+------+
| ID                                   | Label                   | CIDR |
+--------------------------------------+-------------------------+------+
| b9081815-4dbc-4910-b36f-b8b1a9319cb2 | ip-fabric               | None |
| 14ef277b-0409-496d-ac88-5780190973f1 | net1                    | None |
| e549c986-93b8-4268-b9e4-2434f384dd6a | net2                    | None |
| 8a61b1b3-b3a2-4821-97d0-9ae7c21ef985 | default-virtual-network | None |
| a2ad148f-2b1b-40e0-be7c-bdfe090b07c1 | __link_local__          | None |
+--------------------------------------+-------------------------+------+
```

Create the neutron SI:    
```
nova boot --flavor nano --image cirros \
     --nic net-id=14ef277b-0409-496d-ac88-5780190973f1 \
     --nic net-id=e549c986-93b8-4268-b9e4-2434f384dd6a c1
```
The Cirros image per default only activates the first network interface.    
The second interface must be activated from within the VM. Also    
ip_forwading must be activated.    


Start ping between alp1 and alp2 (will not work, yet):    
```
docker exec -it alp1 ping 10.1.2.3
PING 10.1.2.3 (10.1.2.3): 56 data bytes
```

#Create Service Chaining Policy

Create Service Template:    
![ScreenShot](/pics/CreateServiceTemplate.png?raw=true "Create Service Template")

Create CNI SI:    
![ScreenShot](/pics/cniSI1.png?raw=true "Create Service Template")
![ScreenShot](/pics/cniSI2.png?raw=true "Create Service Template")

Create CNM SI:    
![ScreenShot](/pics/cnmSI_1.png?raw=true "Create Service Template")
![ScreenShot](/pics/cnmSI2.png?raw=true "Create Service Template")

Create neutron SI:    
![ScreenShot](/pics/neutronSI1.png?raw=true "Create Service Template")
![ScreenShot](/pics/neutronSI2.png?raw=true "Create Service Template")

Create Network Policy:    
![ScreenShot](/pics/createPolicy.png?raw=true "Create Service Template")

Attach NW Policy to VN net1:    
![ScreenShot](/pics/attachPolNet1.png?raw=true "Create Service Template")

Attach NW Policy to VN net2:    
![ScreenShot](/pics/attachPolNet2.png?raw=true "Create Service Template")

Ping starts to work now:
```
docker exec -it alp1 ping 10.1.2.3
PING 10.1.2.3 (10.1.2.3): 56 data bytes
64 bytes from 10.1.2.3: seq=0 ttl=61 time=4.924 ms
64 bytes from 10.1.2.3: seq=1 ttl=61 time=0.225 ms
64 bytes from 10.1.2.3: seq=2 ttl=61 time=0.205 ms
64 bytes from 10.1.2.3: seq=3 ttl=61 time=0.146 ms
64 bytes from 10.1.2.3: seq=4 ttl=61 time=0.326 ms
64 bytes from 10.1.2.3: seq=5 ttl=61 time=0.165 ms
64 bytes from 10.1.2.3: seq=6 ttl=61 time=0.148 ms
64 bytes from 10.1.2.3: seq=7 ttl=61 time=0.141 ms
64 bytes from 10.1.2.3: seq=8 ttl=61 time=0.212 ms
```

In order to double check that traffic is routed through the Service Instances    
the ports of the SIs must be identified:    
![ScreenShot](/pics/ports.png?raw=true "Get SI Ports")

The cnmSI and cniSI port can be related:    
```
docker exec -it cnmSI ip addr sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
308: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:a5:8d:8e:28:e8 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::a5:8dff:fe8e:28e8/64 scope link
       valid_lft forever preferred_lft forever
310: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:e2:6a:99:33:16 brd ff:ff:ff:ff:ff:ff
    inet 10.1.2.2/24 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::e2:6aff:fe99:3316/64 scope link
       valid_lft forever preferred_lft forever


ip netns exec cniSI ip addr sh
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
7: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1492 qdisc noqueue state UP group default
    link/ether 2a:a4:b6:1d:4f:ba brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.254/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::28a4:b6ff:fe1d:4fba/64 scope link
       valid_lft forever preferred_lft forever
9: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1492 qdisc noqueue state UP group default
    link/ether 0a:92:10:55:a6:f7 brd ff:ff:ff:ff:ff:ff
    inet 10.1.2.254/24 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::892:10ff:fe55:a6f7/64 scope link
       valid_lft forever preferred_lft forever
```

A tcpdump on the cniSI and cnmSI veth interfaces show the traffic.    
cnmSI ports:    
```
tcpdump -nS -i veth52ea6c92p0
listening on veth52ea6c92p0, link-type EN10MB (Ethernet), capture size 65535 bytes
01:24:46.666913 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 7424, seq 0, length 64
01:24:46.668164 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 7424, seq 0, length 64
01:24:47.664270 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 7424, seq 1, length 64
01:24:47.664327 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 7424, seq 1, length 64

tcpdump -nS -i veth41c972efp0
listening on veth41c972efp0, link-type EN10MB (Ethernet), capture size 65535 bytes
01:25:24.760265 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 8448, seq 0, length 64
01:25:24.761422 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 8448, seq 0, length 64
01:25:25.757015 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 8448, seq 1, length 64
01:25:25.757071 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 8448, seq 1, length 64
```

cniSI ports:
```
tcpdump -nS -i vethe0a9d196
listening on vethe0a9d196, link-type EN10MB (Ethernet), capture size 65535 bytes
01:27:07.390304 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 9472, seq 0, length 64
01:27:07.405934 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 9472, seq 0, length 64
01:27:08.385644 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 9472, seq 1, length 64
01:27:08.385710 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 9472, seq 1, length 64

tcpdump -nS -i veth91ea13b4
listening on veth91ea13b4, link-type EN10MB (Ethernet), capture size 65535 bytes
01:27:36.404685 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 10496, seq 0, length 64
01:27:36.412126 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 10496, seq 0, length 64
01:27:37.405485 IP 10.1.1.3 > 10.1.2.3: ICMP echo request, id 10496, seq 1, length 64
01:27:37.405666 IP 10.1.2.3 > 10.1.1.3: ICMP echo reply, id 10496, seq 1, length 64
```

neutronSI ports:
```
tcpdump -nS -i tapc5e3a80e-d7
listening on tapc5e3a80e-d7, link-type EN10MB (Ethernet), capture size 65535 bytes
18:53:14.946087 IP 10.1.1.2 > 10.1.2.2: ICMP echo request, id 2304, seq 2995, length 64
18:53:14.947383 IP 10.1.2.2 > 10.1.1.2: ICMP echo reply, id 2304, seq 2995, length 64
18:53:15.946345 IP 10.1.1.2 > 10.1.2.2: ICMP echo request, id 2304, seq 2996, length 64
18:53:15.947661 IP 10.1.2.2 > 10.1.1.2: ICMP echo reply, id 2304, seq 2996, length 64

tcpdump -nS -i tapcd11d589-57
listening on tapcd11d589-57, link-type EN10MB (Ethernet), capture size 65535 bytes
18:53:42.954621 IP 10.1.1.2 > 10.1.2.2: ICMP echo request, id 2304, seq 3023, length 64
18:53:42.955295 IP 10.1.2.2 > 10.1.1.2: ICMP echo reply, id 2304, seq 3023, length 64
18:53:43.954896 IP 10.1.1.2 > 10.1.2.2: ICMP echo request, id 2304, seq 3024, length 64
18:53:43.955575 IP 10.1.2.2 > 10.1.1.2: ICMP echo reply, id 2304, seq 3024, length 64
```

In order to show that multi-host networking works a third container is    
started on a different host:   
```
docker run -itd --name alp3 --net net2 alpine:3.1 /bin/sh
docker exec -it alp3 ip addr sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
25: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:1d:63:f3:2b:a2 brd ff:ff:ff:ff:ff:ff
    inet 10.1.2.4/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1d:63ff:fef3:2ba2/64 scope link
       valid_lft forever preferred_lft forever
```

The container connected to net1 on the first host can ping the new container    
on the second host:
```
docker exec -it alp1 ping 10.1.2.4
PING 10.1.2.4 (10.1.2.4): 56 data bytes
64 bytes from 10.1.2.4: seq=0 ttl=59 time=9.696 ms
64 bytes from 10.1.2.4: seq=1 ttl=59 time=1.019 ms
64 bytes from 10.1.2.4: seq=2 ttl=59 time=0.843 ms
```

The physical interfaces on first and second host show the encapsulated traffic.
Host 1:    
```
compute109:~# tcpdump -nS -i eth0 proto gre
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
01:50:09.074988 IP 192.168.1.109 > 192.168.1.220: GREv0, length 92: MPLS (label 17, exp 0, [S], ttl 59) IP 10.1.1.3 > 10.1.2.4: ICMP echo request, id 12800, seq 0, length 64
01:50:09.077755 IP 192.168.1.220 > 192.168.1.109: GREv0, length 92: MPLS (label 26, exp 0, [S], ttl 63) IP 10.1.2.4 > 10.1.1.3: ICMP echo reply, id 12800, seq 0, length 64
01:50:10.073415 IP 192.168.1.109 > 192.168.1.220: GREv0, length 92: MPLS (label 17, exp 0, [S], ttl 59) IP 10.1.1.3 > 10.1.2.4: ICMP echo request, id 12800, seq 1, length 64
01:50:10.074108 IP 192.168.1.220 > 192.168.1.109: GREv0, length 92: MPLS (label 26, exp 0, [S], ttl 63) IP 10.1.2.4 > 10.1.1.3: ICMP echo reply, id 12800, seq 1, length 64
```

Host 2:    
```
compute220:~# tcpdump -nS -i eth0 proto gre
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
18:49:33.888287 IP 192.168.1.109 > 192.168.1.220: GREv0, length 92: MPLS (label 17, exp 0, [S], ttl 59) IP 10.1.1.3 > 10.1.2.4: ICMP echo request, id 12800, seq 0, length 64
18:49:33.890634 IP 192.168.1.220 > 192.168.1.109: GREv0, length 92: MPLS (label 26, exp 0, [S], ttl 63) IP 10.1.2.4 > 10.1.1.3: ICMP echo reply, id 12800, seq 0, length 64
18:49:34.886729 IP 192.168.1.109 > 192.168.1.220: GREv0, length 92: MPLS (label 17, exp 0, [S], ttl 59) IP 10.1.1.3 > 10.1.2.4: ICMP echo request, id 12800, seq 1, length 64
18:49:34.886937 IP 192.168.1.220 > 192.168.1.109: GREv0, length 92: MPLS (label 26, exp 0, [S], ttl 63) IP 10.1.2.4 > 10.1.1.3: ICMP echo reply, id 12800, seq 1, length 64
```
