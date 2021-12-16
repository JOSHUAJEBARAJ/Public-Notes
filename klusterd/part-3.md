## Video Link
https://www.youtube.com/watch?v=Ps2CQm6_aZU&list=PLz0t90fOInA5IyhoT96WhycPV8Km-WICj&index=24






## Kluster 03 

```
kubectl version
```

> Shows some certificate error

```
kubectl get nodes
```

> Got some error

SSH into the control plane node

Try to see the process

```
ps aux | grep api-server
```

Figured out the there is clue is `etc/motd`



Now go to the pki folder

```
cd /etc/kubernetes/pki
```

Try to see the expiry date of the certificate


```bash
openssl x509 -enddate -noout -in <file-name>
```

> Find out the certificate is expired


Find out the there is hidden certificate file 

```
mv .new-crt old-crt
mv .new-key old-key
```


> Its not working 

Now restart the kubelet  to see make the changes 

> It's not working

Now kill the api server

```
kill -9 <api-sercer>
```

> Api server is not working

Now move out the api server static pod manifest outside of the folder and again put it back to get picket by the kubelet

Now go to the other control plane node 


Now run do the same for the copying the key 

Now instead of killing the server try to SIGHUP it will reload the server with certificate

```
kill -1 <api-server>
```

> Its started woking 


Now try to look at the nginx deployment

```
kubectl get pods 
```

> nginx working fine 

Now try to look at the service 

```
kubectl get svc
```

> Loadbalancer is pending state

Examine the svc

```
kubectl describe svc <name>
```

> IP allocation failed

There is problem is metallab


```
kubeclt get pods -n metallb
```

Try to look at the all resources

```
kubectl get all -n metallb
```

> Controller is in nodeAffinity 

```
kubectl logs deploy <controller-deploy>
```

> There is nothing could be found

> Hypotheseis there is  problem with configuration 


Now try to look at the configuration 

```
kubectl get cm -n kube-system
```

Now try to look at the secrets

```
kubectl get secrets -n kube-system
```

> There is packet-cloud config


```
kubectl -n kube-system edit secrets <packet-config>
```

Now try to edit the deployment to use the token 

```
kubectl -n kube-system edit deploy <packet-cloud-config>
```


> Changing the value resulted in the crahLoopBackOff and Couldn't have the work it done 

### Second problem 

kubectl run -i --tty dns --image=public.ecr.aws/i1j2j4g5/dnsutils -- bash

and then trying to `dig kubernetes`


Now try to look in the `resolve.conf`


> Skipping the cluster due to the timing issue


Complete solution can be found here 

## Kluster 006 


https://gitlab.com/rawkode/klustered/-/tree/main/006


```
kubectl get pods -A
```

The above command is working but when ssh we couldn't see anything 

SSH into the control plane node 

```
ps aux
```

> Hypothesis on ssh we are getting into the container not the host of the control plane 

Now ssh into the host machine by specfiying the port `2222`

- We get to the host 

Now the next problem
```
what creates pods objects
```


Now see the rs 

```
kubectl get rs
```

> there is something changes 45 hrs ago 

Now try to delete the rs and fix the issue 

> Getting error permission denied

Now see the cluster-rolebing

```
kubectl get clusterrolebinding
```

> Getting the permission error 

Now try to list the permission 

```
kubectl auth can-i --list
```

> There is nothing interesting 

Now try to find the api-server configuration

```
cd /etc/kuberentes/manifests
```

```
vim <kube-api-server>
```

> Found out the Image is different Image 

Restart the api server
```
kill -1 <api-server>
```

Repeat for the other two control plane 

Now looking at the `issuereport` file again 

```
Are we sure that is where the "real" manifest are ?
```

Now look at the kubelet configuration 

```
cd /etc/systemd/system
```

Found out the static pod path is
```
/var/lib/honk
```

Now change the value to the 
```
/etc/kuberentes/manifests
```

Restart the kubelet

```
systemctl restart kubelet
```

> Do it same for the other two control plane 


Now try to delete the rs it is working fine now 

```
kubectl delete rs <wp>
```

Now on port forwarding we are getting error 

> Now try to change the image pull policy

```
kubectl edit dp <wp>
```

Change it to the Always


Still Not working 

Hint
```
what turns a replicaset into pods?
```


> Guessing there is some problem in the control manager

Now try to look at the static pod

```
cd /etc/kubernetes/manifests/
```


```
vim kube-control-managet.yaml
```

Remove the negative value under the  controller flag flag

```
-replicaset
```

'foo' enables the controller named 'foo', '-foo' disables the controller named 'foo'.

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/


> Repeat the same for the other two nodes
