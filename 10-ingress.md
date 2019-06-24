# Ingress

## Module Objectives

1. Deploy AWS EKS ALB Ingress controller
1. Serve app traffic from the Ingress instead of LoadBalancer service
1. Specify app domain using external DNS
1. Add TLS/SSL support

---

## Theory

Ingress is an API object that manages external access to the services in a cluster, typically HTTP.

Ingress can provide load balancing, SSL termination and name-based virtual hosting.

---

## Deploy AWS EKS ALB Ingress controller

You can install EKS ALB ingress controller via Kubectl.

1. Download sample ALB ingress controller manifest

  ```sh
  wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.0/docs/examples/alb-ingress-controller.yaml
  ```

1. Configure the ALB ingress controller manifest, edit the following variables:

  * `--cluster-name=k8s-workshop`: name of the cluster.

1. Deploy the RBAC roles manifest.

  ```sh
  kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.0/docs/examples/rbac-role.yaml
  ```

1. Deploy the ALB ingress controller manifest.

  ```sh
  kubectl apply -f alb-ingress-controller.yaml
  ```

1. Verify the deployment was successful and the controller started.

  ```sh
  kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)
  ```

  Should display output similar to the following.

  ```sh
  -------------------------------------------------------------------------------
  AWS ALB Ingress controller
  Release:    v1.1.0
  Build:      git-72962fcb
  Repository: https://github.com/kubernetes-sigs/aws-alb-ingress-controller.git
  -------------------------------------------------------------------------------
  ```

## Use Ingress

1. Change the `frontend` Service type from `LoadBalancer` to `NodePort`.

    ```shell
    kubectl edit svc/frontend
    ```

    * Find the line `type: LoadBalancer` and change it to `type: NodePort`.

    * Save the file `Esc :wq`.

    > Note: Ingress forwards traffic to a Service (not directly to Pods). That's why we still need a Service wrapping our frontend Pod. However, it is not necessary for this Service to be of a LoadBalancer type, because now we will be accessing the frontend through Ingress and not through a dedicated frontend load balancer. The Service has to be of a NodePort type instead.

1. Check the Service type.

    ```shell
    kubectl get svc
    ```

    ```
    NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
    backend          ClusterIP   10.100.58.231   <none>        8080/TCP       4h
    frontend         NodePort    10.100.33.79    <none>        80:30996/TCP   4h
    galera-cluster   ClusterIP   None            <none>        3306/TCP       31m
    kubernetes       ClusterIP   10.100.0.1      <none>        443/TCP        6h
    ```

