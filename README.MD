
# Running Ballerina with Istio 
[Istio](https://istio.io/) provides an easy way to create a network of deployed services with load balancing, service-to-service authentication, monitoring, and more, without requiring any changes in service code. 

> In this guide you will learn about building a Ballerina service and deploying it on Istio service mesh.  

The following are the sections available in this guide.

- [What you'll build](#what-youll-build)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
- [Deployment](#deployment)
- [Testing](#testing)
- [Observability](#observability)

## What you’ll build 
In this guide, you will build a simple Ballerina service that gives the current time and you will deploy that service on Istio. 
This service is deployed alongside Istio's Envoy sidecar and you will use observability capabilities of Istio to observe the Ballerina service using Grafana and Jaegar. 

![Ballerina on Istio](images/Istio.svg "Ballerina on Istio")
The Ballerina service is exposed via the sidecar proxy and the client accesses the service via the sidecar. All the metrics and tracing related activities are enforced at the sidecar and they are transparent to the service code.

## Compatibility
| Ballerina Language Version | Istio version                                                            |
| -------------------------- | ----------------------------------------------------------------------- |
| 0.980.0                    | [Istio 0.7.1 - March 2018](https://github.com/istio/istio/releases/tag/0.7.1) |

## Prerequisites
 
- [Ballerina Distribution](https://ballerina.io/learn/getting-started/)
- A Text Editor or an IDE 
- [Docker](https://docs.docker.com/engine/installation/)
- [Install Kubernetes.](https://kubernetes.io/docs/setup/)  
- [Istio installation](https://archive.istio.io/v0.7/docs/setup/kubernetes/quick-start.html). 
- Install [Jaegar](https://archive.istio.io/v0.7/docs/tasks/telemetry/distributed-tracing.html) and [Grafana](https://archive.istio.io/v0.7/docs/tasks/telemetry/using-istio-dashboard.html) addons for Istio. 


### Optional requirements
- Ballerina IDE plugins ([IntelliJ IDEA](https://plugins.jetbrains.com/plugin/9520-ballerina), [VSCode](https://marketplace.visualstudio.com/items?itemName=WSO2.Ballerina), [Atom](https://atom.io/packages/language-ballerina))

## Implementation

As the first step, you can build a Ballerina service that gives the current time as the output. You can simply create a file `time_service.bal` and add the following content to the service code.

```ballerina
import ballerina/http;
import ballerina/io;
import ballerina/time;

endpoint http:Listener listener {
    port:9095
};

@http:ServiceConfig {basePath:"/localtime"}
service<http:Service> time bind listener {
    @http:ResourceConfig{
        path: "/",  methods: ["GET"]
    }
    getTime (endpoint caller, http:Request request) {
        http:Response response = new;
        time:Time currentTime = time:currentTime();
        string customTimeString = currentTime.format("yyyy-MM-dd'T'HH:mm:ss");
        json timeJ = {currentTime : customTimeString };
        response.setJsonPayload(timeJ);
        _ = caller -> respond(response);
    }
}
```

Now you can add the Kubernetes annotations that are required to generate the Kubernetes deployment artifacts. 

```ballerina
import ballerina/http;
import ballerina/io;
import ballerina/time;
import ballerinax/kubernetes;


@kubernetes:Ingress {
    name:"ballerina-time-service",
    path:"/localtime",
    ingressClass:"istio"
}

@kubernetes:Service {
    serviceType:"NodePort",
    name:"ballerina-time-service"
}
endpoint http:Listener listener {
    port:9095
};


@kubernetes:Deployment {
    image: "ballerina-time-service",
    name: "ballerina-time-service",
    singleYAML:true
}

@http:ServiceConfig {basePath:"/localtime"}
service<http:Service> time bind listener {
    @http:ResourceConfig{
        path: "/",  methods: ["GET"]
    }
    getTime (endpoint caller, http:Request request) {
        http:Response response = new;
        time:Time currentTime = time:currentTime();
        string customTimeString = currentTime.format("yyyy-MM-dd'T'HH:mm:ss");
        json timeJ = {currentTime : customTimeString };
        response.setJsonPayload(timeJ);
        _ = caller -> respond(response);
    }
}
```

- Please note that we have to override the default Ingress class to `ingressClass:"istio"` 
- If you are using Minikube, you need to specify `dockerHost` and `dockerCertPath` in you deployment annotations. For example, you can have the following Kubernetes annotations.
 

``` ballerina
@kubernetes:Ingress {
    name:"ballerina-time-service",
    path:"/localtime",
    ingressClass:"istio"
}

@kubernetes:Service {
    serviceType:"NodePort",
    name:"ballerina-time-service"
}
endpoint http:Listener listener {
    port:9095
};

@kubernetes:Deployment {
    image: "ballerina-time-service",
    name: "ballerina-time-service",
    dockerHost:"tcp://<minikube ip>:2376", // IP can be obtained via `minikube ip` command
    dockerCertPath:"<Home Dir>/.minikube/certs",
    singleYAML:true
}

```

By default, Ballerina build creates Kubernetes artifacts in multiple .yaml files. You can override this by adding `singleYAML:true` to the Kubernetes deployment annotation of your Ballerina service, so that it creates a single Kubernetes deployment artifact. 

You can build the Ballerina service using `$ ballerina build time_service.bal`. You should be able to see the following output. 

```bash
$ ballerina build time_service.bal
Compiling source
    time_service.bal

Generating executable
    ./target/time_service.balx
        @kubernetes:Service                      - complete 1/1
        @kubernetes:Ingress                      - complete 1/1
        @kubernetes:Deployment                   - complete 1/1
        @kubernetes:Docker                       - complete 3/3 

        Run following command to deploy kubernetes artifacts: 
        kubectl apply -f /Users/anuruddha/workspace/ballerinax/guide-test/ballerina-with-istio/src/kubernetes/
```
    
## Deployment                                                                                                                                    
- Before deploying the service on Istio, you need to do few modifications to the deployment descriptors.
- Open generated ``kubernetes/time_service.yaml`` file. 
- Updating ingress definition : 
    - Remove `` host: "listener.com"`` and ``tls - hosts: [] :`` section. 
    - Final ingress definition should look like below. 
    
    ```yaml
        ---
        apiVersion: "extensions/v1beta1"
        kind: "Ingress"
        metadata:
          annotations:
            kubernetes.io/ingress.class: "istio"
          finalizers: []
          labels:
            app: "time_service"
          name: "ballerina-time-service"
          ownerReferences: []
        spec:
          rules:
          - http:
              paths:
              - backend:
                  serviceName: "ballerina-time-service"
                  servicePort: 9095
                path: "/localtime"
    ```
- Updating service definition : 
    - Replace ``protocol: "TCP"`` with ``name: "http"`` in service definition. 
    - Final service definition should look like below.
    
    ```yaml
      ---
      apiVersion: "v1"
      kind: "Service"
      metadata:
        annotations: {}
        finalizers: []
        labels:
          app: "time_service"
        name: "ballerina-time-service"
        ownerReferences: []
      spec:
        externalIPs: []
        loadBalancerSourceRanges: []
        ports:
        - port: 9095
          name: "http"
          targetPort: 9095
        selector:
          app: "time_service"
        type: "NodePort"
    ```
- Please refer [time_service.yaml](./kubernetes/time_service.yaml), which contains all the aforementioned changes. 

Now you are all set to deploy your Ballerina service on Istio. To do that you need to inject the sidecar into your service's deployment descriptors. You can do that by executing the following.

```bash 
$ cd kubernetes/
$ istioctl kube-inject -f time_service.yaml -o time_service_k8s_istio_injected.yaml
```

Finally, you can deploy the Istio sidecar injected service using the following command. 

``` bash
$ kubectl apply -f time_service_k8s_istio_injected.yaml
service "ballerina-time-service" created
ingress.extensions "ballerina-time-service" created
deployment.extensions "ballerina-time-service" created 
```

You can verify the deployment by checking the pod, service, and ingress. Also, you can verify that your service is running along with a Istio sidecar proxy using `kubectl describe pod < include your ballerina-time-service pod name>`. 

## Testing 

You can invoke the Ballerina service that you have deployed above via Istio. 

If you are running Istio on Minikube you need to find the ingress URL of Istio to access your service. For that you have to determine the Minikube host name (you can find this with `$minikube ip`) and the NodePort for Istio ingress using `kubectl get svc istio-ingress -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}'`. 

Then you can access the Ballerina service as follows. 

``` bash
$ curl http://<minikube_ip>:<istio_ingress_node_port>/localtime
{"currentTime":"2018-05-01T21:47:53"}
``` 

If you are using Docker for Mac OSX you should be able to access it via `http://localhost/localtime`. 

## Observability 
Now you can use Istio's observability capabilities to observe your Ballerina service. 

### Distributed Tracing with Jaeger 
- To enable distributed tracing, you need to install the Jaeger addon for Istio as mentioned in the prerequisites section. 
- Setup port-forwarding with `kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686 &`. 
- Now you can access Jaeger at `http://localhost:16686/search`. 
- You should be able to see the tracers for your Ballerina time-service as shown below. 
![Tracing with Jaeger](images/jaeger1.png "Jaeger Tracing 1")
- Also you can drill down further to see that traffic is flowing through the Istio sidecar to the Ballerina service. 
![Tracing with Jaeger](images/jaeger2.png "Jaeger Tracing 2")


### Metrics with Istio Dashboard on Grafana  
- You can install the Grafana and Prometheus addons as mentioned in the prerequisites section. 
- Once you have added it, setup port-forwarding using `kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &`. 
- Now you can access the Istio Dashboard via `http://localhost:3000/dashboard/db/istio-dashboard`. 
- You can observe the metrics for your Ballerina service as shown below. 
![Tracing with Istio Dashboard](images/istio-graf1.png "Tracing with Istio Dashboard")
![Tracing with Istio Dashboard](images/istio-graf2.png "Tracing with Istio Dashboard")

