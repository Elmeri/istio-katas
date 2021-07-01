[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #sentences #kiali)

# Routing Traffic with Istio

## Learning goals

- Using Virtual Services
- Using Destination Rules

## Introduction

These exercises introduce you to the basics of traffic routing with Istio. 
We are going to deploy all the services for our sentences application 
and a new version `v2` of the **name** service. This will demonstrate normal 
kubernetes load balancing between services. 

Then we are going to use two Istio custom resource definitions(CRD's) which are
the building blocks of Istio's traffic routing functionality to route traffic to 
the desired workloads.

These are the VirtualService and the DestinationRule CRD's.

## Exercise 1

With Istio on kubernetes you use virtual services to route traffic to kubernetes 
services. 

A VirtualService defines a set of traffic routing rules to apply when a host 
is addressed. Each routing rule defines matching criteria for traffic of a 
specific protocol. If the traffic is matched, then it is sent to a named 
destination **service** or subset/version of it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service-route
spec:
  hosts:
  - my-service
  http:
  - route:
    - destination:
        host: my-service-v1
    - destination:
        host: my-service-v2
```
The **http** block is an [HTTPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute) 
containing the routing rules for HTTP/1.1, HTTP/2 and gRPC traffic. 

You can also use [TCPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#TCPRoute) 
and [TLSRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#TLSRoute) 
blocks for configuring routing.

The **hosts** field is the user addressable destination that the routing rules 
apply to. This is **virtual** and doesn't actually have to exist. For example 
You could use it for consolidating routes to all services for an application. 

The **destination** field specifies the **actual** destination of the routing 
rule and **must** exist. In kubernetes this is a **service** and generally 
takes a form like `reviews`, `ratings`, etc.

### Overview

- Deploy the sentences app and a second version (`v2`) of the name service. 

- Run the script `scripts/loop-query.sh` to produce traffic.

- Use the version app graph in Kiali to observe the traffic flow.

- Route **all** traffic to version 1 of the name service with a Virtual Service.

> :bulb: A virtual service lets you configure how requests are routed 
> to a **service** within an Istio service mesh.

- Observe route precedence by adding a route to version 2 of the name service.

### Step by Step
<details>
    <summary> More Details </summary>

* **Deploy sentences app and 2 versions of name services**

```console
kubectl apply -f 001-basic-traffic-routing/start/
```

* **Run loop-query.sh**

```console
./scripts/loop-query.sh
```

* **Observe traffic flow with the **version app graph** in Kiali**

![50/50 split of traffic](images/kiali-blue-green-anno.png)

What you are seeing here is kubernetes load balancing between PODS.
Kubernetes, or more specifically the `kube-proxy`, will load balance in 
either a *round robin* or *random* pattern depending on whether it is 
running in *user space* proxy mode or *IP tables* proxy mode.

You rarely want traffic routed to two version in an uncontrolled 
fashion.

So why is this happening?

> :bulb: Take a look at the label selector for the name service.
> It doesn't specify a version...

* **Route ALL traffic to version 1 of the name service** 

Create a new service called `name-svc-v1.yaml` in `001-basic-traffic-routing/start/` 
and apply it. Make sure it has a version (`v1`) in the label selector.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sentences
    mode: name
    version: v1
  name: name-v1
spec:
  ports:
  - port: 5000
    protocol: TCP
    targetPort: 5000
  selector:
    app: sentences
    mode: name
    version: v1
  type: ClusterIP
```

```console
kubectl apply -f 001-basic-traffic-routing/start/name-svc-v1.yaml
```

Create a virtual service called `name-virtual-service.yaml` in 
`001-basic-traffic-routing/start/` and apply it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: name-route
spec:
  hosts:
  - name
  http:
  - route:
    - destination:
        host: name-v1
```

> The `host` field in the above yaml is the kubernetes short name for the service. 
> Istio will translate the short name based one the **namespace** of the rule. 
> E.g. if the virtual service is in namespace `default` the short name name will 
> be interpreted as `name.default.svc.cluster.local`. What will happen if the 
> **name** service is in the namespace `user1`?

```console
kubectl apply -f 001-basic-traffic-routing/start/name-virtual-service.yaml
```

Observe the traffic flow in Kiali using the **versioned app graph**. It may 
take a minute before fully complete but you should see the traffic being routed 
to the `name-v1` **service**.

> :bulb: Make sure to select `Idle Edges` and `Service Nodes` in the Display 
drop down.

![Basic virtual service route](images/basic-route-vs.png)

* **Observe route precedence**

Create a new service called `name-svc-v2.yaml` in `001-basic-traffic-routing/start/` 
and apply it. Make sure it has a version (`v2`) in the label selector.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sentences
    mode: name
    version: v2
  name: name-v2
spec:
  ports:
  - port: 5000
    protocol: TCP
    targetPort: 5000
  selector:
    app: sentences
    mode: name
    version: v2
  type: ClusterIP
```

```console
kubectl apply -f 001-basic-traffic-routing/start/name-svc-v2.yaml
```

Add a destination to the new service in the `name-virtual-service.yaml` you 
created before. But place it **before** the `name-v1` service and apply it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: name-route
spec:
  hosts:
  - name
  http:
  - route:
    - destination:
        host: name-v2
  - route:
    - destination:
        host: name-v1
```

```console
kubectl apply -f 001-basic-traffic-routing/start/name-virtual-service.yaml
```

Observe the traffic flow in Kiali using the **versioned app graph**.
You will see that traffic is now being routed to the version 2 service.

![Routing precedence](images/basic-route-precedence-vs.png)

Routing rules are evaluated in sequential order from top to bottom, with the 
first rule in the virtual service definition being given highest priority. 

Reorder the destination rules so that service `name-v1` will be evaluated 
first and apply the changes.

```console
kubectl apply -f 001-basic-traffic-routing/start/name-virtual-service.yaml
```

Traffic should now be routed to the `name-v1` service.

</details>

## Exercise 2

Destination rules configure **what** happens to traffic for a destination 
defined in a virtual service.

You can think of virtual services as **how** you route your traffic to a given 
destination, and then you use destination rules to configure **what** happens 
to traffic for that destination.

One of the most common uses of `DestinationRule` is to specify named service **subsets**.

For example, grouping all of a service instances **versions**. You can then 
use these **subsets** in a virtual service to control traffic to different versions.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

> :bulb: Destination rules are applied **after** virtual service routing rules are evaluated, so they apply 
> to the traffic’s “real” destination.


### Overview

- Create a destination rule with **subsets** for the `name-v1` and `name-v2` workloads and apply it.

- Run the script `scripts/loop-query.sh` to produce traffic.

- Update the virtual service from the previous exercise to include the subsets and apply it.

- Remove the `name-v1` and `name-v2` services. 

- Use the version app graph in Kiali to observe the traffic flow.

### Step by Step
<details>
    <summary> More Details </summary>

* **Create a destination rule and apply it**

Create a destination rule called `name-destination-rule.yaml` in 
`001-basic-traffic-routing/start/` and apply it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: name-destination-rule
spec:
  host: name
  subsets:
  - name: name-v1
    labels:
      version: v1
  - name: name-v2
    labels:
      version: v2
```
The above destination rule says, when combined with a virtual service, **what** 
I want to do is send traffic to a workload **labeled** with either `v1` or `v2`.

```console
kubectl apply -f 001-basic-traffic-routing/start/name-destination-rule.yaml
```
Applying the destination rule has no effect at this point because the virtual 
service has not yet been updated to use the subsets.

> :bulb: To avoid 503 errors **always** apply destination rules and changes to 
> destination rules **prior** to changing virtual services.

* **Update the virtual service**

Update the virtual service `001-basic-traffic-routing/start/name-virtual-service.yaml` 
you created in exercise 1 and apply it.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: name-route
spec:
  hosts:
  - name
  gateways:
  - mesh
  http:
  - route:
    - destination:
        host: name
        subset: name-v1
  - route:
    - destination:
        host: name
        subset: name-v2
```

Notice that the `destination` of the route's are no longer the two new services 
you created in exercise 1. Instead it is the original `name` service 
which has no version in the selector. The virtual service is going to route all 
traffic to the generic `name` service and **what** is going to happen after it 
has been routed to this service is determined by the destination rule. In this case 
the traffic is going to be directed to a workload based on version label.

```console
kubectl apply -f 001-basic-traffic-routing/start/name-virtual-service.yaml
```

* **Remove the un-needed services**

```console
kubectl delete -f 001-basic-traffic-routing/start/name-svc-v1.yaml
kubectl delete -f 001-basic-traffic-routing/start/name-svc-v2.yaml
```

* **Observe traffic flow with version app graph in Kiali**

You can see in Kiali that the virtual service combined with the destination 
rule subsets routes traffic to the name workload labeled `v1` even though the 
name service has no versions defined in the selector.

![Virtual service and destination rule](images/kiali-vs-dr.png)

> :bulb: Order of precedence in the virtual service still applies. The destination 
> rule does **not** affect it.

</details>

## Summary

In exercise 1 you learned what a virtual service is and how to route traffic 
to a destination service in kubernetes with an [HTTPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRoute). 
You can also leverage [TLSRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#TLSRoute) 
and [TCPRoute](https://istio.io/latest/docs/reference/config/networking/virtual-service/#TCPRoute).

There is a lot more virtual services can be do for traffic distribution like 
match conditions for HTTPRoute on headers, uri, schemes, etc. HTTP redirects and 
rewrites. We will looking at some of these in following exercises.

See the [documentation](https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService) 
for more details.

> :bulb: Kubernetes only supports traffic distribution based on instance scaling. 

In exercise 2 you saw how a destination rule could be used to determine **what** 
happened to traffic routed to a kubernetes service using labels identifying 
workload versions. But you can also set traffic policies on a destination rule 
to apply load balancing policies, connection pool settings, etc.

See the [documentation](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule) 
for more details.

# Cleanup

```console
kubectl delete -f 001-basic-traffic-routing/start/
```