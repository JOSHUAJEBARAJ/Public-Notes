## Video Link 

https://www.youtube.com/watch?v=JzGv36Pcq3g&list=PLz0t90fOInA5IyhoT96WhycPV8Km-WICj&index=25

### Kluster -4

```
kubeclt get nodes 
```

> Found out two nodes is not working 

```
kubectl get nodes -o wide
```

ssh into the nodes

- Check kubelet is up and running

```
ps aux | grep kubelet
```
 
 > Kubelet is running 

See the kubelet logs
```
journalctl -fu kubelet.service
```

> saw some connection error 

Check the components is running 

```
ps aux | grep "kube"
```

> Found out the scheduler , controller and kubelet working fine


Check for etcd 

```
ps aux | grep "etcd"
```

> Etcd is not running 

Check the etcd logs
```
cd /var/log/containers
```

> It looks like the etcd binding issues

Looks for the etcd configuration 

```
cd /etc/kubenetes/manifests
```

> FInd out the etcd version is difererent and update it 


```
ps aux | grep "etcd"
```

> Etcd is working but api is not working 

Try to look at the api-server logs 

```
cd /var/log/containers
```

> Api server came up after minutes


#### Fixing the worker node
- Ssh into the worker node
- Check the status of kubectl 
```
systemctl status kubectl
```
> Kubelet is nor working 

checking the logs 
```
journalctl -fu kubelet.service
```

> Found out the unknow flag

Fix the systemd service file

```
cd /etc/systemd/system/kubelet.service.d/
```

```
vim 10-kubeadm-conf
```

> Kubelet is not working

Instead of running as the service try to run on the command line

```
kubelet --kubeconfig=/etc/kuberentes/kubelet.conf --conf=/var/lib/kubelet/config.yaml
```

> Cannot connect the docker 


look at the kubelet configuration

> Hypothesis the docker is used instead of using containerd

Figured out the worker node is using the old version of kubelet


### Takeways
- Checks for version 
- If api server is not working then probably checks for the etcd

----
### Kluster -5 

Check for the api server 


```
kubectl get pods
```

> Api server not responding

Check for  resources using ps

```
ps aux | grep "kublet"
```

> kubelet is working 

```
ps aux | grep "etcd"
```

> etcd is working 

Check for the admin.conf

```
vim /etc/kubernetes/admin.conf
```

Get the ip and port 


```
netstat -an | grep 6443
```

Api server is running 


```
kubectl get nodes
```

> etcd timeout 

Try to look out the etcd log 

```
cd /var/log/containers
```

```
cat <etcd-container>
```

> Could't find anything suspicious

Try to see the etcd manifest 

```
cd /etc/kubernetes/manifests
```

```
vi etcd.yaml
```

Install the etcdctl

```
 ETCD_VER=v3.5.1

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
/tmp/etcd-download-test/etcdutl version

```

https://www.devops.buzz/public/kubernetes/etcd-cheat-sheet
```
etcdctl endpoint health 
```

> We could able to interact with the etcd using the client 


https://etcd.io/docs/v3.2/op-guide/maintenance/

```
etcdctl alarm list 
```

> No space 

```
etcdctl --write-out-table endpoint status 
```


> Find out there is no space 


Add the below in the static manifest of etcd

```
-quota-backend-bytes=$((1*1024*1024))

```

> Pod will be automatically restarted by the kubelet 

After sometimes on looking logs its found out that etcd is messed up 

Again try to add the new value


```sh
-quota-backend-bytes=1000000
```

> its still not working 

The below commands still throw the alarm

```
etcdctl --write-out=table endpoint status 
```

Now try to compact 

Get the revision using the below command 

```
etcdctl --write-out=json endpoint status | grep revision 
```

Now compact the revision 

```
etcdctl compact <revision-number>
```

> but still throwing error 

Now try to go into the other machine 

Perform the compact 

> But still same isssue 


No remove the quota that we applied in the previous step 

```
-quota-backend-bytes=$((1*1024*1024))
```

Now on executing the below comand

```
kubectl get pods 
```

> Its  working  But kubelet pod is crashing

```
kubeclt -n kube-system describe <kubelet-pod>
```

> Shows some permission error 


Now get the node ip using the below command 

```
kubeclt -n kube-system describe <kubelet-pod> -o wide
```

SSH into the node

Looking on the etcd logs still show the space issue 


```
etcdctl --write-out=table endpoint status 
```

> Still shows the space issue 


Later found out that etcd alarm should be removed manualy 

```
etcdctl alarm list 
```

```
etcdctl alarm disarm
```

Try to port-forward the worpress and see it working 
> No it didn't there is some error with service 


```
kubectl get pods -n kube-system | grep "dns"
```

> Coredns is not there 


```
kubectl get deploy -n kube-system
```

> Two replica is not creating 


Now killing the api-server

```
kill -9 <api-server-pid>
```

> Api-server again comes online but still the same issue


Now figured out the service is not reachable 

Delete the Network policy
```
kubectl get netpol -A
```

```
kubectl delete  netpol <name>
```

>Still not working 

Looking for the psp

```
kubectl get psp 
```

Delete the no-privelege psp


```
kubectl delete psp <no-privilege>
```


Read the complete one here 

https://gitlab.com/rawkode/klustered/-/tree/main/005



### Takeaway

- Look for etcd space
- Always remove the alarm in the etcd its not removed by default 
- Look for psp and Network policy if the service discovery is not working 
