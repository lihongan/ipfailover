# ipfailover

create ipfailover resources in OpenShift 4.x

```

### create ServiceAccount ipfailover and update clusterrole.
$ oc create sa ipfailover
$ oc adm policy add-scc-to-user privileged -z ipfailover

### create ipfailover DeploymentConfig (v3.x style)
$ oc create -f https://raw.githubusercontent.com/lihongan/ipfailover/main/dc-ipfailover.yaml

```
ensure a rule to iptables chain INPUT to accept 224.0.0.28 multicast packets
```
-A INPUT -d 224.0.0.18/32 -j ACCEPT
```

### use unicast

```
# oc set env dc/ipfailover OPENSHIFT_HA_UNICAST_PEERS="10.0.152.208,10.0.183.251,10.0.219.207"     ## eth0 interface IP

# oc set env dc/ipfailover OPENSHIFT_HA_USE_UNICAST="true"                                         ## useless, default "false" works as well

# tcpdump -i any vrrp -nn
09:11:35.701300 IP 10.0.183.251 > 10.0.219.207: VRRPv2, Advertisement, vrid 1, prio 132, authtype simple, intvl 1s, length 20
09:11:36.708204 IP 10.0.183.251 > 10.0.219.207: VRRPv2, Advertisement, vrid 1, prio 132, authtype simple, intvl 1s, length 20
```

note: 10.0.183.251 hosts the VIP (MASTER STATE)


see also: https://sites.google.com/site/springblend/home/deploy-ipfailover-router-with-unicast-mode
