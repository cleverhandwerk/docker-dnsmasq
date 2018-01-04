# docker-dnsmasq

based on andyshinn/docker-dnsmasq docker image

Purpose:
* make it fully configurable through environment variables
** use one image to run them all
** run stateless, environment configured containers (see https://12factor.net/)
* use primarily to setup DNS/DHCP for simple/home environments

## Usage

Try it in 3 steps

### create your own docker.env
`docker run --rm -it drpsychick/docker-dnsmasq:latest --test`
`docker run --rm -it drpsychick/docker-dnsmasq:latest --export > dnsmasq.env`

### run and test
Run in a separate teminal
```
docker run --rm -it --cap-add NET_ADMIN --env-file dnsmasq.env --name dnsmasq-1 drpsychick/docker-dnsmasq:latest -k -q --log-facility=-
```

Test it
```
# test DNS and DHCP
container_ip=$(docker inspect dnsmasq-1 -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
docker_interface=$(docker network inspect bridge -f '{{index .Options "com.docker.network.bridge.name"}}')
nslookup google.com $container_ip

sudo ip link add test0 link docker0 type macvlan mode bridge
sudo dhclient -1 -d -s $container_ip test0
sudo ip link del test0 link docker0 type macvlan mode bridge
```

If that ruins your routing because of a new default gateway (check `route -n`):
```
sudo route del -net 172.17.10.0/24 gw 172.10.10.1 dev $docker_interface
```

## Use case: run in local network
Some additional work is needed in order to run the docker container with an IP from your local subnet and to server request for your subnet.
If you don't need DHCP, you can skip most part of it. 

Read:
http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/
https://docs.docker.com/engine/userguide/networking/get-started-macvlan/#macvlan-bridge-mode-example-usage

**Important**
DHCP will only work if the DHCP range is on the interface it runs on
In other words: running DHCP on the ip of the docker container will not work, it needs to have an IP on the subnet it will serve DHCP requests on

### Example: 
* 192.168.1.253 IP of the DNS/DHCP server
* 192.168.1.1   IP of the gateway
* 192.168.1.110-120 is an unused range of IPs
* eth0 is the network device on the local subnet

#### dnsmasq.env:

> DMQ_HOST1=192.168.1.1 gateway.local gateway
> DMQ_DHCP_GATEWAY=dhcp-option=3,192.168.1.1
> DMQ_DHCP_RANGES=dhcp-range=192.168.1.110,192.168.1.120,24h
> DMQ_DHCP_DNS=dhcp-option=6,192.168.1.253,8.8.8.8,8.8.4.4

test configuration:
`docker run --rm -it --env-file dnsmasq.env drpsychick/docker-dnsmasq:latest --test`

#### Network
To run the container with an IP from the local subnet, we need the "macvlan" driver. 
And in order to be able to interact with the container even from the host, we need to create a virtual interface.

```
# create linked interface mac0 and use this instead of the parent eth1
sudo ip link add mac0 link eth1 type macvlan mode bridge
sudo ip addr flush dev eth1
sudo dhclient mac0

# create macvlan network with our subnet
docker network create --driver macvlan --subnet 192.168.1.0/24 --gateway 192.168.1.1 -o parent=mac0 local-net
```

#### Run in a separate shell:
```
docker run --rm -it --net local-net --ip 192.168.1.253 --cap-add NET_ADMIN --env-file dnsmasq.env --publish 53:53 --publish 53:53/udp --publish 67:67/udp --name dnsmasq-1 drpsychick/docker-dnsmasq:latest -k -q --log-facility=-
```

#### Test it
```
nslookup google.de 192.168.1.253

sudo ip link add mac1 link eth1 type macvlan mode bridge
sudo dhclient -1 -d -s 192.168.1.253 mac1
sudo ip link del mac1
```

#### All good, now lets see it in production:
```
# run it
Same "run" command as above, but with "-d" and "--restart always" instead of "--rm -it" (run as daemon)
docker run -d --net local-net --ip 192.168.1.253 --cap-add NET_ADMIN --env-file dnsmasq.env --restart always --publish 53:53 --publish 53:53/udp --publish 67:67/udp --name dnsmasq-1 drpsychick/docker-dnsmasq:latest -k -q --log-facility=-

# watch it
docker attach --sig-proxy=false dnsmasq-1
```

## The simple way
For services other than DHCP you still have to some manual tweaking, but its much easier to do

Try this:
```
sudo ip addr add 192.168.1.253/32 dev eth1
docker run ... --publish 192.168.1.253:53:53 ... (for every port)
```





### automate
- see ansible-galaxy DrPsychick/ansible-docker


## Use case: Home DNS+DHCP

