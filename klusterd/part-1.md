
## Video Link 
https://www.youtube.com/watch?v=teB22ZuV_z8&list=PLz0t90fOInA5IyhoT96WhycPV8Km-WICj&index=24

#cni , #kubelet

## Kluster -1 
Based on **CNI**


The below command verifies the api working 
```
kubectl get nodes
```



The below command is used to see the state components
```
kubectl get cs
```

> Scheduler and control manager is not working  and its depricated 	
https://github.com/kubernetes/enhancements/issues/553

See the whether the scheduler and control manager pod pod is in pending state 

```
kubectl get pods -A | grep Pending
```

> No pod is in pending state



Now run the simple nginx pod 

```
kubectl run nginx --image=nginx
```

> Container can't able to run 

Let's see the probes is correctly configured

```
kubectl describe -n kube-system <scheduler-pod> | less
```

Lets Investigate more on the API-Server

- SSH into the Master Node

```
cd /etct/kubernetes/manifests
```

check the configuration of the control manager
```
vi <kube-control-manager>
```
> found no error


```
kubectl get events
```

Found out there is problem is. CNI


```
kubectl get pods -A | grep cilium
```

> Cilium Pod is in crashloop back 

```
kubectl -n cilium describe pod
```
Try to find the reason 
```
kubectl -n cilium describe pod | grep Reason
```
Get the logs of the previous logs
```
kubeclt -n cilium logs -p -f pod-name
```

> Hypothesis there must be some error in the cilium configuration 


```
kubectl get all -n cilium
```

Edit the configmap

```
kubectl -n cilium edit cm <config-map-name>
```

> Found out iptables-rules is repeated twice , remove the complication 

Remove the false one 
Delete the all pods  in the cilium ns
```
kubectl -n cilium delete pod -all
```

#### Takeaway
- Always looks for the crashing pods
- Always try to looks at the events and logs (including previous one)
- Try to look at the configuration for typos

## Kluster -2 

https://gitlab.com/rawkode/klustered/-/tree/main/002
 Based on kubelet

See whether the api-server is running
```
kubectl get pods 
```

> Api server is running

Get the API-address

```
kubectl config view
```

Curl the API

```
curl -V <API>
```

Couldn't able to ssh into the nodes

Scan the ip address
```
nmap -sU <ip-address>
```

See the ipatables

```
iptables -L
```

See  all the running pods

```
crictl pods
```

> No result

See the status of continerd

```
systemctl status containerd
```

See the status of the kubelet 

```
systemctl status kubelet
```

> Kubelet is not running

See the logs

```
journctl -u kubelet.service --no-pager
```


See the kubelet config

```
cd /var/lib/kubelet
```

See the configuration

```
ls config.yaml
```


Discussion on why swap off is needed 

https://discuss.kubernetes.io/t/swap-off-why-is-it-necessary/6879

> Found out the error swap
```
lsblk
```

```
swapoff /dev/sdb2
```


Restart the kubelet

```
systemctl status kubelet
```


### Alternative methods

https://github.com/kubernetes/kubernetes/issues/53533

You can start with the kubelet with the swap enabled  using the below flag




```
--fail-swap-on=false
```

- Add the swap on
-   containers which do _not_ specify a memory requirement will then by default be able to use all of the machine memory, including swap.


Install the strace

```
apt install strace
```


Find out why the command is not running

```
strace crictl ps
```

Try to look at the kubelet configuration service 

```
cd /etc/systemd/system/kubelet.service.d/
```
Look at the file 
> Sample service targel file 
```
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

> The Api server is started and died again


Try to look at the logs

```
cd /var/log/containers
```

```
tail -f <container-name>
```

```
ps aux | grep kubelet
``` 

> Found out the evict hard parameter is set on the kubelet

Fix the kubelet using the** EnvironmentFile**

```
vim /etc/default/kubelet
```

Restart the kubelet 
```
systemctl deamon-reload & systemctl restart kubelet
```

Stop the ufw  firewall
```
ufw distable 
iptables -F
```

> One control manager is in crashloopback phase

```
kubectl logs --previous -f <pod-name>
```

> find outs there is some error in cluster-cidr


```
cd /etc/kubernetes/manifests
```

```
vim kube-controller-manger.yaml
```

#### Takeaway
- Look at the host configuration 
- If the pods is deleted looks for eviction rules
- Try to look at the network policy