# Namespaces RBAC and IAM

## Module Objectives

1. Create a Namespace
1. Add a user to the cluster
1. Create a Role, assign it to the user and make sure it is enforced
1. Create a ClusterRole, assign it to the user and make sure it is enforced

---

## Create a Namespace

Namespaces provide for a scope of Kubernetes objects. You can think of it as a workspace you are sharing with other users.

With namespaces one may have several virtual clusters backed by the same physical cluster. Names are unique within a Namespace, but not across Namespaces.

The cluster administrator can divide physical resources between Namespaces using quotas.

Namespaces cannot be nested.

Low-level infrastructure resources like Nodes and PersistentVolumes are not associated with a particular Namespace.

1. List all Namespaces in the system

    ```sh
    kubectl get ns
    ```

1. Use `describe` to learn more about a particular Namespace.

    ```sh
    kubectl describe ns default
    ```

1. Save the following file as `manifests/workshop-ns.yaml`.

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: workshop
    ```

1. Apply the manifest to create a new Namespace called `workshop`.

    ```sh
    kubectl apply -f manifests/workshop-ns.yaml
    ```

1. List Namespaces again.

    > Note: You should see the Namespace `workshop` in the list.

## Add a User to the Cluster

Kubernetes has two types of users. The first is human users. They are managed outside the cluster in AWS IAM. The second type is service accounts. Service accounts are used by processes to access the Kubernetes API.

In this exercise you will create a service account and configure `kubectl` to use it.

>Note: Typically you login as a human user, not as a service account.

Adding human users is out of scope for this workshop as we focus on Kubernetes not AWS, however we have included an example of how this is configured. You would normally create additional human users in AWS IAM.

  ```sh
  TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

  echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

  aws iam create-role --role-name EksWorkshopCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

  aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy

  ROLE="    - rolearn: arn:aws:iam::$ACCOUNT_ID:role/EksWorkshopCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

  kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

  kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
  ```

1. Create a service account in the new workspace.

    ```sh
    kubectl create serviceaccount workshop-user --namespace workshop
    ```

1. Set up authentication for `kubectl` with the service account. You will use the context `arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop` for normal operations and `limited` for operations as `workshop-user`.

    ```sh
    kubectl config set-credentials workshop-user --token=$(kubectl get secret $(kubectl get secret -n workshop | grep workshop-user-token | cut -f 1 -d " ") -n=workshop -o jsonpath={.data.token} | base64 --decode)
    ```

1. Edit `~/.kube/config`.

   1. Search for the contexts section and duplicate one the contexts objects, name the new context `limited`.

   1. Rename the original context `arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop`.

   1. Add `namespace: workshop` on the `limited` context.

   1. Change the user to `workshop-user` on the `limited` context.

   1. Set `current-context: arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop`.

   It should look similar to this:

    ```yaml
    contexts:
    - context:
        cluster: arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop
        user: arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop
      name: arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop
    - context:
        cluster: arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop
        namespace: workshop
        user: workshop-user
      name: limited
    current-context: arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop
    ```

    > Note: With each context you can set a default Namespace to use for all kubectl commands. This can be overwritten by specifying an explicit Namespace in the metadata of a manifest file.

1. Check if you can get Pods with the `arn:aws:eks:ap-northeast-1:<ACCOUNT-ID>:cluster/k8s-workshop` context.

    ```sh
    kubectl auth can-i get pods
    ```

    > Note: Output should be yes

1. Switch between contexts.

    ```sh
    kubectl config use-context limited
    ```

1. Check if you can get Pods with the `limited` context.

    ```sh
    kubectl auth can-i get pods
    ```

    > Note: Output should be no. The user can do nothing as you didn't associate it with any role

1. Switch back to the normal context.

    ```sh
    kubectl config use-context arn:aws:eks:ap-northeast-1:$AWS_ACCOUNT_ID:cluster/k8s-workshop
    ```

## Create a Role, Assign it to the User and Make Sure it is Enforced

A Role is a set of rules that are applied to the Namespace. ClusterRole is applied to the whole cluster.

In both cases you describe the objects you want to grant access to and operations user may execute against these objects.

For the Role to take effect you must bind it to the user.

1. Create `manifests/worker-role.yaml` file which grants permissions to create Pods and Deployments.

    ```yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      namespace: workshop
      name: worker
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "watch", "list", "create"]
    - apiGroups: ["extensions", "apps"]
      resources: ["deployments"]
      verbs: ["get", "watch", "list", "create"]
    ```

    > Note: The `nginx` deployment created later might be of either `extensions` or `apps` API groups depending on your client version. For simplicity, we will use both in this workshop.

1. Apply the manifest.

    ```sh
    kubectl apply -f manifests/worker-role.yaml
    ```

1. Create a RoleBinding `manifests/worker-rolebinding.yaml` between user and the Role.

    ```yaml
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: worker
      namespace: workshop
    subjects:
    - kind: User
      name: system:serviceaccount:workshop:workshop-user
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: worker
      apiGroup: rbac.authorization.k8s.io
    ```

    > Note: This RoleBinding is for the `workshop` Namespace only.

1. Apply the manifest.

    ```sh
    kubectl apply -f manifests/worker-rolebinding.yaml
    ```

1. Switch context to `limited`.

    ```sh
    kubectl config use-context limited
    ```

1. Create an `nginx` Deployment.

    ```sh
    kubectl run nginx --image=nginx -n workshop
    ```

1. Check that the Deployment and Pod was created successfully.

    ```sh
    kubectl get pods -n workshop
    ```

    ```
    NAME                     READY   STATUS    RESTARTS   AGE
    nginx-64f497f8fd-kqgzr   1/1     Running   0          18s
    ```

1. Check that the user still can't read nodes in the cluster.

    ```sh
    kubectl get nodes
    ```

## Create a ClusterRole, Assign it to the User and Make Sure it is Enforced

1. Switch to the normal context.

    ```sh
    kubectl config use-context arn:aws:eks:ap-northeast-1:$AWS_ACCOUNT_ID:cluster/k8s-workshop
    ```

1. Create a `ClusterRole` to list nodes `manifests/cr-node-reader.yaml`.

    ```yaml
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: node-reader
    rules:
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["get", "watch", "list"]
    ```

    > Note: Nodes are cluster-wide resources and are not associated with a particular Namespace.

1. Apply the manifest.

    ```sh
    kubectl apply -f manifests/cr-node-reader.yaml
    ```

1. Bind the `node-reader` ClusterRole to the service account `manifests/crb-node-reader.yaml`.

    ```yaml
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: read-nodes
    subjects:
    - kind: User
      name: system:serviceaccount:workshop:workshop-user
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: node-reader
      apiGroup: rbac.authorization.k8s.io
    ```

1. Apply the manifest.

    ```sh
    kubectl apply -f manifests/crb-node-reader.yaml
    ```

1. Now verify that user may list nodes.

    ```sh
    kubectl config use-context limited
    ```

    ```sh
    kubectl get nodes
    ```

	```
	NAME                             STATUS    ROLES     AGE       VERSION
	ip-192-168-134-44.ec2.internal   Ready     <none>    1d        v1.11.5
	ip-192-168-240-81.ec2.internal   Ready     <none>    1d        v1.11.5
	ip-192-168-78-36.ec2.internal    Ready     <none>    1d        v1.11.5
	```

## Optional Exercise

1. Switch back to the unrestricted context and delete the `nginx` Deployment.

    ```sh
    kubectl delete deployment nginx -n workshop
    ```

1. Edit the `manifests/worker-role.yaml` and delete the `pods` permissions. Reapply the worker role then try to deploy `nginx` again. What happens? Can you explain the outcomes?

> Note: Ensure you switch back to the unreestricted context when you have finished the exercises

---

Next: [Services](09-services.md)
