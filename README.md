# Exploring use of HCP and RHEL worker nodes

Exploring HCP and a use case of exclusively RHEL worker nodes.

## Background

### HCP Background

[HCP][1] is a feature of [MCE][2] which is a component of RHACM for managing cluster lifecycle. RHACM depends on MCE but the inverse is not true.

```bash
oc api-resources --api-group=multicluster.openshift.io
NAME                  SHORTNAMES   APIVERSION                     NAMESPACED   KIND
multiclusterengines   mce          multicluster.openshift.io/v1   false        MultiClusterEngine

oc api-resources --api-group=hypershift.openshift.io
NAME                  SHORTNAMES   APIVERSION                        NAMESPACED   KIND
awsendpointservices                hypershift.openshift.io/v1beta1   true         AWSEndpointService
hostedclusters        hc,hcs       hypershift.openshift.io/v1beta1   true         HostedCluster
hostedcontrolplanes   hcp,hcps     hypershift.openshift.io/v1beta1   true         HostedControlPlane
nodepools             np,nps       hypershift.openshift.io/v1beta1   true         NodePool
```

Diagrams

* [Standalone Control Plane](img/standalone-cp.png)
* [Hosted Control Plane](img/hosted-cp.png)

**HostedCluster Platform / Provider Types**

```bash
hypershift create cluster -h
Creates basic functional HostedCluster resources

Usage:
  hypershift create cluster [command]
Available Commands:
  agent       Creates basic functional HostedCluster resources on Agent
  aws         Creates basic functional HostedCluster resources on AWS
  azure       Creates basic functional HostedCluster resources on Azure
  kubevirt    Creates basic functional HostedCluster resources for KubeVirt platform
  none        Creates basic functional HostedCluster resources on None
  powervs     Creates basic functional HostedCluster resources on PowerVS PowerVS
  ...
```
 
These 3 platforms are of mosed interest for evaluation

 * [Agent][5] - Uses BMC to provision bare metal nodes
 * [None][4] - Leaves node creationg to user
 * Kubevirt - Uses OpenShift virtualization to provision nodes

Can none be used to form an exclusively RHEL worker node cluster?

### RHEL Worker Background

[Requirements][6]

* Removing a cluster node requires destroying the OS so you must use an exclusively dedicated machine
* Swap memory is disabled
* host must have OpenShift Container Platform subscription
* Base OS: RHEL 8.6, 8.7, or 8.8 with "Minimal" installation option.
* Min 1vCPU, 8GB RAM

# Tests

## [Kubevirt](#deploying-a-kubevirt-hostedcluster)

* [x] Deploy a platform=Kubervirt HostedCluster
* [ ] [Add RHEL Workers][6] to Kubevirt cluster

## None

* [ ] Deploy a [platform=None][4] HostedCluster
* [ ] [Add RHEL Workers][6] to None cluster

## Agent

* [ ] Deploy a [plaform=Agent][5] Hosted Cluster (_maybe_)



# Hosted Control Plane Prerequisites

## Enable Hosted Control Plane Feature

* Deploy MultiClusterEngine Operator - Installing RHACM can accomplish this

* Enable HyperShift Add-on to MultiClusterEngine. This will likely be on by default around RHACM 2.9 release.

```bash
# are these redundant? not sure.
oc patch MultiClusterEngine multiclusterengine \
  --type=json \
  -p='[{"op": "add", "path": "/spec/overrides/components/-","value":{"name":"hypershift-preview","enabled":true}},
       {"op": "add", "path": "/spec/overrides/components/-","value":{"name":"hypershift-local-hosting","enabled":true}}
  ]'
```

* Note these OCP versions are supported

```bash
oc extract configmap/supported-versions -n hypershift --to=- | jq
# supported-versions
{
  "versions": [
    "4.13",
    "4.12",
    "4.11"
  ]
}
```

## Enable Features Supporting Ingress to HCP

* Enable a load balancer solution

_A load balancer is required._
Install MetalLB Operator <https://docs.openshift.com/container-platform/4.13/networking/metallb/metallb-operator-install.html>, <https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html/clusters/cluster_mce_overview#hosting-service-cluster-configure-metallb-config>

```bash
oc apply -k prereqs/metallb/instance
namespace/metallb-system created
ipaddresspool.metallb.io/lab-192-168-4-224-b29 created
l2advertisement.metallb.io/l2advertisement created
metallb.metallb.io/metallb created
operatorgroup.operators.coreos.com/metallb created
subscription.operators.coreos.com/metallb created
```

