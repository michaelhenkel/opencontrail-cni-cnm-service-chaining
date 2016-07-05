# Service chaining with OpenContrail using CNI and CNM based container

CNI (Container Network Interface) and CNM (Container Network Model) are two competing    
container networking frameworks. CNI can be used for k8s, OpenShift and Mesos,    
whereas CNM is the default for Dockers libnetwork.    
Both provide a well documented APIs and support a driver plugin model.    
The driver plugin model allows to use a SDN backend providing the overlay network between    
containers located on the same or different hosts.    
A SDN backend providing drivers for both can abstract the difference in the networking    
models and provide a common networking infrastructure.    
OpenContrail is the perfect candidate as it is OpenSource and provides very rich APIs    
for different programming languages. The CNM driver is written in Python and the CNI    
in Go:    

https://github.com/michaelhenkel/opencontrail-cni-plugin    
https://github.com/michaelhenkel/opencontrail-docker-libnetwork    

In order to show the full potential of the abstraction this example uses one CNI    
and one CNM based container as Service Instances (SI) in a Service Chain between    
two other containers.    
Two additional containers will be connected to VNs net1 and net2. A network policy    
will force traffic between the containers through the CNM and CNI SI.    
Therefore two virtual networks (VNs) (net1 and net2) will be created. The VNs    
must be created using the CNM command line as CNM doesn't support discovery    
of VNs not being created by CNM.    


# Logical Network Setup
```
                      +-------------+
                      |    cnmSI    |
                      |   Docker    |
                      |.3         .3|
                      +-+---------+-+
                        |         |
             +----------++       ++----------+
+---------+  |net1:      |       |net2:      |  +---------+
|Docker .2+--+10.1.1.0/24|       |10.2.2.0/24+--+.2 Docker|
+---------+  +----------++       ++----------+  +---------+
                        |         |
                      +-+---------+-+
                      |.254     .254|
                      |  Namespace  |
                      |    cniSI    |
                      +-------------+
```

# Communication Flow
```
              +-------------+
              |    cnmSI    |
 +---------+  |   Docker    |
 |Docker|.2+-->.3         .3|
 +---------+  +------------++
                           |
                 +---------+
                 |
              +--v----------+  +---------+
              |.254     .254+-->.2 Docker|
              |  Namespace  |  +---------+
              |    cniSI    |
              +-------------+
```
#Setup Instructions:

Install Kernel headers:    
```
apt-get install -y linux-headers-`uname -r` linux-image-`uname -r` linux-headers-`uname -r`-generic linux-image-`uname -r`-generic
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
``
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

Create a CNI source file:    
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
export CNI_CONTAINERID=cniSI
export CNI_NETNS=/var/run/netns/cniSI
EOF
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
docker-compose run -e CNI_COMMAND=ADD -e CNI_NETNS=/var/run/netns/cniSI --rm cni  < /etc/cni/net.d/10-opencontrail-multi.conf
```

Check create vrouter virtual interfacesL    
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

Start ping between alp1 and alp2 (will not work, yet):    
```

```
docker-compose run -e CNI_COMMAND=DEL -e CNI_NETNS=/var/run/netns/cniSI --rm cni  < /etc/cni/net.d/10-opencontrail-multi.conf
