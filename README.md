# vcluster-rancher-op

First class vCluster support in Rancher

## Description

Deploying a vCluster in rancher should provide the same great experience that provisioning any other cluster in Rancher would. vCluster Rancher Operator automatically integrates vCluster deployments with Rancher. This integration combines the great cluster management of Rancher with the security, cost-effectiveness, and speed of vClusters.

**Features**
* Automatically creates a Rancher Cluster that corresponds to each virtual cluster
* Rancher users do not need cluster management permissions. The operator will create the Rancher cluster.
* Project owners and project members of the virtual cluster's project, as well as the cluster owner, will be added to the new Rancher cluster as cluster owners.

**Contents**
- [Installation](#installation)
- [How to Use](#how-to-use)
- [Technical Details](#technical-details)
- [Development](#development)

## Installation

To install, add the loft charts repository to Rancher, and then install this vCluster Rancher Operator chart in the local cluster.
1. Select the local cluster in the Rancher clusters overview page.
2. In the sidebar, navigate to "Apps" -> "Repositories".
3. Select "Create".
4. Set the name to any value and the Index URL to `https://loft.sh/charts`.
5. In the sidebar, navigate to "Apps" -> "Charts".
6. Find and select the "vCluster Rancher Operator" chart.
7. Follow the installation process and install the chart.
8. In the sidebar, navigate to "Workloads" -> "Deployments". Confirm that the deployment named "vcluster-rancher-op" has the State "Active".

Once the operator is installed:
* All vClusters deployed in any downstream cluster will cause a corresponding Rancher cluster to be created
* The vCluster will connect to the corresponding Rancher cluster
* Any project-member or project-owner will be added as a cluster-owner

## Uninstall

1. Select the local cluster in the Rancher clusters overview page.
2. In the sidebar, navigate to "Apps" -> "Installed Apps".
3. Delete the vcluster-rancher-op app.

Note that Rancher clusters will not be affected by removing this operator, outside of operations it manages such as RBAC syncing.

## Technical Details


*Downstream Client*

The vCluster Rancher operator runs in Rancher's local cluster and watches clusters.management.cattle.io. 
When a cluster has not been seen by the handler before, it creates a client with a token, using the Rancher reverse proxy to talk to the downstream cluster.

*Rancher Cluster Creation*

Once a client is created for each downstream cluster, it is used to watch all services with the label `app=vcluster`. If a virtual cluster install is found which does not have a corresponding Rancher cluster, the installation process begins. 
The installation process creates a provisioning cluster, waits for Rancher to create the `clusterregistrationtoken.cattle.io`,
and then extracts the command from the `clusterregistrationtoken` status. This command contains everything that is needed to deploy Rancher's cluster agent to the vCluster.
A job is then deployed in the vCluster's host cluster that creates a kubeconfig pointing to the vCluster service in the host cluster. The earlier extracted command is executed using the newly created kubeconfig.

Deletion events for the vCluster service will trigger the controller to delete the corresponding Rancher cluster.

*Cluster RBAC*

After provisioning a cluster, a `clusters.management.cattle.io` handler is created. This handler acts on `clusters.management.cattle.io` but also watches `projectroletemplatebindings.cattle.io`. The watcher will check if any virtual-cluster originated Rancher clusters are associated with the project targeted by a prtb, and enqueues those clusters if so.

All Rancher users that have a `projectroletemplatebidning.cattle.io` for the project with either the `project-owner` or `project-member` roles are added as cluster owners, using a `clusterroletemplatebinding.management.cattle.io`, to the virtual cluster's Rancher cluster.


## Development

### General Prerequisites
- docker
- devespace
- rancher installed
- access to local cluster in rancher

### To Deploy on the cluster

**Deploy the manager using devspace**

Note: This is the recommended way to develop on vcluster-rancher-op

```shell
devspace dev
```
**Build and push your image to the location specified by `IMG`:**

```sh
make docker-build docker-push IMG=<some-registry>/vcluster-rancher-op:tag
```

**Deploy the Manager to the cluster using Helm**

```sh
KUBECONFIG=/home/ricardo/dev/rancher-integration/temp-kc.yaml helm install chart --generate-name --create-namespace
```

**Deploy the Manager to the cluster with a custom image**

```sh
KUBECONFIG=/home/ricardo/dev/rancher-integration/temp-kc.yaml helm install chart --generate-name --create-namespace --set image.registry=<REGISTRY> --set image.repository=<REPO/REPO> --set tag=<TAG>
```

### Update RBAC for installed service account
1. Update controller-gen tags in the file `controller/cluster-controller.go`.
2. Run:
```sh
make manifests
```

### To Uninstall
**Delete devspace install:**
```sh
devspace purge
rm -rf .devspace/*
```

**Delete helm install:**

```sh
helm delete <release-name>
```

## License

Copyright 2025.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
