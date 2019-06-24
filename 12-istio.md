# Istio

## Module Objectives

1. Configuring & installing Istio
1. Deploying a microservice with an Istio sidecar
1. Monitoring and tracing
1. Traffic Shifting
1. Fault Injection
1. Retries
1. Control Egress Traffic

---

## Configure & Install Istio

In this exercises you will deploy Istio onto the EKS cluster.

1. Delete everything from the default Namespace to ensure you have a clean state.

    ```sh
    kubectl delete deployment --all
    kubectl delete svc --all
    kubectl delete statefulset --all
    kubectl delete hpa --all
    ```

1. Download the Istio release.

    ```sh
    wget https://github.com/istio/istio/releases/download/1.1.4/istio-1.1.4-linux.tar.gz
    tar -zxvf istio-1.1.4-linux.tar.gz
    cd istio-1.1.4/
    ```

1. Add the `istioctl` client to your PATH, we can do this by adding the following line to the `~/.bashrc` file:

    ```sh
    export PATH=$HOME/istio-1.1.4/bin:$PATH
    ```

1. Install Istio's core components:

    ```sh
    kubectl create namespace istio-system
    ```

    ```sh
    helm template install/kubernetes/helm/istio-init \
    --name istio-init --namespace istio-system | kubectl apply -f -

    helm template install/kubernetes/helm/istio --name istio --namespace istio-system \
    --values install/kubernetes/helm/istio/values-istio-demo-auth.yaml \
    --set global.configValidation=false \
    --set sidecarInjectorWebhook.enabled=false --set grafana.enabled=true \
    --set servicegraph.enabled=true > istio.yaml
    ```

    ```sh
    kubectl apply -f istio.yaml
    ```

    This does the following:

    * Creates the `istio-system` Namespace along with the required RBAC permissions

    * Deploys the core Istio components:

        * `Istio-Pilot` is responsible for service discovery and for configuring the Envoy sidecar proxies in an Istio service mesh

        * The Mixer components `Istio-Policy` and `Istio-Telemetry` enforce usage policies and gather telemetry data across the service mesh

        * `Istio-Ingressgateway` provides an ingress point for traffic from outside the cluster

        * `Istio-Citadel` automates key and certificate management for Istio

    * Deploys plugins for metrics, logs, and tracing

    * Enables mutual TLS authentication between Envoy sidecars

1. Verify that Istio is correctly installed, also need to delete meshpolicies.authentication.istio.io

    ```sh
    kubectl get service -n istio-system
    kubectl get pods -n istio-system
    ```

    All pods should be in `Running` or `Completed` state.

Now you are ready to deploy the sample application to the Istio cluster.

## Deploying a microservice with an Istio sidecar

1. Delete and recreate each service to use type `ClusterIP`.

    ```sh
    kubectl get svc
    ```

    For the `frontend` service this would be the following:

    > Note: You need to delete the spec->ports[\*]->nodePort key, if it exists

    ```yaml
    kind: Service
    apiVersion: v1
    metadata:
      name: frontend
    spec:
      type: ClusterIP
      ports:
      -  port: 80
      selector:
        app: awsme
        role: frontend
    ```

1. Inject the sidecar container to the sample app from the previous section.

    ```sh
    istioctl kube-inject -f manifests/sample-app.yaml  > manifests/sample-app-istio.yaml
    ```

    Inspect the generated file.

1. Redeploy the sample app.

    ```sh
    kubectl apply -f manifests/sample-app-istio.yaml
    ```

