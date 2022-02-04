[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #TLS #mutual-tls #PKI-domains #ingress-gateway #VirtualService #Gateway #PeerAuthentication #DestinationRule)

# Securing with Mutual TLS

## Learning goals

- Secure communication between services
- Secure communication with mesh external services

## Introduction

These exercises will demonstrate how to use mutual TLS inside the mesh between
PODs with the Istio sidecar injected and also from mesh-external services
accessing the mesh through an ingress gateway.

You will be using several Istio custom resource 
definitions([CRD's](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)) 
for this.

- [PeerAuthentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/#PeerAuthentication-MutualTLS) - 
Applies to requests that a service **receives**

- [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule) - 
What type of TLS sidecar **sends**

- [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings) - 
Specify TLS settings for external service

Istio provides two types of authentication policies of which *peer 
authentication* is one of them. 

Peer authentication is used for **service-to-service** authentication 
to verify the client making the connection and secure the 
service-to-service communication. Istio uses 
[mutual TLS](https://en.wikipedia.org/wiki/Mutual_authentication)(mTLS) 
for transport authentication. 

This provides a **strong identity** for each workload. 

> Istio has it's own certificate authority(CA) and securely provisions strong 
> identities to every workload with X.509 certificates.

<details>
    <summary> More About Workload Identity </summary>

Istio provisions keys and certificates through the following flow:

- istiod offers a gRPC service to take certificate signing requests (CSRs).

- When started, the Istio agent creates the private key and CSR, and then 
sends the CSR with its credentials to istiod for signing.

- The CA in istiod validates the credentials carried in the CSR. Upon 
successful validation, it signs the CSR to generate the certificate.

- When a workload is started, Envoy requests the certificate and key from 
the Istio agent in the same container via the Envoy secret discovery 
service (SDS) API.

- The Istio agent sends the certificates received from istiod and the private 
key to Envoy via the Envoy SDS API.

- Istio agent monitors the expiration of the workload certificate. 

> The above process repeats periodically for certificate and key rotation.

</details>

Peer authentication is normally enabled at the `mesh` level, as in our 
training infrastructure, which means traffic is secured by **default** 
for all services in the cluster **without** requiring code changes. 

There are four modes that can be set for mTLS. They are `UNSET`, `DISABLE`, 
`PERMISSIVE` and `STRICT`.

<details>
    <summary> More About mTLS Modes </summary>

- `UNSET` - Modes is inherited from parent, defaults to PERMISSIVE

- `DISABLE` - Connection is **not** tunneled 

- `PERMISSIVE` - Connection can be either plaintext or mTLS tunnel

- `STRICT` - Connection is an mTLS tunnel (TLS with client cert must be presented)

> PeerAuthentication defines how traffic will be tunneled (or not) to the sidecar.

</details>

> :bulb: If you have not completed exercise 
> [00-setup-introduction](00-setup-introduction.md) you **need** to label 
> your namespace with `istio-injection=enabled`.

## Exercise: Mutual TLS Inside the Mesh

You will deploy all the services for the sentences application to observe 
the affects of mTLS configuration **without** sidecars. Afterwards you will 
inject sidecars and observe the effects of the mTLS configuration.

### PeerAuthentication

The PeerAuthentication CRD is used to specify the mTLS settings for a workloads 
**requests**.

An example of a peer authentication policy is seen below:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: foo
spec:
  mtls:
    mode: PERMISSIVE
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: foo
spec:
  selector:
    matchLabels:
      app: myapp
  mtls:
    mode: STRICT
```

The above policy says to allow **both** plain-text and mTLS traffic for **all** 
workloads in the `foo` namespace but **require** mTLS for the workload `myapp` 
in the `foo` namespace.

- The `namespace` definition scopes the policy to the defined namespace.
If a policy is **not** namespace scoped it applies to **all** workloads 
within the mesh.

> There can be only one **mesh-wide** peer authentication policy, and only one 
> **namespace-wide** peer authentication policy per namespace. If you configure 
> more then Istio will **ignore the newer policies**.

- The `selector` determines the workload to apply the ChannelAuthentication to.

> You can also specify the mTLS mode at the **port** level **if** you specify 
a `selector`. 

### DestinationRule

The DestinationRule CRD is used to specify the TLS settings for upstream 
connections. E.g. traffic the workload sends. These settings are common for 
both HTTP and TCP connections. 

In a destination rule you use the `tls` keyword instead of the `mtls` keyword 
under `spec.trafficPolicy`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: upstream-mtls
spec:
  host: upstream-app
  trafficPolicy:
    tls:                        # tls instead of mtls
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```

The modes are `DISABLE`, `SIMPLE`, `MUTUAL` and `ISTIO_MUTUAL`. For services 
within the mesh `ISTIO_MUTUAL` is the natural choice as it uses Istio's 
generated certs and they do not need to be specified in the destination rule.

<details>
    <summary> More About TLS Modes </summary>

- `DISABLE` - Do not setup a TLS connection to the upstream endpoint.

- `SIMPLE` - Originate a TLS connection to the upstream endpoint.

- `MUTUAL` - Secure connections to the upstream using **mutual** TLS by 
presenting client certificates for authentication.

> You **must** specify the certificates when modes is set to MUTUAL.

- `ISTIO_MUTUAL` - Same as MUTUAL except you do **not** specify the 
client certificates as the certs generated for mTLS by Istio are used.

</details>

### Overview

A general overview of what you will be doing in the **Step By Step** section.

- Deploy the sentences application services **without** sidecars and see 
the effect of mTLS modes

- Deploy the sentences application services **with** sidecars and see the
effect of mTls modes

- Configure TLS to an upstream service

- Observe the traffic flow with Kiali between different steps

### Step by Step

Expand the **Tasks** section below to do the exercise.

<details>
    <summary> Tasks </summary>

#### Task: Deploy the sentences application and observe sidecars

___


Run a for loop to substitute placeholders for environment variables and deploy 
the services along with an ingress gateway and virtual service for routing 
inbound traffic.

```console
for file in 03-ingress-traffic/start/*.yaml; do envsubst < $file | kubectl apply -f -; done
```

Execute `kubectl get pods` and observe that we have one container per POD, 
i.e. **no Istio sidecars injected**.

```
NAME                          READY   STATUS    RESTARTS   AGE
age-v1-b69df859b-m996r        1/1     Running   0          30s
name-v1-6f44875ccd-sz5zp      1/1     Running   0          30s
name-v2-7755ddbd74-4ppgh      1/1     Running   0          30s
sentences-v1-fc7dbd55-zx8qs   1/1     Running   0          30s
```

#### Task: Run the script `./scripts/loop-query.sh`

___


The sentence service we deployed in the first step has a type of `ClusterIP` 
now. In order to reach it we will need to go through the `istio-ingressgateway`. 

Run the `loop-query.sh` script with the option `-g`.

```console
./scripts/loop-query.sh -g $STUDENT_NS.sentences.$TRAINING_NAME.eficode.academy
```

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Select the `istio-ingress` namespace along with your namespace and select the
display checkboxes as shown below.

Select the **arrow** from the ingress gateway to the sentences service.

If we observe the result in Kiali, we will see that we only have information
about traffic from the ingress gateway towards the frontend sentences service
as none of the sentences services have sidecars, only the ingress gateway.

![Kiali with no sidecars](images/kiali-no-sidecar-no-mtls.png)

#### Task: Create peer authentication requiring `STRICT` mTLS

___


Create a file called `peer-authentication.yaml` in 
`05-securing-with-mtls/start/`.

Paste the following yaml into the file.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: $STUDENT_NS
spec:
  mtls:
    mode: STRICT
```

> :bulb: Note `$STUDENT_NS` in the above yaml. It is important that the 
> PeerAuthentication CRD is scoped to **your individual** namespace. There can 
> only be one mesh wide and one namespace wide policy at a time.

Substitute the placeholder with the value of the environment variable and apply 
the output with kubectl.

```console
envsubst < 05-securing-with-mtls/start/peer-authentication.yaml | kubectl apply -f -
```

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Select the `istio-ingress` namespace along with your namespace and select the
display checkboxes as shown below.

Select the **arrow** from the ingress gateway to the sentences service.

We see, that traffic still flows. This is because the sentences services do not
have an Istio sidecar and the strict peer-authentication policy we created only
applies to the namespace where we created it and it is only applied when
validating requests made *towards* a workload with an Istio sidecar. 

> Even if we created a strict policy in the namespace of the ingress gateway, 
> traffic would still flow. **No sidecar means no policy enforcement**.

![Kiali with no sidecars](images/kiali-no-sidecar-no-mtls.png)

#### Task: Enable sidecar for the age service

___


Lets inject Istio sidecar into the **age** service by removing the 
`sidecar.istio.io/inject: 'false'` annotation from the deployment.

```console
cat 05-securing-with-mtls/start/age.yaml |grep -v inject | kubectl apply -f -
```

Wait for the pod to be redeployed. 

```console
watch kubectl get pods
```

You should see it being terminated and new instantiated. Once it has been 
redeployed hit `ctrl+c` to exit the watch.

```console
NAME                          READY   STATUS    RESTARTS   AGE
age-v1-698cf6fd9d-kgvt2       2/2     Running   0          81s
name-v1-6f44875ccd-bsnrm      1/1     Running   0          57m
name-v2-7755ddbd74-mxlbt      1/1     Running   0          57m
sentences-v1-fc7dbd55-9smjb   1/1     Running   0          57m
```

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Select the `istio-ingress` namespace along with your namespace and select the
display checkboxes as shown below.

Select the **arrow** from the ingress gateway to the sentences service.

After this we see, that traffic no longer flows. We have applied a `STRICT` 
policy saying **all** traffic **must** be mTLS. But **only** the `age` service 
has a sidecar and we therefor have **both** plain-text and mTLS traffic.

![Kiali with no sidecars](images/kiali-mtls-error.png)

#### Task: Allow un-encrypted and un-authenticated traffic using `PERMISSIVE` mTLS

___


Modify the `peer-authentication.yaml` in `05-securing-with-mtls/start/` 
to use `PERMISSIVE` mode.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: $STUDENT_NS
spec:
  mtls:
    mode: PERMISSIVE    # <---- Change mode.
```

> While migrating an application to full mTLS, it may be useful to start with 
> a `PERMISSIVE` mTLS mode which allow a mix of mTLS and un-encrypted and 
> un-authenticated traffic.

Substitute the placeholder with the value of the environment variable and apply 
the output with kubectl.

```console
envsubst < 05-securing-with-mtls/start/peer-authentication.yaml | kubectl apply -f -
```

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. 

Select the `istio-ingress` namespace along with your namespace and select the
display checkboxes as shown below.

Select the **arrow** from the ingress gateway to the sentences service.

The traffic is now flowing but you **may** see a disjointed graph and `unknown` 
traffic. This is because the `age` service is the **only** service which has a 
sidecar.

![Disjointed graph](images/kiali-disjointed-graph.png)

#### Task: Inject sidecars for all services

___


Lets inject Istio sidecars into all sentences services by running a for loop 
removing the `sidecar.istio.io/inject: 'false'` annotation from all deployments.

```console
for file in 05-securing-with-mtls/start/*.yaml; do grep -v inject < $file | kubectl apply -f -; done
```

Wait for the pods to be redeployed.

```console
watch kubectl get pods
```

You should see them being terminated and new instantiated. Once the have all 
been redeployed hit `ctrl+c` to exit the watch.

```console
NAME                            READY   STATUS        RESTARTS   AGE
age-v1-698cf6fd9d-kgvt2         2/2     Running       0          35m
name-v1-6f44875ccd-bsnrm        1/1     Terminating   0          91m
name-v1-7f7bcf7fb8-btpff        2/2     Running       0          24s
name-v2-6886569bfb-lxb9z        2/2     Running       0          24s
name-v2-7755ddbd74-mxlbt        1/1     Terminating   0          91m
sentences-v1-6f9578db77-xbbzf   2/2     Running       0          24s
sentences-v1-fc7dbd55-9smjb     1/1     Terminating   0          91m
```

#### Task: Re-enabled `STRICT` mTLS

___


Since **all** services now have an Istio sidecar, we can enable strict mTLS:

Modify the `peer-authentication.yaml` in `05-securing-with-mtls/start/` 
to use `STRICT` mode.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: $STUDENT_NS
spec:
  mtls:
    mode: STRICT    # <-- Change mode.
```

Substitute the placeholder with the value of the environment variable and apply 
the output with kubectl.

```console
envsubst < 05-securing-with-mtls/start/peer-authentication.yaml | kubectl apply -f -
```

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. Select the checkboxes as shown in the below image.

Now we can see in Kiali, that mTLS is enabled between all services of the
sentences application. Kiali will denote it with a **lock** icon on the graph. 
In the view below, the link between the frontend and the `age` service has 
been selected and you can see that mTLS is enabled in the details view.

![Full mTLS](images/kiali-mtls-anno.png)

#### Task: Configure TLS to an upstream service

___


To show how we can control upstream mTLS settings with a DestinationRule, we
create one that uses mTLS towards `v2` of the `name` service and no mTLS for
`v1`. 

> :bulb: Note that we now need to use a `PERMISSIVE` Policy.

Modify the `peer-authentication.yaml` in `05-securing-with-mtls/start/` 
to use `PERMISSIVE` mode.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: $STUDENT_NS
spec:
  mtls:
    mode: PERMISSIVE
```

Substitute the placeholder with the value of the environment variable and apply 
the output with kubectl.

```console
envsubst < 05-securing-with-mtls/start/peer-authentication.yaml | kubectl apply -f -
```

Note, that a DestinationRule *will not* take effect until a route rule
explicitly sends traffic to a subset. So you need both a virtual 
service and a destination rule. 

Create a file called `name-vs-dr.yaml` in `05-securing-with-mtls/start/` 
directory.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: name
spec:
  hosts:
  - name
  exportTo:
  - "."
  gateways:
  - mesh
  http:
  - route:
    - destination:
        host: name
        port:
          number: 5000
        subset: name-v1
      weight: 50
    - destination:
        host: name
        port:
          number: 5000
        subset: name-v2
      weight: 50
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: name
spec:
  host: name
  exportTo:
  - "."
  subsets:
  - name: name-v1
    labels:
      version: v1
    trafficPolicy:
      tls:
        mode: DISABLE         # Plain text traffic
  - name: name-v2
    labels:
      version: v2
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL    # Mutual TLS using Istio generated certs
```

Apply the virtual service and destination rule with the mTLS settings.

```Console
kubectl apply -f 05-securing-with-mtls/start/name-vs-dr.yaml
```

#### Task: Observe the traffic flow with Kiali

___


Go to Graph menu item and select the **Versioned app graph** from the drop 
down menu. Select the checkboxes as shown in the below image.

Now we see a missing padlock on the traffic towards `v1` of the name service.

![No mTLS towards v1](images/kiali-mtls-destrule-anno.png)

</details>

## Exercise: Mutual TLS From External Clients

This exercise extends what we have done so far to demonstrate a common 
use case of configuring mTLS with a service **external** to the mesh. A 
service running in a different mesh for example. 

In order to do this we will create a Certificate authority and certificates 
to enable strong workload identity.

The TLS settings to control this will be defined in the 
[Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/#Gateway) 
CRD.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-unique-gateway
  namespace: istio-ingress
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: MUTUAL
      credentialName: server-side-tls-secret
    hosts:
    - "myapp.example.com"

```

- The port and tls blocks define the protocol and TLS mode to apply the 
configuration to. In this exercise you will be using the `SIMPLE` and 
the `MUTUAL` TLS modes. See this link for more information about 
[Server TLS modes](https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode).

- The credentialName represents a kubernetes secret containing the server side 
certificate.

> :bulb: The secret **must** be in the **same** namespace as the ingress gateway 
> controller. The gateway must be **namespaced** to be in the same namespace as 
> the kubernetes secret. So **both** must be in the `istio-ingress` namespace in 
> the above example and the gateway **must** be unique for this exercise.

### Overview

A general overview of what you will be doing in the **Step By Step** section.

- Generate needed certificate authority(CA) and certificates 

- Require `MUTUAL` TLS for port `443`

- Modify the virtual service to point to the gateway in `istio-ingress` namespace

- Test the traffic using a script in both TLS and mTLS modes

### Step by Step

Expand the **Tasks** section below to do the exercise.

<details>
    <summary> Tasks </summary>

#### Task: Delete the gateway created in the previous exercise

___


You must have the sentences application deployed with sidecars. This should 
already be the case, unless you skipped some of the first exercise.

```console
for file in 05-securing-with-mtls/start/*.yaml; do grep -v inject < $file | kubectl apply -f -; done
```

Ensure that the gateway and virtual service for the `sentences` 
service from the first exercise is removed.

```console
kubectl delete -f 05-securing-with-mtls/start/sentences-ingress-gw.yaml
kubectl delete -f 05-securing-with-mtls/start/sentences-ingress-vs.yaml
```

#### Task: Generate needed certificate authority(CA) and certificates

___


Execute the `generate-certs.sh`script.

```console
./scripts/generate-certs.sh
```
You should see namespace specific certs for both the server side and 
client side in your workspace.

There should also be a kubernetes secret in the `istio-ingress` namespace. 

```console
kubectl get secrets -n istio-ingress
```

It should be pre-pended with your namespace and look something like below.

```console
NAME                              TYPE      DATA   AGE
student1-sentences-tls-secret        Opaque    3      112m
```
> :bulb: DO NOT touch any other secrets in the `istio-ingress` namespace!

#### Task: Modify the gateway to configure `MUTUAL` TLS for port `443`

___


Modify the file `05-securing-with-mtls/start/sentences-ingress-gw.yaml` so 
that it will be namespaced to the `istio-ingress` namespace and configure it 
for `MUTUAL` TLS on port 443.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: $STUDENT_NS-sentences                           # <----- Use your namespace to make it unique
  namespace: istio-ingress                              # <----- Must be in the istio-ingress namespace
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:                                               # <----- Add the port block with the port and protocol
      number: 443
      name: https
      protocol: HTTPS
    tls:                                                # <----- Add the tls block
      mode: MUTUAL                                      # <----- TLS mode
      credentialName: $STUDENT_NS-sentences-tls-secret  # <----- Add the kubernetes secret
    hosts:
    - "$STUDENT_NS.sentences.$TRAINING_NAME.eficode.academy"
```

#### Task: Modify the virtual service to point to the gateway in `istio-ingress` namespace

___


The gateway is now namespaced to the `istio-ingress` namespace so you need 
to tell the virtual service which namespace the gateway is located in.

Modify the file `05-securing-with-mtls/start/sentences-ingress-vs.yaml`.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: sentences
spec:
  hosts:
  - "$STUDENT_NS.sentences.$TRAINING_NAME.eficode.academy"
  gateways:
  - istio-ingress/$TRAINING_NAME-sentences  # <----- Use the gateway in the istio-ingress namespace
  http:
  - route:
    - destination:
        host: sentences
```

#### Task: Apply the changes to the virtual service and gateway

___


Once you have done the modifications to the gateway and virtual service, 
substitute the placeholders with environment variable(s) and apply with kubectl.

```console
envsubst < 05-securing-with-mtls/start/sentences-ingress-gw.yaml | kubectl apply -f -
envsubst < 05-securing-with-mtls/start/sentences-ingress-vs.yaml | kubectl apply -f -
```

#### Task: Run `loop-query-mtls.sh https+mtls`

___


The `loop-query-mtls.sh` script uses the `client` certificates that were 
generated for the mtls connection. 

```console
scripts/loop-query-mtls.sh https+mtls
```

Note the curl options printed when the script starts. The certificate 
authority **and** the client cert, which is required for mTLS, are specified.

It will look *something* like below but with the CA and client certs that can 
be found in your workspace as the below has been edited for simplicity.

```console
-------------------------------------
Using ingress gateway with label: app=istio-ingressgateway
Using URL: https://student1.sentences.istio.eficode.academy:443/
Using curl options: '--resolve sentences.istio.eficode.academy:443 --cacert eficode.academy.crt --cert client.crt --key client.key'
-------------------------------------
```

#### Task: Run `loop-query-mtls.sh https`

___


```console
scripts/loop-query-mtls.sh https
```

You should see an OpenSSL error. Note the curl options printed when the 
script starts. **Only** the **certificate authority** is specified as 
simple TLS only requires server side authentication.

```console
-------------------------------------
Using ingress gateway with label: app=istio-ingressgateway
Using URL: https://student1.sentences.istio.eficode.academy:443/
Using curl options: '--resolve student1.sentences.istio.eficode.academy:443 --cacert eficode.academy.crt'
```

If you change the TLS mode to `SIMPLE` then plain `https` will work. You can 
still pass the `https+mtls` argument to the script and a connection will be 
made. Istio will simply ignore the client certs passed through the curl options.

</details>

### Summary

PKI (Public Key Infrastructure) does not necessarily mean, that we are using
internet-scoped public/private key encryption. In this exercise we have seen how
we can leverage Istio's **internal PKI** to implement mTLS inside the Istio mesh
between PODs with Istio sidecars. We have also seen how to setup mTLS for Istio
ingress gateways.

The main takeaways are:

- Peer authentication policies apply to traffic a workload **receives**

- Peer authentication policies only apply to workloads with a sidecar

- Peer authentication policies apply to the **all** workloads within the 
mesh if **not** namespaced

- The `STRICT` policy means strict and a `PERMISSIVE` policy is a good way to get started

- Destination rules configure TLS for a workloads upstream traffic

- When securing external traffic through ingress gateways you need to 
consider namespaces in relation to kubernetes secrets and gateway controllers.

## Cleanup

#Todo: Check this as we now are dynamically naming gatways based of env vars -> Use envsubst?

```console
kubectl delete -f 05-securing-with-mtls/start/
```