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
  namespace: myoperator-vms
spec:
  # Add fields here
  cpu: 2
  memory: 1
  replicas: 3
  template: my-operator-template
```

This CRD will yield the following configuration on vCenter:

![Image](/images/vcenter-final-result.png "vCenter with final result of a CRD desired state.")

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

To start a kind cluster on your mac, run the following command, setting as an arbitrarily name for your cluster (this name will be used for kubectl context):

```bash
kind create cluster --name operator-dev
```

By default, kind will start a cluster with the latest version of Kubernetes. You can have as many kind clusters you wish, as long as the cluster name is unique.
The following example starts a kind cluster with k8s 1.16 version:

```bash
kind create cluster --image=kindest/node:v1.16.4 --name myother-operator-dev
```

You can switch from one cluster to another using the kubectl config use-context <name>. 
For this exercise, you will need only one cluster.

Additionally, let's make sure the cluster is responsive and create a namespace where we will create our VmGroup object:

```bash
$ kubectl create namespace myoperator-vms
namespace/myoperator-vms created
```

## vCenter Environment

You will need a vCenter, a user with privilege of creating and deleting Virtual Machines and VM folders.
Other vCenter requirements:
- The k8s cluster must have direct access to the vCenter (no proxy, no jumpbox).
- You will need to create a "vm-operator" folder under the datacenter. The controller will create VMs under this folder.
- Limitation: you can't have more than one DataCenter managed by the vCenter.
- A default Resource Group.
- At least one VM template configured, preferrably under the "vm-operator" folder. The template name must be unique.

Just an example of how to create a VM folder on any given vCenter configuration:

![Image](/images/vcenter-new-vmfolder.png "vCenter new folder creation.")


Once "vm-operator" folder is created, populate with templates, in this case the template is called "vm-operator-template".
It should look like this:

![Image](/images/vcenter-vm-operator-folder-template.png "vCenter vm-operator folder.")


# Downloading the sample code

To follow the next instructions, you will need to git clone this repo to your machine. 

```bash
cd ~/go/src
git clone https://github.com/embano1/codeconnect-vm-operator.git
```

# Kubebuilder Scaffolding

"scaffolding" is the first step of building an operator which kubebuilder will initialize the operator from a brand-new directory.

## Creating the directory

For academic purposes, we will call the directory as "myoperator" but choose a more meaningful name because this directory name will be used as part of the default name of the k8s namespace and container of your operator (this is default behavior and it can be changed).

```bash
cd ~/go/src
mkdir myoperator
cd myoperator
```

At this time, we recommend you to open the code editor (using VScode as screenshots from now on) with both directories: the new directory you just created and this source code directory.
It should look like this:

![Image](/images/vscode-empty-directory.png "VScode Screenshot with two directories.")


## Initializing the Directory

Now, define the go module name of your operator. This module name will be used inside of the go code.
We will call it myoperator as well (same as the directory name).

```bash
cd ~/go/src/myoperator/
go mod init myoperator
```

At this point, you will have a single file under myoperator folder: go.mod.
![Image](/images/vscode-go-module-name.png "VScode Screenshot with go.mod after kubebuilder scaffolding.")

## API Group Name, Version, Kind

API Group, its versions and supported kinds are the part of the DNA of k8s. You can read about these concepts [here](https://kubernetes.io/docs/concepts/overview/kubernetes-api/).

For academic purposes, this example creates a kind "VmGroup" that belongs to the API Group "vm.codeconnect.vmworld.com" with v1alpha as the initial version.

These are all parameters for the kubebuilder scaffolding: please note the parameters --domain and then --group, --version --kind).

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

Look again the now the content of myoperator folder: you will have a typical directory with go code.
The screenshot below shows the default VmGroup type created under the api directory.

![Image](/images/vscode-kubebuilder-create-api.png "VScode Screenshot with operator module name.")

Pay attention on the new version of the go.mod: kubebuilder populated all the dependencies for the new controller you are building.

![Image](/images/vscode-go-mod-updated.png "VScode Screenshot with go.mod after kubebuilder scaffolding.")


# Custom Resource Definition (CRD)

## Concept

The Custom Resource Definition [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) is how you extend Kubernetes. Any custom controller/operator requires at least one CRD for its functionality (we will call it "root object").
When you design your operator, you will need to spend time to define what kind of root object(s) you will need and the data.

Each K8s object has at least three sections: [metadata, spec and status](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/).

On Spec section is where you declare the state of the object you wish to have. On Status section is where the current state of the object. Your operator's job is reconcile these two sections.

## Our example: vmgroup_types.go

For our academic example, we want to define a root object type "Vmgroup" that will create a VM folder on vCenter.
For such, we need to provide the following information to vCenter (besides the name of the folder):
- cpu: how many CPUs that each VM will be created with. It is an integer.
- memory: how much memory (in GB) that each VM will be created with. Integer.
- replicas: How many VMs under the folder. Integer.
- template: the name of the VM template that VMs will be created. It is a string.

Scaffolding already created a vmgroup_types.go for us. However, it is empty for these desired fields (on both spec and status sections). We will need to populate these sections with our business logic.

```bash
cd ~/go/src
cp codeconnect-vm-operator/api/v1alpha1/vmgroup_types.go myoperator/api/v1alpha1/vmgroup_types.go
```

Look at the body of the vmgroup_types.go, the screenshot below has the new Status section.

![Image](/images/vscode-vmgroup-types-go.png "VScode Screenshot with vmgroup_types.go")

## Version of CRDs

CRDs are evolving like any Kubernetes functionality. As of K8s 1.16, the default version of CRD is v1 which is the latest version.
The latest version of CRD introduced many features, such as error control on the CRDs.
We want to onboard our controller with the latest version of CRDs, For such, we need to set v1 version in makefile (if you plan to run your operator on older version of K8s cluster, skip this part). 

```bash
# edit ~/go/src/myoperator/Makefile and change to the following line
CRD_OPTIONS ?= "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true"
```

## Creating the CRD

The following will generate the CRD based on the spec/status that we copied (from last section):

```bash
make manifests && make generate
```
You might have some issues in compiling with mismatching modules.
See section common errors if you encounter those.

Example of a clean run:

```bash
$ make manifests && make generate
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/Users/rbrito/go//bin/controller-gen "crd:preserveUnknownFields=false,crdVersions=v1,trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/Users/rbrito/go//bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
```

The success criteria is the CRD generated as config/crd/bases/vm.codeconnect.vmworld.com_vmgroups.yaml file.

## Installing the CRD

Before onboarding our CRDs, let's inspect the cluster to make sure they do not exist.
The following command lists all API Groups of the cluster.

```bash
# show all existing API resources on this cluster
$ kubectl api-resources
# making sure they do not exist
$ kubectl api-resources | grep codeconnect
# empty
```

After check the vm.codeconnect.vmworld.com does not exist, let's install them:

```bash
$ kubectl create -f config/crd/bases/vm.codeconnect.vmworld.com_vmgroups.yaml 
customresourcedefinition.apiextensions.k8s.io/vmgroups.vm.codeconnect.vmworld.com created

