# Amazon EKS Auto Mode Cluster w/ Istio (`Ambient` mode)

This example demonstrates provisioning an EKS Auto Mode cluster with Istio in `Ambient` mode.

- Deploy an EKS Auto Mode cluster in a VPC. Auto Mode automatically manages compute, networking, and security group configurations.
- Install Istio in `Ambient` mode using Helm resources in Terraform.
- Deploy/Validate Istio communication using a sample application.

Refer to the [Istio documentation](https://istio.io/latest/docs/concepts/) for detailed explanations of Istio concepts.

## Deploy

Refer to the [prerequisites](https://aws-ia.github.io/terraform-aws-eks-blueprints/getting-started/#prerequisites) and run the following command to deploy this pattern:

```sh
cd terraform-blueprint/ambient
terraform init
terraform apply --auto-approve
aws eks --region us-west-2 update-kubeconfig --name ambient
```

Once the resources have been provisioned, you will need to replace the `istio-ingress` pods due to a [`istiod` dependency issue](https://github.com/istio/istio/issues/35789). Use the following command to perform a rolling restart of the `istio-ingress` pods:

```sh
kubectl rollout restart deployment istio-ingress -n istio-ingress
```

### Observability Add-ons

Use the following code snippet to add the Istio Observability Add-ons (Kiali and Prometheus) on the EKS cluster with deployed Istio.

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/prometheus.yaml \
  -f https://raw.githubusercontent.com/istio/istio/release-1.28/samples/addons/kiali.yaml
```

### Kubernetes Gateway API CRDs (Optional)

EKS clusters don't include [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) custom resource definitions (CRDs) by default. The Gateway API is an open source standard interface for Kubernetes application networking and represents the next generation of managing ingress and service mesh traffic within a cluster. Istio supports the Kubernetes Gateway API, and you need these resources to allow ingress traffic into your cluster and to manage ambient mesh traffic.

> **Note:** There is a Gateway resource in the Istio APIs, but this walkthrough doesn't use that resource. There are [key differences](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/) between the two.

Install the Gateway API CRDs if they are not already present on your cluster:

```sh
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

**Why do you need this?**

- **Gateway**: A Gateway helps route traffic from outside the cluster to services running within it. Each Gateway is associated with a GatewayClass, which indicates the gateway controller (in this case, Istio) that handles the traffic for that Gateway. By default, Istio creates a ServiceAccount, Service, and Deployment that correspond to the Gateway configuration. If you need to adjust the settings of the underlying resources, you can create a ConfigMap and associate it with the Gateway resource.
- **HTTPRoute**: Route resources define rules for mapping requests through a Gateway to backend Kubernetes services. The HTTPRoute is specifically for the HTTP protocol and routes requests to your application services (e.g., the UI service).

## Validate

1. List out all pods and services in the `istio-system` namespace:

    ```sh
    kubectl get pods,svc -n istio-system
    kubectl get pods,svc -n istio-ingress
    ```

    ```text
    NAME                              READY   STATUS    RESTARTS   AGE
    pod/grafana-657df88ffd-89nxn      1/1     Running   0          16s
    pod/istio-cni-node-42mjk          1/1     Running   0          22m
    pod/istio-cni-node-24ctm          1/1     Running   0          22m
    pod/istiod-6768c599c5-6rwd9       1/1     Running   0          23m
    pod/jaeger-697d898d6-32gsr        1/1     Running   0          22s
    pod/kiali-5899548ff7-1xx5h        1/1     Running   0          24s
    pod/prometheus-777db476b6-3gvg7   2/2     Running   0          18s
    pod/ztunnel-56s7h                 1/1     Running   0          22m
    pod/ztunnel-7wbr5                 1/1     Running   0          22m

    NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                          AGE
    service/grafana            ClusterIP   172.20.79.2      <none>        3000/TCP                                         16s
    service/istiod             ClusterIP   172.20.90.17     <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP            23m
    service/jaeger-collector   ClusterIP   172.20.221.206   <none>        14268/TCP,14250/TCP,9411/TCP,4317/TCP,4318/TCP   21s
    service/kiali              ClusterIP   172.20.241.225   <none>        20001/TCP,9090/TCP                               24s
    service/prometheus         ClusterIP   172.20.39.12     <none>        9090/TCP                                         18s
    service/tracing            ClusterIP   172.20.195.31    <none>        80/TCP,16685/TCP                                 22s
    service/zipkin             ClusterIP   172.20.92.216    <none>        9411/TCP                                         22s
    NAME                                READY   STATUS    RESTARTS   AGE
    pod/istio-ingress-94f46b75b-w5pch   1/1     Running   0          22m

    NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                     PORT(S)                                      AGE
    service/istio-ingress   LoadBalancer   172.20.249.189   k8s-istioing-istioing-21ba5f8e50-56515edc7fdae5d5.elb.us-west-2.amazonaws.com   15021:32477/TCP,80:32556/TCP,443:32006/TCP   23m
    ```

2. Verify all the Helm releases installed in the `istio-system` and `istio-ingress` namespaces:

    ```sh
    helm list -n istio-system
    ```

    ```text
    NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART               APP VERSION
    istio-base      istio-system    1               2024-06-03 15:49:08.443242104 -0700 PDT deployed        base-1.22.0         1.22.0
    istio-cni       istio-system    1               2024-06-03 15:49:02.186964057 -0700 PDT deployed        cni-1.22.0          1.22.0
    istiod          istio-system    1               2024-06-03 15:49:07.609140674 -0700 PDT deployed        istiod-1.22.0       1.22.0
    ztunnel         istio-system    1               2024-06-03 15:49:11.624277009 -0700 PDT deployed        ztunnel-1.22.0      1.22.0
    ```

    ```sh
    helm list -n istio-ingress
    ```

    ```text
    NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART               APP VERSION
    istio-ingress   istio-ingress   1               2024-06-03 15:49:14.784086208 -0700 PDT deployed        gateway-1.22.0      1.22.0
    ```

### Observability Add-ons

Validate the setup of the observability add-ons by running the following commands
and accessing each of the service endpoints using this URL of the form
[http://localhost:\<port>](http://localhost:<port>) where `<port>` is one of the
port number for the corresponding service.

```sh
# Visualize Istio Mesh console using Kiali
kubectl port-forward svc/kiali 20001:20001 -n istio-system

# Get to the Prometheus UI
kubectl port-forward svc/prometheus 9090:9090 -n istio-system

# Visualize metrics in using Grafana
kubectl port-forward svc/grafana 3000:3000 -n istio-system

# Visualize application traces via Jaeger
kubectl port-forward svc/tracing 16686:80 -n istio-system
```

### Example

1. Create the `sample` namespace and enable the sidecar injection on it

    ```sh
    kubectl create namespace sample
    kubectl label namespace sample istio.io/dataplane-mode=ambient
    ```

    ```text
    namespace/sample created
    namespace/sample labeled
    ```

2. Deploy `helloworld` app

    ```sh
    cat <<EOF | kubectl apply -n sample -f -
    apiVersion: v1
    kind: Service
    metadata:
      name: helloworld
      labels:
        app: helloworld
        service: helloworld
    spec:
      ports:
      - port: 5000
        name: http
      selector:
        app: helloworld
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: helloworld-v1
      labels:
        app: helloworld
        version: v1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: helloworld
          version: v1
      template:
        metadata:
          labels:
            app: helloworld
            version: v1
        spec:
          containers:
          - name: helloworld
            image: docker.io/istio/examples-helloworld-v1
            resources:
              requests:
                cpu: "100m"
            imagePullPolicy: IfNotPresent #Always
            ports:
            - containerPort: 5000
    EOF
    ```

    ```text
    service/helloworld created
    deployment.apps/helloworld-v1 created
    ```

3. Deploy `sleep` app that we will use to connect to `helloworld` app

    ```sh
    cat <<EOF | kubectl apply -n sample -f -
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: sleep
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: sleep
      labels:
        app: sleep
        service: sleep
    spec:
      ports:
      - port: 80
        name: http
      selector:
        app: sleep
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: sleep
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: sleep
      template:
        metadata:
          labels:
            app: sleep
        spec:
          terminationGracePeriodSeconds: 0
          serviceAccountName: sleep
          containers:
          - name: sleep
            image: curlimages/curl
            command: ["/bin/sleep", "infinity"]
            imagePullPolicy: IfNotPresent
            volumeMounts:
            - mountPath: /etc/sleep/tls
              name: secret-volume
          volumes:
          - name: secret-volume
            secret:
              secretName: sleep-secret
              optional: true
    EOF
    ```

    ```text
    serviceaccount/sleep created
    service/sleep created
    deployment.apps/sleep created
    ```

4. Check all the pods in the `sample` namespace

    ```sh
    kubectl get pods -n sample
    ```
    
    ```text
    NAME                             READY   STATUS    RESTARTS   AGE
    helloworld-v1-64674bb6c8-5szqq   1/1     Running   0          26s
    sleep-5577c64d7c-htrf8           1/1     Running   0          10s
    ```

5. Connect to `helloworld` app from `sleep` app and verify if the connection uses envoy proxy

    ```sh
    kubectl exec -n sample -c sleep \
        "$(kubectl get pod -n sample -l \
        app=sleep -o jsonpath='{.items[0].metadata.name}')" \
        -- curl -sv helloworld.sample:5000/hello
    ```

    ```text
    * Host helloworld.sample:5000 was resolved.
    ...
    * Connection #0 to host helloworld.sample left intact
    Hello version: v1, instance: helloworld-v1-64674bb6c8-43qfx
    ```

## Destroy


```sh
terraform destroy --auto-approve
```