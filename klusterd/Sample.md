In this episode, Talos team starts to fix the Kubernetes cluster broken by the redhat team 



### Corrupted binary

First, they configured the kubectl to connect to the cluster

```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```
Next, they tried to investigate the status of the nodes.  

```shell
kubectl get nodes -o wide
```
On running the `kubectl` command they got a permission denied error message. 

In order to fix the issue , they tried to give permission to the `kubectl` binary using the `chmod` command

```shell
chmod +x /usr/local/bin/kubectl
```
Surprisingly they got the permission denied for the `chmod` binary too  !

They tried to verify whether they have enough permission to run the `chmod` command

```shell
whoami
```

On running the above command they got the output as `root` , which means they have enough permission to run the `chmod` command. 

The team tried to figure out how to give permission to the `chmod` binary, since the chmod binary itself doesn't have the necessary permission to run

The team tried to reinstall the coreutils package

```shell
apt reinstall coreutils
```

They found out they can't reinstall the package due to lack of permission on the `echo` binary.

Based on the hint present in the host machine , the team decided to leverage the `ld-linux-x86-64.so.2` dynamic linker to run the `chmod` binary

```shell
/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 /bin/chmod +x /bin/chmod
```

They still facing the issue on executing the `chmod` binary  . But they found out without using the +x flag , they can run the chmod binary

```shell
/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 /bin/chmod
```

Now they tried to list the extended attributes of the `chmod` binary

```shell
lsattr /bin/chmod
```

They found out the binary file is immutable

They tried to remove the immutable attribute

```shell
chattr -i /bin/chmod
```

They are not able to remove the immutable attribute

So they tried to remove the immutable attribute from the file using the help of dynamic linker 

```shell
/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 /bin/chattr -i /bin/chmod
```

Now they are able to remove the immutable attribute from the binary file

Now they have the permission to run the `chmod` binary

```shell
/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 /bin/chmod +x /bin/chmod
```

Now they are able to run the `chmod` command 

After that they tried to give the permission to the `kubectl` binary

```shell
chmod +x /usr/local/bin/kubectl
```
> They got the permission denied error

The team found out that the `kubectl` binary is immutable, so they tried to remove the immutable attribute from the binary file

```shell
chattr -i /usr/local/bin/kubectl
```

Now they are able to remove the immutable attribute from the binary file and set the permission to the binary file

```shell
chmod +x /usr/local/bin/kubectl
```

On running the kubectl get nodes command , they got the error message. 

The team decided to investigate the error by verifying the authenticity of the kubectl binary

```shell
cat /usr/local/bin/kubectl
```
They found out the kubectl binary is a simple shell script that prints out the error message 

