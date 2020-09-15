# codeconnect-vm-operator
Toy VM Operator using kubebuilder for educational purposes presented at VMware Code Connect 2020

# what are we trying to achieve?

Build sample Kubernetes Operator using kubebuilder that will create/delete Virtual Machines based on **declarative** desired state configuration over **imperative** scripting/automation (i.e. PowerCLI or `govc`)

This declarative configuration is exemplified below by a standard Kubernetes contruct called Customer Resource Definition (CRD):

```yaml
apiVersion: vm.codeconnect.vmworld.com/v1alpha1
kind: VmGroup
metadata:
  name: vmgroup-sample
  namespace: default
spec:
  # Add fields here
  cpu: 4
  memory: 1
  replicas: 3
  template: vm-template
```

This CRD will yield the following configuration on vCenter:

<add a screenshot here>.

The flow: the user creates a CRD via kubectl command, the operator picks up the CRD object and interact with the vCenter over the govmomi (GO library).
kubectl -> operator (lib:govmomi) -> vCenter

# Requirements for building the operator

All examples and components used on this demo are on a mac osx.

## Developer Software

Install all the following:

- git client
- Go lang - https://golang.org/dl/
- Docker Desktop - https://www.docker.com/products/docker-desktop
- Kubebuild - https://go.kubebuilder.io/quick-start.html
- Kustomize - https://kubernetes-sigs.github.io/kustomize/installation/
- [Optional but Recommended] - Code editor such as [VScode](https://code.visualstudio.com/download) or goland.
- [Optional but Recommended] - have access to a public registry such as quay.io or hub.docker.com

## Kubernetes Cluster
Any Kubernetes 1.16 and above. For this exercise, we will be using a standalone cluster Kind - https://kind.sigs.k8s.io/docs/user/quick-start/

## vCenter 
A vCenter, a user with privilege of creating and deleting Virtual Machines and VM folders.
Example:
The mac osx (or the k8s cluster) must have direct access to the vCenter.

# Downloading the sample code

To follow these instructions, you will need to git clone this repo. 

```bash
cd ~/go/src
git clone https://github.com/embano1/codeconnect-vm-operator.git
```

# Scaffolding

"scaffolding" is the first step of building an operator with kubebuilder starting with a brand-new directory.

## Creating the directory

For academic purposes, we will call the directory as "myoperator" but choose a more meaningful name because this directory name will be used as part of the default name of the k8s namespace and container of your operator (default behavior and it can be changed).

```bash
cd ~/go/src
mkdir myoperator
cd myoperator
```

At this time, we recommend you to open the code editor with both directories: the new directory you just created and this source code directory.
It should look like this:

![Image](/images/vscode-empty-directory.png "VScode Screenshot with two directories")

Now, define the go module name of your operator. This module name will be used inside of the go code. Module names are how packages make reference to each other.
We will call it vmworld/codeconnect.

## Initializing the Directory

```bash
cd ~/go/src/myoperator/
go mod init vmworld/codeconnect
```

Look now the content of myoperator folder: you will have the go.mod for this module.

![Image](/images/vscode-go-module-name.png "VScode Screenshot with operator module name")

## API Group Name, Version, Kind

API Group, its versions and supported kinds are the part of the DNA of k8s. You can read about these concepts [here](https://kubernetes.io/docs/concepts/overview/kubernetes-api/).

For academic purposes, this example creates a kind "VmGroup" that belongs to the API Group "vm.codeconnect.vmworld.com" with the initial version of v1alpha.

These are all parameters for the kubebuilder sccafolding (note the parameters --domain and then --group, --version --kind):

```bash
cd ~/go/src/myoperator/
kubebuilder init --domain codeconnect.vmworld.com
# answer yes for both questions
kubebuilder create api --group vm --version v1alpha1 --kind VmGroup
Create Resource [y/n]
y
Create Controller [y/n]
y
```

Look again the now the content of myoperator folder: you will have a typical directory with go code and other directories.
The screenshot below shows the default VmGroup type created by scaffolding.

![Image](/images/vscode-kubebuilder-create-api.png "VScode Screenshot with operator module name")

# Custom Resource Definition (CRD)

```bash
# show existing API resources before creating our CRD
kubectl api-resources
```

set crd v1 version in makefile: 

```bash
# Makefile
CRD_OPTIONS ?= "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true"
```

add spec/status in `vmgroup_types.go`

make manifests && make generate

# vmgroup_controller.go

add vc client to controller struct

```go
type VmGroupReconciler struct {
	client.Client
    VC *govmomi.Client
	Log     logr.Logger
	Scheme  *runtime.Scheme
}
```

# main.go

# first run of the manager

install CRD

```bash
# show API resources including our CRD
kubectl api-resources
```

```bash
kubectl create -f config/crd/bases/vm.codeconnect.vmworld.com_vmgroups.yaml
customresourcedefinition.apiextensions.k8s.io/vmgroups.vm.codeconnect.vmworld.com created

kubectl get vg
No resources found in default namespace.
```

export environment variables

```bash
export VC_USER=administrator@vsphere.local
export VC_PASS='Admin!23'
export VC_HOST=10.78.126.237
```

open watch in a separate window

```bash
watch -n1 "kubectl get vg,deploy,secret"
```

run the operator locally

```bash
make manager && bin/manager -insecure
```


# deploy to kubernetes

show the `manager.yaml`  manifest

```bash
# create the secret in the target namespace
kubectl create ns codeconnect-vm-operator-system
kubectl -n codeconnect-vm-operator-system create secret generic vc-creds --from-literal='VC_USER=administrator@vsphere.local' --from-literal='VC_PASS=Admin!23' --from-literal='VC_HOST=10.78.126.237'
```

delete `suite_test.go` since we're not writing any unit/integration tests and
deployment will otherwise fail

```bash
# build and deploy the manager to the cluster
make docker-build docker-push IMG=embano1/codeconnect-vm-operator:latest
make deploy IMG=embano1/codeconnect-vm-operator:latest
```

open watch in a separate window

```bash
watch -n1 "kubectl -n codeconnect-vm-operator-system get vg,deploy,secret"
```

create some VmGroups

```bash

```

fix the vg-2 template issue

```bash
kubectl patch vg vg-2 --type merge -p '{"spec":{"template":"vm-operator-template"}}'
```


# cleanup

```bash
kubectl delete vg --all -A
kubectl delete ns codeconnect-vm-operator-system
```

# Advanced Topics (we could not cover)

Feel free to enhance the code and submit PR(s) :)

- Code cleanup (DRY), unit and integration testing
- Sophisticated K8s/vCenter error handling and CR status representations
- Configurable target objects, e.g. datacenter, resource pool, cluster, etc.
- Supporting multi-cluster deployments and customizable namespace-to-vCenter mappings
- Generated object name verification and truncation within K8s/vCenter limits
- Advanced RBAC and security/role settings
- Controller local indexes for faster lookups
- vCenter object caching (and change notifications via property collector) to
  reduce network calls and round-trips
- Using hashing or any other form of compare function for efficient CR (event)
  change detection
- Using
  [expectations](https://github.com/elastic/cloud-on-k8s/blob/cf5de8b7fd09e55b74389128fbf917897b6bf17a/pkg/controller/common/expectations/expectations.go#L11)
  for more robust CR object handling (interleaving operations)
detection  
- Smarter controller (re)queuing mechanisms
- vCenter task management (tracking) and async vCenter operations to not block too long during `Reconcile()`
- Robust vCenter session management (keep-alive, gracefully handle expired
  sessions, other forms of authentication against vCenter, etc.)
- Multi-VC topologies (one-to-one for now)
- Sophisticated leader election and availability (HA) concerns
- Fancy `kustomize`(ation)
- Production readiness (resources, certificate management, webhooks, quotas, [etc.](https://github.com/mercari/production-readiness-checklist))