```bash
oc api-resources --api-group=metallb.io
NAME                SHORTNAMES   APIVERSION           NAMESPACED   KIND
addresspools                     metallb.io/v1beta1   true         AddressPool
bfdprofiles                      metallb.io/v1beta1   true         BFDProfile
bgpadvertisements                metallb.io/v1beta1   true         BGPAdvertisement
bgppeers                         metallb.io/v1beta2   true         BGPPeer
communities                      metallb.io/v1beta1   true         Community
ipaddresspools                   metallb.io/v1beta1   true         IPAddressPool
l2advertisements                 metallb.io/v1beta1   true         L2Advertisement
metallbs                         metallb.io/v1beta1   true         MetalLB
```

# Tests

## Deploying a KubeVirt HostedCluster

* First test a kubevirt HCP cluster

```bash
export CLUSTER_NAME=example
export PULL_SECRET_PATH=/Users/dale/.credz/redhat/pull-secret/dbewley@redhat.com.json
export MEM="6Gi"
export CPU="2"
export WORKER_COUNT="2"

#  use the --release-image flag to set up the hosted cluster with a specific OpenShift Container Platform release.
# use the --base-domain flag avoid being in a subdomain of the hub cluster
hypershift create cluster kubevirt \
  --name $CLUSTER_NAME \
  --node-pool-replicas=$WORKER_COUNT \
  --pull-secret $PULL_SECRET_PATH \
  --memory $MEM \
  --cores $CPU
2023-08-29T13:27:46-07:00       INFO    Applied Kube resource   {"kind": "Namespace", "namespace": "", "name": "clusters"}
2023-08-29T13:27:46-07:00       INFO    Applied Kube resource   {"kind": "Secret", "namespace": "clusters", "name": "example-pull-secret"}
2023-08-29T13:27:46-07:00       INFO    Applied Kube resource   {"kind": "", "namespace": "clusters", "name": "example"}
2023-08-29T13:27:47-07:00       INFO    Applied Kube resource   {"kind": "Secret", "namespace": "clusters", "name": "example-etcd-encryption-key"}
2023-08-29T13:27:47-07:00       INFO    Applied Kube resource   {"kind": "NodePool", "namespace": "clusters", "name": "example"}

oc -n clusters-$CLUSTER_NAME get pods
NAME                                                  READY   STATUS    RESTARTS   AGE
capi-provider-c45497547-wqs9f                         1/1     Running   0          143m
catalog-operator-77bcd7d784-mv5nm                     2/2     Running   0          4m39s
certified-operators-catalog-86df6799ff-vwv4t          1/1     Running   0          4m36s
cluster-api-bdf46484-f2nks                            1/1     Running   0          143m
cluster-autoscaler-59885677c6-pz55w                   1/1     Running   0          4m31s
cluster-image-registry-operator-6759698f64-5lwmx      2/2     Running   0          4m38s
cluster-network-operator-9cfb4bc98-pslb8              1/1     Running   0          4m44s
cluster-node-tuning-operator-5f7d86c598-pqgzf         1/1     Running   0          4m43s
cluster-policy-controller-5885cc7dd8-29v6n            1/1     Running   0          4m45s
cluster-storage-operator-7c787bc7d4-rkb4z             1/1     Running   0          4m37s
cluster-version-operator-79674cfcc6-n7j6m             1/1     Running   0          4m45s
community-operators-catalog-5bffccb695-r7nhw          1/1     Running   0          4m36s
control-plane-operator-5c578df76c-qbnpm               1/1     Running   0          143m
csi-snapshot-controller-58f5bb94bb-5hxbn              1/1     Running   0          3m7s
csi-snapshot-controller-operator-d7c9c6f78-6hzss      1/1     Running   0          4m33s
csi-snapshot-webhook-548b85c5d-rrrqd                  1/1     Running   0          3m7s
dns-operator-84fc6c9fbb-hzg67                         1/1     Running   0          4m42s
etcd-0                                                2/2     Running   0          7m17s
hosted-cluster-config-operator-6964bdc4d6-bhqw7       1/1     Running   0          4m41s
ignition-server-57f569fc6-9vzbm                       1/1     Running   0          4m27s
ignition-server-proxy-7647d8f777-xccsj                1/1     Running   0          4m26s
importer-example-dt69f-rhcos                          1/1     Running   0          4m
importer-example-fdwg4-rhcos                          1/1     Running   0          4m
ingress-operator-56cc8596dd-4t9x7                     2/2     Running   0          4m42s
konnectivity-agent-5cc56f548-8qzwg                    1/1     Running   0          4m47s
konnectivity-server-58f448c7fd-9f5tn                  1/1     Running   0          4m47s
kube-apiserver-8585567465-764z6                       3/3     Running   0          6m50s
kube-controller-manager-597c59778d-ctmtl              1/1     Running   0          2m18s
kube-scheduler-795dfdc6c7-5tzh2                       1/1     Running   0          5m48s
kubevirt-cloud-controller-manager-75b776f4bb-blvvg    1/1     Running   0          4m40s
kubevirt-csi-controller-897bdcd6b-5tshd               4/4     Running   0          4m33s
machine-approver-54cfcfd56f-p76lz                     1/1     Running   0          4m30s
oauth-openshift-c755f7f6-bqx5j                        2/2     Running   0          3m6s
olm-operator-85d6f75886-j4x9s                         2/2     Running   0          4m39s
openshift-apiserver-5f44d7dfc5-s5rj7                  3/3     Running   0          5m48s
openshift-controller-manager-76b7bdcdf4-swcpt         1/1     Running   0          4m46s
openshift-oauth-apiserver-6fb779d6d4-k7nhs            2/2     Running   0          4m46s
openshift-route-controller-manager-676fcc6f88-vqlcv   1/1     Running   0          4m45s
packageserver-66657c84ff-6lxmz                        2/2     Running   0          4m39s
redhat-marketplace-catalog-54d4cc44c9-tz5xt           1/1     Running   0          4m36s
redhat-operators-catalog-769fc8f758-rk86z             1/1     Running   0          4m36s
```


