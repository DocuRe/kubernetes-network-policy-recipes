# ALLOW traffic only to a port number of an application

This NetworkPolicy lets you define ingress rules for specific port
numbers of an application. If you do not specify a port number in the
ingress rules, the rule applies to all port numbers.

**Use Cases**
- Allow monitoring system to collect the metrics by querying the diagnostics
  port of your application, without giving it access to the rest of the
  application.
  
### Example

Run a web server deployment:

    kubectl run web --image=nginx --labels=app=web
    
Expose the deployment as Service on ports 8000 (for web access)
and on port 5000 (for internal monitoring).

    kubectl create service clusterip web \
        --tcp 8000:80 \
        --tcp 5050:5000

> ![#c5f015](https://placehold.it/15/c5f015/000000?text=+) **NOTE:**
> Network Policies will not know the port numbers you exposed the application,
> such as 8080 and 5050. This is because they control inter-pod traffic and
> when you expose Pod as Service, ports are remapped like above. Therefore,
> you need to use the container port numbers (such as 80 and 5050) in the 
> NetworkPolicy specification.

Save this Network Policy as `web-allow-5000.yaml` and apply to
the cluster.
  
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-allow-5000
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - ports:
    - port: 5000
    from:
    - podSelector:
        matchLabels:
          role: monitoring
```

```sh
$ kubectl apply -f web-allow-5000.yaml
networkpolicy "web-allow-5000" created
```

This network policy will:

- Drop all traffic to `app=web` by default.
- Allow traffic on port `5000` from pods with label
  `role=monitoring` in the same namespace.

### Try it out

Run a pod with no custom labels, observe the traffic to ports
5000 and 8000 are blocked:

```sh
$ kubectl run test-$RANDOM --rm -i -t --image=alpine -- sh
/ # wget -qO- --timeout=2 http://web:8000
wget: download timed out

/ # wget -qO- --timeout=2 http://web:5000
wget: download timed out
```

Run a pod with `role=monitoring` label, observe the traffic to
port 5000 is allowed, but port 8000 is not:

