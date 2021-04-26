# Kubernetes HA
## Usage
```bash
# on first node
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=master \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['192.168.1.231', '192.168.1.232']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service

# on second node
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=backup \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['192.168.1.231', '192.168.1.232']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service

# on third node
docker run -d --net=host --volume $(pwd)/config/haproxy:/usr/local/etc/haproxy \
    -e HAPROXY_PORT=7443 \
    haproxy:2.3
docker run -d --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
    -e KEEPALIVED_INTERFACE=eth0 \
    -e KEEPALIVED_PASSWORD=pass \
    -e KEEPALIVED_STATE=backup \
    -e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['192.168.1.231', '192.168.1.232']" \
    -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['172.16.217.171', '172.16.217.172', '172.16.217.173']" \
    osixia/keepalived:2.0.20 --copy-service
```
Finally, point your dns to either `192.168.1.231` or `192.168.1.232` and set api server endpoint to `https://your.domain.com:6443` in kubeconfig. From now on, every `kubectl` request will be pointed to one of our virtual ip and mapped to the current master peer. The request will eventually be routed to one of the three kube api servers by the HAProxy on the master peer.
## Reference
- https://kubernetes.io/docs/tasks/administer-cluster/highly-available-master/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
- https://www.kubernetes.org.cn/6964.html
- https://www.mdeditor.tw/pl/pRgX/zh-tw