```bash
hypershift create kubeconfig --name $CLUSTER_NAME > $CLUSTER_NAME-kubeconfig
```

```bash
oc get hostedcluster/$CLUSTER_NAME -n clusters
NAME      VERSION   KUBECONFIG                 PROGRESS    AVAILABLE   PROGRESSING   MESSAGE
example   4.13.10   example-admin-kubeconfig   Completed   True        False         The hosted control plane is available

oc get nodepool/$CLUSTER_NAME -n clusters
NAME      CLUSTER   DESIRED NODES   CURRENT NODES   AUTOSCALING   AUTOREPAIR   VERSION   UPDATINGVERSION   UPDATINGCONFIG   MESSAGE
example   example   2               2               False         False        4.13.10

oc get virtualmachineinstances -n clusters-$CLUSTER_NAME
NAME            AGE   PHASE     IP             NODENAME              READY
example-dt69f   84m   Running   10.130.4.166   hub-fpkcn-cnv-2q786   True
example-fdwg4   84m   Running   10.129.4.96    hub-fpkcn-cnv-6r66k   True
```

Success. See [screenshot](img/overview-screenshots.png)

### Adding a RHEL Node to KubeVirt HostedCluster

[Add a RHEL node][6] to example KubeVirt HostedCluster. For this test the playbook will be run from the RHEL node its self.

**TODO**
* Mount kubeconfig via configmap

The cloud-init [user data](rhel-vm/overlays/ocp-node/scripts/userData) will configure the RHEL node.

* Deploying a RHEL compute node as VM

```bash
oc apply -k rhel-vm/overlays/ocp-node
secret/cloudinitdisk-rhel-node created
networkattachmentdefinition.k8s.cni.cncf.io/vlan-1924 created
virtualmachine.kubevirt.io/rhel-node-1 created
nodenetworkconfigurationpolicy.nmstate.io/ens224-v1924 unchanged
```

#### FAIL

* First test while manually running the playbook to add node to HCP cluster failed due to lack of `machineconfigpool` API.

