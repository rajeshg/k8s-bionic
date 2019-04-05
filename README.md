# k8s-bionic
Kubernetes cluster setup using 3 ubuntu (bionic 18.04) nodes. Nodes are provisioned using Vagrant

## Vagrant file
- Vagrant file provisions 3 nodes (master, worker1, worker2) with bionic64 base vm
- Installs docker, kubernetes
- Sets up the necessary services
- Initializes the kubenetes cluster with kubeadm
- Sets up a calico network for the cluster

## Selenium grid is setup using 4 files
They can be run using the following commands

```shell
kubectl create --filename=examples/selenium-hub-deployment.yaml
watch -n 1 kubectl get pods # wait for the selenium hub pod to be in Running state
kubectl create --filename=examples/selenium-hub-svc.yaml
kubectl create --filename=examples/selenium-node-chrome-deployment.yaml
kubectl create --filename=examples/selenium-node-firefox-deployment.yaml
watch -n 1 kubectl get pods # wait for the selenium firefox and chrome pods to be in Running state. The necessary containers are fetched from the registry if required
```

## Miscellaneous
By now, the selenium hub and the associated nodes should be up and running. The selenium hub runs on one of the nodes. To identify the hub url, run the following on the shell:
```shell
NODENAME=`kubectl get pod -ojsonpath='{.items[0].spec.nodeName}' -lapp=selenium-hub`
NODEPORT=`kubectl get svc selenium-hub -ojsonpath="{.spec.ports[0].nodePort}"`
echo "http://$NODENAME:$NODEPORT/grid/console"  # this is the selenium grid URL
```
