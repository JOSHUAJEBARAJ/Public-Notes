## Klusterd-011
https://www.youtube.com/watch?v=tmsqYWBTxEQ&t=1s

```bash
kubectl get nodes
```

> One worker node and one control plane is in  pending


```bash
kubectl get pods
```

> One pods is in terminating working

```
crictl pods
```

> Not working

```
cat > /etc/crictl.yaml

runtime-endpoint: unix:///run/containerd/containerd.sock

```


```
crictl pods
```

https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/

Port forward the pod service
```
kubectl port-forward  <workload> 8080:800
```

> Failed to connect to the database

Look at the DNS configurations
```
kubectl get pods -n kube-system
```

> core dns is in terminating state

Look at the configuration of the coredns
```
kubectl get cm -n kube-system coredns -o yaml
```


Now try to run the simple bash image

```
kubectl run -ti image=bash bash
```

> Permission denied

Now look at the api server configuration

```
cd /etc/kubernetes/manifests
```

Look at the api-server file 

Found out the there is `--admission-control-config-file`

Now look at the content of the file admission controller file 

```
cat /etc/kubernetes/demo/admission.json
```

Now look at the image pull policy file , found out there is configuration 

Decided to the remove the file since it is not related the configuration

> Remove the --admission-control-config-file flag

Now remove the `ImagepolicyWebhook`  enable-admission-plugins

#### Noderestriction  plugins
Every node has the same certificate , the above plugins allows to only access the resources belongs to the nodes

```
kubectl get secrets -A
```

> The above command will throw error due to the this plugin

Now again run the bash image 

> We get again another permission error

Now try to look at the custom resources 
```
kubectl get crd
```

> We have found kyverno running

```
kubectl get all -n kyverno
```

Now try to look at the policy 

```
kubectl get cpol
```

> found out the there is policy

Found out that there is policy denies the pod creation 

Now try to find out how to run the enable pod running

```
kubectl explain cpol --recursive
```

> Decided to delete the policy insetead of editing

Now delete the cpol 

```
kubectl delete cpol -A -all
```

Now try to run the pods, we are still not able to run the pods

Now try to see why the  pod is not in NotReady state, now ssh into the one of the one 

```
systemctl status kubelet
```

> found out it is not in good status

Now try to see ths logs

```
journalctl -flu kubelet
```

Found out there is some typo in the config

```
vim /var/lib/kubelet/config.yaml
```

Fix the `api-server` typo and fix the cert location


Now restart the kubelet

```
systemctl restart kubelet
```


Now its working :) 

Now ssh into the other nodes

```
systemctl status kubelet
```

> Found out there is problem in the kubelet

Now see the logs

```
journalctl -flu kubelet
```


Found out there is typo in the configuration

```
systecmctl cat kubelet
```

Try to look at the environment variable

```
vim /var/lib/kubelet/kubeadm-flags.env
```

Fix the typo and restart the kubelet 

Now the nodes is ready but scheduling disabled

```
kubectl uncordon <node-name>
```

Now the node is in working state,Still the database is unreachable

Now try to look at the chaos metrics

```
kubectl get all -n litmus
```

Hypothesis Chaos operator take all the resources , so we are going to delete the dep 

```
kubectl delete deployement.apps -n litmus
```

Now still the the bash pod in  pending state, Now look at the scheduler config

```
vim /etc/kuberentes/<scheduler>
```

> Couldn't find anything supscious

Now try to run the dep instead of pods

```
kubectl create deployment test --image=nginx
```

Now on describe the deployement

```
kubectl describe deployment test
```

> Found out rs is created but not the pod

Now try to looks at the logs 

```
kubectl logs -n kube-system kube-scheduler
```

Found out port number is wrong

Now try to the see the config

```
vim /etc/kuberentes/scheduler.conf
```

Fix the port number 

Now restart the scheduler 

```
mv manifests/kubescheduler.yaml .
```

Now again move it to the manifest folder


Now try to see the pods it is still in creating state

```
kubectl describe pod <bash>
```

Now we will see some error regarding registry 

Now ssh into the worker node now try to see the resolve.conf file 

```
cat /etc/resolve.conf
```

> found out the worker node is having different ip 


Fix it and now the bash is running successfully , Now get the shell inside the bash pod

```
apt update 
```

Found out it is not working 

> Now try to look at the netpol


```
kubectl get netpol
```

Found out there is netpolicy that denies ingress and delete it 

```
kubectl delete netpol <policy-name>
```

Now the application is working fine 


## Kluster 013

```
ln -sfn /etc/kuberentes/admin.conf ./kube/config
```



```
cat > /etc/crictl.yaml

runtime-endpoint: unix:///run/containerd/containerd.sock

```


```
crictl ps
```
https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/

> found out there is no api server


```
cd /etc/kubernetes
```

Found out there is couple of file folder  move the etcd and api server  to the manifest folder 

```
crictl pods
```

Now the pods is working

```
kubectl get nodes
```

> Still not working

Now try to see the logs of the containers

```
cd /var/lib/conatiners/
```

Now look at the logs of the containers 

We found the etcd gets the permission denied


```
cd /var/lib/etcd
```

```
ls -al 
```

> Found out the file has 


```
chattr --help
```

```
lsattr db
```

> This tells whether the file is mutable or not 

Found out it is immutable

```
chattr -i db
```
Now try to restart the static pods 

```
touch /etc/kuberentes/manifests/etcd.yaml
```


Now try to list out the pods

```
crictl pods
```

> System starts behaving very slow and assuming its due to low memory

```
top
```

This shows there is process `kswap`

so we are trying to turn off the swap

```
swapoff --all
```

We are still facing the same issue

Now stopping the kubelet , and move the api server out of the manifests folder

Now turn restart the kubelet , still facing the same issue 

Now trying to find the systemd process taking more resources

```
systemd-cgtop
```

Now try to see the free space

```
free -h 
```

*Hint :That's a lot of memory, and a lot of free memory, but not a lot of available memory,you can tune the kernel virtual memory*

```
sysctl -a | grep -i mem
```

```
sysctl -a | grep -i vm
```

> Found out there is min free kbytes

Now echo the value

```
echo <large int> >> /proc/sys/vm/min_free_kytes .
```


https://linuxhint.com/vm_min_free_kbytes_sysctl/

Now everything working fine now move back the api server manifest 

```
kubectl get nodes
```

> Two nodes are not ready

Controller and scheduler is not working 

Tried look at the manifest couldn't able to find anything 

Now try to look at the resources deployed at the pods
```
kubectl api-resrources
```


Now ssh into the other nodes

```
systemctl status kubelet
```

> Kubelet working fine 

Now look at the  logs

```
journalctl -flu <kubelet>
```

> Found out containerd is not running

Now restart the containerd
```
systemctl restart containerd
```

Now try to look at the pods

```
crictl pods
```

> Still containers is not running

One Node is working , Now ssh into the other node

```
systemctl status kubelet
```

> Kubelet working fine

Now see the logs

```
journalctl -flu kubelet
```

> The service is getting killed

Now look at the unit file 

```
systemctl cat kubelet
```

We have found the service has memory limit of 64k, remove the memory and restart the service 

Now the one node is working and other one is not working 

ssh into the previous node

```
journactl -flu kubelet
```

> Found out there is cni error says too many open file 

Restarting the kubelet fixes the issue
