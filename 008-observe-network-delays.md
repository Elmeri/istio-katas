[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #delay #network-delay #kiali)

# Observing Network Delays

## Learning goal

- 
- 

## Introduction

This exercise will show that some forms of delays can be observed with the
**metrics** that Istio tracks. Metrics are statistical and not specific to
e.g. a certain request, i.e. we can only observe statistical data about
observations like sums and averages. This is however, in some cases very useful.

## Exercise

### Overview

- 
- 

### Step by Step
<details>
    <summary> More Details </summary>

First, deploy the following, which creates three versions of the `name` service
`v1`, `v2` and `v3`:

```console
kubectl apply -f 008-observe-network-delays/start/v1
kubectl apply -f 008-observe-network-delays/start/v2
kubectl apply -f 008-observe-network-delays/start/v3
```

In another shell, run the following to continuously query the sentence service
and observe the effect of deployment changes:

```console
scripts/loop-query.sh
```

In this exercise we have not created any Istio Kubernetes resources to affect
routing, i.e. requests to the `name` services are approximately evenly
distributed across the three version. However, from the output of
`loop-query.sh` we will observe an occasional delay.

If we open Kiali and select to display 'response time', we see the following,
which shows that `v3` have a significantly higher delay than the two other
versions.

![Canary Traffic in Kiali](images/kiali-request-delays-anno.png)

This scenario in this exercise was rather simple, and the problem was isolated
to something we could isolate in Kubernetes i.e. a specific version of a
service. If the delay was caused by something more complicated, e.g. an
unforeseen interaction between services, it would have been difficult to
diagnose purely from metrics due to their statistical nature.

The following exercise will show how to diagnose cross-service issues using
distributed tracing.


In the exercise [Observing Delays](request-delays.md) we saw how we could
identify service delays when this was related to a specific service (or PODs if
we use labels). This was without any instrumentation in the application so this
is a nice possibility.

However, in larger application such an approach may prove more difficult. With a
larger application, multiple teams may be involved and e.g.:

- The misbehaving service might be owned by another team

- The misbehaving application might not be the immediate one from which you are
  observing a delay. In fact, it might be deep in the application tree

Istio and Jaeger can help us get a clearer picture on the problem.

Delete the sentence applications services.

```console
kubectl delete -f 008-observe-network-delays/start/
kubectl delete -f 008-observe-network-delays/start/v1/
kubectl delete -f 008-observe-network-delays/start/v2/
kubectl delete -f 008-observe-network-delays/start/v3/
```

Deploy the following version of the `sentence` application - now with
three tiers to simulate a slightly more complex application:

```console
kubectl apply -f 008-observe-network-delays/start/three-tiers/
```

In another shell, run the following to continuously query the sentence service
and observe the effect of deployment changes:

```console
scripts/loop-query.sh
```

![No delays with v1](images/kiali-three-tiers-1.png)

Next, deploy `v2` of the `sentences` service:

```console
kubectl apply -f 008-observe-network-delays/start/three-tiers/v2/
```

This version has a (simulated) bug, that cause large delays on the combined
service as we can see from the following Kiala application graph.

![Delays with v2](images/kiali-three-tiers-2.png)

Now the SRE team for the `random` service is being paged, and they might find it
difficult to understand what have changed. Remember, the `sentences` service
might be developed by another team. How can the SRE team for `random` figure out
that they need to contact the responsible for `sentences` version `v2`?

If we search for traces in Jaeger where the trace time is high and inspect the
trace, we will find that the top-level service is indeed `sentences` version
`v2`:

![Traces in Jaeger](images/jaeger-three-tiers-1-anno.png)

</details>

## Summary


## Cleanup

```console
kubectl delete -f 008-observe-network-delays/start/three-tiers/
```