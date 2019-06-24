# Security

## Module Objectives

1. Install Calico on EKS
1. Limit egress with a Network Policy
1. Limit ingress with a Network Policy

## Install Calico on EKS

1. Deploy the Calico DaemonSet to the EKS cluster

    ```sh
    kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/v1.5/calico.yaml
    ```

1. Verify the DaemonSet is running correctly on all worker nodes

    ```sh
    kubectl get daemonset calico-node --namespace=kube-system
    ```

## Network Policy Egress

Let's see how to use network policy for blocking the external traffic for a `Pod`

1. Create file called `manifests/deny-egress.yaml`:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: foo-deny-egress
    spec:
      podSelector:
        matchLabels:
          app: foo
      policyTypes:
      - Egress
      egress:
      - ports: # allow DNS resolution
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    ```

1. Apply the network policy

    ```sh
    kubectl apply -f manifests/deny-egress.yaml
    ```

    This file blocks all the outgoing traffic except DNS resolution.

1. Start the Pod that matches label `app=foo`

    ```sh
    kubectl run --rm --restart=Never --image=alpine -i -t -l app=foo test -- ash
    ```

1. In container try to access a website

    ```sh
    wget --timeout 1 -O- http://www.example.com
    ```

    ```sh
    Connecting to www.example.com (93.184.216.34:80)
    wget: download timed out
    ```

    You see the name resolution works fine but external connections are dropped.

1. Now remove the `-l app=foo` and try again, see what happens.

## Network policy Ingress

Now we will create a service and set policy that will restrict access to it.

1. Create nginx and expose the service

    ```sh
    kubectl run nginx --image=nginx --replicas=2
    kubectl expose deployment nginx --port=80

    kubectl get svc,pod
    ```

1. Execute a pod and see if you can connect

    ```sh
    kubectl run busybox --rm -ti --image=busybox /bin/sh
    ```

    ```sh
    wget --spider --timeout=1 nginx

    Connecting to nginx (10.104.90.248:80)
    remote file exists
    ```

1. Create file called `manifests/nginx-policy.yaml` to restrict access

    ```yaml
    kind: NetworkPolicy
    apiVersion: networking.k8s.io/v1
    metadata:
      name: access-nginx
    spec:
      podSelector:
        matchLabels:
          run: nginx
      ingress:
      - from:
        - podSelector:
            matchLabels:
              access: "true"
    ```

    ```sh
    kubectl apply -f manifests/nginx-policy.yaml
    ```

1. Try to connect via the Pod

    ```sh
    kubectl run busybox --rm -ti --image=busybox /bin/sh
    ```

    ```sh
    wget --spider --timeout=1 nginx
    Connecting to nginx (10.100.0.16:80)
    wget: download timed out
    ```

1. Try to connect via the Pod again but add the access label

    ```sh
    kubectl run busybox --rm -ti --labels="access=true" --image=busybox /bin/sh
    ```

    ```sh
    wget --spider --timeout=1 nginx
    Connecting to nginx (10.109.50.139:80)
    remote file exists
    ```

> Note: Delete the Network Policies so that they don't interfere with other exercises

```sh
kubectl delete -f manifests/deny-egress.yaml
kubectl delete -f manifests/nginx-policy.yaml
```

---

Next: [Namespaces RBAC and IAM](08-namespaces-rbac.md)
