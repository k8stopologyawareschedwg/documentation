# Resource Topology Scheduling Installation

This guide will walk through the process of installing Resource Topology Aware Scheduling on a single-node Kubernetes cluster.

## Prerequisites
The following are required:
- Single-node Kubernetes cluster of 1.21 or higher installed with kubeadm. Note: kubeadm installation isn't a prerequisite for Topology Aware Scheduling, but assumptions on configuration of Kubernetes components are made in this guide that rely on Kubeadm being the installation mechanism.
- Untainted master node in cluster.(`kubectl taint nodes $(kubectl get nodes -o=jsonpath='{.items[0].metadata.name}') node-role.kubernetes.io/master:NoSchedule-`) 
- Go 1.15+ on the system where components are to be built

## Step by step
The following steps are required to get the Resource Topology Aware Scheduling solution up and running.

1) Configure Kubelet to use Topology Manager, CPU Manager and the Pod Resources endpoint extensions
2) Install Node Feature Discovery with Topology Exporter
3) Replace default scheduler with Topology Aware Scheduler enabled Kubernetes scheduler
4) Create scheduling policy

### Configure Kubelet to use Topology Manager and CPU Manager
Kubelet requires a CPU Manager policy of static and a Topology manager policy other than 'None' to be set in order to enable Topology Scheduling. It also requires the Pod Resources endpoint, enabled in Kubernetes 1.21, to be enabled through its feature gate.

In kubeadm these variables can be set in a file that sits in 
- Ubuntu: /etc/default/kubelet
- CentOS: /etc/sysconfig/kubelet

 Replace the line KUBELET_EXTRA_ARGS with the following:

```
KUBELET_EXTRA_ARGS=\
--cpu-manager-policy=static \
---reserved-cpus=0 \
--topology-manager-policy=single-numa-node \
--feature-gates="TopologyManager=true,KubeletPodResourcesGetAllocatable=true"

```

The next step is to remove the kubelet cpu checkpoint and restart the service.

```
systemctl stop kubelet
systemctl daemon-reload
rm -f /var/lib/kubelet/cpu_manager_state
systemctl start kubelet
systemctl restart kubelet
```
To check that the above has take effect run `systemctl status kubelet` and review the arguments passed to kubelet. The above args, including topology manager and cpu manager policies, should now be in place.