The team decided to fix the issue by replacing the kubectl binary with the original binary file

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl
```

```bash
chmod +x ./kubectl
```

```bash
mv ./kubectl /usr/local/bin/kubectl
```

### Invalid certificates

On running the kubectl get nodes command , they got the error message saying the certificate is invalid

The team tried to resolve the  issue by renewing  the certificate using the kubeadm command 

```bash
kubeadm certs renew admin.conf
```

They still got  the same error message

The team decided to renew all the certificates using the kubeadm command

```bash
kubeadm certs renew all
```
Inorder to take effect of the renewed certificates, they have to restart the kubernetes components

They decided to use the crictl command to restart the kubernetes components by killing the pods 

Based on the suggestions ,the team decided to restart the services by moving the files from the /etc/kubernetes/manifests directory to the /tmp directory

```bash
mv /etc/kubernetes/manifests/* /tmp
```
By making the changes , the team lost access to the teleport server

David decided to fix the issue by moving back the files from the /tmp directory to the /etc/kubernetes/manifests directory

```bash
mv /tmp/* /etc/kubernetes/manifests
```
But still the kubernetes components are not running 

The team decided to investigate the issue by checking the logs of the kubelet service

```bash
journalctl -flu kubelet
```

On investigating the logs they couldn't finding anything useful in the logs.They decided to look upon the kubelet config

```bash
cat /var/lib/kubelet/config.yaml
```

They checked for the `manifest` attribute in the config file to make sure they are pointing to the correct directory
but it was pointing to the correct directory.

The team decided to restart the kubelet service

```bash
systemctl restart kubelet
```

On restarting the kubelet service the Kubernetes components are running fine

Now on running the kubectl get nodes command , they got the below error message

```bash
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

They fixed it by setting the `KUBECONFIG` environment variable

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

Now on looking at the status of the nodes, Looks like everything is working fine 


### Cilium Custom Network Policy

Now on looking at the application they got the error saying `Failed to connect to the database

The team decided to investigate the service manifest file

```bash 
kubectl get services postgres -o yaml
```

Everything looks fine there, The team decide to look for the network policies 

```bash
kubectl get networkpolicies
```
They found out that there is no network policy present in the default namespace

The team decided to look at the kubelet config to check whether there is any missing configuration, but they didn't find anything useful. The team decided to get the shell inside the pods to investigate the issue

```
kubectl exec -ti deployment/klustered -- bash 
```

They tried to ping the Postgres service from the klustered pod , but they are not able to ping the service

``bash
ping postgres
```

They tried the same thing for the Kubernetes but they are still not able to ping the service

```bash
ping kubernetes
```

The team decided to check the cilium config 

```bash
kubectl get configmap cilium-config -o yaml
```

They don't find anything interesting there, Based on the chat feedback they decided to check the custom network policy from the cilium


```bash
kubectl get crd
```

They found that there is a custom resource definition for the cilium network policy

```bash
kubectl get ciliumnetworkpolicies -A
``` 
They found that there are custom network policies to restrict the traffic to the postgres service

They decided to delete the cilium network policy and check whether the traffic is working fine or not

```bash
kubectl delete ciliumnetworkpolicies postgres
```


They still got the same error message, so they decided to delete all the cilium network policies and check whether the traffic is working fine or not

```bash
kubectl delete ciliumnetworkpolicies --all -n kube-system
```

The application is still not working 

The team decided to look for the cilium config for the policy parameter

```bash
kubectl get configmap cilium-config -o yaml | grep policy 
```

On looking at the `enable-policy` parameter , they found that its set to `default` which blocks the traffic to the postgres service , on changing the values to `never` and restarting the cilium pods, the application is working fine

Now in the second part of the challenge, they have to find the upgrade of the version of the application from v1 to v2 . 

The team decided to edit the manifest and change the version from v1 to v2 by changing the image name this fixed the issue


## Team Redhat 


The RedHat team started with the investigation by checking the status of the nodes

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## Unreachable API server

```bash
kubectl get nodes
```

They got the below error message

```bash
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```


> The team configured the alias for the kubectl command

```bash
vi ~/.aliases
```

```bash
alias oc=kubectl
```

```bash
source ~/.aliases
```

The team decided to verify the ip address of the API server using ip addr command

```bash
ip addr
```

The team decided to check the process running on the host 

```bash
ps -ef | grep kube
```

Looks like the api server is not running on the host. The team verified the presence of the kube-apiserver manifest file in the /etc/kubernetes/manifests directory

```bash
ls -l /etc/kubernetes/manifests
```

And also looked at the api server manifest file , everything looks fine .

The team decided to check the logs of the kubelet service

```bash
journalctl -flu kubelet
```
They found out there is an issue with starting the api server

The team further decided to dig deep into the kube api server manifest file

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

The team checked for the `image` name and verified that the image is a valid image


The team decided to look at the logs of the kube api server

```bash
cd /var/log/containers
```

```bash
tail -f kube-apiserver-*.log
```

## Broken etcd server

The team found out that the api server is not able to connect to the etcd server

The team decided to check the etcd logs

```bash
tail -f etcd-*.log
```
On looking at the logs they got the below error message

```bash
no leader (status code 503)
```

The team decided to check the etcd manifest file

```bash
vim /etc/kubernetes/manifests/etcd.yaml
```

They found out that the `--listen-client-urls` parameter is not set to the correct value , they fixed the issue by removing the localhost and keep only the private ip address

After making the changes they found  out the etcd server is running fine

```bash
tail -f etcd-*.log
```

The team decided to change the `--etcd-servers` parameter in the kube api server manifest file to the private ip address of the etcd server

After looking at the logs of the kube api server they found out that the api server is still not running fine. 

They tried to connect to the etcd using the netcat command

```bash
nc -z 10.80.73.137 2379
```

They found out it is working fine

But after some time they found out that the etcd server is still  not running fine and showing the same error message


The team decided to use the etcdctl command to check the status of the etcd server

```bash
export ETCDCTL_API=3
export ETCDCTL_CA_FILE=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT_FILE=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY_FILE=/etc/kubernetes/pki/etcd/server.key
```

```bash
etcdctl get /registry/services/specs/default/kubernetes
```
https://lzone.de/cheat-sheet/etcd


The team found out that the etcdctl is not able to connect to the etcd server due to the previous changes in the etcd manifest file. 

They decided to change the `--listen-client-urls` parameter in the etcd manifest file to the correct value ie with localhost and private ip 

They repeated the same for the kube api server manifest file 

Now on trying to connect to the etcd server using the etcdctl command they found out that the etcd server is still not running fine

The team decided to check the health of the etcd server using the etcdctl command

```bash
etcdctl endpoint health
```

They found out that the etcd server is not healthy

The team again checked the logs , but they could not find anything interesting. 


The team decided to pick up the hints, on sshing to the worker node they found there are some hints files present there.

On looking at the first hint they found the below text 

```bash 
#Hint 1
- Quorum praesentia sufficit
```

The team figured out that etcd is currently working in the HA mode with multiple members, they decided to fix the issue by removing the defective etcd member from the cluster.

On looking at the logs they found one of the etcd member is not running fine

They decided to remove the etcd member from the cluster

```bash
etcdctl -C https://<surving host IP address>:2379 --ca-file=/etc/etcd/ca.crt --cert-file=/etc/etcd/peer.crt member remove <defective member ID>
```
They got the below error message

```bash
Unknown short flag '-C'
```

The team fixed the iuu by changing the `-C` parameter to `--endpoints`

```bash
etcdctl --endpoints=https://<surving host IP address>:2379  member remove <defective member ID>
```

The team decided to look upon the another hint on looking at the second hint they found the below text

```bash
- `etcd` snapshot stored in `/var/etcd.db`
```

The team decided to restore the etcd snapshot 

```bash
etcdctl snapshot restore /var/etcd.db
```

> They got the error on trying to restore the snapshot

They tried to list the members of the etcd cluster

```bash
etcdctl member list
```

> They again got the error on trying to list the members of the etcd cluster

The team tried to restart the etcd server by moving the etcd manifest file to the /tmp directory

```bash
mv /etc/kubernetes/manifests/etcd.yaml /tmp
```

Now they again moved the etcd manifest file to the /etc/kubernetes/manifests directory

```bash
mv /tmp/etcd.yaml /etc/kubernetes/manifests
```

Now the team tried to restore the etcd snapshot

```bash
etcdctl snapshot restore /var/etcd.db
```

They got the below message. 
```bash
Deprecated; Use "etcdutl snapshot restore" instead.
```

The team decided to remove the existing etcd data directory and restore the snapshot

```bash
rm -rf /var/lib/etcd
```

```bash
etcdutl snapshot restore /var/etcd.db --data-dir /var/lib/etcd
```

Now the team tried to list the members of the etcd cluster

```bash
etcdctl member list
```

Now they are able to list down the members of the etcd cluster

Now on running the below kubectl command they found out that they are not able to connect to the API server

```bash
kubectl get nodes
```

Now the team decided to check the manifest of the kube api server

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

They couldn't find anything useful . 

They tried to look at the logs of the kube api server

```bash
tail /var/log/containers/kube-apiserver-*.log
```

They found out that everything is fine with the kube api server

Now again they tried to connect to the API server using the kubectl command

```bash
kubectl get nodes
```

Now they are able to list down the nodes.

Now the team decided to look at the all running pods

```bash 
kubectl get pods --all-namespaces
```

They found out at the cilium pod is restarting every 5 seconds

But they are able to access the application pod successfully. 

## Corrupted CNI

Now they tried to solve the challenge by replacing the application image with version 2

On editing the `image` tag in the deployment manifest file they found out that the application is not running fine

They found at worker node 2 is not running fine. They decided to override the scheduling of the application pod to worker node 1 using Nodeselector.

On looking at the pod status , they found out it is still in the pending state

On describing the worker node 2 they found at the kubelet is not running fine

The team decided to ssh into worker node 2 and check the kubelet logs

On looking upon the status of the kubelet they found out that the kubelet is running fine

```
systemctl status kubelet
```

On looking at the logs of the kubelet they found nothing interesting , so they decided to restart the kubelet

```bash
systemctl restart kubelet
```

The team decided to look at the events on the kubernetes, they found out there is some issue with the CNI plugin

```bash
kubectl get events 
```

The team decided to look at the Hints file on worker node 2 , they found the below text

```bash
Its not CNI, there's no way its CNI
, it was CNI.
```

On looking at the other hints file they found the below text

```bash
In `/opt/cni/bin` there are a lot of files. Running `loopback`, gives an output that's wild
```

On looking at the `/opt/cni/bin` directory they found out there are couple of files , they decided to run the `loopback` file

```bash
/opt/cni/bin/loopback
```
On running the `loopback` file they got the below message

```bash
CNI bridge plugin version 0.8.2
```

On running the `./loopback.bak` file they got the below message

```bash
CNI loopback plugin version 0.8.2
```

The team decided to replace the `loopback` file with the `loopback.bak` file

```bash
mv /opt/cni/bin/loopback.bak /opt/cni/bin/loopback
```

On replacing the file it worked fine , they were able to access the application pod successfully.










