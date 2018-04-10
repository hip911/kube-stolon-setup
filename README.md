Running stolon under Kubernetes (Rancher)
=========================================

#### Requirements
* a distributed backend KV store for storing cluster state - [Consul](https://www.consul.io/)
* [Helm](https://github.com/kubernetes/helm#install) Kubernetes package manager installed where you run kubectl

#### INSTALL
* Depending on your cluster it might be practical to adjust the namespace you are installing stolon into. If you end up installing the KV store in a different namespace, the DNS name of that should change also (eg. consul-consul.namespace) 

###### Volume provisioning
Dynamic provisioning can be achieved with the 
[external-storage](https://github.com/kubernetes-incubator/external-storage) plugins 
but integration with Rancher 2.0 platform is not proven to be skimmed off, 
so let's use the [fabric8](https://fabric8.io/guide/getStarted/persistence.html) binary to bind PersistentVolumeClaims to PersistentVolumes. 

Install onto you control box

    `curl -sS https://get.fabric8.io/download.txt | bash`
    `export PATH=$PATH:$HOME/.fabric8/bin`

Once installed

    `gofabric8 volumes`

will create the volume which blocks bringing up our pods. The volume creation should be repeated manually over and over.
   

###### Consul

The default [helm chart](/consul/helm) worked well for this setup, however 
the [values.yaml](/consul/helm/values.yaml) can be tweaked for further requirements.

    `helm install --name consul -f values.yaml stable/consul`


###### Stolon
The non-helm install for Stolon worked better, please find the resources in the [stolon](/stolon/kubernetes) dir.
First we need a one-off cluster initialization

    `kubectl run -i -t stolonctl --image=sorintlab/stolon:master-pg9.6 --restart=Never --rm -- /usr/local/bin/stolonctl --cluster-name=kube-stolon --store-backend=consul --store-endpoints=consul-consul:8500 init`
     
then install all the pieces

    `kubectl apply -f rbac/`    
    `kubectl apply -f components`
    
Notes: 
* The resource files for the components expect to name your consul cluster `consul`. The internal kube DNS name will come from ${yourname}-${helmchartname}
* The setup uses an example `sorintlab/stolon:master-pg9.6`, for production this should be exchanged depending the needs and built from scratch using the provided [Dockerfile](/consul/kubernetes/image/docker), then referenced by the stolon yaml resource files.
    
    `make PGVERSION=9.6 TAG=stolon:v0.7.0-pg9.6`
  
* All keepers will need a PersistentVolume so repeat creation steps with fabric8
* Usernames and passwords kept as default, [secrets](https://kubernetes.io/docs/concepts/configuration/secret/) should be used for production environments.
* consul-ui is available as a NodePort service     
  
#### FAILOVER TESTING

* Postgres Cluster info:

    `kubectl run -i -t stolonctl --image=sorintlab/stolon:master-pg9.6 --restart=Never --rm -- /usr/local/bin/stolonctl --cluster-name=kube-stolon --store-backend=consul --store-endpoints=consul-consul:8500 status`

* Get proxy service IP (on the node):

    `PROXY_IP=$(kubectl describe svc stolon-proxy-service | grep IP: | awk '{print $2}')`
    `watch pg_isready -h $(PROXY_IP)`

* In another shell manual failover on the Master pod:

`watch kubectl delete pods $(name_of_the_master_pod)`

#### Other Notes

* The architecture of stolon is designed to be resilient until the last postgres instance is up in the cluster.
* It is running as a StatefulSet itself meaning there will be only once instance per node.
* By default all connections go to master, but you can leverage the standby replicas for read only purposes by adding a [service](/stolon/kubernetes/read-only-service/stolon-ro-keeper-service.yaml)
* To prevent data loss at the event of a failover, syncronous replication can be switched on. `kubectl run -i -t stolonctl --image=sorintlab/stolon:master-pg9.6 --restart=Never --rm -- /usr/local/bin/stolonctl --cluster-name=kube-stolon --store-backend=consul --store-endpoints=consul-consul:8500 update --patch '{"syncronousReplication" : true }'`

#### Backup
* Running from a node a full backup is easily achieved with the following command (depends on your postgres version, also substitue the services local ip):
    
    `docker run -e PGPASSWORD=replpassword -v /root/backup/data:/backup/data postgres:9.6.8 pg_basebackup -h $(PROXY_IP) -U repluser -D /backup/data`
    
This uses the replication user (and pass) to pull the data to the `/root/backup/data` directory on the node

* TODO: rewrite the backup command to a Kubernetes Job

* TODO: instructions for WAL Archiving to Object Storage or File System 
`

   