## Helm

## Module Objectives

1. Install Helm on EKS
1. Deploy simple workload with Helm

---

## Install Helm on EKS

Helm is a package manager for Kubernetes that packages multiple Kubernetes resources into a single logical deployment unit called Chart.

Before we can get started configuring helm we’ll need to first install the command line tools that you will interact with.

```sh
cd $HOME
wget https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get -O get_helm.sh
chmod +x get_helm.sh
./get_helm.sh
```

> Note: Once you install helm, the command will prompt you to run ‘helm init’. **Do not run ‘helm init’.** Follow the instructions to configure helm using Kubernetes RBAC and then install tiller as specified below If you accidentally run ‘helm init’, you can safely uninstall tiller by running ‘helm reset –force’

1. Create Helm service account `manifests/helm-rbac.yaml`

    ```yaml

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: tiller
      namespace: kube-system

    ```
2. Bind Helm service account `manifest/helm-rbac-bind.yaml`


    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: tiller
      namespace: kube-system
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: tiller
      namespace: kube-system

    ```

1. Apply the config

    ```sh
    kubectl apply -f helm-rbac.yaml
    kubectl apply -f helm-rbac-bind.yaml
    ```

1. Install Helm

    ```sh
    helm init --service-account tiller
    ```

## Deploy simple workload with Helm

Helm uses a packaging format called Charts. A Chart is a collection of files that describe Kubernetes resources.

Instead of installing k8s resources manually via `kubectl`, we can use Helm to install pre-defined Charts faster, with less chance of typos or other operator errors.

1. Update Helm local list of Charts

    ```sh
    helm repo update
    ```

1. List all avaiable charts

    ```sh
    helm search
    ```

    That should output something similiar

    ```sh
    NAME                                    CHART VERSION   APP VERSION                     DESCRIPTION
    stable/acs-engine-autoscaler            2.2.2           2.1.1                           DEPRECATED Scales worker nodes within agent pools
    stable/aerospike                        0.2.1           v3.14.1.2                       A Helm chart for Aerospike in Kubernetes
    ```

1. Add the Bitnami Chart repo to our local list of searchable charts.

    ```sh
    helm repo add bitnami https://charts.bitnami.com/bitnami
    ```

1. Search for Nginx chart

    ```sh
    helm search nginx
    ```

    ```sh
    NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
    bitnami/nginx                           2.1.3           1.14.2          Chart for the nginx server
    bitnami/nginx-ingress-controller        3.2.0           0.22.0          Chart for the nginx Ingress controller
    ```

1. Use the Helm utility to install Nginx chart

    ```sh
    helm install --name workshop-webserver bitnami/nginx
    ```

    Once you run this command, the output confirms the types of Kubernetes objects that were created as a result

    ```sh
    RESOURCES:
    ==> v1/Service
    NAME                      TYPE          CLUSTER-IP      EXTERNAL-IP  PORT(S)       AGE
    workshop-webserver-nginx  LoadBalancer  10.100.154.253  <pending>    80:32212/TCP  0s

    ==> v1beta1/Deployment
    NAME                      DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
    workshop-webserver-nginx  1        1        1           0          0s

    ==> v1/Pod(related)
    NAME                                       READY  STATUS             RESTARTS  AGE
    workshop-webserver-nginx-6569f48788-swrhs  0/1    ContainerCreating  0         0s
    ```

1. Get the complete URL of this Service

    ```sh
    kubectl get svc workshop-webserver-nginx -o wide
    ```

    ```sh
    NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)        AGE       SELECTOR
    workshop-webserver-nginx   LoadBalancer   10.100.154.253   a1b6ddc8d273111e990940a1304ea8e7-1802906149.ap-northeast-1.elb.amazonaws.com   80:32212/TCP   36s       app=workshop-webserver-nginx
    ```

1. Open in your browser 'http://EXTERNAL-IP' and check that Nginx is working.

---

Next: [Downward API](../../misc/downward_api.md)
