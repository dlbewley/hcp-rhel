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

oc api-resources --api-group=agent-install.openshift.io
NAME                            SHORTNAMES   APIVERSION                           NAMESPACED   KIND
agentclassifications                         agent-install.openshift.io/v1beta1   true         AgentClassification
agents                                       agent-install.openshift.io/v1beta1   true         Agent
agentserviceconfigs                          agent-install.openshift.io/v1beta1   false        AgentServiceConfig
hypershiftagentserviceconfigs   hasc         agent-install.openshift.io/v1beta1   true         HypershiftAgentServiceConfig
infraenvs                                    agent-install.openshift.io/v1beta1   true         InfraEnv
nmstateconfigs                               agent-install.openshift.io/v1beta1   true         NMStateConfig
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
 
These 3 platforms are of most interest for evaluation:

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

# Test Cases

## Kubevirt HCP

[Results](#deploying-a-kubevirt-hostedcluster)

* [x] Deploy a [platform=Kubervirt][13] HostedCluster
  * Success
* [-] Add a [RHEL VM](rhel-vm/overlays/hcp-node) to `Kubevirt` cluster
  * **Failure** - Not possible, due to lack of Machine Config Operator on HostedClusters. [CAPI][14] is used instead.

## None HCP

Results - Did not test

* [-] Deploy a [platform=None][4] HostedCluster
  * TBD
* [-] Add [RHEL VM](rhel-vm/overlays/hcp-node) Worker to `None` cluster
  * TBD

## Agent HCP

* Results - Did not test

## Standalone Control Plane OCP

[Results](#test-adding-rhel-node-to-openshift-cluster)

* [x] Deploy a OCP 4.13 platform=vSphere cluster
* [x] Deploy a [RHEL9 VM](rhel-vm/overlays/ocp-node)
* [ ] Add as worker to OCP
  * **Workaround** OCP 4.13 docs list RHEL8 yum repository names which lack `crio-tools` and more importantly does not match the cluster OS.
  * Bug filed <https://issues.redhat.com/browse/OCPBUGS-18557>
  * **Workaround** The playbook does not account for RHEL9. Source <https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_node/tasks/install.yml#L2-L6>
  * Bug filed <https://issues.redhat.com/browse/OCPBUGS-18558>
  * **Failure** Bootstrap ignition attempts to fetch from api-int which is not resolvable.
  * **Workaround** Force `api` endpoint
  * **Success** Node joins cluster and is Ready
* [ ] Schedule workload to RHEL worker
  * **Failure** Node joins cluster and is Ready but network is not fully initialized so node is tainted

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

# Test Runs

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

**Success.** See [screenshot](img/overview-screenshots.png)

* Obtain credentials to talk to hosted control plane

```bash
hypershift create kubeconfig --name $CLUSTER_NAME > $CLUSTER_NAME-kubeconfig
# or
oc extract -n clusters secret/${CLUSTER_NAME}-admin-kubeconfig --to=- > ${CLUSTER_NAME}-kubeconfig
# or
oc extract -n clusters-${CLUSTER_NAME} secret/admin-kubeconfig --to=- > ${CLUSTER_NAME}-kubeconfig

oc get clusterversion --kubeconfig=${CLUSTER_NAME}-kubeconfig
```

### Adding a RHEL Node to KubeVirt HostedCluster

[Add a RHEL node][6] to example KubeVirt HostedCluster. For this test the playbook will be run from the RHEL node its self.


* Deploying a RHEL compute node as [a VM](rhel-vm/overlays/hcp-node/kustomization.yaml)

The cloud-init [user data](rhel-vm/overlays/hcp-node/scripts/userData) will configure the RHEL node.

```bash
oc apply -k rhel-vm/overlays/hcp-node
secret/cloudinitdisk-rhel-node created
networkattachmentdefinition.k8s.cni.cncf.io/vlan-1924 created
virtualmachine.kubevirt.io/rhel-node-1 created
nodenetworkconfigurationpolicy.nmstate.io/ens224-v1924 unchanged
```

#### FAIL This is not supported

* Running the playbook to [add a RHEL node][6] to HCP cluster failed due to lack of `machineconfigpool` API.

_The Machine Config Operator is not present in the control plane of a HostedCluster, so HyperShift and RHEL nodes are incompatible._

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

Source <https://github.com/openshift/openshift-ansible/blob/master/roles/openshift_node/tasks/install.yml#L2-L6>

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

# Questions

* How to [upgrade HCP Clusters][12]?

```bash
oc -n clusters patch hostedcluster/example --patch '{"spec":{"release":{"image": "ocp-pull-spec"}}}' --type=merge
```

# Test adding RHEL Node to OpenShift Cluster

* Playbook failed

```
No match for argument: cri-tools
Error: Unable to find a match: cri-tools
```

That was because I was using RHEL8 per [the docs][6] however OCP 4.13 is based on RHEL 9.2.

* Workaround: Update repo names in [userData](rhel-vm/overlays/ocp-node/scripts/userData) 

* Bug Filed <https://issues.redhat.com/browse/OCPBUGS-18557>

* Retry with above workaround.

* **Failure** of playbook which hints at a failure to update the playbook to support RHEL 9.

```
TASK [openshift_node : Install openshift packages] *****************************
fatal: [rhel-node-1.lab.bewley.net]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: {{ (openshift_node_packages + openshift_node_support_packages) | join(',') }}: {{ openshift_node_support_packages_base + openshift_node_support_packages_by_os_major_version[ansible_distribution_major_version] + openshift_node_support_packages_by_arch[ansible_architecture] }}: 'dict object' has no attribute '9'. 'dict object' has no attribute '9'. {{ openshift_node_support_packages_base + openshift_node_support_packages_by_os_major_version[ansible_distribution_major_version] + openshift_node_support_packages_by_arch[ansible_architecture] }}: 'dict object' has no attribute '9'. 'dict object' has no attribute '9'. {{ (openshift_node_packages + openshift_node_support_packages) | join(',') }}: {{ openshift_node_support_packages_base + openshift_node_support_packages_by_os_major_version[ansible_distribution_major_version] + openshift_node_support_packages_by_arch[ansible_architecture] }}: 'dict object' has no attribute '9'. 'dict object' has no attribute '9'. {{ openshift_node_support_packages_base + openshift_node_support_packages_by_os_major_version[ansible_distribution_major_version] + openshift_node_support_packages_by_arch[ansible_architecture] }}: 'dict object' has no attribute '9'. 'dict object' has no attribute '9'\n\nThe error appears to be in '/usr/share/ansible/openshift-ansible/roles/openshift_node/tasks/install.yml': line 91, column 5, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n- block:\n  - name: Install openshift packages\n    ^ here\n"}
```

* Yep. No entry for 9 here: <https://github.com/openshift/openshift-ansible/blob/release-4.13/roles/openshift_node/defaults/main.yml#L78-L87>

* **Workaround** This patch works around above:

```diff
--- openshift-ansible/roles/openshift_node/defaults/main.yml.orig       2023-07-14 03:07:17.000000000 -0400
+++ openshift-ansible/roles/openshift_node/defaults/main.yml    2023-09-01 20:06:53.986175752 -0400
@@ -85,6 +85,7 @@
   "8":
     - openvswitch2.17
     - policycoreutils-python-utils
+  "9": []

 openshift_conflict_packages:
   - openvswitch
```

* Bug filed <https://issues.redhat.com/browse/OCPBUGS-18558>

* Retry with above workaround.

* **Failed** to download the machineconfig. 

```

TASK [openshift_node : Fetch bootstrap ignition file locally] **************************************************************************************
FAILED - RETRYING: [rhel-node-1.lab.bewley.net]: Fetch bootstrap ignition file locally (60 retries left).
...
fatal: [rhel-node-1.lab.bewley.net]: FAILED! => {"attempts": 60, "changed": false, "elapsed": 0, "msg": "Status code was -1 and not [200]: Request failed: <urlopen error [Errno -2] Name or service not known>", "redirected": false, "status": -1, "url": "https://api-int.hub.lab.bewley.net:22623/config/worker"}
```

Need to investigate ignition fetching from api-int which is not resolvable.

  * Task is here <https://github.com/openshift/openshift-ansible/blob/release-4.13/roles/openshift_node/tasks/config.yml#L70-L83>
  * Vars are here <https://github.com/openshift/openshift-ansible/blob/release-4.13/roles/openshift_node/defaults/main.yml#L6-L10>

  * **Workaround** Apply this patch to stop forcing api-int.<cluster-domain>, and force api.<cluster-domain>.

```diff
--- openshift-ansible/roles/openshift_node/defaults/main.yml.orig       2023-07-14 03:07:17.000000000 -0400
+++ openshift-ansible/roles/openshift_node/defaults/main.yml    2023-09-05 14:16:31.487090591 -0400
@@ -6,7 +6,7 @@
 openshift_node_kubeconfig_path: "{{ openshift_kubeconfig_path | default('~/.kube/config') | expanduser | realpath }}"
 openshift_node_kubeconfig: "{{ lookup('file', openshift_node_kubeconfig_path) | from_yaml }}"
 openshift_node_bootstrap_port: 22623
-openshift_node_bootstrap_server: "{{ openshift_node_kubeconfig.clusters.0.cluster.server.split(':')[0:-1] | join(':') | regex_replace('://api-int|://api', '://api-int') }}:{{ openshift_node_bootstrap_port }}"
+openshift_node_bootstrap_server: "{{ openshift_node_kubeconfig.clusters.0.cluster.server.split(':')[0:-1] | join(':') | regex_replace('://api-int|://api', '://api') }}:{{ openshift_node_bootstrap_port }}"
 openshift_node_bootstrap_endpoint: "{{ openshift_node_bootstrap_server }}/config/{{ openshift_node_machineconfigpool }}"
 openshift_package_directory: '/tmp/openshift-ansible-packages'

@@ -85,6 +85,7 @@
   "8":
     - openvswitch2.17
     - policycoreutils-python-utils
+  "9": []

 openshift_conflict_packages:
   - openvswitch
```

* Retry with above workaround got farther, but failed to apply bootstrap ignition due to missing openvswitch service.

```
TASK [openshift_node : Apply ignition manifest] **********************************************
...
 "F0905 14:23:42.499261   39224 start.go:104] error enabling units: Failed to enable unit: Unit file openvswitch.service does not exist."]}
```

  * **Workaround** Update diff to include openvswitch pkgs compatible with OCP 4.13 and RHEL9

```diff
--- /usr/share/ansible/openshift-ansible/roles/openshift_node/defaults/main.yml.orig    2023-07-14 03:07:17.000000000 -0400
+++ /usr/share/ansible/openshift-ansible/roles/openshift_node/defaults/main.yml 2023-09-05 14:32:33.161233539 -0400
@@ -6,7 +6,7 @@
 openshift_node_kubeconfig_path: "{{ openshift_kubeconfig_path | default('~/.kube/config') | expanduser | realpath }}"
 openshift_node_kubeconfig: "{{ lookup('file', openshift_node_kubeconfig_path) | from_yaml }}"
 openshift_node_bootstrap_port: 22623
-openshift_node_bootstrap_server: "{{ openshift_node_kubeconfig.clusters.0.cluster.server.split(':')[0:-1] | join(':') | regex_replace('://api-int|://api', '://api-int') }}:{{ openshift_node_bootstrap_port }}"
+openshift_node_bootstrap_server: "{{ openshift_node_kubeconfig.clusters.0.cluster.server.split(':')[0:-1] | join(':') | regex_replace('://api-int|://api', '://api') }}:{{ openshift_node_bootstrap_port }}"
 openshift_node_bootstrap_endpoint: "{{ openshift_node_bootstrap_server }}/config/{{ openshift_node_machineconfigpool }}"
 openshift_package_directory: '/tmp/openshift-ansible-packages'

@@ -85,6 +85,9 @@
   "8":
     - openvswitch2.17
     - policycoreutils-python-utils
+  "9":
+    - openvswitch3.1
+    - policycoreutils-python-utils

 openshift_conflict_packages:
   - openvswitch
```

* Soft **FAIL** Rerun with above workaround. Mostly worked but the playbook reboots the node. That failed since it was run from localhost.

```
[ansible@rhel-node-1 openshift-ansible]$ cd /usr/share/ansible/openshift-ansible
[ansible@rhel-node-1 openshift-ansible]$ ansible-playbook -c local -i /home/ansible/inventory/hosts playbooks/scaleup.yml
...
TASK [openshift_node : Pull MCD image] ************************************************************************************************************************
changed: [rhel-node-1.lab.bewley.net]

TASK [openshift_node : Apply ignition manifest] ***************************************************************************************************************
changed: [rhel-node-1.lab.bewley.net]

TASK [openshift_node : Remove temp directory] *****************************************************************************************************************
changed: [rhel-node-1.lab.bewley.net]

TASK [openshift_node : Reboot the host and wait for it to come back] ******************************************************************************************
fatal: [rhel-node-1.lab.bewley.net]: FAILED! => {"changed": false, "elapsed": 0, "msg": "Running reboot with local connection would reboot the control node.", "rebooted": false}

TASK [openshift_node : fail] **********************************************************************************************************************************
fatal: [rhel-node-1.lab.bewley.net]: FAILED! => {"changed": false, "msg": "Ignition apply failed"}

PLAY RECAP ****************************************************************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
rhel-node-1.lab.bewley.net : ok=47   changed=20   unreachable=0    failed=1    skipped=6    rescued=1    ignored=0

[ansible@rhel-node-1 openshift-ansible]$ uptime
 14:38:01 up 3 days, 19:43,  1 user,  load average: 2.67, 1.40, 0.71
```


* **Workaround** Rebooted host by hand and did not re-run playbook nor check for any missing subsequent tasks.

Manually approved 3 CSRs

```
 oc get csr
NAME        AGE    SIGNERNAME                                    REQUESTOR                                                                   REQUESTEDDURATION   CONDITION
csr-6qgqt   4m5s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
csr-lv56x   19m    kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Approved,Issued
csr-qf8ks   22s    kubernetes.io/kubelet-serving                 system:node:rhel-node-1                                                     <none>              Pending
```

After approving those, the node shows up as Not Ready and journal complains about networking problem for a few minutes until finally becoming Ready.

Success in that now the node is Ready, but failure in that it is not fully schedullable

```
$ oc get node rhel-node-1 -o wide
NAME          STATUS   ROLES    AGE   VERSION           INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                              KERNEL-VERSION                 CONTAINER-RUNTIME
rhel-node-1   Ready    worker   36m   v1.26.7+0ef5eae   192.168.4.191   <none>        Red Hat Enterprise Linux 9.2 (Plow)   5.14.0-284.28.1.el9_2.x86_64   cri-o://1.26.4-3.rhaos4.13.git615a02c.el9

$ oc get nodes -o custom-columns="NODE:.metadata.name,OS-IMAGE:.status.nodeInfo.osImage,IP:.status.addresses[0]"
NODE                       OS-IMAGE                                                       IP
hub-fpkcn-cnv-2q786        Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.103 type:InternalIP]
hub-fpkcn-cnv-6r66k        Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.128 type:InternalIP]
hub-fpkcn-cnv-f9gcp        Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.196 type:InternalIP]
hub-fpkcn-cnv-qb67t        Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.137 type:InternalIP]
hub-fpkcn-master-0         Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.195 type:InternalIP]
hub-fpkcn-master-1         Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.138 type:InternalIP]
hub-fpkcn-master-2         Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.182 type:InternalIP]
hub-fpkcn-store-1-s8cz5    Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.197 type:InternalIP]
hub-fpkcn-store-2-n5vvm    Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.193 type:InternalIP]
hub-fpkcn-store-3-7hbz8    Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.39 type:InternalIP]
hub-fpkcn-worker-0-nrc9w   Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.47 type:InternalIP]
hub-fpkcn-worker-0-zkntn   Red Hat Enterprise Linux CoreOS 413.92.202308091852-0 (Plow)   map[address:192.168.4.180 type:InternalIP]
rhel-node-1                Red Hat Enterprise Linux 9.2 (Plow)                            map[address:192.168.4.191 type:InternalIP]

$ oc get vmis -n ocp-rhel -o wide
NAME          AGE     PHASE     IP              NODENAME              READY   LIVE-MIGRATABLE   PAUSED
rhel-node-1   3d21h   Running   192.168.4.191   hub-fpkcn-cnv-qb67t   True    True

$ oc get pods -n ocp-rhel -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP            NODE                  NOMINATED NODE   READINESS GATES
virt-launcher-rhel-node-1-npfn8   1/1     Running   0          3d21h   10.131.5.99   hub-fpkcn-cnv-qb67t   <none>           1/1
```

**FAIL** Even though node is status Ready it is tainted to reject workloads. TBD

```
oc describe node rhel-node-1
Name:               rhel-node-1
Roles:              worker
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=rhel-node-1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/worker=
                    node.openshift.io/os_id=rhel
Annotations:        alpha.kubernetes.io/provided-node-ip: 192.168.4.191
                    k8s.ovn.org/host-addresses: ["192.168.4.191"]
                    k8s.ovn.org/l3-gateway-config:
                      {"default":{"mode":"shared","interface-id":"br-ex_rhel-node-1","mac-address":"02:f8:4a:00:00:20","ip-addresses":["192.168.4.191/24"],"ip-a...
                    k8s.ovn.org/node-chassis-id: ac10dd01-fee1-4792-ae81-c92fe4afdc30
                    k8s.ovn.org/node-gateway-router-lrp-ifaddr: {"ipv4":"100.64.0.14/16"}
                    k8s.ovn.org/node-mgmt-port-mac-address: d2:6d:e0:30:4e:a2
                    k8s.ovn.org/node-primary-ifaddr: {"ipv4":"192.168.4.191/24"}
                    k8s.ovn.org/node-subnets: {"default":["10.128.6.0/23"]}
                    machineconfiguration.openshift.io/controlPlaneTopology: HighlyAvailable
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 05 Sep 2023 13:15:20 -0700
Taints:             node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
                    UpdateInProgress:PreferNoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  rhel-node-1
  AcquireTime:     <unset>
  RenewTime:       Tue, 05 Sep 2023 13:42:52 -0700
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 05 Sep 2023 13:42:40 -0700   Tue, 05 Sep 2023 13:15:20 -0700   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 05 Sep 2023 13:42:40 -0700   Tue, 05 Sep 2023 13:15:20 -0700   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 05 Sep 2023 13:42:40 -0700   Tue, 05 Sep 2023 13:15:20 -0700   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Tue, 05 Sep 2023 13:42:40 -0700   Tue, 05 Sep 2023 13:19:59 -0700   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.4.191
  Hostname:    rhel-node-1
Capacity:
  cpu:                1
  ephemeral-storage:  30728172Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             7841692Ki
  pods:               250
Allocatable:
  cpu:                500m
  ephemeral-storage:  27245341445
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             6690716Ki
  pods:               250
System Info:
  Machine ID:                 65bdb56817b1596fb2526befa9e1a29a
  System UUID:                65bdb568-17b1-596f-b252-6befa9e1a29a
  Boot ID:                    8d39b128-fc04-4fe1-9107-72775385010c
  Kernel Version:             5.14.0-284.28.1.el9_2.x86_64
  OS Image:                   Red Hat Enterprise Linux 9.2 (Plow)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  cri-o://1.26.4-3.rhaos4.13.git615a02c.el9
  Kubelet Version:            v1.26.7+0ef5eae
  Kube-Proxy Version:         v1.26.7+0ef5eae
Non-terminated Pods:          (7 in total)
  Namespace                   Name                            CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                            ------------  ----------  ---------------  -------------  ---
  openshift-cnv               bridge-marker-455pj             10m (2%)      0 (0%)      15Mi (0%)        0 (0%)         27m
  openshift-dns               node-resolver-npm5t             5m (1%)       0 (0%)      21Mi (0%)        0 (0%)         27m
  openshift-multus            multus-m992h                    10m (2%)      0 (0%)      65Mi (0%)        0 (0%)         27m
  openshift-multus            network-metrics-daemon-7m4fk    20m (4%)      0 (0%)      120Mi (1%)       0 (0%)         27m
  openshift-ovn-kubernetes    ovnkube-node-mmzf5              50m (10%)     0 (0%)      660Mi (10%)      0 (0%)         27m
  openshift-vsphere-infra     coredns-rhel-node-1             200m (40%)    0 (0%)      400Mi (6%)       0 (0%)         27m
  openshift-vsphere-infra     keepalived-rhel-node-1          200m (40%)    0 (0%)      400Mi (6%)       0 (0%)         26m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                495m (99%)    0 (0%)
  memory             1681Mi (25%)  0 (0%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:
  Type     Reason                Age                 From             Message
  ----     ------                ----                ----             -------
  Warning  ErrorReconcilingNode  27m                 controlplane     nodeAdd: error adding node "rhel-node-1": could not find "k8s.ovn.org/node-subnets" annotation
  Normal   RegisteredNode        27m                 node-controller  Node rhel-node-1 event: Registered Node rhel-node-1 in Controller
  Warning  ErrorReconcilingNode  23m (x22 over 27m)  controlplane     [k8s.ovn.org/node-chassis-id annotation not found for node rhel-node-1, macAddress annotation not found for node "rhel-node-1" , k8s.ovn.org/l3-gateway-config annotation not found for node "rhel-node-1"]
  Warning  ErrorReconcilingNode  23m                 controlplane     error creating gateway for node rhel-node-1: failed to init shared interface gateway: failed to create MAC Binding for dummy nexthop rhel-node-1: error getting datapath GR_rhel-node-1: object not found
```

Maybe failure due to mixing platform=vsphere and a non-vmware guest?

Result: Not going to pursue this any further at this time.

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
[12]: <https://docs.openshift.com/container-platform/4.13/hosted_control_planes/hcp-managing.html>
[13]: <https://hypershift-docs.netlify.app/how-to/kubevirt/create-kubevirt-cluster/>
[14]: <https://cluster-api.sigs.k8s.io/>