```
[ansible@rhel-node-1 openshift-ansible]$ ansible-playbook -c local -i /home/ansible/inventory/hosts playbooks/scaleup.yml

...
TASK [openshift_node : Retrieve rendered-worker name] *********************************************************************************************************
FAILED - RETRYING: [rhel-node-1.lab.bewley.net -> localhost]: Retrieve rendered-worker name (3 retries left).
FAILED - RETRYING: [rhel-node-1.lab.bewley.net -> localhost]: Retrieve rendered-worker name (2 retries left).
FAILED - RETRYING: [rhel-node-1.lab.bewley.net -> localhost]: Retrieve rendered-worker name (1 retries left).
fatal: [rhel-node-1.lab.bewley.net -> localhost]: FAILED! => {"attempts": 3, "changed": false, "cmd": ["oc", "get", "machineconfigpool", "worker", "--kubeconfi
g=/home/ansible/example-kubeconfig", "--output=jsonpath={.status.configuration.name}"], "delta": "0:00:00.521666", "end": "2023-08-31 12:00:21.349878", "msg":
"non-zero return code", "rc": 1, "start": "2023-08-31 12:00:20.828212", "stderr": "error: the server doesn't have a resource type \"machineconfigpool\"", "stde
rr_lines": ["error: the server doesn't have a resource type \"machineconfigpool\""], "stdout": "", "stdout_lines": []}

[ansible@rhel-node-1 openshift-ansible]$ oc api-resources --api-group=machineconfiguration.openshift.io
NAME   SHORTNAMES   APIVERSION   NAMESPACED   KIND
[ansible@rhel-node-1 openshift-ansible]$
```

There is no machine config operator present.

```
[ansible@rhel-node-1 openshift-ansible]$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
console                                    4.13.10   True        False         False      22h
csi-snapshot-controller                    4.13.10   True        False         False      41h
dns                                        4.13.10   True        False         False      39h
image-registry                             4.13.10   True        False         False      39h
ingress                                    4.13.10   True        False         False      22h
insights                                   4.13.10   True        False         False      40h
kube-apiserver                             4.13.10   True        False         False      41h
kube-controller-manager                    4.13.10   True        False         False      41h
kube-scheduler                             4.13.10   True        False         False      41h
kube-storage-version-migrator              4.13.10   True        False         False      40h
monitoring                                 4.13.10   True        False         False      39h
network                                    4.13.10   True        False         False      40h
node-tuning                                4.13.10   True        False         False      40h
openshift-apiserver                        4.13.10   True        False         False      41h
openshift-controller-manager               4.13.10   True        False         False      41h
openshift-samples                          4.13.10   True        False         False      39h
operator-lifecycle-manager                 4.13.10   True        False         False      41h
operator-lifecycle-manager-catalog         4.13.10   True        False         False      41h
operator-lifecycle-manager-packageserver   4.13.10   True        False         False      41h
service-ca                                 4.13.10   True        False         False      40h
storage                                    4.13.10   True        False         False      41h
```

* Approve CSRs for new node

## Deploying a None HostedCluster
### Adding a RHEL Node

# Questions

* What is roadmap for HCP? When is it GA? [FAQ][8]
* How to upgrade HCP Clusters?
** <https://docs.openshift.com/container-platform/4.13/hosted_control_planes/hcp-managing.html#updating-node-pools-for-hcp_hcp-managing> 

```bash
oc -n clusters patch hostedcluster/example --patch '{"spec":{"release":{"image": "ocp-pull-spec"}}}' --type=merge
```

# References


[1]: <https://docs.openshift.com/container-platform/4.13/architecture/mce-overview-ocp.html>
[2]: <https://docs.openshift.com/container-platform/4.13/architecture/control-plane.html>
[3]: <https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.8/html/clusters/cluster_mce_overview>
[4]: <https://hypershift-docs.netlify.app/how-to/none/create-none-cluster/>
[5]: <https://hypershift-docs.netlify.app/how-to/agent/create-agent-cluster/>
[6]: <https://docs.openshift.com/container-platform/4.13/machine_management/adding-rhel-compute.html> "Adopting RHEL Nodes"
[7]: <https://docs.openshift.com/container-platform/4.13/updating/updating-cluster-rhel-compute.html> "Updating RHEL Nodes"
[8]:  <https://docs.google.com/document/d/1cvvFShz3AZ24VAHZvcEZImmiqn3sQPpHBvIGvgMdhCo/edit#> "HCP FAQ"
[9]: <https://docs.google.com/document/d/1H_hmY_r1dQjAv163OIWeAQpkaULOYHSNzWvEh7_TkpU/edit#heading=h.38yju8gvhgz4a> "MCE FAQ"
[10]: <https://issues.redhat.com/browse/OCPSTRAT-9> "Jira tracking Self Managed HCP"
[11]: <https://pp.engineering.redhat.com/hypershift/overview/> "Engr HyperShift"
