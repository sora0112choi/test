# Persistence

## Module Objectives

1. Use Persistent Volumes to store data
1. Create a Storage Class
1. Convert a Persistent Volume to Persistent Volume Claim
1. Use StatefulSet to deploy a MySQL galera cluster

---

## Use a Persistent Volume to Store Data

1. Create a data disk.

    ```sh
    aws ec2 create-volume \
      --availability-zone=ap-northeast-1a \
      --size=10 \
      --volume-type=gp2
    ```

    > Note: The disk must be in the same zone and account as EKS cluster.

    Look for the following key in the output:

    ```
    "VolumeId": "vol-068c0000fe3b7cc41"
    ```

1. Edit the database manifest to add the Volume to the Pod `spec`.

    ```yaml
    volumes:
    - name: my-data
      awsElasticBlockStore:
        fsType: ext4
        volumeID: vol-068c0000fe3b7cc41

    ```

    > Note: Add it to the Pod spec not the Deployment spec!

1. Now mount the Volume inside the MySQL container.

    ```yaml
    containers:
      - image: mysql:5.6
        name: mysql
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: my-data
    ```

    > Note: The disk and pod should be in the same availability zone.

1. To be sure that db pod will be created on worker in availability zone where aws ebs located we can make next
    ```shell
    aws ec2 describe-instances --filters \
    "Name=availability-zone,Values=ap-northeast-1a" \
    "Name=tag:aws:cloudformation:logical-id, Values=NodeGroup" \
    "Name=tag:aws:cloudformation:stack-id,Values=*$AWS_ACCOUNT_ID*" \
    | grep PrivateDnsName | head -1


    ```
1. Then create label for this worker node e.x.
    ```shell
    kubectl label nodes ip-192-168-87-212.ec2.internal azs=ap-northeast-1a

    ```
1. In `db.yaml` add to spec section following:
    ```yaml
    nodeSelector:
      azs: ap-northeast-1a
    ```

1. Re-create the Deployment.

    ```shell
    kubectl apply -f manifests/db.yaml
    ```

1. Test the data persists between Pod restarts.

    * Add some notes to the frontend app and then delete the mysql Pod.

    * The Deployment Controller will automatically re-create the Pod and mount the Persistent Volume.

## Create a Storage Class

AWS offers several disk types. They differ based on performance, reliability and price. How can I use particular disk type for Kubernetes workloads?

Kubernetes defines resource type called StorageClass. By specifying StorageClass in you PVCs you can provision different disk types.

1. Create a Storage Class that uses gp2 storage type `manifests/storage-class.yaml`.

	```yaml
	kind: StorageClass
	apiVersion: storage.k8s.io/v1
	metadata:
	  name: gp2
	  annotations:
	    storageclass.kubernetes.io/is-default-class: "true"
	provisioner: kubernetes.io/aws-ebs
   parameters:
      type: gp2
      fsType: ext4
	```

    `type: gp2` tells to create an SSD disk instead of standard one.

1. Apply configuration.

    ```shell
    kubectl apply -f manifests/storage-class.yaml
    ```

1. Verify the StorageClass was created.

    ```shell
    kubectl get storageClass
    ```

    ```
    NAME            PROVISIONER            AGE
    gp2 (default)   kubernetes.io/aws-ebs   7s
    ```

## Convert Persistent Volume to Persistent Volume Claim

A Persistent Volume Claim automates disk provisioning.

