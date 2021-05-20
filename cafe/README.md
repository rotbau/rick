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
or
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/coffee```

    - Output will look like.  Note server name and URI at the end of the output.
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

Instead of the Kubernetes standard ingress you could deploy HTTPproxy object instead.  Functionality is the same.

- Clean up previous ingress object
```kubectl delete -f 02-cafe-ingress.yaml```

- Deploy HTTPproxy object for applicaton
```kubectl apply -f 03-cafe-httpproxy.yaml```

- Validate HTTPproxy object is created
```kubectl get httpproxy```
```
NAME              FQDN                   TLS SECRET   STATUS   STATUS DESCRIPTION
cafe-ingress      cafe.rtbsystems.com                 valid    Valid HTTPProxy
```

- Test Ingress
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/tea```
or
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/coffee```

Output will be the same as above depending if you specified the /tea or /coffee URI.

## Deploy HTTPProxy rewrite path object 

This example show how to rewrite a path using the HTTPproxy from Contour.  This example stil has the /coffee and /tea URI, but also will rewrite the / URI to the coffee service because I like coffee more than tea.  This is just one example of how to use the path rewrite function.  More examples on the project contour site.

- Clean up previous HTTPproxy object
```kubectl delete -f 03-cafe-httpproxy.yaml```

- Deploy HTTPproxy rewrite ingress
```kubectl apply -f 04-cafe-rewrite-httpproxy.yaml```

- Validate HTTPproxy object is created
```kubectl get httpproxy```
```
NAME              FQDN                   TLS SECRET   STATUS   STATUS DESCRIPTION
rewrite-ingress   cafe.rtbsystems.com                 valid    Valid HTTPProxy
```
- Test rewrite
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/```

Note that you will get the coffee service even though you are going to the root URI. 

## Deploy Blue/Green Weighted HTTPproxy object

This example allows you to demonstrate the abilty to load balance between 2 different services.  Typicaly this is used for blue/green deployments where you have two different versions of a application.  You can use the httpproxy to deploy a weighted ingress to send a percentage of traffic to each service.  We can use the /tea and /coffee services to demonstrate this.  We also cover this using the hello-world app.

- Clean up previous HTTPproxy object
```kubectl delete -f 04-cafe-rewrite-httpproxy.yaml```

- Deploy Contour HTTPproxy weighted ingress object
```kubectl apply -f 05-cafe-weight-httpproxy.yaml```

- Validate the HTTPproxy object is created
```kubectl get httpproxy```
```
NAME              FQDN                  TLS SECRET   STATUS   STATUS DESCRIPTION
weight-shifting   cafe.rtbsystems.com                valid    Valid HTTPProxy
```

-  Test Blue/Green deployment
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/```

You will see the `Server name` change between coffee and tea pods with coffee being preferred 80% of the time in this example

```
Server address: 100.96.4.112:8080
Server name: coffee-6f4b79b975-6gpx8
Date: 20/May/2021:19:43:02 +0000
URI: /
```
```
Server address: 100.96.5.13:8080
Server name: tea-6fb46d899f-n4xhh
Date: 20/May/2021:19:43:04 +0000
URI: /
```

## TLS HTTPproxy Ingress and permitting insecure

This example covers assigning a TLS secret to the cafe.rtbsystems.com site but allowing the /coffee to be served insecure while /tea requires secure.

- Clean up previous HTTPproxy object
```kubectl delete -f 05-cafe-weight-httpproxy.yaml```

- Deploy cafe TLS secret
```kubectl apply -f 06-cafe-secret.yaml```

- Deploy Contour HTTPproxy object with TLS and permit insecure
```kubectl apply -f 07-cafe-tls-httpproxy.yaml```

- Test to validate SSL is enforced on /tea service
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/tea```

You will note you do not get to the /tea service but instead get an HTTP 301 stating the URI has a permanent redirect to https.
```
HTTP/1.1 301 Moved Permanently
location: https://cafe.rtbsystems.com/tea
```

- Test to validate you do not need SSL for the /coffee service
```curl -v --resolve cafe.rtbsystems.com:172.31.3.191 cafe.rtbsystems.com/coffee```

This time you will not receive a HTTP 301 redirect but instead get the coffee service

```
HTTP/1.1 200 OK
....
Server address: 100.96.5.10:8080
Server name: coffee-6f4b79b975-vpwbw
Date: 20/May/2021:20:18:42 +0000
URI: /coffee
```