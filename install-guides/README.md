# Resource Topology Scheduling Installation

This guide will walk through the process of installing Resource Topology Aware Scheduling on a single-node Kubernetes cluster for basic validation.

## Prerequisites
The following are required:
- Single-node Kubernetes cluster of 1.20 or higher installed with kubeadm.
- Untainted master node in cluster.(`kubectl taint nodes $(kubectl get nodes -o=jsonpath='{.items[0].metadata.name}') node-role.kubernetes.io/master:NoSchedule-`) 
- Go 1.15+ on the system where Kubernetes components are to be built

## Step by step
The following steps are required to get the Resource Topology Aware Scheduling solution up and running.
1) Install modified Kubelet
2) Configure Kubelet to use Topology Manager and CPU Manager
3) Install Topology Exporter
4) Install Topology Aware Scheduler
5) Create scheduling policy

### Install modified Kubelet
The Kubelet PodResource endpoint is being changed to enable Resource Topology Scheduling. A preview of those changes is available from [this repo.](https://github.com/fromanirh/kubernetes)

To pull and build the modified version of Kubelet:
```
git clone https://github.com/fromanirh/kubernetes
cd kubernetes
git checkout podresources-concrete-resources-apis
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

### Configure Kubelet to use Topology Manager and CPU Manager
Kubelet requires a CPU Manager policy of static and a Topology manager policy other than 'None' to be set in order to enable Topology scheduling.

In kubeadm these variables can be set in a file that sits in 
- Ubuntu: /etc/default/kubelet
- CentOS: /etc/sysconfig/kubelet

 Replace the line KUBELET_EXTRA_ARGS with the following:

```
KUBELET_EXTRA_ARGS="\
--cpu-manager-policy=static \
--kube-reserved=cpu=1,memory=2Gi,ephemeral-storage=1Gi \
--system-reserved=cpu=1,memory=2Gi,ephemeral-storage=1Gi \
--feature-gates=TopologyManager=true  --topology-manager-policy=single-numa-node"
```

The next step is to remove the kubelet cpu checkpoint and restart the service.

```
systemctl stop kubelet
systemctl daemon-reload
rm -f /etc/kubernets
rm -f /var/lib/kubelet/cpu_manager_state
systemctl start kubelet
```
To check that the above has take effect run `systemctl status kubelet` and review the arguments passed to kubelet. The above args, including topology manager and cpu manager policies, should now be in place.

### Install Topology Exporter
The topology exporter is part of Node Feature Discovery. It reads information from the Kubelet's endpoint and creates a representation of the Kubernetes node as a result.

To install Node Feature Discovery:
```
git clone https://github.com/k8stopologyawareschedwg/node-feature-discovery.git
cd node-feature-discovery
git checkout rte-pr-2
make image
kubectl apply -f  manifests/noderesourcetopologies.yaml
kubectl apply -f kubectl create -f manifests/nfd-daemonset-master-topology-updater.yaml
```
Once Node Feature discovery is up and running it should begin reporting on the resources per numa node for the node.

To see these descriptions run:
```
kubectl describe  noderesourcetopologies
```

### Install Topology Aware Scheduler
At this point our kubelet is exposing information about resource Topology and our Node Feature Discovery agent is making this information available in the Kubernetes API. The next step is to schedule based on the per-node resource topology information. To get and build our scheduler run:

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


### Create scheduling policy
In order to select which workloads require Topology Awareness at scheduling time scheduling policies - part of the Kubernetes Scheduling Framework - are used. Scheduling policies allow a single kube-scheduler binary or pod to perform many different types of scheduling job through a set of plugins defined by a policy.

For Topology Aware Scheduling the policy is defined in a configmap:
```
kubectl describe  cm topo-aware-scheduler-config -n kube-system
```
This file has a number of profiles which can be called on directly by a workload using the schedulerName field in a pod spec. Our pre-loaded policy is using the `single-numa-node-scheduler` policy which enables our NodeResourceTopologyMatch plugin.

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

Note that the components in Topology Aware Scheduling are still under active development. If anything in this guide doesn't work as exected please open an issue on the repo describing your problem along with details about your environment.

## Topology Scheduling on multiple nodes
The above demonstrates how to use Resource Topology Scheduling on a single node cluster. To expand the above to multiple nodes the modified kubelet will need to be installed and configured on each node in the cluster. The topology exporter can then be run on each node. 
