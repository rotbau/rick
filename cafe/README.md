# Coffee / Tea application with multiple Contour HTTPproxy and Ingress Examples

This demo shows how to deploy the cafe application that exposes two services called coffee and tea.  These services are exposed at /coffee and /tea prefix.  We are using Contour ingress controller to demonstrate some of the advanced HTTPproxy ingress options like weighted deployment, rewrite, etc.

## Pre-requisites

1. Contour Ingress Controller up and running
2. Access to container registry with the v1.10 and v1.11 container images running
3. Load Balancer for Kubernetes to provide routeable IP for Envoy ServiceType Loadbalancer

## Deploy cafe application and services

- Deploy core application and services
```kubectl apply -f 01-cafe-deployment.yaml```

- Validate deployment was successful
```kubectl get po,svc```
```
NAME                                READY   STATUS    RESTARTS   AGE
coffee-6f4b79b975-6gpx8             1/1     Running   0          97s
coffee-6f4b79b975-vpwbw             1/1     Running   0          97s
tea-6fb46d899f-bfczq                1/1     Running   0          32s
tea-6fb46d899f-kswdg                1/1     Running   0          34s
tea-6fb46d899f-n4xhh                1/1     Running   0          30s

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
coffee-svc         ClusterIP   100.64.242.82   <none>        80/TCP      107s
tea-svc            ClusterIP   100.66.205.68   <none>        80/TCP      107s
```

## Deploy Standard Ingress or HTTPProxy object

Contour can handle both the standard Kubernetes Kind Ingress (networking.k8s.io) as well as Envoy HTTPproxy type.  Both of the following acheive the exact same end result and you would choose one or the other method in the real world.

### Standard Kubernetes Ingress

- Deploy standard ingress object
```kubectl apply -f 02-cafe-ingress.yaml```

- Validate ingress objects
```kubectl get ing```
```
NAME                    CLASS    HOSTS                     ADDRESS        PORTS   AGE
cafe-ingress            <none>   cafe.rtbsystems.com       172.31.3.191   80      14m
```

- Test ingress
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/tea```
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/coffee```

    - Output will look like
```
    *   Trying 172.31.3.191...
* TCP_NODELAY set
* Connected to cafe.rtbsystems.com (172.31.3.191) port 80 (#0)
> GET /tea HTTP/1.1
> Host: cafe.rtbsystems.com
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< server: envoy
< date: Thu, 20 May 2021 16:24:25 GMT
< content-type: text/plain
< content-length: 155
< expires: Thu, 20 May 2021 16:24:24 GMT
< cache-control: no-cache
< x-envoy-upstream-service-time: 0
< vary: Accept-Encoding
<
Server address: 100.96.5.13:8080
Server name: tea-6fb46d899f-n4xhh
Date: 20/May/2021:16:24:25 +0000
URI: /tea
```

### Deploy Contour/Envoy HTTPproxy Object
