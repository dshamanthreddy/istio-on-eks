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
    ```

    ```text
    NAMESPACE      NAME                          READY   STATUS    RESTARTS   AGE
    istio-system   grafana-6c689999f9-5lk9b      1/1     Running   0          37s
    istio-system   istio-cni-node-28w2s          1/1     Running   0          10m
    istio-system   istio-cni-node-v4fc8          1/1     Running   0          12m
    istio-system   istiod-759544898-n84g5        1/1     Running   0          12m
    istio-system   kiali-95cffb658-8dp42         1/1     Running   0          10m
    istio-system   prometheus-6bd68c5c99-z6flt   2/2     Running   0          10m
    istio-system   ztunnel-82xvq                 1/1     Running   0          12m
    istio-system   ztunnel-csb26                 1/1     Running   0          10m

    NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                 AGE
    service/grafana      ClusterIP   172.20.210.200   <none>        3000/TCP                                2m14s
    service/istiod       ClusterIP   172.20.80.137    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP   14m
    service/kiali        ClusterIP   172.20.65.49     <none>        20001/TCP,9090/TCP                      12m
    service/prometheus   ClusterIP   172.20.141.251   <none>        9090/TCP                                12m


2. Verify all the Helm releases installed in the `istio-system` and `istio-ingress` namespaces:

    ```sh
    helm list -n istio-system
    ```

    ```text
    NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART             APP VERSION
    istio-base      istio-system    1               2026-03-19 21:14:27.275765 -0400 EDT    deployed        base-1.28.1       1.28.1
    istio-cni       istio-system    1               2026-03-19 21:14:19.6922 -0400 EDT      deployed        cni-1.28.1        1.28.1
    istiod          istio-system    1               2026-03-19 21:14:17.901337 -0400 EDT    deployed        istiod-1.28.1     1.28.1
    ztunnel         istio-system    1               2026-03-19 21:14:23.912003 -0400 EDT    deployed        ztunnel-1.28.1    1.28.1
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

```

### Deploy Sample EKS Application

To demonstrate some of the features of Istio, deploy a retail store sample application. This sample application uses a microservices architecture with components written in various programming languages and uses a variety of data stores. By default, the UI service is set to type=LoadBalancer, but you update this to ClusterIP and let Istio handle traffic into the cluster later. Run the following commands in a second terminal session.

```sh
helm install cart oci://public.ecr.aws/aws-containers/retail-store-sample-cart-chart --version 1.3.0

helm install catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version 1.3.0
```

```sh
cat > checkout-values.yaml <<EOF
redis:
  create: true
app:
  persistence:
    provider: redis
  endpoints:
    orders: 'http://orders:80'
EOF
```

<!-- 1. Create the `sample` namespace and enable the sidecar injection on it

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
    ``` -->

## Destroy


```sh
terraform destroy --auto-approve
```