1. Create a Persistent Volume Claim for 50Gi disk `manifests/pvc.yaml`.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: dynamic-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 50Gi
    ```

1. Apply configuration.

    ```shell
    kubectl apply -f manifests/pvc.yaml
    ```

1. Verify PVC is created.

    ```shell
    kubectl describe pvc/dynamic-data
    ```

    ```
    Events:
    Type    Reason                 Age   From                         Message
    ----    ------                 ----  ----                         -------
    Normal  ProvisioningSucceeded  20s   persistentvolume-controller  Successfully provisioned volume pvc-a7e7a72e-256a-11e9-b9c8-0ab9e71b7fc6 using kubernetes.io/aws-ebs
    ```

1. Let's look at the Volume.

    ```shell
    kubectl get pv
    ```

    ```
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
    pvc-a7e7a72e-256a-11e9-b9c8-0ab9e71b7fc6   50Gi       RWO            Delete           Bound     default/dynamic-data   gp2                      50s
    ```

1. You may find the disk using the AWS CLI output as well.

    ```shell
    aws ec2 describe-volumes --output text
    ```

    ```
    VOLUMES ap-northeast-1b      2019-01-31T15:12:52.864Z        False   150     50              available       vol-028f2b9a574762238   gp2
    ```

    Kubernetes provisioned 50Gi and made it available to use in Pods.

1. Reconfigure MySQL to use PVC instead of PV. Don't forget to remove NodeSelector.

    The data will be lost - it is another disk!

    Edit the `manifests/db.yaml` file.

    ```yaml
    spec:
      volumes:
      - name: my-data
        persistentVolumeClaim:
          claimName: dynamic-data
    ```

1. You need to re-create the Pod or update the Deployment.

    ```shell
    kubectl delete -f manifests/db.yaml
    kubectl apply -f manifests/db.yaml
    ```

1. Find the line in the event stream from the Pod telling that Volume was mounted.

    ```shell
    kubectl describe pod db
    ```

    ```
    Normal  SuccessfulAttachVolume  2s    attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-a7e7a72e-256a-11e9-b9c8-0ab9e71b7fc6"
    ```

1. Go to frontend and verify the previous notes are gone. That mean you use a new disk to store data.

## Converting mysql Deployment to a Stateful Set

When you are using Deployments to manage your Pods it's very easy to scale stateless applications, like our backend and frontend, but scaling stateful applications is more difficult. If you want to scale a mysql database you have to start the first node in a bootstrap mode, then you have to wait until this node is ready and finally you have to start all other nodes and add them to the cluster. If you try to use a Deployment to scale your database, it will create 3 instances of the database Pod simultaneously and you will not be able to create a cluster. StatefulSet is the right object to do such kind of Deployment. Now let's use a StatefulSet to convert our mysql Pod to a 3-node galera mysql cluster.

1. In the `sample-app-aws` folder create a subfolder `mysql-galera`.

1. Inside `mysql-galera` folder create a Dockerfile with the following content.

    ```
    FROM ubuntu:16.04
    ENV DEBIAN_FRONTEND noninteractive

    RUN apt-get update
    RUN apt-get install -y software-properties-common
    RUN apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 BC19DDBA
    RUN add-apt-repository 'deb http://releases.galeracluster.com/mysql-wsrep-5.7/ubuntu xenial main'
    RUN add-apt-repository 'deb http://releases.galeracluster.com/galera-3/ubuntu xenial main'
    RUN apt-get update
    RUN apt-get install -y galera-3 galera-arbitrator-3 mysql-wsrep-5.7 rsync lsof host
    COPY my.cnf /etc/mysql/my.cnf
    COPY start.sh /start.sh
    ENTRYPOINT ["/start.sh"]
    ```

    This Dockerfile simply installs Galera using the Codership repository and copies `my.cnf` and `start.sh` over.

1. Add `my.cnf` file to the `mysql-galera` folder.

    ```
    [mysqld]
    user = mysql
    bind-address = 0.0.0.0
    wsrep_provider = /usr/lib/galera/libgalera_smm.so
    wsrep_sst_method = rsync
    default_storage_engine = innodb
    binlog_format = row
    innodb_autoinc_lock_mode = 2
    innodb_flush_log_at_trx_commit = 0
    query_cache_size = 0
    query_cache_type = 0
    ```

    This file configures galera replication.

1. Add `start.sh` file to the `mysql-galera` folder.

    ```shell
    #!/bin/bash -e

    mkdir /var/run/mysqld
    chown mysql:mysql /var/run/mysqld

    node_list=$(host $SERVICE_NAME | grep "has address" |  awk '{print $4}' | paste -s -d, -)

    if [ -z "$node_list" ]; then
        # if we are in a bootstrap mode we want to set root password first
        mysqld --wsrep-cluster-address=gcomm://$node_list &
        pid="$!"
        # waiting for mysql to become ready
        while !(mysqladmin ping)
        do
           sleep 3
           echo "waiting for mysql ..."
        done
        # setting root password to $MYSQL_ROOT_PASSWORD
        mysql -u root  -e "use mysql; update user set authentication_string=password(\"$MYSQL_ROOT_PASSWORD\") where user='root';"

        # Note: By default, galera-cluster resricts access to localhost only, we are updating this to allow from all IPs
        mysql -u root -e "GRANT ALL ON *.* to root@'%' IDENTIFIED BY 'root';"

        # stopping mysql because we have to restart after setting root password
        if ! kill -s TERM "$pid" || ! wait "$pid"; then
            echo >&2 'MySQL init process failed.'
            exit 1
        fi
    fi
    mysqld --wsrep-cluster-address=gcomm://$node_list
    ```

    `$SERVICE_NAME` is the DNS name of the service that wraps all 3 mysql nodes. This is a headless service, so instead of a virtual IP address it resolves to the list of all IP addresses of the underlying Pods. When the first node is started `host $SERVICE_NAME` command returns and empty list. In this case `node_list` variable is empty. After the first node is started the command returns its IP address, after the second one is started it returns 2 IP addresses separated by comma. This is exactly what we need to start the galera cluster: the first node should be started with `--wsrep-cluster-address=gcomm://` parameter, the second with `--wsrep-cluster-address=gcomm://<first-node-ip>`, the third with `--wsrep-cluster-address=gcomm://<first-node-ip>,<second-node-ip>` and so on.

