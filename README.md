# Arpspoof proof-of-concept

This is intended to demonstrate how we can support the arpspoof example in
Workbench, but running in raw Kubernetes.

## Bootstrap
Bootstrap Kubernetes on an Ubuntu VM following the data8 kubeadm-bootstrap
instructions. We've forked it to modify the default network in
`init-master.bash`, changing from Flannel to Weave. Weave seems to support
Network Policy required for this project but is not as restrictive as Calico.

```
git clone git clone https://github.com/cheese-hub/kubeadm-bootstrap
cd kubeadm-bootstrap
sudo ./install-kubeadm.bash
sudo -E ./init-master.bash
```

At this point, you should have a Kubernetes "cluster" running with Weave. 
	
```
$ kubectl get nodes
NAME          STATUS    ROLES     AGE       VERSION
hostname   Ready     master    1h        v1.9.2
```

```
$ kubectl get pods --all-namespaces 
NAMESPACE     NAME                                  READY     STATUS    RESTARTS   AGE
kube-system   etcd-xxxxxxx-dev                      1/1       Running   0          1h
kube-system   kube-apiserver-xxxxxxx-dev            1/1       Running   0          1h
kube-system   kube-controller-manager-xxxxxxx-dev   1/1       Running   0          1h
kube-system   kube-dns-6f4fd4bdf-lrp4f              3/3       Running   0          1h
kube-system   kube-proxy-w747d                      1/1       Running   0          1h
kube-system   kube-scheduler-xxxxxxx-dev            1/1       Running   0          1h
kube-system   weave-net-wjkwj                       2/2       Running   0
```

## Create the test namespaces
We're going to create two namespaces to deploy the `arpspoof` example
containers to test whether we can isolate network communications.

```
kubectl create ns test1
kubectl create ns test2
```

## Deploy the arpspoof pods
```
kubectl create -n test1 -f arpspoof
kubectl create -n test2 -f arpspoof
```

Wait for them to enter a `Running` state:
```
$ kubectl get pods -n test1 -o wide
NAME      READY     STATUS    RESTARTS   AGE       IP          NODE
hacker    1/1       Running   0          1h        10.32.0.3   hostname
server    1/1       Running   0          1h        10.32.0.4   hostname
victim    1/1       Running   0          1h        10.32.0.5   hostname

$ kubectl get pods -n test2 -o wide
NAME      READY     STATUS    RESTARTS   AGE       IP          NODE
hacker    1/1       Running   0          1h        10.32.0.7   hostname
server    1/1       Running   0          1h        10.32.0.6   hostname
victim    1/1       Running   0          1h        10.32.0.8   hostname
```

## Test the arpspoof scenario (via `exec`):

`exec` into the `victim` pod and run `traceroute` to `server`:
```
kubectl exec -it -n test1 victim bash
root@victim:~# traceroute 10.32.0.4
 1  10.32.0.4 (10.32.0.4)  0.109 ms  0.048 ms  0.044 ms
```

In a separate shell, `exec` into `hacker` pod and run `arpspoof`:
```
kubectl exec -it -n test1 hacker bash
root@hacker:/app# arpspoof -i eth0 -t 10.32.0.5 10.32.0.4
c2:e7:56:5d:18:bb 2a:c:5b:b6:52:1c 0806 42: arp reply 10.32.0.4 is-at c2:e7:56:5d:18:bb
c2:e7:56:5d:18:bb 2a:c:5b:b6:52:1c 0806 42: arp reply 10.32.0.4 is-at c2:e7:56:5d:18:bb
c2:e7:56:5d:18:bb 2a:c:5b:b6:52:1c 0806 42: arp reply 10.32.0.4 is-at c2:e7:56:5d:18:bb
```

In the `victim` shell, re-run the `traceroute`:
```
root@victim:~# traceroute 10.32.0.4
traceroute to 10.32.0.4 (10.32.0.4), 30 hops max, 60 byte packets
 1  10.32.0.3 (10.32.0.3)  0.170 ms  0.158 ms  0.098 ms
 2  10.32.0.4 (10.32.0.4)  0.123 ms  0.116 ms  0.133 ms
```

In the hacker shell, try to access the server pod running in the `test1` and
`test2` namespaces:

```
root@hacker:/app# curl 10.32.0.4
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="https://10.32.0.4/">here</a>.</p>
<hr>
<address>Apache/2.4.7 (Ubuntu) Server at 10.32.0.4 Port 80</address>
</body></html>

# curl 10.32.0.6
....
Nada
```

This is achieved by using the [arpspoof/network.yml] Network Policy. Note that
it appears to be possible to use simple labels to restrict both ingress and
egress on individual pods.



## See also
* http://try.cybersecurity.ieee.org/trycybsi/help/sslstrip
* http://try.cybersecurity.ieee.org/trycybsi/help/arpspoof
