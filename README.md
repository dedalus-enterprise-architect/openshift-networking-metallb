# OpenShift networking with MetalLB

This project collects some procedures on how to setup a MetalLB deployment on OpenShift based on the following minimum requirements:

* RedHat Operators Catalog available
* MetalLB Operator
* 1 subnet available to assign virtual IPs to the load balancer service (must not overlap with the hosts network)
* OpenShift client utility: ```oc```
* OpenShift cluster admin and application namespace admin privileges
* Git project local clone

Explore files provided in this project:

* ```deploy/metallb-namespace.yaml``` : _namespace_ definition
* ```deploy/metallb-operator.yaml``` : _OperatorGroup_ definition
* ```deploy/metallb-operator-sub.yaml``` : _Subscription_ definition
* ```deploy/metallb-instance.yaml``` : _MetalLB_ instance definition
* ```deploy/metallb-addresspool.yaml``` : example _IPAddressPool_ definition
* ```deploy/metallb-l2advertise.yaml``` : _L2Advertise_ definition
* ```examples/tcp-single-service.yaml``` : example provisioning 1 virtual IP balancing a single TCP port
* ```examples/tcp-multiple-service.yaml``` : example provisioning 1 virtual IP balancing multiple TCP ports

## MetalLB resources: setup

> WARNING: cluster admin role is required to proceed on this section.

Deploy MetalLB namespace:

```bash
oc apply -f deploy/metallb-namespace.yaml
```

Deploy MetalLB _OperatorGroup_ and its _Subscription:

```bash
oc apply -f deploy/metallb-operator.yaml
oc apply -f deploy/metallb-operator-sub.yaml
```

Check that the _InstallPlan_ has been approved

```bash
oc get installplan -n metallb-system
 
NAME            CSV                                     APPROVAL    APPROVED
install-xxxxx   metallb-operator.v4.xxxx-xxxxxxxxxxxx   Automatic   true
```

Check that the Operator installation is successful:

```bash
oc get clusterserviceversion -n metallb-system -o custom-columns=Name:.metadata.name,Phase:.status.phase
 
Name                                    Phase
metallb-operator.v4.xxxx-xxxxxxxxxxxx   Succeeded
```

Deploy a MetalLB instance:

```bash
oc apply -f deploy/metallb-instance.yaml
```


Note that:

* a _nodeSelector_ ```node-role.kubernetes.io/worker``` has been applied to the _DaemonSet_ ```speaker``` to deploy speaker pods only on worker nodes

Check the status of the _Controller_ and of the _Speaker_ pods (should have a pod for every worker node):

```bash
oc get deployment -n metallb-system controller
 
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
controller   1/1     1            1           61s
```

```bash
oc get daemonset -n metallb-system speaker
 
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                            AGE
speaker   2         2         2       2            2           kubernetes.io/os=linux,node-role.kubernetes.io/worker=   2m41s
```

Provision one or more _IPAddressPool_ resources from which MetalLB will assign virtual IPs for the load balanced services by customizing the provided yaml according to your needs:

```bash
oc apply -f deploy/metallb-addresspool.yaml
```

Provision one or more _L2Advertisement_ resources to announce networks described in the address pools:

```bash
oc apply -f deploy/metallb-addresspool.yaml
```

## Load balanced services: setup

> WARNING: application namespace admin role is required to proceed on this section.

Provision a _Service_ of type _LoadBalancer_ for every application that has to be load-balanced, as shown in the following examples

* 1 virtual IP balancing a single TCP port; virtual IP is assigned manually: ```examples/tcp-single-service.yaml```
* 1 virtual IP balancing multiple TCP ports; virtual IPs are assigned manually: ```examples/tcp-multiple-service.yaml```

Tips:

* remove key ```loadBalancerIP``` to get an auto-assigned IP from ```metallb.universe.tf/address-pool``` if available
* key ```externalTrafficPolicy```: can be set to ```Cluster``` or ```Local```, refer to K8S docs about the difference <https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip>
* the Service API definition (<https://v1-23.docs.kubernetes.io/docs/concepts/services-networking/service/>) doesn't support ranges as port values 
* the annotation ```metallb.universe.tf/allow-shared-ip``` must be added when a virtual IP is shared between 2 or more services, and its value must be the same for all the sharing services
