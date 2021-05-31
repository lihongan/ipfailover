# ipfailover

### Two types of monitoring appliacation
1. Use hostNetwork:
Application pods working with hostNetwork node, since ipfailover pods working with hostNetwork mode as well so they are in same network namespace and ipfailover can monitor the application port directly, so needn't to create a nodePort service for the application. And we can trigger the failover by restarting the node as well as the application pod.

2. Use Nodeport:
Application pods working with pods networking mode, since ipfailover pods working with hostNetwork mode so they are in different network namespace, to make ipfailover can monitor the port we have to create a NodePort service for the application pods. But because nodeport is always alive/reachable on all nodes, so ipfailover checking scripts always return true and we should trigger a failover by reboot the node (block ipfailover to send keepalive/heartbeat packets) 

### When failover ocuurs
1. MASTER cannot reach the monitor port 
If MASTER cannot reach the port that it monitored, then it will enter FAULT state immediately and remove the Virtual IP from the Interface and stop sending multicast/unicast VRRP packets.

2. BACKUP cannot receive VRRP packets from MASTER
If ipfailover can reach the monitored port then it will enter BACKUP state initially and start sending multicast/unicast VRRP packets, only the BACKUP with highest priority becomes the MASTER. If BACKUP doesn't receive the VRRP packets from MASTER for a period longer than three times of the advertisement timer, the BACKUP takes the MASTER state and assigns the VIP(s) to itself.

note: If two ipfailover instances don't see each other (e.g. multicast VRRP packets are blocked), both will have the MASTER state and both will carry the same VIP(s), that's and issue should be solved. 

### How to observe failover
Tips:
Set `OPENSHIFT_HA_PREEMPTION` to `nopreempt` to avoid the MASTER changing back in a short period. 
```
        - name: OPENSHIFT_HA_PREEMPTION
          value: nopreempt
```

1. Check the ipfailover pod logs, we can see the state change, e.g.
```
Thu May 27 04:21:24 2021: VRRP_Script(chk_ipfailover) succeeded
Thu May 27 04:21:24 2021: (ipfailover_VIP_1) Entering BACKUP STATE
Thu May 27 04:23:56 2021: (ipfailover_VIP_1) Backup received priority 0 advertisement
Thu May 27 04:23:57 2021: (ipfailover_VIP_1) Entering MASTER STATE

```


2. Check the owner of VIP, only the MASTER has the VIP. e.g.
```
$ oc rsh ipfailover-xxxxx
$ ip a | grep $VIP
```


### Create ipfailover resources in OpenShift 4.x

```

### create ServiceAccount ipfailover and update clusterrole.
$ oc create sa ipfailover
$ oc adm policy add-scc-to-user privileged -z ipfailover

### create ipfailover by Deployment
$ oc create -f https://raw.githubusercontent.com/lihongan/ipfailover/main/deploy-ipfailover.yaml

### create example Appliacations that monitored by ipfailover
$ oc create -f https://raw.githubusercontent.com/lihongan/ipfailover/main/web-server-rc.yaml

### check the logs of ipfailover pods
$ oc logs ipfailover-xxxx

### check the configuration of keepalived
$ oc rsh ipfailover-xxxx
$ ps -ef
$ cat /etc/keepalived/keepalived.conf

### Reconfigure the ipfailover by updating the ENV. 
### e.g. change the virtual IP to use 192.168.10.10 and 192.168.10.11. 
$ oc set env deploy/ipfailover OPENSHIFT_HA_VIRTUAL_IPS=192.168.10.10-11

```

more Environment variables please see:
https://docs.openshift.com/container-platform/3.11/admin_guide/high_availability.html#options-environment-variables


Note: ipfailover/VRRP uses multicast by default, but multicast is not allowed in many Cloud Platforms, so please use Unicast instead for your testing.

To use multicast, ensure below rule is added to iptables INPUT chain.
```
-A INPUT -d 224.0.0.18/32 -j ACCEPT
```

### use unicast

```
### Find the worker node's IP
$ oc get node -o wide

### Specify node's IP as unicast peers
$ oc set env deploy/ipfailover OPENSHIFT_HA_UNICAST_PEERS="10.0.152.208,10.0.183.251,10.0.219.207"

### Please skip below, seems default "false" works as well
$ oc set env deploy/ipfailover OPENSHIFT_HA_USE_UNICAST="true"                                         

### capture the vrrp unicast packets on nodes.
# tcpdump -i any vrrp -nn
09:11:35.701300 IP 10.0.183.251 > 10.0.219.207: VRRPv2, Advertisement, vrid 1, prio 132, authtype simple, intvl 1s, length 20
09:11:36.708204 IP 10.0.183.251 > 10.0.219.207: VRRPv2, Advertisement, vrid 1, prio 132, authtype simple, intvl 1s, length 20
```

note: 10.0.183.251 hosts the VIP (MASTER STATE)


see also: https://sites.google.com/site/springblend/home/deploy-ipfailover-router-with-unicast-mode
