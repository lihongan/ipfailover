# ipfailover

create ipfailover resources in OpenShift 4.x

$ oc adm policy add-scc-to-user privileged -z ipfailover

$ create ServiceAccount ipfailover

$ create deploymentconfig (3.x style) or deployment

ensure a rule to iptables chain INPUT to accept 224.0.0.28 multicast packets

-A INPUT -d 224.0.0.18/32 -j ACCEPT
