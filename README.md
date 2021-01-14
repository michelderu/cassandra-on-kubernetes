# Run Cassandra on Kubernetes using the cass-operator (with Minikube)
This project aims to help usetting up and understanding some of the basics of running Cassandra on Kubernetes using a local Minikube installation.

It is assumed that Docker is already installed, up and running and has enough resources available (personally run it with 8 CPUs and 12 Gi of memory).
Also this guide assumes an osx environment with Homebrew installed.

Relevant links:
- cass-operator: https://docs.datastax.com/en/cass-operator/doc/cass-operator/cassOperatorAbout.html
- Minikube: https://minikube.sigs.k8s.io/docs/start/
- Docker: https://www.docker.com/
- Brew: https://brew.sh/

## Preparations

### Install Minikube
This installs the local Kubernetes service
```sh
brew install minikube
```

### Install Kubectl
This installs the command-line tool for Kubernetes
```sh
brew install kubectl
```

### Provision the Kubernetes cluster using Minikube
This configures a Kubernetes cluster with three worker nodes
```sh
minikube start --nodes 3 --memory=2048 --cpus=2 -p cassandra
minikube profile cassandra
```

### Check cluster health
Now test the availability of the cluster
```sh
kubectl get nodes
minikube dashboard &
``` 

## Provision Cassandra

### Load the cass operator
Load the operator and check it's availability
```sh
kubectl apply -f https://raw.githubusercontent.com/datastax/cass-operator/v1.5.0/docs/user/cass-operator-manifests-v1.16.yaml
kubectl -n cass-operator get pods --selector name=cass-operator
```

### Now create a storage class
```sh
kubectl apply -f storage-minikube.yaml
```

### Finally create a 3 node cassandra cluster (using the example that uses the storage class)
Create the cluster and check it's availability
```sh
kubectl -n cass-operator apply -f cassdc.yaml
watch kubectl -n cass-operator get pods --selector cassandra.datastax.com/cluster=cluster1
kubectl -n cass-operator get statefulset
```

### Describe the pods / dc
```sh
kubectl -n cass-operator describe pods
kubectl -n cass-operator get pod cluster1-dc1-default-sts-0 -o yaml
kubectl -n cass-operator get cassdc dc1 -o yaml
```

### Use NodeTool to check the cluster status
```sh
kubectl -n cass-operator exec -it -c cassandra cluster1-dc1-default-sts-0 -- nodetool status
```

### Log in using cqlsh
```sh
export CASS_USER=$(kubectl -n cass-operator get secret cluster1-superuser -o json | jq -r '.data.username' | base64 --decode)
export CASS_PASS=$(kubectl -n cass-operator get secret cluster1-superuser -o json | jq -r '.data.password' | base64 --decode)
kubectl -n cass-operator exec -ti cluster1-dc1-default-sts-0 -c cassandra -- sh -c "cqlsh -u '$CASS_USER' -p '$CASS_PASS'"
```

## Or just bash into a pod
```sh
kubectl -n cass-operator exec -ti cluster1-dc1-default-sts-0 -- bash
cd /etc/cassandra
cat cassandra.yaml
```

## Debugging

### Get information about the statefullset
```sh
kubectl -n cass-operator get statefulset
kubectl -n cass-operator get statefulset cluster1-dc1-default-sts -o yaml > statefullset.yaml
```

### Get current state of the pods
```sh
kubectl -n cass-operator describe pods > pods.txt
kubectl -n cass-operator describe pods cluster1-dc1-default-sts-1 > pod.txt
```

### Check health of persistant volume claims
```sh
kubectl -n cass-operator get pvc
kubectl -n cass-operator describe pvc server-data-cluster1-dc1-default-sts-0
```

### Get node capacities
```sh
kubectl get nodes -o yaml > capacity.yaml
```

## Tear down Cassandra

### Uninstall the cassandra cluster
```sh
kubectl delete cassdcs --all-namespaces --all
```

### Uninstall the storage class
```sh
kubectl delete -f https://github.com/datastax/cass-operator/raw/master/operator/k8s-flavors/minikube/storage.yaml
```

### Uninstall the cass-operator
```sh
kubectl delete -f https://raw.githubusercontent.com/datastax/cass-operator/v1.5.0/docs/user/cass-operator-manifests-v1.16.yaml
```

### Check the uninstallation
```sh
watch kubectl -n cass-operator get pods
```