1. Create an Istio Gateway as `manifests/istio-gateway.yaml` and apply the chagnes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: awsme-gateway
    spec:
      selector:
        istio: ingressgateway # Use Istio default controller
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
          - "*"
    ```

1. Create a VirtualService for the frontend as `manifests/frontend-vs.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: frontend-vs
    spec:
      hosts:
      - "*"
      gateways:
      - awsme-gateway
      http:
      - match:
        - uri:
            exact: /
        - uri:
            exact: /add-note
        - uri:
            exact: /healthz
        route:
        - destination:
            host: frontend
            port:
              number: 80
    ```

1. Check that the Gateway is created.

    ```shell
    kubectl get gateway
    ```

    ```sh
    NAME            AGE
    awsme-gateway   3m
    ```

1. Get Ingress IP info (from the load balancer).

    ```shell
    export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
    export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
    export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
    ```

    > Note: You can add this to your `~/.profile` file to automatically load on startup of new shell sessions.

1. Now the app should be reachable through the Istio gateway on `$GATEWAY_URL`.

You can write notes and save them in the database.

## Monitoring and Tracing

1. Set up a temporary tunnel to Grafana by using port-forwarding.

    > Note: When you CTRL+C from this process, the tunnel will be closed and you will no longer be able to access the UI.

    ```sh
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 8080:3000 &
    ```

1. Open the `awsme` app in a web browser and send a couple of requests.

1. Open 'Preview' -> 'Previw Running Application' at port `8080`, you should see the Grafana interface.

1. In the left panel click on `Dashboards -> Manage` and then select the `istio` folder. You should see a lot of dashboards, let's check a couple of them.

    * `Istio Mesh Dashboard` - Displays the overall volume of the requests as well as number of failed requests and track requests latency. Refresh the frontend page several times, add a couple of notes and make sure you see a spike in global request volume graph and changes in overall request statistics.
    * `Istio Service Dashboard` - Contains more detail information about request statistics. You can use this dashboard to see the request statistics per service.
    * `Istio Workload Dashboard` - Gives details about metrics for each workload and then inbound workloads (workloads that are sending request to this workload) and outbound services (services to which this workload send requests) for that workload.

1. Set up a temporary tunnel to ServiceGraph by using port-forwarding.

    ```sh
    kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=servicegraph -o jsonpath='{.items[0].metadata.name}') 8081:8088 &
    ```

1. Open 'Preview' -> 'Previw Running Application' at port `8081` and append `dotviz` to the path. You should see sample-app's service topology.

You can create Services if you want to permanently expose Grafana and ServiceGraph for managing and monitoring your service mesh.

## Traffic Shifting

Let's now see how Istio can help us to add new features to our application. Let's imagine that we want to add a new feature to the app and test it on a small percent of our users (this is called 'Canary deployment').

> Note: If you have already done this in exercise 5, skip to step 4.

1. In the `sample-app` folder open `main.go` file.

1. At line 60 find `version` constant and change it from `1.0.0` to `1.0.1`

1. Now rebuild the app image with a new version tag and push it to the AWS ECR. (These commands should be executed in the `sample-app` folder)

    ```sh
    export IMAGE=<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/sample-app:1.0.1
    docker build . -t $IMAGE
    docker push $IMAGE
    ```

1. Edit `manifests/sample-app.yaml`.

    1. Duplicate the `backend` deployment section.

    1. Keep the name for the first Deployment (`backend`) and name the second one `backend-v1`.

    1. Modify the first Deployment, add `version: v0` label to the `spec -> selectors -> matchLabels` and to the `spec -> templates -> metadata -> labels` elements.

    1. Modify the second Deployment, but this time use `version: v1` label instead.

1. Change the second Deployment image. Change the image tag from `1.0.0` to `1.0.1`

1. Delete the old sample application.

    ```sh
    kubectl delete -f manifests/sample-app.yaml
    ```

1. Apply changes.

    ```sh
    kubectl apply -f <(istioctl kube-inject -f manifests/sample-app.yaml)
    ```

    > Note: The backend container may crash a few times until it can establish a connection with the database, this is normal and can be fixed by implementing readiness checks for the backend.

1. Create a destination rule for the backend Service as `manifests/backend-dr.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: backend-dr
    spec:
      host: backend
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
      subsets:
      - name: v0
        labels:
          version: v0
      - name: v1
        labels:
          version: v1
    ```

1. Create a VirtualService for the backend as `manifests/backend-vs.yaml` and apply the changes.

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: backend-vs
    spec:
      hosts:
      - backend
      http:
      - route:
        - destination:
            host: backend
            subset: v0
          weight: 75
        - destination:
            host: backend
            subset: v1
          weight: 25
    ```

1. Open the app and refresh the page several times. You should see `1.0.0` backend version in 75% of cases and `1.0.1` in 25%.

## Fault Injection

One of the most difficult aspects of testing microservice applications is verifying that the application is resilient to failures. Each service should not assume that all its dependencies are available 100% of the time, instead it should be ready to handle any unexpected failure.

Usually people manually shut down application instances or block application ports in order to simulate failures, Istio provides us with a much better way: Fault Injection.

1. Modify `manifests/backend-vs.yaml` and add the following lines to `spec -> http[0]` (if you simply append them to the end of the file (including the spaces) it should work fine). Apply the changes.

    ```yaml
        fault:
          delay:
            fixedDelay: 3s
            percent: 50
    ```

1. Open the app and verify that in 50% of the times it should take 3 seconds to complete the request.

In a similar way you can inject not only delays, but also failures.

## Retries

Now let's inject a native failure to the backend application to demonstrate how Istio can help make microservices more resilient.

1. Modify `manifests/sample-app.yaml` and add `-fail-percent=50` parameter to the `backend` Deployment `command` property (leaving the second `backend_v1` deployment untouched) then apply the changes.

    ```sh
    kubectl apply -f <(istioctl kube-inject -f manifests/backend.yaml)
    ```

1. Delete the fault definition added from the previous exercise from the `manifests/backend-vs.yaml` file then apply the changes.

1. Observe that the application is failing 50% of the times (Failures should happen only if frontend connects to the `1.0.0` version of the backend).

1. Add retries to the `spec -> http[0]` section of the `manifests/backend-vs.yaml` then apply the changes.

    ```yaml
        retries:
          attempts: 3
          perTryTimeout: 2s
    ```

1. Test the app. You should no longer see any failures. Each failed request now retries up to 3 times.

---

Next: [Jenkins](13-jenkins.md)