1. Create the file `manifests/ingress.yaml`.

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: awsme-ingress
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
    spec:
      rules:
      - http:
          paths:
          - path: /*
            backend:
              serviceName: frontend
              servicePort: 80
    ```

    This will expose the Service `frontend` using the root path `/`.

1. Create the Ingress.

    ```shell
    kubectl get ingress --watch -o wide  # in the first terminal
    ```

    ```shell
    kubectl apply -f manifests/ingress.yaml  # in the second terminal
    ```

    ```sh
    NAME           HOSTS   ADDRESS                                                                       PORTS     AGE
    awsme-ingress  *       203122f2-default-awsmeingr-b6ac-1626324477.ap-northeast-1.elb.amazonaws.com   80        1m
    ```

    > Note: It may take up to 10 minutes for EKS to allocate an external IP address and set up forwarding rules until the load balancer is ready to serve your application. In the meanwhile, you may get errors such as HTTP 404 or HTTP 500 until the load balancer configuration is propagated across the globe.

1. In the Amazon Management Console go to 'EC2' -> 'Load Balancers' and examine the created load balancer.

1. The application will be available as `http://<ingress-address>/`.

## Setting up external DNS

1. Create a DNS zone which will contain the managed DNS records.

  ```sh
  aws route53 create-hosted-zone --name "external-dns-test.my-org.com." --caller-reference "external-dns-test-$(date +%s)"
  ```

1. Make a note of the ID of the hosted zone you just created.

  ```sh
  aws route53 list-hosted-zones-by-name --output json --dns-name "external-dns-test.my-org.com." | jq -r '.HostedZones[0].Id'
  ```

1. Make a note of the nameservers that were assigned to your new zone.

  ```sh
  aws route53 list-resource-record-sets --output json --hosted-zone-id "/hostedzone/Z33CUP2K3I2SMW" \
      --query "ResourceRecordSets[?Type == 'NS']" | jq -r '.[0].ResourceRecords[].Value'
  ns-1768.awsdns-29.co.uk.
  ...
  ```

1. Apply the following manifests file to deploy ExternalDNS `manifests/external-dns.yaml`.

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: external-dns
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRole
    metadata:
      name: external-dns
    rules:
    - apiGroups: [""]
      resources: ["services"]
      verbs: ["get","watch","list"]
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get","watch","list"]
    - apiGroups: ["extensions"]
      resources: ["ingresses"]
      verbs: ["get","watch","list"]
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["list"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: external-dns-viewer
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: external-dns
    subjects:
    - kind: ServiceAccount
      name: external-dns
      namespace: default
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: external-dns
    spec:
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: external-dns
        spec:
          serviceAccountName: external-dns
          containers:
          - name: external-dns
            image: registry.opensource.zalan.do/teapot/external-dns:latest
            args:
            - --source=service
            - --source=ingress
            - --domain-filter=external-dns-test.my-org.com # will make ExternalDNS see only the     hosted zones matching provided domain, omit to process all available hosted zones
            - --provider=aws
            - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to   enable full synchronization
            - --aws-zone-type=public # only look at public hosted zones (valid values are public,   private or no value for both)
            - --registry=txt
            - --txt-owner-id=my-identifier
    ```

1. Update ingress resource manifest file `manifests/ingress.yaml`.

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: awsme-ingress
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
    spec:
      rules:
      - host: frontend.external-dns-test.my-org.com
        http:
          paths:
          - path: /*
            backend:
              serviceName: frontend
              servicePort: 80
    ```

1. After roughly two minutes check that a corresponding DNS record for your service was created.

    ```sh
    aws route53 list-resource-record-sets --output json --hosted-zone-id "/hostedzone/Z33CUP2K3I2SMW" \
        --query "ResourceRecordSets[?Name == 'frontend.external-dns-test.my-org.com.']"

    [
        {
            "AliasTarget": {
                "HostedZoneId": "Z35SXDOTRQ7X7K",
                "EvaluateTargetHealth": true,
                "DNSName": "203122f2-default-awsmeingr-b6ac-149951395.ap-northeast-1.elb.amazonaws.com."
            },
            "Type": "A",
            "Name": "frontend.external-dns-test.my-org.com."
        },
        {
            "ResourceRecords": [
                {
                    "Value": "\"heritage=external-dns,external-dns/owner=my-identifier, external-dns/resource=ingress/default/awsme-ingress\""
                }
            ],
            "Type": "TXT",
            "Name": "frontend.external-dns-test.my-org.com.",
            "TTL": 300
        }
    ]
    ```

    Note created TXT record alongside ALIAS record. TXT record signifies that the corresponding ALIAS record is managed by ExternalDNS. This makes ExternalDNS safe for running in environments where there are other records managed via other means.

    Let's check that we can resolve this DNS name. We'll ask the nameservers assigned to your zone first.

    1. Get ingress address.

        ```sh
        kubectl get ingress -o wide
        ```

    1. Get inside the backend pod.

        ```sh
        kubectl get pods
        kubectl exec -i -t backend-c97756b74-c7m7v bash
        ```

    1. Install `dig` utility inside the container.

        ```sh
        apt-get update && apt-get install dnsutils
        ```

    1. Check that we can resolve this DNS name. We'll ask the nameservers assigned to your zone.

        ```sh
        dig +short @ns-1768.awsdns-29.co.uk. frontend.external-dns-test.my-org.com.
        52.4.69.70
        54.87.94.60
        52.3.188.212
        ```

    1. Modify `/etc/hosts` inside the backend container and set `frontend.external-dns-test.my-org.com` domain to be resolved to the ingress addresses.

    1. Access `frontend.external-dns-test.my-org.com` from backend container.

        ```sh
        curl -s frontend.external-dns-test.my-org.com
        ```

## Optional Exercises

### Use TLS

1. Create a self-signed certificate for `awsme` app for the `frontend.external-dns-test.my-org.com` domain [link](https://stackoverflow.com/questions/10175812/how-to-create-a-self-signed-certificate-with-openssl).
1. Import a certificate to AWS Certificate Manager.
1. Redeploy, open app in a web browser and examine certificate details. Use [this](https://www.ssl2buy.com/wiki/how-to-view-ssl-certificate-details-on-chrome-56) link to see how a certificate can be viewed in chrome.

<details><summary>SOLUTION - CLICK ME</summary>
<p>

1. Create a self-signed certificate.

    ```shell
    openssl req -nodes -x509 -newkey rsa:2048 -keyout awsme_key.pem -out awsme_cert.pem -days 365 -subj "/C=US/ST=California/L=Sunnyvale/O=Altoros/OU=Training/CN=frontend.external-dns-test.my-org.com"
    ```

1. Import a certificate to AWS Certificate Manager. Note the `Arn` in the output.

    ```shell
    aws iam upload-server-certificate --server-certificate-name k8s-workshop-certificate \
      --certificate-body file://awsme_cert.pem --private-key file://awsme_key.pem
    ```

1. Add the following annotations to setup an ingress to redirect http traffic into https.

    ```yaml
    metadata:
        annotations:
            alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:xxxx:certificate/xxxxxx
            alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
            alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
    ```

1. The final `manifests/ingress.yaml` should look like the following:

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
        alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:xxxx:certificate/xxxxxx
        alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
      name: awsme-ingress
      labels:
        app: awsme
    spec:
      rules:
      - http:
          paths:
          - backend:
              serviceName: ssl-redirect
              servicePort: use-annotation
            path: /*
          - backend:
              serviceName: frontend
              servicePort: 80
            path: /*
    ```

    > Note: the `ssl-redirect` action must be be first rule(which will be evaluated first by ALB)

* You can troubleshoot certificate issues by viewing the Ingress events.

    ```shell
    kubectl describe ing awsme-ingress
    ```

</p>
</details>

---

Next: [Helm](11-helm.md)
