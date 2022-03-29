---
layout: post
title: "Adventures with On-prem Kubernetes (Kubespray) with Ingress (ingress-nginx) and a Load Balancer (MetalLB)"
date: 2020-05-21 14:45
last_modified_at: 2022-03-29 18:23
---

Setting up a on-prem Kubernetes cluster with all the bells and whistles

**Update 29/03/2022:** I'd mistakenly called `"ingress-nginx"` `"NGINX Ingress"`, 
the 2 are different products not to be confused.

`ingress-nginx` is made by Google and is FOSS while `NGINX Ingress` is made by F5 and is commercial.

**Update 11/07/2020:** Include information about MetalLB changes

# Preface

I honestly wish I'd found [this blog](https://medium.com/@futuredon/kubernetes-on-prem-demo-using-gns3-kubespray-nginx-ingress-frr-and-metallb-part-1-4b872a6fa89e) 
before I started my journey, it's a little old but still basically holds ups in 2020.

[Part 2](https://medium.com/@futuredon/kubernetes-on-prem-demo-using-gns3-kubespray-nginx-ingress-frr-and-metallb-part-2-4f11ace36c00) 
is a little out of date with regards to how to deploy the ingress 
and load balancer components, so I'll cover off what I assume is 
the way to do it these days.


# Kubernetes via [Kubespray](https://github.com/kubernetes-sigs/kubespray)

To be frank; the Kubespray documentation isn't great at 
explaining everything, it very much expects you to know a fair 
amount about kubernetes before you use it, which is fair enough.

I'll list **some** of the changes I made along the way

`group_vars/all/all.yml`

```
kubeconfig_localhost: true # Saves the kubectl conf file to artifacts/admin.conf

etcd_kubeadm_enabled: true # Decided to enable this new experimental feature

# Don't rely on a external loadbalancer
# Install one on every worker node
# I'm sure this could be replaced down the line with MetalLB and ingress-nginx...
loadbalancer_apiserver_localhost: true
loadbalancer_apiserver_type: nginx

cert_management: script # Generate certs for us
```


`group_vars/k8s-cluster/k8s-cluster.yml`

```
kube_proxy_strict_arp: true #Needed for MetalLB

dns_min_replicas: 3 #I had three masters so went with one dns replica for master

kubeadm_control_plane: true # Decided to enable this new experimental feature
```


`group_vars/k8s-cluster/addons.yml`

```
# Enable the deployment of ingress-nginx
# But don't enable the HostNetwork stuff as we'll be using MetalLB as LoadBalancer
ingress_nginx_enabled: true
ingress_nginx_host_network: false


# MetalLB Config
# See https://github.com/kubernetes-sigs/kubespray/tree/master/roles/kubernetes-apps/metallb
metallb_enabled: true
metallb_ip_range:
  - "10.0.70.100-10.0.70.132" # Choose IP range MetalLB can give out on the L2 network segment
metallb_protocol: "layer2"
```

Once deployed you'll have a working Kubernetes cluster (hopefully).


# Load Balancer via [MetalLB](https://github.com/kubernetes-sigs/kubespray/tree/master/roles/kubernetes-apps/metallb)

As of [29/06/2020](https://github.com/kubernetes-sigs/kubespray/commit/25bab0e9760bdad922863ac95bb2944fabe1c3a4) 
MetalLB is a core part of Kubespray and therefore no longer requires any extra deployment steps. Kubespray 2.13 is the last version to require 
any extra steps beyond setting the config options in `group_vars/k8s-cluster/addons.yml`.

~~Super easy to deploy as you did the config in `group_vars/k8s-cluster/addons.yml`, just follow the [readme](https://github.com/kubernetes-sigs/kubespray/tree/8213b1802b3971722514e96ead1333572a529d50/contrib/metallb) 
in the kubespray docs.~~ 


# Ingress via [ingress-nginx](https://github.com/kubernetes-sigs/kubespray/tree/master/roles/kubernetes-apps/ingress_controller/ingress_nginx)

This one is mostly already done via kubespray apart from the 
service of `type: LoadBalancer`.

We want to use a **similar** `service` definition from the [cloud config](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml) 
rather than the [baremetal](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml) one because the baremetal one uses `NodePort` 
instead of `LoadBalancer` which we just setup via MetalLB.

We also have to modify the `selector` to match what kubespray has 
deployed.

Create and deploy a file called `ingress-nginx-service.yaml` with 
the contents:

```
# Source: ingress-nginx/templates/controller-service-webhook.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller-admission
  namespace: ingress-nginx
spec:
  type: ClusterIP
  ports:
    - name: https-webhook
      port: 443
      targetPort: webhook
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
# Source: ingress-nginx/templates/controller-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  externalTrafficPolicy: Cluster
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

You should now have a ingress service with a `EXTERNAL-IP` that 
was obtained from our load balancer, verify this by making sure 
the IP is in the range that you specified MetalLB config when 
looking at services in the `ingress-nginx` namespace.


# Test the setup

Lets deploy a simple container that says hello world over http on 
port `3000`.

We'll make a deployment, a service using `ClusterIP` using port 
`8008` and then setup an ingress to point to it on the path 
`/test`.

`hello-world-d_s_i.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-deployment
spec:
  selector:
    matchLabels:
      app: hello-world-debug
  replicas: 3
  template:
    metadata:
      labels:
        app: hello-world-debug
    spec:
      containers:
      - name: hello-world-debug
        image: "imranh2/hello-world-debug"
        env:
        - name: "PORT"
          value: "3000"

---
apiVersion: v1
kind: Service
metadata:
  name: hello-world-service
spec:
  type: ClusterIP
  selector:
    app: hello-world-debug
  ports:
  - protocol: TCP
    port: 8008
    targetPort: 3000

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-world-ingress
  annotations: 
     nginx.ingress.kubernetes.io/rewrite-target: /
     nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /test
        backend:
          serviceName: hello-world-service
          servicePort: 8008

```


`curl http://$(kubectl get services ingress-nginx-controller -n ingress-nginx --output jsonpath='{.status.loadBalancer.ingress[0].ip}')/test`


You should see the container output :)


# Closing Thoughts

* Source IP to containers - The direct TCP/IP connection you see 
is the Pod IP of a Ingress Controller, as that's where the 
external connection is being terminated. For HTTP(S) 
there's `X-FORWARDED-FOR`. Setting 
`externalTrafficPolicy: Cluster` to `Local` ***and*** using BGP 
instead of layer2 to announce will let you see the real IP for 
non-HTTP application [allegadly](https://metallb.universe.tf/usage/#local-traffic-policy-1).

* Kubernetes API `EXTERNAL-IP` - As we now have a load balancer 
we could give the kubernetes API itself a `EXTERNAL-IP`. Then 
possibly reconfigure the entire cluster to use that VIP via the 
`loadbalancer_apiserver` config options, also editing 
`supplementary_addresses_in_ssl_keys` to include the VIP and 
domain. How sensible it is to have your API LB as part of your 
cluster is left up to you to decide....


I think I had some more thoughts but they've escaped me for now...