1. Make `start.sh` executable.

    ```shell
    chmod +x start.sh
    ```

1. Build the image and push it to the container registry.

    ```shell
    aws ecr create-repository --repository-name mysql-galera
    docker build . -t $AWS_ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/mysql-galera
    docker push $AWS_ACCOUNT_ID.dkr.ecr.ap-northeast-1.amazonaws.com/mysql-galera
    ```

1. Delete the `db` Deployment and `db` Service.

    ```shell
    kubectl delete deployment db
    kubectl delete svc db
    ```

1. Save the following file as `manifests/mysql-galera-svc.yaml` and apply the changes.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: galera-cluster
    spec:
      ports:
      - port: 3306
        name: mysql
      clusterIP: None
      selector:
        app: awsme
        role: db
      publishNotReadyAddresses: false
    ```

    The most important parameter here is `clusterIP: None`. This is called a [Headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services). In this case Kubernetes will not create a cluster IP for the service and DNS name `galera-cluster` will point directly to the list of underlying Pods.

1. Save the following file as `manifests/mysql-galera.yaml` and apply the changes.

    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: db
    spec:
      serviceName: "galera"
      replicas: 3
      selector:
        matchLabels:
          app: awsme
          role: db
      template:
        metadata:
          labels:
            app: awsme
            role: db
        spec:
          containers:
          - name: mysql
            image: <REPLACE_WITH_YOUR_OWN_MYSQL_IMAGE>
            ports:
            - containerPort: 3306
              name: mysql
            - containerPort: 4444
              name: sst
            - containerPort: 4567
              name: replication
            - containerPort: 4568
              name: ist
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql
                  key: password
            - name: SERVICE_NAME
              value: galera-cluster
            readinessProbe:
              exec:
                command:
                - sh
                - -c
                - "mysql -u root -p$MYSQL_ROOT_PASSWORD -e 'show databases;'"
              initialDelaySeconds: 5
              timeoutSeconds: 2
              successThreshold: 2
    ```

    > Note: Replace the image name with the image just created.

