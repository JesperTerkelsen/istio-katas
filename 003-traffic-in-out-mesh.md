[//]: # (Copyright, Eficode )
[//]: # (Origin: https://github.com/eficode-academy/istio-katas)
[//]: # (Tags: #sentences #kiali)

# Traffic in and out of mesh

## Learning goals

- Understand Istio gateways (ingress)
- Understand how to access external services
- Understand Istio gateways (egress)

## Introduction

These exercises will introduce you to Istio concepts and CRD's for configuring 
traffic **into** the service mesh (Ingress) and out of the service mesh (Egress). 

You will use two Istio CRD's for this. The **Gateway** and the **ServiceEntry** 
CRD's. 

## Exercise 1

The previous exercises used a Kubernetes **NodePort** service to get traffic 
to the sentences service. E.g. the **ingress** traffic to `sentences` **was 
not** flowing through the Istio service mesh. From the `sentences` 
service to the `age` and `name` services traffic **was** flowing through the 
Istio service mesh. We know this to be true because we have applied virtual 
services and destination rules to the `name` service.

Ingressing traffic directly from the Kubernetes cluster network to a frontend
service means that Istio features **cannot** be applied on this part of the 
traffic flow.

<details>
    <summary> More Info </summary>

A Gateway **describes** a load balancer operating at the **edge** of the mesh 
receiving incoming or outgoing **HTTP/TCP** connections. The specification 
describes the ports to be expose, type of protocol, configuration for the 
load balancer, etc.

An Istio **Ingress** gateway in a Kubernetes cluster consists, at a minimum, of a 
Deployment and a Service. Istio ingress gateways are based on the Envoy and have a 
standalone Envoy proxy. 

Inspecting our course environment would show something like:

```console
NAME                                        TYPE                                   
istio-ingressgateway                        deployment  
istio-ingressgateway                        service
istio-ingressgateway-69c77d896c-5vvjg       pod
```

Inspecting the POD would show something like:

```console
NAME                                    CONTAINERS
istio-ingressgateway-69c77d896c-5vvjg   istio-proxy
```

</details>

In this exercise we are going rectify this by **configuring** ingress traffic 
to the sentences service through a dedicated **ingress** 
gateway provided by **Istio** `istio-ingressgateway`.

You are going to do this by defining a gateway.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-app-gateway
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "my-app.example.com"
```

The servers block is where you define the port configurations, protocol 
and the hosts exposed by the gateway. A host entry is specified as a dnsName 
and should be specified using the FQDN format. 

> You can use a wildcard character in the **left-most** component of the 
> `hosts`field. E.g. `*.example.com`. You can also **prefix** the `hosts` field 
> with a namespace. 
> See the [documentation](https://istio.io/latest/docs/reference/config/networking/gateway/#Server) 
> for more details.

The **selectors** above are the labels on the `istio-ingressgateway` POD which 
is running a standalone Envoy proxy.

The gateway defines and **entry point** to be exposed in the 
`istio-ingressgateway`. That is it. Nothing else. This entry point knows 
nothing about how to route the traffic to the desired destination within the 
mesh. In order to route the traffic we, of course, use a virtual service. 

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - "my-app.example.com"
  gateways:
  - my-app-gateway
  http:
  - route:
    - destination:
        host: my-app-frontend
```

Note how it specifies the hostname and the name of the gateway 
(in `spec.gateways`). A gateway definition can define an entry for many 
hostnames and a VirtualService can be bound to multiple gateways, i.e. these 
are not necessarily related one-to-one.

### Overview

- Deploy the sentences app

- Create an entry point for the sentences service

> :bulb: The FQDN you will use should be 
> `<YOUR_NAMESPACE>.sentences.istio.eficode.academy`.

- Create a route from the entry point to the sentences service

- Run the loop query script with the `-g` option and FQDN

- Observe the traffic flow with Kiali

### Step by Step
<details>
    <summary> More Details </summary>

**Deploy the sentences app**

```console
kubectl apply -f 003-traffic-in-out-mesh/start/
kubectl apply -f 003-traffic-in-out-mesh/start/name-v1/
```

**Create an entry point for the sentences service**

Create a file called `sentences-ingressgateway.yaml` in 
`003-traffic-in-out-mesh/start` directory.

It should look like the below yaml. 

> :bulb: Replace <YOUR_NAMESPACE> in the yaml below with the namespace you 
> have been assigned in this course. Otherwise you might not hit the 
> `sentence` service in your namespace.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: sentences
spec:
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "<YOUR_NAMESPACE>.sentences.istio.eficode.academy"
```

Apply the resource:

```console
kubectl apply -f 003-traffic-in-out-mesh/start/sentences-ingressgateway.yaml
```

**Create a route from the gateway to the sentences service**

Create a file called `sentences-ingressgateway-vs.yaml` in 
`003-traffic-in-out-mesh/start` directory.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: sentences
spec:
  hosts:
  - "<YOUR_NAMESPACE>.sentences.istio.eficode.academy"
  gateways:
  - sentences
  http:
  - route:
    - destination:
        host: sentences
```

The VirtualService routes all traffic for the given hostname
to the `sentences` service (the two last lines specifying the Kubernetes
`sentences` service as destination).

Apply the resource:

```console
kubectl apply -f 003-traffic-in-out-mesh/start/sentences-ingressgateway-vs.yaml
```

**Run the loop query script with the `hosts` entry**

The sentence service we deployed in the first step has a type of `ClusterIP` 
now. In order to reach it we will need to go through the `istio-ingressgateway`. 

Run the `loop-query.sh` script with the option `-g` and pass it the `hosts` entry.

```console
./scripts/loop-query.sh -g <YOUR_NAMESPACE>.sentences.istio.eficode.academy
```

**Observe the traffic flow with Kiali**

Now we can see that the traffic to the `sentences` service is no longer 
**unknown** to the service mesh. 

![Ingress Gateway](images/kiali-ingress-gw.png)

</details>

## Exercise 2

A ServiceEntry allows you to apply Istio traffic management for services 
running **outside** of your mesh. Your service might use an external API 
as an example. Once you have defined a service entry you can configure 
virtual services and destination rules to apply Istio features like 
redirecting or forwarding traffic to external destinations, defining 
retries, timeouts and fault injection policies. 

<details>
    <summary> More Info </summary>

By default, Istio configures Envoy proxies to **passthrough** requests to 
unknown services. So, technically they are not required. But without them 
you can't apply Istio features. 

There is also the security aspect to consider. While securely controlling 
ingress traffic is the highest priority, it is good policy to securely control 
egress traffic also. As part of this many clusters will have the 
`outBoundTrafficPolicy` set to `REGISTRY_ONLY`. This will force you to define 
your external services with a service entry.

</details>

In this exercise we will define a service entry for [httpbin](https://httpbin.org/) 
and create a virtual service with a timeout of 3 seconds to prove we can use 
Istio features.

> :bulb: In our environment we have set the outBoundTrafficPolicy to 
> `REGISTRY_ONLY`. You will not be able to reach external services without 
> defining a service entry.

When you create a service entry it is added to Istio's internal service 
registry.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - external-api.example.com
  exportTo:
  - "."
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
```

The `hosts` field is used to select matching `hosts` in virtual services 
and destination rules. The `resolution` field is used to determine how the 
proxy will resolve IP addresses of the end points. The `exportTo` field scopes 
the service entry to the namespace where it is defined.

> :bulb: The `exportTo` field is important for the exercises in this course. 
> **Not** scoping the service entry to your namespace will open the external 
> service for **all** attendees. 

### Overview

- Deploy multitool which will be used to reach httpbin from within the mesh

- Run `./scripts/external-service-query.sh` and observe response

- Define a service entry for httpbin.org

- Run `./scripts/external-service-query.sh` and observe response

- Create a virtual service with a timeout of 3 seconds

- Run `./scripts/external-service-query.sh http://httpbin.org/delay/5`

### Step by Step
<details>
    <summary> More Details </summary>

**Deploy multitool**

We want to generate traffic **through** the service mesh. In order to do that 
we will deploy an image we built for container/network testing and 
troubleshooting. Our script will then use the `exec`command to have this 
workload curl the external service.

```console
kubectl apply -f 003-traffic-in-out-mesh/start/multitool/
```

**Run `./scripts/external-service-query.sh`**

```console
./scripts/external-service-query.sh http://httpbin.org
```

You should see something like below because the exists no service entry 
for the external service httpbin.

```console
Using multitool-66d9d48d44-s7wcb to query http://httpbin.org
HTTP/1.1 502 Bad Gateway
HTTP/1.1 502 Bad Gateway
HTTP/1.1 502 Bad Gateway
```

**Define a service entry for httpbin.org**

Create a service entry called `httpbin-service-entry.yaml`in 
`003-traffic-in-out-mesh/start/`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https-port
    protocol: HTTPS
  resolution: DNS
```

Apply the service entry.

```console
kubectl apply -f 003-traffic-in-out-mesh/start/httpbin-service-entry.yaml
```

**Run `./scripts/external-service-query.sh`**

```console
./scripts/external-service-query.sh http://httpbin.org
```

Now you should be getting a 200 OK response from httpbin. 

```console
Using multitool-66d9d48d44-s7wcb to query http://httpbin.org
HTTP/1.1 200 OK
HTTP/1.1 200 OK
HTTP/1.1 200 OK
```

Basically all we have done so far is to add an entry for httpbin to Istio's 
internal service registry. But we can now apply some of the Istio features to 
external service. 

**Create a virtual service with a timeout of 3 seconds**

Create a file called `httpbin-virtual-service.yaml` in 
`003-traffic-in-out-mesh/start/`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
```

Apply the virtual service.

```console
kubectl apply -f 003-traffic-in-out-mesh/start/httpbin-virtual-service.yaml
```

**Run `./scripts/external-service-query.sh http://httpbin.org/delay/5`**

We are going to ask httpbin to delay the response for 5 seconds.

```console
./scripts/external-service-query.sh http://httpbin.org/delay/5
```

Since you have injected a timeout of 3 seconds on the virtual service you 
should be seeing a 504 Gateway timeout.

```console
Using multitool-66d9d48d44-s7wcb to query http://httpbin.org/delay/5
HTTP/1.1 504 Gateway Timeout
HTTP/1.1 504 Gateway Timeout
HTTP/1.1 504 Gateway Timeout
```

If you ask httpbin for a delay **under** 3 seconds you should get a 200 OK.

</details>

## Exercise 3 <------ **SKIP FOR NOW! WORK IN PROGRESS!**

In a previous exercise you configured Istio to allow access to an external 
service([httpbin](http://httpbin.org)). You then applied a simple virtual 
service with a timeout to prove that Istio traffic management features can 
be applied.

The traffic was flowing directly from the workloads Envoy sidecar to the 
external service. 

But there are use cases where you need to have traffic leaving the mesh 
routed **via** a dedicated **egress** gateway.

In this exercise we will route outbound traffic for [httpbin](https://httpbin.org/) 
through the dedicated **egress** gateway provided by Istio `istio-egressgateway`.

<details>
    <summary> More Info </summary>


</details>

> :bulb: In our environment we have set the outBoundTrafficPolicy to 
> `REGISTRY_ONLY`. You will not be able to reach external services without 
> defining a service entry.

You are going to do this by defining a gateway. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: myapp-egressgateway
spec:
  selector:
    app: istio-egressgateway
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - myapp.org
```
The fields are the same as for the gateway you defined for the sentences 
service in a previous exercise. The notable difference being that the 
**selectors** are now the labels on the `istio-egressgateway` POD. It also 
is running a standalone Envoy proxy.

The gateway defines an **exit point** to be exposed in the `istio-egressgateway`. 
That is it. Nothing else. Again the exit point knows nothing about how traffic 
is routed to it. In order to route the traffic we, of course, use a virtual 
service. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: external-api
spec:
  hosts:
  - external-api.example.com
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: external-api.example.com
        port:
          number: 80
      weight: 100
```

In order to get the outbound traffic **from** the workload to our external 
service you need to define a route to the `istio-egressgateway`, which is 
located in the `istio-system` namespace. Then you need to define a route 
from the egress gateway to the external service. 

 The first route uses the reserved keyword `mesh` implying that this rule 
 applies to the sidecars in the mesh. E.g any sidecar wanting to hit the 
 external service will be routed to the `istio-egressgateway`.

 The second route is the one directing traffic from the egress gateway to 
 the external service.

### Overview

- Create an exit point for httpbin traffic

- Deploy multitool which will be used to reach httpbin from within the mesh

- Modify the `httpbin-virtual-service.yaml` from previous exercise

- 

### Step by Step
<details>
    <summary> More Details </summary>

**Create an exit point for httpbin traffic**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-egressgateway
spec:
  selector:
    app: istio-egressgateway
    istio: egressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - httpbin.org
```

**Deploy multitool**

We want to generate traffic **through** the service mesh. In order to do that 
we will deploy an image we built for container/network testing and 
troubleshooting. Our script will then use the `exec`command to have this 
workload curl the external service.

```console
kubectl apply -f 003-traffic-in-out-mesh/start/multitool/
```

**Run `./scripts/external-service-query.sh`**

```console
./scripts/external-service-query.sh http://httpbin.org
```

**Modify the `httpbin-virtual-service.yaml` from previous exercise**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-egress
spec:
  hosts:
  - httpbin.org
  gateways:
  - istio-egressgateway
  - mesh
  http:
  - match:
    - gateways:
      - mesh
      port: 80
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local
        #subset: httpbin
        port:
          number: 80
      weight: 100
  - match:
    - gateways:
      - istio-egressgateway
      port: 80
    route:
    - destination:
        host: httpbin.org
        port:
          number: 80
      weight: 100
```

</details>


# Summary

In exercise 1 you saw how to route incoming traffic through an ingress gateway. 
This allows you to apply istio traffic management features to the sentences 
service. For example, you could do a blue/green deploy of two different versions 
of the sentence service. 

In exercise 2 you created a service entry to allow access to an external service. 
This is a pretty common use case. A lot of service meshes will have a `REGISTRY_ONLY` 
policy defined for security reasons. So you should be aware of what a service entry does.

The important takeaway from these exercises is this.

**If traffic is not flowing through the mesh, e.g through the envoy sidecars, 
then you cannot leverage Istio features. Regardless of whether it is ingress or 
egress traffic.**

# Cleanup

```console
kubectl delete -f 003-traffic-in-out-mesh/start/multitool/
kubectl delete -f 003-traffic-in-out-mesh/start/
```