# checking again
$ kubectl api-resources | grep codeconnect
vmgroups                          vg           vm.codeconnect.vmworld.com     true         VmGroup
```

Congrats, your cluster is now able to understand your CRD. 
Let's create one VmGroup object under our namespace myoperator-vms:

```bash
cat <<EOF | kubectl -n myoperator-vms create -f -
apiVersion: vm.codeconnect.vmworld.com/v1alpha1
kind: VmGroup
metadata:
  name: vmgroup-sample
spec:
  # Add fields here
  cpu: 2
  memory: 1
  replicas: 3
  template: vm-operator-template
EOF

# expected output
vmgroup.vm.codeconnect.vmworld.com/vmgroup-sample created
```

Let's list the CRD using a standard kubectl command. Note the some fields are empty. 
They are this way because we have not written our controller yet:

```bash
$ kubectl get vmgroups -n myoperator-vms
NAME             PHASE   CURRENT   DESIRED   CPU   MEMORY   TEMPLATE          LAST_MESSAGE
vmgroup-sample                     3         2     1        photon-template   
```


# Controller: vmgroup_controller.go

It is time to compile the controller. 
For such, we will copy some files from this source repo to our myoperator directory.
We will go over the code later.

```bash
cd ~/go/src
# limiters.go, vmgroup_controller.go and vsphere.go
cp codeconnect-vm-operator/controllers/* myoperator/controllers/
cp codeconnect-vm-operator/main.go myoperator/main.go
```

After the copy, we need make sure the import directives point to our myoperator directory.

See screenshot below the orginal main.go:

![Image](/images/vscode-old-main-go.png "VScode Screenshot with old main.go")

Change the import to the "myoperator" directory:

![Image](/images/vscode-new-main-go.png "VScode Screenshot with new main.go")

Repeat the same changes on vmgroup_controller.go and vsphere.go.
On the latter, you will need to change the constant to match your vCenter datacenter.
See example below:

![Image](/images/vscode-new-vsphere-go.png "VScode Screenshot with new vsphere.go")

Since this code is for academic purposes, delete `suite_test.go` since we're not writing any unit/integration tests.

```bash
cd ~/go/src/myoperator
rm controllers/suite_test.go
```

## Compiling

```bash
$ make manager
go: creating new go.mod: module tmp
go: found sigs.k8s.io/controller-tools/cmd/controller-gen in sigs.k8s.io/controller-tools v0.2.5
/Users/rbrito/go//bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
```

## Running the controller in development mode

For the first run, we will run our controller on our desktop.

Finally, let's set the environmental variables for the vCenter connection:

```bash
export VC_HOST=10.186.34.28
export VC_USER=administrator@vsphere.local
export VC_PASS='Admin!23'
```

Other vCenter (pending a single DC):

```bash
export VC_HOST=lab02-m01-vc01.lab02.vsanpe.vmware.com
export VC_USER=rbrito@vsphere.local
export VC_PASS=P@ssw0rd
```

Now, let's start controller locally on your machine. This is known running the controller in development mode (since it is not yet as a pod).
See common errors section to troubleshoot, but you should see the following messages:

```bash
$ bin/manager -insecure

"level":"info","ts":1600289927.772952,"logger":"controller-runtime.metrics","msg":"metrics server is starting to listen","addr":":8080"}
{"level":"info","ts":1600289929.190078,"logger":"setup","msg":"starting manager"}
{"level":"info","ts":1600289929.190326,"logger":"controller-runtime.manager","msg":"starting metrics server","path":"/metrics"}
{"level":"info","ts":1600289929.190331,"logger":"controller","msg":"Starting EventSource","reconcilerGroup":"vm.codeconnect.vmworld.com","reconcilerKind":"VmGroup","controller":"vmgroup","source":"kind source: /, Kind="}
{"level":"info","ts":1600289929.293183,"logger":"controller","msg":"Starting Controller","reconcilerGroup":"vm.codeconnect.vmworld.com","reconcilerKind":"VmGroup","controller":"vmgroup"}
{"level":"info","ts":1600289929.293258,"logger":"controller","msg":"Starting workers","reconcilerGroup":"vm.codeconnect.vmworld.com","reconcilerKind":"VmGroup","controller":"vmgroup","worker count":1}
{"level":"info","ts":1600289929.293513,"logger":"controllers.VmGroup","msg":"received reconcile request for \"vmgroup-sample\" (namespace: \"myoperator-vms\")","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289929.683297,"logger":"controllers.VmGroup","msg":"VmGroup folder does not exist, creating folder","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289929.68333,"logger":"controllers.VmGroup","msg":"creating VmGroup in vCenter","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289930.958067,"logger":"controllers.VmGroup","msg":"no VMs found for VmGroup, creating 3 replica(s)","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289930.958116,"logger":"controllers.VmGroup","msg":"creating clone \"vmgroup-sample-replica-lbzgbaic\" from template \"vm-operator-template\"","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289930.9581382,"logger":"controllers.VmGroup","msg":"creating clone \"vmgroup-sample-replica-mrajwwht\" from template \"vm-operator-template\"","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289930.9581592,"logger":"controllers.VmGroup","msg":"creating clone \"vmgroup-sample-replica-hctcuaxh\" from template \"vm-operator-template\"","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289938.199923,"logger":"controllers.VmGroup","msg":"received reconcile request for \"vmgroup-sample\" (namespace: \"myoperator-vms\")","vmgroup":"myoperator-vms/vmgroup-sample"}
{"level":"info","ts":1600289939.453033,"logger":"controllers.VmGroup","msg":"replica count in sync, checking power state","vmgroup":"myoperator-vms/vmgroup-sample"}
```








# deploy to kubernetes


```bash
# build and deploy the manager to the cluster
make docker-build docker-push IMG=embano1/codeconnect-vm-operator:latest
make deploy IMG=embano1/codeconnect-vm-operator:latest
```

show the `manager.yaml`  manifest

```bash
# create the secret in the target namespace
kubectl create ns codeconnect-vm-operator-system
kubectl -n codeconnect-vm-operator-system create secret generic vc-creds --from-literal='VC_USER=administrator@vsphere.local' --from-literal='VC_PASS=Admin!23' --from-literal='VC_HOST=10.78.126.237'
```

open watch in a separate window

```bash
watch -n1 "kubectl -n codeconnect-vm-operator-system get vg,deploy,secret"
```

fix the vg-2 template issue

```bash
kubectl patch vg vg-2 --type merge -p '{"spec":{"template":"vm-operator-template"}}'
```

# Code Deep Dive

This section will highlight some of the go code that made the controller functional.

```go
type VmGroupReconciler struct {
	client.Client
    VC *govmomi.Client
	Log     logr.Logger
	Scheme  *runtime.Scheme
}
```

# main.go

open watch in a separate window

```bash
watch -n1 "kubectl get vg,deploy,secret"
```


## Reconcile

TBD

## Limit

TBD

## Finalizer

TBD

# Common Errors you might encounter during this exercise

## go.mod incompability

You might have some issues in compiling with mismatching modules.
For example, I had the following errors:

```
$ make manifests && make generate
go: finding module for package k8s.io/api/auditregistration/v1alpha1
go: finding module for package k8s.io/api/auditregistration/v1alpha1
../../../pkg/mod/k8s.io/kube-openapi@v0.0.0-20200831175022-64514a1d5d59/pkg/util/proto/document.go:24:2: case-insensitive import collision: "github.com/googleapis/gnostic/openapiv2" and "github.com/googleapis/gnostic/OpenAPIv2"
../../../pkg/mod/k8s.io/client-go@v11.0.0+incompatible/kubernetes/scheme/register.go:26:2: module k8s.io/api@latest found (v0.19.1), but does not contain package k8s.io/api/auditregistration/v1alpha1

# As documented on
# https://github.com/kubernetes/client-go/issues/741
```

I fixed it running:
```
go get -u k8s.io/client-go@v0.17.2 github.com/googleapis/gnostic@v0.3.1 ./...
# and editing go.mod for client-go and apimachinery to match versions:
# k8s.io/apimachinery v0.17.2
# k8s.io/client-go v0.17.2
# then running "go get <module>" until there is a clean execution
```

## CRDs are not applied yet

```
$ kubectl create vmgroup
error: unable to recognize "STDIN": no matches for kind "VmGroup" in version "vm.codeconnect.vmworld.com/v1alpha1"
```

## Cluster not running / kubeconfig misconfigured

```
"level":"error","ts":1600288746.2496572,"logger":"controller-runtime.client.config","msg":"unable to get kubeconfig","error":"invalid configuration: no configuration has been provided, try setting KUBERNETES_MASTER environment variable","errorCauses":[{"error":"no configuration has been provided, try setting KUBERNETES_MASTER environment variable"}],"stacktrace":"github.com/go-logr/zapr.(*zapLogger).Error\n\t/Users/rbrito/go/pkg/mod/github.com/go-logr/zapr@v0.2.0/zapr.go:132\nsigs.k8s.io/controller-runtime/pkg/client/config.GetConfigOrDie\n\t/Users/rbrito/go/pkg/mod/sigs.k8s.io/controller-runtime@v0.6.3/pkg/client/config/config.go:159\nmain.main\n\t/Users/rbrito/go/src/myoperator/main.go:70\nruntime.main\n\t/usr/local/Cellar/go/1.14.2_1/libexec/src/runtime/proc.go:203"}
```

## Lack of environmental variables

```
{"level":"error","ts":1600286316.549897,"logger":"setup","msg":"could not connect to vCenter","controller":"VmGroup","error":"could not parse URL (environment variables set?)","errorVerbose":"could not parse URL (environment variables set?)\nmain.newClient\n\t/Users/rbrito/go/src/myoperator/main.go:137\nmain.main\n\t/Users/rbrito/go/src/myoperator/main.go:90\nruntime.main\n\t/usr/local/Cellar/go/1.14.2_1/libexec/src/runtime/proc.go:203\nruntime.goexit\n\t/usr/local/Cellar/go/1.14.2_1/libexec/src/runtime/asm_amd64.s:1373","stacktrace":"github.com/go-logr/zapr.(*zapLogger).Error\n\t/Users/rbrito/go/pkg/mod/github.com/go-logr/zapr@v0.2.0/zapr.go:132\nmain.main\n\t/Users/rbrito/go/src/myoperator/main.go:92\nruntime.main\n\t/usr/local/Cellar/go/1.14.2_1/libexec/src/runtime/proc.go:203"}
```

## Need the -insecure parameter

If your vCenter does not have a signed cert, you will need to pass the -insecure flag to the controller (manager binary).

```
"level":"error","ts":1600288944.8231418,"logger":"setup","msg":"could not connect to vCenter","controller":"VmGroup","error":"could not get vCenter client: Post \"https://10.186.34.28/sdk\": x509: cannot validate certificate for 10.186.34.28 because it doesn't contain any IP SANs","stacktrace":"github.com/go-logr/zapr.(*zapLogger).Error\n\t/Users/rbrito/go/pkg/mod/github.com/go-logr/zapr@v0.2.0/zapr.go:132\nmain.main\n\t/Users/rbrito/go/src/myoperator/main.go:92\nruntime.main\n\t/usr/local/Cellar/go/1.14.2_1/libexec/src/runtime/proc.go:203"}
```

## vCenter has multiple DataCenters

TBD


# cleanup after testing

```bash
kubectl delete vg --all -A
kubectl delete ns codeconnect-vm-operator-system

# or destroying the entire kind cluster
kind destroy cluster --name operator-dev
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
