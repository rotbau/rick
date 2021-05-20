# Demonstrate Network Policies using Yelb

Native Kubernetes network polices can be used to control ingress and egress to your namespaces and the applications running in them.  This is a pretty simple example of an Ingress policy.

## Deploy Yelb Application

- Deploy core components
```
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-yelb-app.yaml
kubectl apply -f 03-yelb-db.yaml
kubectl apply -f 04-yelb-redis.yaml
```

- Deploy Frontend web service using either Ingress or LB

Choose *ONE* of the two options
    - Deploy UI and expose as Load Balancer
    ```kubectl apply -f 05a-yelb-ui-lb.yaml```

    - Deploy UI and expose as Ingress
    ```kubectl apply -f 05b-yelb-ui-ingress.yaml```

## Test access to Yelb

- Verify Pods are running and services created and get UI external IP addres
```kubectl get po,svc -n yelb```

- If you deployed as Ingress view ingress and get IP address for Ingress
```kubectl get ing -n yelb```

- Use curl or browser to test access
```curl -i -k http://{external IP of LB}```
or
```curl -H "Host: yelb.rtbsystems.com" {external IP of Envoy Service}```
or Open in Browser using IP if service type LB or DNS/Host if ingress

## Apply Default Network Policy

This should block all Ingress in Yelb Namespace

```kubectl apply -f 06-network-policy-deny-al.yaml```

Test using curl or browser again.  It may take a couple seconds but traffic show be blocked and access will fail

## Apply Yelb Network Policy

Kubernetes network policies are addative so be applying the yelb policy we can add allow pod-pod and ingress/egress traffic
```kubectl apply -f 07-network-policy-yelb.yaml```

Retest using curl or browser and application.  The application should now function again