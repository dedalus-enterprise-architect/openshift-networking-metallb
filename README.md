# OpenShift networking with MetalLB

This project collects some procedures on how to setup a MetalLB deployment on OpenShift based on the following minimum requirements:

* 1 subnet available to assign virtual IPs to the load balancer service (must not overlap with the hosts network)
* availability of docker image ```quay.io/metallb/controller:v0.13.5```
* availability of docker image ```quay.io/metallb/speaker:v0.13.5```
* OpenShift client utility: ```oc```
* OpenShift cluster admin and application namespace admin privileges
* Git project local clone

Explore files provided in this project:

* ```manifests/metallb-native.yaml``` : MetalLB deployment definition
* ```manifests/metallb-addresspool.yaml``` : _IPAddressPool_ example definition
* ```manifests/metallb-l2advertise.yaml``` : _L2Advertisement_ definition
* ```examples/tcp-single-service.yaml``` : example provisioning 1 virtual IP balancing a single TCP port
* ```examples/tcp-multiple-service.yaml``` : example provisioning 1 virtual IP balancing multiple TCP ports

## MetalLB resources: setup

> WARNING: cluster admin role is required to proceed on this section.

Deploy MetalLB resources from manifest:

```bash
oc apply -f manifests/metallb-native.yaml
```

Note that:

* a namespace ```metallb-system``` is created
* a _nodeSelector_ ```node-role.kubernetes.io/worker``` has been applied to the _DaemonSet_ ```speaker``` to deploy speaker pods only on worker nodes
* keys ```spec.template.spec.securityContext.runAsUser``` and ```spec.template.spec.securityContext.fsGroup``` have been removed for OCP compliance <https://metallb.universe.tf/installation/clouds/#metallb-on-openshift-ocp>

Assign the following policy

```bash
oc adm policy add-scc-to-user privileged -n metallb-system -z speaker
```

Provision one or more _IPAddressPool_ resources from which MetalLB will assign virtual IPs for the load balanced services by customizing the provided yaml according to your needs:

```bash
oc apply -f manifests/metallb-addresspool.yaml
```

Provision one or more _L2Advertisement_ resources to announce networks described in the address pools:

```bash
oc apply -f manifests/metallb-addresspool.yaml
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