1. Edit the backend deployment `manifests/backend.yaml`.

    1. If you still have the init and multi container remove them, leaving only the backend container, remember to re-add `-run-migrations` to the original backend container.

    1. In the startup command, change `-db-host=db` to `-db-host=galera-cluster`.

    1. Apply the changes.

1. Connect to the app and add some notes.

1. Exec inside one of the db Pods.

    ```shell
    kubectl exec -it db-0 bash
    ```

1. Connect to the mysql database.

    ```shell
    mysql -u root -p$MYSQL_ROOT_PASSWORD
    ```

1. Show databases.

    ```shell
    mysql> show databases;
    ```

    ```
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | mysql              |
    | performance_schema |
    | sample_app         |
    | sys                |
    +--------------------+
    ```

1. Show your notes.

    ```sql
    select * from sample_app.notes;
    ```

    ```
    +----+---------------------+---------------------+------------+---------+
	| id | created_at          | updated_at          | deleted_at | note    |
	+----+---------------------+---------------------+------------+---------+
	|  1 | 2019-01-31 16:00:14 | 2019-01-31 16:00:14 | NULL       | note1   |
	|  4 | 2019-01-31 16:00:18 | 2019-01-31 16:00:18 | NULL       | note2   |
	|  7 | 2019-01-31 16:02:29 | 2019-01-31 16:02:29 | NULL       | note3   |
	+----+---------------------+---------------------+------------+---------+
	3 rows in set (0.00 sec)
    ```

## Optional Exercises

### Use Persistent Volumes in the Stateful Set

Modify `mysql-galera.yaml` to use Persistent Volume Claims. Make sure the data survives after you delete and recreate the database.

  <details><summary>SOLUTION - CLICK ME</summary>
  <p>

  We need to modify mysql-galera.yaml and add following mount to mysql container

  ```yaml
          volumeMounts:
          - name: pv-data
            mountPath: /var/lib/mysql
  ```
  and following to the end of the file.

  ```yaml
    volumeClaimTemplates:
    - metadata:
        name: pv-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  ```
  These will dynamically create volumes and mount them to node/pod

  For some reason when PVC is mounted, related folder (/var/lib/mysql) is cleared, so we need to adjust out start.sh script to fix this. You could make this in different way. It could looks like this:

  ```shell
  #!/bin/bash -e

  mkdir /var/run/mysqld
  chown mysql:mysql /var/run/mysqld
  node_list=$(host $SERVICE_NAME | grep "has address" |  awk '{print $4}' | paste -s -d, -)
  if [ -d "/var/lib/mysql/mysql" ]
  then
          echo "/var/lib/mysql/mysql directory  exists, skipping initialization."
          sed -i 's/safe_to_bootstrap: 0/safe_to_bootstrap: 1/g' /var/lib/mysql/grastate.dat
  else
          rm -rf /var/lib/mysql/*
          mysqld --initialize-insecure
          chown -R mysql:mysql /var/lib/mysql/
          if [ -z "$node_list" ]; then
              # if we are in a bootstrap mode we want to set root password first
              mysqld --wsrep-cluster-address=gcomm://$node_list &
              pid="$!"
              # waiting for mysql to become ready
              while !(mysqladmin ping)
              do
                 sleep 3
                 echo "waiting for mysql ..."
              done

              mysql -u root  -e "use mysql; update user set authentication_string=password(\"$MYSQL_ROOT_PASSWORD\") where user='root';"

              # Note: By default, galera-cluster resricts access to localhost only, we are updating this to allow from all IPs
              mysql -u root -e "GRANT ALL ON *.* to root@'%' IDENTIFIED BY 'root';"

              # stopping mysql because we have to restart after setting root password
              if ! kill -s TERM "$pid" || ! wait "$pid"; then
                  echo >&2 'MySQL init process failed.'
                  exit 1
              fi
          fi


  fi
  chown -R mysql:mysql /var/lib/mysql/

  mysqld --wsrep-cluster-address=gcomm://$node_list
  ```
  After you made changes in start.sh, don't forget to build and push new container to ECR

  </p>
  </details>

---

Next: [Security](07-security.md)
