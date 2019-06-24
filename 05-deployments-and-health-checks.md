# Deployments and Health Checks

## Module Objectives

- Convert a pod to a deployment
- Scale/update/rollout/rollback the deployment
- Define custom health checks and liveness probes

---

## Convert a Pod to a Deployment

There are a couple of problems with our current setup.

* If the application inside a Pod crashes, nobody will restart the Pod.
* We already use Services to connect our Pods to each other, but we don't have an efficient way to scale the Pods behind a Service and load balance traffic between Pod instances.
* We don't have an efficient way to update our Pods without a downtime. We also can't easily rollback to the previous version if we need to do so.

    **Deployments** can help to eliminate all these issues. Let's convert all our 3 Pods to Deployments.

1. Delete all running Pods.

    ```shell
    kubectl delete pod --all
    ```

1. Open the `manifests/db.yaml` file.

1. Delete the **top 2 lines**.

    ```yaml
    apiVersion: v1
    kind: Pod
    ```

1. **Indent/add 4 spaces** in the beginning of each line. If you use vim for editing you can check [this](http://vim.wikia.com/wiki/Shifting_blocks_visually) link to learn how to easily shift blocks of text in vim.

    > Tip: You can configure VIM to indent spaces with Tab using the following command `:set ts=4 sw=4 sts=4 et`

1. After indenting, add the following block to the **top** of the file:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: db
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: awsme
          role: db
      template:
    ```

    Your Pod definition becomes a template for the newly created Deployment. The Deployment Controller handles creating a ReplicaSet and creating the Pods.

1. **Repeat these steps** for the `backend` and `frontend` Pods. Don't forget to change the name of the Deployment and the `machLabels` element.

1. Apply the manifest files and list Deployments.

    ```shell
    kubectl get deployments
    ```

    ```
    NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    backend    1         1         1            0           1m
    db         1         1         1            1           1m
    frontend   1         1         1            1           6s
    ```

1. List ReplicaSets.

    ```shell
    kubectl get rs
    ```

    ```
    NAME                  DESIRED   CURRENT   READY     AGE
    backend-5c46cc4bb9    1         1         1         12m
    db-77df47c5dd         1         1         1         12m
    frontend-654b5ff445   1         1         1         11m
    ```

1. List Pods.

    ```shell
    kubectl get pod
    ```

    ```
    NAME                        READY     STATUS    RESTARTS   AGE
    backend-5c46cc4bb9-wf7qj    1/1       Running   0          12m
    db-77df47c5dd-ssgs6         1/1       Running   0          12m
    frontend-654b5ff445-b6fgs   1/1       Running   0          11m
    ```

1. List Services and connect to the `frontend` LoadBalancer service's IP in a web browser. Verify that the app is working fine.

    ```shell
    kubectl get services
    ```

    > Tip: When troubleshooting, start at the most abstract level and drill down as required. Service > Deployment > ReplicaSet > Pod > Logs. Use describe for more information if you notice something is not correct.

## Scale/Update/Rollout/Rollback the Deployment

### Scale the Deployment

1. Edit `manifests/backend.yaml`. Update the number of replicas to 3 and apply changes.

1. In your browser refresh the application several times. Notice that `Container IP` field sometimes changes. This indicates that the request comes to a different backend Pod.

### Update and Rollout the Deployment

1. Edit `main.go` file. Change `version` to `1.0.1` and save the file. (Around line 60)

1. Rebuild the container image with a new tag.

    ```shell
    export AWS_ACCOUNT_ID="1234567890"
    export IMAGE=$AWS_ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/sample-app:1.0.1
    docker build . -t $IMAGE
    docker push $IMAGE
    ```

1. Update `manifests/backend.yaml` to use the new version of the container image and apply the changes.

    Replace the image version in both the `initContainers` and `containers` sections.

    ```yaml
    image: $AWS_ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/sample-app:1.0.1
    ```

    > Note: Remember to replace `<AWS_ACCOUNT_ID>` with your AWS Account Id.

1. Watch how the Pods are rolled out in real time.

    ```shell
    watch kubectl get pod
    ```

1. Open the application in the browser and make sure the `backend` version is updated.

1. View the Deployment rollout history.

    ```shell
    kubectl rollout history deployment/backend
    ```

    ```
    deployments "backend"
    REVISION  CHANGE-CAUSE
    1         <none>
    2         <none>
    ```

### Rollback the Deployment

1. Rollback to the previous revision

    ```shell
    kubectl rollout undo deployment/backend
    ```

1. Make sure the application now shows version `1.0.0` instead of `1.0.1`.

## Define Custom Health Checks and Liveness Probes

By default, Kubernetes assumes that a Pod is ready to accept requests as soon as its container is ready and the main process starts running. Once this condition is met, Kubernetes Services will redirect traffic to the Pod. This can cause problems if the application needs some time to initialize. Let's reproduce this behavior.

1. Add `-delay=60` to the `command` property in the `manifests/backend.yaml` file and apply changes. This will cause our app to sleep for a minute on startup.

1. Open the app in the web browser. You should see the following error for about a minute:

    ```
    Error: Get http://backend:8080: dial tcp 10.100.58.231:8080: connect: connection refused
    ```

Now let's fix this problem by introducing a `readinessProbe` and a `livenessProbe`.

The `readinessProbe` and `livenessProbe` are used by Kubernetes to check the health of the containers in a Pod. The probes can check either HTTP endpoints or run shell commands to determine the health of a container. The difference between readiness and liveness probes is subtle, but important.

If an app fails a `livenessProbe`, kubernetes will restart the container.

> Caution: A misconfigured `livenessProbe` could cause deadlock for an application to start. If an application takes more time than the probe allows, then the livenessProbe will always fail and the app will never successfully start. See [this document](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) for how to adjust the timing on probes.

If an app fails a `readinessProbe`, Kubernetes will consider the container unhealthy and send traffic to other Pods.

> Note: Unlike liveness, the Pod is not restarted.

For this exercise, we will use only a `readinessProbe`.

1. Edit `manifests/backend.yaml`, add the following section into it and apply the changes.

    ```yaml
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8080
    ```

    This section should go under `spec -> template -> spec -> containers[name=backend]` and should be aligned with `image`, `env` and `command` properties.

1. Run `watch kubectl get pod` to see how Kubernetes is rolling out your Pods. This time it will do it much slower, making sure that previous Pods are ready before starting a new set of Pods.

1. **(Optional Bonus Challenge)** Define a HTTP healthcheck for the backend app. Convert the healthcheck to Container Exec and TCP. Verify that each type of healthcheck is working corectly. Use [official docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#container-v1-core) for guidance.

---

Next: [Persistence](06-persistence.md)
