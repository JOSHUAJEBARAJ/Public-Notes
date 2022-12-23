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

