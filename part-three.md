# ArangoDB k8s operator #

## What is Kubernetes? ##

* Open-source system for complex **container orchestration**
* Kind of a **cluster operating system**
  * Manage **resources**
  * **Schedule** services
  * At scale (**very large scale**)
* Moving extremely fast
* **(Currently) the winning orchestration**
  * dc/os mesosphere
  * `docker` swarm
* **Helps devops**
  * ... development
  * ... deployment
  * ... self healing
  * ... monitoring



## How is it done?

* Create resource specification
* Talk to API server
* Kubernetes cluster does stuff

* Many different resource types
  * `Pod`: smallest deployable unit
  * `Service`: logical set of pods and an access policy
  * `Deployment`: Higher level including replicas and change management policy
  * `Node`: Nomes es omen
  * `Secret`: Tokens, certificates

## ArangoDB on Kubernetes the hard way

### Single server ###
* Bare Kubernetes
* Run ArangoDB single server
* Resources
  * 1 pod
  * 1 persistent volume
  * 1 service
* 60 lines YAML file. Not nice but manageable

### Cluster ###
* Bare Kubernetes
* Resources
  * 9 pods (3 agents, 3 coordinators, 3 database servers)
  * 6 persisten volumes (agents and db servers only)
  * 1 service
* 450+ lines YAML file. Starting to solidly head for nightmare terrain.
  * Many YAML files needed. Risk of mistakes high.
  * Mostly stateful, pods can die
  * Upgrading needs human intervention and lots of care
* True for any such complex system

### What about stateful sets?
* Good:
  * Really stateful - needed of course.
* Not good:
  * Databases, especially distributed ones are complex.
  * Special upgrade requirements, when considering 0 downtime
  * Tons of YAML

## The easy way ##
* Needed obviously simplification
* Upgrade needed be stream lined
* YAML generation?
  * Scaling?
  * Upgrading?

## Kubernetes **custom resources**

* Introducing Kube-ArangoDB
  * Operator for running ArangoDB deployments
  * Write a `CustomResouce` and `apply`
    ```
    apiVersion "database.arangodb.com/v1alpha"
    kind "ArangoDeployment"
    metadata
      name "example-simple-cluster"
    spec
      mode "Cluster"
    ```  
  * Scaling up? Add the following to bottom and `apply`
    ```
    dbservers
      count 7
    ```
  * Scaling down? Add the following to bottom and `apply`
    ```
    dbservers
      count 4
    ```
  * Upgrading?
    ```
    image arangodb/arangodb:3.3.14
    ```

## Access the service
* Internal access through cluster IP
* External access through `LoadBalancer` or `NodePort`
  * E.g. `my-deployment` will have additional service `my-deployment-ea`
  * Service type automatically set

## Storage options
* Locally attached SSD best case
* Network volumes work but are fast
* Not all providers provide locally attached storage
  * `ArangoLocalStorage` operator

    ```
    apiVersion: "storage.arangodb.com/v1alpha"
    kind: "ArangoLocalStorage"
    metadata:
      name: "arangodb-local-storage"
    spec:
      storageClass:
        name: my-local-ssd
        isDefault: true
      localPath:
        - /var/lib/arango-storage
    ```

## Feature summary
* Support all deployment modes: `single`, `active failover` and `cluster`
* Automatic TLS
* Automatic authentication
* Locally attached persistent storage with custom resources
* `DC2DC` support
  * `sync.enabled: true`
* support `Kubernetes Federation` pending
* Easy configuration

## Installation
* Operators running in Kubernetes cluster
* Deployment options
  * `helm install ...` (preferred, easier to tweak and upgrade)
  * `kube apply -f ...`

## Monitoring
* Of course logs of operator pods
* Status part of custom resources
  `kubectl get -o ...`
* Events attached to custom resources
  `kubectl describe ...`
* continuous monitoring via `prometheus`

## Suported providers
* Kubernetes version >= `1.10`
* However providers are not equal
  * Big differences in authentication
  * Big differences in ease of use
* Small features to deal with
  * Preemptive nodes
  * load balancer limitations
  * regional IP addresses ...
* Production ready on GKE
* Other platforms more testing
* Federation support coming

## Deep dive into `kube-arangodb`
* Multiple operators in a ''single'' package
  * One per custom resource type
  * Easier deployment and releasing
  * Released `0.3.0` last week
* Custom resource definitions
  * Go code -> lots of additionally generated
* Leadership election per operator
  * Hot standby for failover
* Finalizers

## ArangoDeployment layers of control
* Resource
  * Ensure all resources are created as needed
  * Inspect resources for state change
* Reconciliation
  * Ensure that we have the members to adhere to specs
  * Create and execute a plan else
* Resilience
  * Monitor system behaviour over time
  * Take action is needed

## Note on `Finalizers`
* Generated resources need to be guarded from removal
* Only remove after strict checking

## What did I do for this course

* Setup 3 machines `kube01`, `kube02`, `kube03`
* `kube01`
  * `kubeadm init --pod-network-cidr=10.244.0.0/16`
  * `mkdir -p $HOME/.kube`
  * `cp /etc/kubernetes/admin.conf .`
  * `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml`
  * `kubectl taint nodes --all node-role.kubernetes.io/master-`
* `kube02` & `kube03`
  * `kubeadm join kube01:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-sha256>`
* Et voila I have a Kubernetes cluster

## The good old way
* kubectl apply -f https://raw.githubusercontent.com/arangodb/kube-arangodb/0.3.0/manifests/crd.yaml
* kubectl apply -f https://raw.githubusercontent.com/arangodb/kube-arangodb/0.3.0/manifests/arango-deployment.yaml
* kubectl apply -f https://raw.githubusercontent.com/arangodb/kube-arangodb/0.3.0/manifests/arango-storage.yaml
* kubectl apply -f
https://raw.githubusercontent.com/arangodb/kube-arangodb/0.3.0/manifests/arango-deployment-replication.yaml

## The new way
* `helm install https://github.com/arangodb/kube-arangodb/releases/download/0.3.0/kube-arangodb.tgz`
* `helm install https://github.com/arangodb/kube-arangodb/releases/download/0.3.0/kube-arangodb-storage.tgz`
