## Rough notes 


## Break 1 

In this section Marino starts to fix the cluster 

After configuring the kubectl config file on looking at the status of pods he notices the pods status is `ContainerStatusUnknown`

On trying to describe the pod he notices the following error

```
Message: The container could not located when the pod was terminated. 
```

Next he tries to look at the logs of the pod and notices the following error

```
Error from server (BadRequest): container "klustered" in pod "klustered-7f7f7f7f7f" is terminated 
```

Next he decides to look at the deployment yaml and tries to see if he can find anything wrong with it

```
kubectl get deployment klustered -o yaml | less 
```

On describing the pod he notices the following error

```bash 
The node was low on resource: ephemeral-storage. 
```

> Fun fact Ki is the symbol for Kibibyte which is 1024 bytes

He then tries to look at the deployment yaml and notices the following error

```bash
Message : 'Internal error occurred: failed calling webhook "validate.kyverno.svc-fail": Post https://klustered-webhook.klustered.svc:443/validate?timeout=30s: dial tcp refused
```

Next he tries to look at the kyverno resources 

```bash
kubectl get kyverno -A 
```

Next he tries to look at all the validatingwebhookconfigurations 

```bash
kubectl get validatingwebhookconfigurations -A 
```

On running the above command he notices there are couple of validatingwebhookconfigurations is present so he decided to delete them 

Before deleting the ValidatingWebhookConfigurations he tries to save the yaml of the ValidatingWebhookConfigurations 

```bash
kubectl get validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg -o yaml > kyverno-policy-validating-webhook-cfg.yaml 
```


Next he deletes the ValidatingWebhookConfigurations 

```bash
kubectl delete validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg 
```

Next he deletes the kyverno namespace to remove the kyverno resources 

```bash
kubectl delete ns kyverno 
```


Now he tries to delete the pod again and tries to recreate the pod 

```bash
kubectl delete pod klustered-7f7f7f7f7f 
```


Someone in the chat suggest that he may working on a different cluster so he tries to check the context 

```bash
kubectl config get-contexts 
```

On running the above command he notices that he is working on a staging now he tries to switch to the production cluster 


```bash
kubectl config use-context production 
```

After switching to the production cluster he tries to look at the nodes 

```bash
kubectl get nodes 
```

He found one node is not ready . On describing the node he notices the following error 

```bash
The node was low on resource
```

Now they decided to ssh into the node and tries to look at the disk space 

```bash
df -h 
```

On running the above command he notices the disk space is running of space , next he tries to look at the largest files 

```bash
du -d1 -h 
```

The found /var has occupied 387G of space , next they cd into the /var and tries to look at the largest files 

```bash
du -d3 -h 
``` 

They found the ./log directory has occupied 387G of space , next they cd into the ./log directory and tries to look at the largest files 


```
ls -l 
```

They found the log4j2.log file has occupied 387G of space , next they tries to remove the log4j2.log file 

```bash
rm log4j2.log 
```


Now they tries to look at the node again 

```bash
kubectl describe node node-name
```

Now they notices the following error 

```bash
Invalid capacity 0 on image filesystem 
```

Now again on describing the node they notices the following error 

```bash
Kubelet not ready 
```

Now they ssh into the node and tries to look at the status of the kubelet 

```bash
systemctl status kubelet 
```

Everything looks fine on the kubelet side, now they tries to look at the kubelet logs 

```bash
journalctl -u kubelet 
```

To make logs more readable they tries to use the following command 

```bash
journalctl -flu kubelet 
```

They notices the following error 

```bash 
err=invalid Node allocatable configuration
```

Now they decided to look at the kubelet config file . To find the kubelet config file they tries to look at the kubelet service file 

```bash
systemctl cat kubelet 
```

Now they decided to look at the kubelet config file 

```bash
cat /var/lib/kubelet/config.yaml 
```

They couldn't find anything suspicious on the kubelet config file , next they tries to look at the /var/bin/kubelet/kube-adm.env file 

```bash
cat /var/bin/kubelet/kube-adm.env 
```

They couldn't find anything suspicious on the kubelet config file , next they tries to look at the /etc/default/kubelet file 

```bash
cat /etc/default/kubelet 
```

They found out the kubelet is configured with eviction hard memory.available=memory.available<100Gi , They decided to remove the eviction hard memory.available=memory.available<100Gi from the /etc/default/kubelet file 

```bash
rm /etc/default/kubelet 
```

Now on looking at the node they notices the worker node is ready but the control plane node is not ready . On removing the `/etc/default/kubelet` file the control plane node is ready

Now they tries to change the image from the v1 to v2 

Now on looking at the pod they notices the pod is running with old image without updating to the new image 

Other Guest gave a hint there may be a issue in the controller manager

They tried to manifest of the controller manager 

