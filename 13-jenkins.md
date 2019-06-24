# Jenkins

## Module Objectives

1. Configure Jenkins
1. Deploy Jenkins
1. Access Jenkins

---

## Configure Jenkins

Look at possible configuration options: https://github.com/helm/charts/blob/master/stable/jenkins/README.md#configuration

1. Make an empty directory `jenkins/`

1. Start with empty `jenkins/values.yaml`

1. Add five sections

    - `Master`: configuration for Jenkins master
    - `Agent`: configuration for Jenkins agent
    - `Persistence`: configure Jenkins master disk
    - `NetworkPolicy`: limit network communications
    - `RBAC`: account and permissions configuration

    ```yaml
    Master:
    Agent:
    Persistence:
    NetworkPolicy:
    rbac:
    ```

1. Set disk size to `100Gi`

    ```yaml
    Persistence:
        Size: 100Gi
    ```

1. Disable network policy for now. You will enable it later while doing the Security module.

    ```yaml
    NetworkPolicy:
        Enabled: false
    ```

1. Set the API version for the NetworkPolicy

    ```yaml
    NetworkPolicy:
        Enabled: false
        ApiVersion: networking.k8s.io/v1
    ```

1. Install Default RBAC roles and bindings

    ```yaml
    rbac:
        install: true
        serviceAccountName: cd-jenkins
    ```

1. Enable Kubernetes plugin jnlp-agent podTemplate

    ```yaml
    Agent:
        Enabled: false
    ```

1. Set master resource limits (or requests?). 1 CPU and 3.5 Gb of memory.

    ```yaml
    Master:
        requests:
            cpu: "1"
            memory: "3500Mi"
        limits:
            cpu: "1"
            memory: "3500Mi"
    ```

1. Tell Jenkins to use all the memory available because it is a single process in container

    ```yaml
    Master:
        JavaOpts: "-Xms3500m -Xmx3500m"
    ```

1. Expose Jenkins UI using Cloud Load Balancer

    ```yaml
    Master:
        ServiceType: LoadBalancer
    ```

    Another option is `ClusterIP` which creates virtual IP inside the cluster. You may proxy this IP with `kubeproxy` to your local worstation and access it as if Jenkins was running on `localhost`.

1. Plugins extend Jenkins functionality. We will use several

    - __kubernetes__: launch builds in k8s containers
    - __workflow-aggregator__: A suite of plugins that lets you orchestrate automation, simple or complex
    - __workflow-job__: Defines a new job type for pipelines and provides their generic user interface
    - __credentials-binding__: Allows credentials to be bound to environment variables for use from miscellaneous build steps
    - __git__: check out application code from the repo
    - __google-oauth-plugin__: use credentials from the virtual machine metadata to access GCP services
    - __google-source-plugin__: credential provider to use GCP OAuth Credentials to access source code from Google Source Repositories

    ```yaml
    Master:
        InstallPlugins:
            - kubernetes:1.15.5
            - workflow-aggregator:2.6
            - workflow-job:2.32
            - credentials-binding:1.19
            - git:3.10.0
            - google-oauth-plugin:0.8
            - google-source-plugin:0.3
    ```

    You can find Jenkins plugins at URL: https://plugins.jenkins.io/

1. Now the `jenkins/values.yaml` file should look like

    ```yaml
    Master:
        InstallPlugins:
            - kubernetes:1.15.5
            - workflow-aggregator:2.6
            - workflow-job:2.32
            - credentials-binding:1.19
            - git:3.10.0
            - google-oauth-plugin:0.8
            - google-source-plugin:0.3
        requests:
            cpu: "1"
            memory: "3500Mi"
        limits:
            cpu: "1"
            memory: "3500Mi"
        JavaOpts: "-Xms3500m -Xmx3500m"
        ServiceType: LoadBalancer
    Agent:
        Enabled: false
    Persistence:
        Size: 100Gi
    NetworkPolicy:
        Enabled: false
        ApiVersion: networking.k8s.io/v1
    rbac:
        install: true
        serviceAccountName: cd-jenkins
    ```

## Deploy Jenkins

1. Deploy Jenkins chart using Helm

    ```sh
    helm install --name cd \
        --namespace cd \
        -f jenkins/values.yaml \
        --version 0.16.6 \
        stable/jenkins \
        --wait
    ```

    `--name` sets the name of Jenkins deployment called _release_. Using this name you can update and delete release in future.

    `-f jenkins/values.yaml` tells Helm to override default chart configuration values with the values from `jenkins/values.yaml`.

    With `--version` flag you choose specific versions of Jenkins chart to install. Note that chart version differs from the Jenkins version.

    `stable/jenkins` is the name of Helm chart

    By default `helm install` does not wait until deployment complete. `--wait` flag tells it to wait until Kubernetes creates all of Jenkins resources. It still takes time for Jenkins to boot so dashboard will not be available immediately after the install completes.

1. Wait until Jenkins pod goes to the `Running` state and the container is in the `READY` state:

    ```console
    kubectl -n cd get pods --watch
    ```

    Output:
    ```console
    NAME                          READY     STATUS    RESTARTS   AGE
    cd-jenkins-7c786475dd-vbhg4   1/1       Running   0          1m
    ```

1. Run the following command to get the URL of the Jenkins service

    ```sh
    export SERVICE_IP=$(kubectl get svc --namespace cd cd-jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
    echo http://$SERVICE_IP:8080/login
    ```

    `--watch` streams events from Kubernetes and contantly updates the Pod status

1. Now, check that the Jenkins Service was created properly:

    ```console
    kubectl get svc -n cd
    ```

    Output:
    ```console
    NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    cd-jenkins         LoadBalancer   10.47.255.13    35.236.21.7   8080:31027/TCP   3m
    cd-jenkins-agent   ClusterIP      10.47.246.125   <none>        50000/TCP        3m
    ```

We installed Jenkins with [Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin). This plugin launches Pods with executors each time Jenkins master requests them. After Jenkins executor completed the task Jenkins Kubernetes plugin disposes the Pod and frees the resources.

Note that this service exposes ports `8080` and `50000` for any pods that match the `selector`. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster.

Additionally the `cd-jenkins` services is exposed using a LoadBalancer so that it is accessible from outside the cluster.

## Connect to Jenkins

Jenkins requires username and password. `admin` is default username. Helm generates admin password and stores in the Kubernetes secret `cd-jenkins`.

1. Get an admin password

    ```sh
    printf $(kubectl -n cd get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
    ```

1. Log in with username `admin` and your auto generated password.

![](img/jenkins-login.png)

Optional exercises
------------------

1. What resources were created during Jenkins deployment?
1. If the service was exposed internally using `ClusterIP` how can you access it? How about via `port-forward`?

---

Troubleshooting:
1. If things aren't working we may have to check `Manage Jenkins` on the left navigation pane. Click on that and you may see some warnings about upgrading some plugins and that is probably what we should do.

1. If you upgrade a plugin but want to make sure this is part of your source code like what we have described above then you can track the plugin name by looking at the link name in the `Manage plugin page`, http://<YOURIP>:8080/pluginManager/.

    ![](img/jenkins-plugins.png)

1. For this type of deployment via helm, you can not upgrade, but rather it has to be deleted and recreated.

    ```
    helm upgrade -f jenkins/values.yaml cd stable/jenkins
    Error: UPGRADE FAILED: Deployment.apps "cd-jenkins" is invalid: spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy `type` is 'Recreate'

    helm del --purge cd
    release "cd" deleted
    ```

---

Next: [Pipelines](15-pipelines.md)
