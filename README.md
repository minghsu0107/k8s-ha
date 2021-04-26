# Kubernetes HA
## Overview
The following diagram shows a typical high-availibility Kubernetes cluster with embedded etcd: 

![](https://i.imgur.com/FwEzzWV.png)

Each of K8s controller plane replicas will run the following components in the following mode:
- etcd instance: all instances will be clustered together using consensus;
- API server: each server will talk to local etcd - all API servers in the cluster will be available;
- controllers, scheduler, and cluster auto-scaler: will use lease mechanism - only one instance of each of them will be active in the cluster;
- add-on manager: each manager will work independently trying to keep add-ons in sync

However, HA architecture also brings about another problem - which K8s API server should the client connect to? Simply adding mulitple DNS A records does not solve the problem since DNS server could not check server healthiness in real time. A reverse proxy with upstream health check is still not good enough because it would become a single-point-of-failure.

**The concept of virtual ip comes to the rescue**.

To achieve high availibility, we will use Keepalived, a LVS (Linux Virtual Server) solution based on VRRP protocol. A LVS service contains a master and multiple backup servers while exposing a virtual ip to outside as a whole. The virtual ip will be pointed to the master server. The master server will send heartbeats to backup servers. Once backup servers are not receiving heartbeats, one backup server will take over the virtual ip, ie. the virtual ip will "float" to that backup server.

To sum up, we have the following architecture:

![](https://i.imgur.com/FrwyxCQ.png)

1. A floating vip maps to three keepalived servers (one master and two backups). Each keepalived server runs on a K8s controller plane replica.
2. When keepalived master server fails, vip floats to another backup server automatically.
3. HAProxy load-balances traffic to three K8s API servers.

The following section is hands-on tutorial that implements the above architecture.
## Settings
- Three controller plane nodes in private network: `172.16.0.0/16`
- Node private IPs: `172.16.217.171`, `172.16.217.172`, `172.16.217.173`
- Virtual IP: `172.16.100.100`, `172.16.100.101`
## Usage
```bash
# on first node
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3-alpine
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    --volume $(pwd)/config/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=MASTER \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['172.16.100.100', '172.16.100.101']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service

# on second node
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3-alpine
    --volume $(pwd)/config/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf \
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=BACKUP \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['172.16.100.100', '172.16.100.101']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service

# on third node
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3-alpine
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    --volume $(pwd)/config/keepalived/keepalived.conf:/container/service/keepalived/assets/keepalived.conf \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=BACKUP \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['172.16.100.100', '172.16.100.101']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service
```
Finally, point your dns to either `172.16.100.100` or `172.16.100.101`. From now on, a request with url `<https://your.domain.com:7443>` will be resolved to `https://<172.16.100.100|172.16.100.101>:7443`, rounted to the current keepalived master, and eventually sent to one of the three K8s API servers by HAProxy.

Clean up virtual ip:
```bash
ip addr del 172.16.100.100/32 dev eth0
ip addr del 172.16.100.101/32 dev eth0
```
## Reference
- https://kubernetes.io/docs/tasks/administer-cluster/highly-available-master/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
- https://www.kubernetes.org.cn/6964.html
- https://www.mdeditor.tw/pl/pRgX/zh-tw