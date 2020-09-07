# kubernetes-metallb-rpi-k3s
MetalLB on a Raspberry Pi Kubernetes Setup using k3s

## Demo

Demonstration how to setup [MetalLB](https://metallb.universe.tf/) on your raspberrypi k3s cluster

### View Nodes

Our nodes:

```
$ kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
rpi-07   Ready    master   67d   v1.18.4+k3s1
rpi-06   Ready    master   67d   v1.18.4+k3s1
rpi-05   Ready    master   67d   v1.18.8+k3s1
```

### Deploy MetalLB

Deploy MetalLB:

```
$ kubectl apply -f namespace.yml
$ kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
$ kubectl apply -f metallb.yaml
```

We need to reserve IP's for the load balancer. I am running on a 192.168.0.0/24 range, but my DHCP dont assign addresses from 192.168.0.20-100, so I will use 20-30 for MetalLB:

```
$ kubectl create -f metallb-config.yml
```

You should see something like this:

```
$ kubectl get all --namespace metallb-system
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-57f648cb96-rxk4j   1/1     Running   0          6h46m
pod/speaker-75sgp                 1/1     Running   0          3h25m
pod/speaker-njlwc                 1/1     Running   0          3h26m
```

### Deploy a Sample Application

First we will deploy the application **without** the MetalLB Load Balancer:

```
$ kubectl apply -f example-app/namespace.yml
$ kubectl apply -f example-app/deployment.yml
$ kubectl apply -f example-app/service-without-metallb.yml
```

You should see the following for our deployment:

```
$ kubectl get pods -n rpi-demo -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP           NODE     NOMINATED NODE   READINESS GATES
rpi-demo-6b48b95d9d-sk7p5   1/1     Running   0          104s   10.42.1.11   rpi-06   <none>           <none>
rpi-demo-6b48b95d9d-b8kdc   1/1     Running   0          104s   10.42.2.10   rpi-07   <none>           <none>
rpi-demo-6b48b95d9d-mmkt8   1/1     Running   0          104s   10.42.2.9    rpi-07   <none>           <none>
```

And the following for our service:

```
$ kubectl get service -n rpi-demo -o wide
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
rpi-demo   ClusterIP   10.43.74.108   <none>        80/TCP    79s   app=rpi-demo
```

From the node, we can see the application by using the cluster-ip:

```
$ curl 10.43.74.108
Hostname: rpi-demo-6b48b95d9d-sk7p5
```

But we cant view this outside the cluster. Lets change the service so that it uses the LoadBalancer type, and deploy the service **with** the MetalLB LoadBalancer service:

```
$ kubectl apply -f example-app/service-with-metallb.yml
```

Now we should see that MetalLB assigned a LAN IP to the External-IP section:

```
$ kubectl get service -n rpi-demo -o wide
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE     SELECTOR
rpi-demo   LoadBalancer   10.43.74.108   192.168.0.21   80:31439/TCP   5m42s   app=rpi-demo
```

From a host on the network outside the cluster:

```
$ curl 192.168.0.21
Hostname: rpi-demo-6b48b95d9d-b8kdc
```