```bash
vi /etc/kubernetes/manifests/kube-controller-manager.yaml 
```

Now they tries to look at the controller manager logs 

```bash
kubectl logs -n kube-system kube-controller-manager-node-name 
```

They got the following error 

```bash 
kube-system/kube-controller-manager-node-name: container "kube-controller-manager" leases.coordination.k8s.io is forbidden: User "system:kube-scheduler" cannot list resource "leases" in API group "coordination.k8s.io" at the cluster scope
```

Now on looking at the controller manager manifest they notices the following thing 

```bash
- hostpath:
        path: /etc/kubernetes/scheduler.conf
        type: FileOrCreate
```

On replacing the `scheduler.conf` with `controller-manager.conf` they notices the controller manager is running fine


Now after restarting the kubelet and controller manager they tries to look at the pod again 

```bash
kubectl get pods 
```

This time the pod is running with the new image. On accessing the pod via service they notices the pod is running with the new image


52.57 

## Break 2 

First John is trying to see the health of nodes 

```bash
kubectl get nodes 
```

On running the above command he notices the API server is not responding , then he tries to look at the API server logs 

```bash
cd /var/log/containers 
```

```bash
tail -f kube-apiserver-node-name.log 
```
On looking at the logs they found the below error 

```bash
"command failed" err="open /etc/kubernetes/pki/apisever-etcd-client.key: no such file or directory" 
```


Now they tries to look at the actual file 

```bash
ls -lad /etc/kubernetes/pki/apissever-etcd-client.key 
```
 They fixed the issue by configuring the correct file name in the API server manifest file

 Now on looking at the nodes they notices the api server is not responding

 The api server didn't pick up the new changes. 

Now on they tries to look at the running container 

```bash
crictl ps 
```

They couldn't see the api server container running

Now they tried to look at the kubelet configuration to see if they points to the right manifest directory 

```bash
systemctl cat kubelet 
```


Everything looks fine but still the api server is not up and running

Now on restarting and looking at the kubelet logs they notices the API server in crashloopbackoff 


Next they tried to look at all the pods in node 

```bash
crictl ps -a 
```

They notices that the api server was exited a minute ago . 

Now they tried to look at the api server logs 

```bash
crictl logs 5b9f9e0b3e0f9 
```

They got the below error 

```bash
"command failed" err="x509: trailing data" 
```

Next they tried to look at the api server certificates

```
cd /etc/kubernetes/pki 
```

```bash
ls -lah
```

They notices the apiserver-etcd-client.crt is recently modified

They removed the extra data from the apiserver-etcd-client.crt file and restarted the kubelet

Now on running the crictl ps command we notices the api server is running fine

Now on trying to look at the pods they notices the api server is not responding

Now on trying to look at the admin.conf file they notices the server is pointing to wrong port number

```
grep -r 1337 .
```


On changing the port number to 6443 they notices the api server is responding fine


Now on looking for the control plane component they notices the cilium-operator is not running  and core-dns is in container creating state

Now on describing the klustered pod they notices the following error 

```bash
kubelet network is not ready 
```

Now they tried to delete the pod and recreate it 

```bash
kubectl delete pod --all -force --grace-period=0 
```

Now on looking at the pod log they notices the following error 

```bash
Failed to create pod sandbox: rpc error: Incompatible CNI version
```

Now they ssh into the node and tries to look at the CNI config file 

```bash
cd /etc/cni/net.d 
```

```bash
cat 05-cilium.conf 
```

They didn't find anything suspicious on the CNI config file


Now they tried to find the version of the CNI plugin installed on the node 

```bash
cd /opt/cni/bin 
```

```bash
./cilium-cni --version 
```

The team decided to reinstall the CNI plugin by moving the CNI plugin to a temporary directory and restarting the cilium daemonset

```bash
mkdir /tmp/cni
```

```bash
mv /opt/cni/bin/* /tmp/cni/
```

```bash
kubectl rollout restart ds cilium -n kube-system 
``` 

They deleted the pod and recreated it 

```bash
kubectl delete pod --all -force --grace-period=0 
```

Now on looking at the pod log they notices the same error 

Now they decided to reinstall the kubelet

```bash
apt-get install --reinstall kubelet 
```

Marino dropped a hint that to look at the output of the `kubectl get nodes -o wide` command

 On looking at the output they notices the worker node uses the different containerd version 

 They fixed that by sshing into the worker node and installing the same containerd version as the control plane node

 Once it is done they are able to see the pod running fine 

 In the end Marino reveals that it is the known issue with the containerd version with the cilium CNI plugin


## Interesting links 
1. Kubernetes debugging CLI https://gist.githubusercontent.com/sontek/5b31111d56d30a48dca764fe72fd9b01/raw/e8c51a1e50a5d039b9270e7930c69913c5b87aac/klustered.sh 
