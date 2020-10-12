# ipfailover

create ipfailover resources in OpenShift 4.x

$ oc adm policy add-scc-to-user privileged -z ipfailover

$ create ServiceAccount ipfailover

$ create deploymentconfig (3.x style) or deployment