### Install Topology Exporter
In Kubernetes 1.21 a number of enhancements have been made to the Kubelet [to support Topology Scheduling](https://github.com/kubernetes/website/blob/64ef8768ab9ae25a5338a95225a80ac90423473f/content/en/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins.md#monitoring-device-plugin-resources). These changes are enabled by the feature flag 'KubeletPodResourcesGetAllocatableTrue' and expose information about device topology on a gRPC service running locally on each node. A Resource Topology Exporter can then be used in order to read this information and make it available in the Kubernetes API.

The Resource Topology Exporter is part of Node Feature Discovery. It reads information from the Kubelet's endpoint and creates a representation of the Kubernetes node as a result.
Note: With the development branch only resources in a specified namespace are being watched. The current config watches resources in the ["rte" namespace](https://github.com/k8stopologyawareschedwg/node-feature-discovery/blob/topology-updater-implementation/manifests/nfd-daemonset-master-topology-updater.yaml.template#L109).

To install Node Feature Discovery from the [development branch](https://github.com/kubernetes-sigs/node-feature-discovery/pull/525):

```
git clone https://github.com/k8stopologyawareschedwg/node-feature-discovery.git
cd node-feature-discovery
git checkout topology-updater-implementation 
make image
kubectl apply -f  manifests/noderesourcetopologies.yaml
kubectl apply -f manifests/nfd-daemonset-master-topology-updater.yaml
```

Once Node Feature discovery is up and running it should begin reporting on the resources per numa node for the node.

To see these descriptions run:
```
kubectl describe  noderesourcetopologies
```

### Install Topology Aware Scheduler

At this point our kubelet is exposing information about resource Topology and our Node Feature Discovery agent is making this information available in the Kubernetes API. The next step is to schedule based on the per-node resource topology information.

Node Resource Topology Scheduling is currently enabled in the [community sponsored scheduler plugins repo](https://github.com/kubernetes-sigs/scheduler-plugins/tree/release-1.19).  The image from the scheduler plugins repo has all of the scheduling power of the Kubernetes core scheduler with some added useful plugins - like Resource Topology Scheduling - built in on top.
You can read more about the Node Resource Topology Scheduler [here](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/noderesourcetopology/README.md)

```
git clone https://github.com/kubernetes-sigs/scheduler-plugins 
cd scheduler-plugins
```
Before deploying the image field in manifests/noderesourcetopology/deploy.yaml needs to be updated with:


```
image: k8s.gcr.io/scheduler-plugins/kube-scheduler:v0.19.9
```

Now the manifests can be applied with:
```
kubectl apply -f manifests/noderesourcetopology/cluster-role.yaml 
kubectl apply -f manifests/noderesourcetopology/scheduler-configmap.yaml
kubectl apply -f manifests/noderesourcetopology/deploy.yaml
```

More information on deploying and troubleshooting the Node Resource Topology Scheduler can be found at [the upstream repo](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/pkg/noderesourcetopology/README.md)


#### Run in place of default scheduler
Alternatively the Topology Aware Scheduler can be run in place of the default kubernetes scheduler so that there is only one scheduler in the cluster.
To deploy a scheduler with Resource Topology Scheduling capabilities we can replace the image used by the default scheduler in our Kubernetes manifest.
To deploy the scheduler run `vim /etc/kubernetes/manifests/kube-scheduler.yaml` to open the deployment manifest. Locate the image field and replace the line with:

` 
image: k8s.gcr.io/scheduler-plugins/kube-scheduler:v0.19.9
`
The configmap that configures the scheduler plugins will have to be mounted to the main kube scheduler - similar to the mounting operation in the [scheduler plugins deploy.yaml file](https://raw.githubusercontent.com/kubernetes-sigs/scheduler-plugins/master/manifests/noderesourcetopology/deploy.yaml)

### Create scheduling policy
In order to select which workloads require Topology Aware placement scheduling policies - part of the Kubernetes Scheduling Framework - are used. Scheduling policies allow a single kube-scheduler binary or pod to perform many different types of scheduling job through a set of plugins defined by a policy.

For Topology Aware Scheduling the policy is defined in a configmap:
```
kubectl describe  cm topo-aware-scheduler-config -n kube-system
```
This file has a number of profiles which can be called on directly by a workload using the schedulerName field in a pod spec. Our pre-loaded policy is using the `topo-aware-scheduler` prolicy which enables our NodeResourceTopologyMatch plugin.

## Topology Aware Scheduling in action
With the above components working workloads will now be scheduled according to Topology Requirements as long as those requirements are pointed to in the Pod spec. Pods which do not need to be scheduled according to their resource constraints can continue to use the default scheduler.

From the folder this guide sits in there is a test deployment `test-deployment.yaml`. This deployment contains a spec that calls on the single-numa-node-scheduler policy.
To see it in action run: `kubectl apply -f test-deployment.yaml`. The result should look like the below:
```
test-deployment-798d7f9889-hj5mt   0/1     Pending   0          12m
``` 
Looking at the details of the pod we can see the reason it hasn't been deployed is an issue with alignment of node resources:
```
kubectl describe pod test-deployment-798d7f9889-hj5mt

....
Warning  FailedScheduling  38s (x11 over 13m)  single-numa-node-scheduler  0/1 nodes are available: 1 Can't align container: test-deployment-1-container-1.
```

Note that the components in Topology Aware Scheduling are still under active development. If anything in this guide doesn't work as expected please open an issue on the repo describing your problem along with details about your environment.

## Topology Scheduling on multiple nodes
The above demonstrates how to use Resource Topology Scheduling on a single node cluster. To expand the above to multiple nodes the modified kubelet will need to be installed and configured on each node in the cluster. The topology exporter can then be run on each node through NFD. Scheduling will be performed based on Topology across all nodes where applicable.

 
### Component developer versions
As development of Topology Aware Scheduling continues features will likely be enabled out of tree before being merged into Kubernetes repositories. Below shows the target repos for development and the methods for deploying these out of tree versions.

#### Kubelet
There exist a number of experimental Kubelet branches which enable features that support Topology Aware Scheduling. To build and deploy these custom Kubelet builds from branch $BRANCHNAME:


```
git clone https://github.com/kubernetes/kubernetes
cd kubernetes
git checkout $BRANCHNAME 
make clean && make WHAT=cmd/kubelet
```

Once kubelet has build successfully we'll need to replace the kubelet on our system with the version we've just built. To do so we're going to simply swap out the binary and restart the service:

```
mv _output/bin/kubelet /usr/bin/

systemctl restart kubelet
```

Once kubelet is successfully restarted - this can be seen with `systemctl status kubelet` check if the new version of kubelet is load with:

```
kubectl get nodes
```

The version number of the output should be altered to look something like:

```
silpixa00399863   Ready    master   66m   v1.18.0-alpha.1.9389+521ee88a0f3142-dirty
```


#### Node Feature Discovery
The guide above uses an out of tree build of Node Feature Discovery.

#### Kube Scheduler

```
git clone https://github.com/k8stopologyawareschedwg/scheduler-plugins 
cd scheduler-plugins
git checkout TopologyAwareSchedulerPerNUMA
make local-image
``` 
Now that the scheduler is up and running we can deploy it by running:
```
kubectl apply -f manifests/noderesourcetopology/cluster-role.yaml 
kubectl apply -f manifests/noderesourcetopology/scheduler-configmap.yaml
kubectl apply -f manifests/noderesourcetopology/deploy.yaml
```
