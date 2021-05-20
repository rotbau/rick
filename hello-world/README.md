# Hello World Blue Green Deployment Test

This demo uses 2 different container images (blue and green) to simulate upgrading an application.  Each service can either have its own service deployed which can demonstrate using using FQDN the two distinct applications versions.  Or you can leverage the HTTPproxy component of Contour Ingress controller to do a weighted deployment where 80 percent of traffic will go to blue service and blue container and 20 percent will go to green pod and service.

## Pre-requisites

1. Contour Ingress Controller up and running
2. Access to container registry with the v1.10 and v1.11 container images running

## Deployment

- Deploy hello v1.10 application
```kubectl apply -f 01-hello-v1.10.yaml``

- Deploy hello v1.11 application
```kubectl apply -f 02-hello-v1.11.yaml```

- Validate Pods and Services are created successfully
```kubectl get po,svc```
```
NAME                                    READY   STATUS    RESTARTS   AGE
pod/helloworld-v1.10-7f4d455bdf-hgff2   1/1     Running   0          8m47s
pod/helloworld-v1.11-8446d9bb49-vq62r   1/1     Running   0          8m5s


NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/helloworld-v1-10   ClusterIP   100.66.20.8     <none>        80/TCP      8m47s
service/helloworld-v1-11   ClusterIP   100.71.186.42   <none>        80/TCP      8m5s
```

- Deploy Weighted Httpproxy Object (Contour weighted ingress)
```kubectl apply -f 03-hello-weight-httproxy.yaml``

- Validate contour httpproxy object is created successfully
```kubectl get httpproxy```
```
NAME              FQDN                   TLS SECRET   STATUS   STATUS DESCRIPTION
weight-shifting   hello.rtbsystems.com                valid    Valid HTTPProxy
```

## Testing

To test this you will need to either have a DNS record pointing the FQDN (hello.rtbsystems.com in our example) to the Load Balanced IP of the Envoy Proxy service.  You could also set this up using Hosts file on your local machine or using Curl and passing the headers.  

- Determine IP of Envoy Proxy Service
```kubectl get svc -n projectcontour``` 
    - Note: your namespace for Contour may be different depending on how you depoloyed.  Based on the below output we would equate FQDN of hello.rtbsystems.com with IP 172.31.3.191 in DNS, hosts file or Curl
```
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
contour   ClusterIP      100.69.27.15   <none>         8001/TCP                     20m
envoy     LoadBalancer   100.69.47.57   172.31.3.191   80:30855/TCP,443:30167/TCP   20m
```

- Test using browser.  You should see 80% of the refreshes land on the `Hello World! version 1.10 BLUE` page and 20% on Green.  Note that this is randomized so you will not see same pattern on each refresh, but 20% of the entire traffic will hit the new Green service.

- Testing using Curl
```curl -H "Host: hello.rtbsystems.com" 172.31.3.191```

## OPTIONAL - Deploy Kubernetes ingress for each service version to demonstrate separate services and deployments

Contour can also service the Kubernetes default ingress object (networking.k8s.io/v1 Kind: Ingres) instead of using the HTTPproxy object.  You can optionally deploy the ingress controller for the two sepearate Blue/Green deployments to demonstrate this.

```kubectl apply -f 04-hello-v1.10-ingress.yaml```
```kubectl apply -f 05-hello-v1.11-ingress.yaml```

- View Ingress objects
```kubectl get ing```
```NAME                    CLASS    HOSTS                     ADDRESS        PORTS   AGE
helloworld-v1-ingress   <none>   hello-v1.rtbsystems.com   172.31.3.191   80      7m2s
helloworld-v2-ingress   <none>   hello-v2.rtbsystems.com   172.31.3.191   80      5m10s
```

- Test Ingress using Curl
```
curl -H "Host: hello-v1.rtbsystems.com" 172.31.3.191
Hello World! version 1.10 BLUE
```

```
curl -H "Host: hello-v2.rtbsystems.com" 172.31.3.191
Hello World! version 1.11 GREEN
```
