# Postgres App

A sample K8s Postgres app that is deployed as a stateful set using CSI(Container Storage Interface). CSI enables dynamic provisioning, taking snapshot and restoration of persistent volumes.

#### Objetive
The aim of this guide is to list down the steps required to deploy Postgres statefulset and test out the volume snapshot and restore features. Using the snapshot, the postgres app should be able to restore the data from the restored persistent volume.

A high level breakdown of the workflow:

1. provision K8s cluster
2. deploy Postgres stateful service 
3. create a Snapshot of the persistent volume attached to the Postgres statefulset 
4. create a new persistent volume from the above Snapshot
4. update the Postgres statefulset definition to use the newly created persistent volume and validate the data

The rest of the guide assumes you have a local Minikube Kubernetes cluster installed on your machine, however, any K8s cluster of your choice will work.

#### Deployment

Let's walk through the deployment steps:

Install and configure the CSI driver: You need to install and configure a CSI driver that supports volume snapshots. For K8s cluster other than Minikube, please refer to your storage provider's documentation.

For Minikube, please follow below steps:

```
minikube addons list
minikube addons enable csi-hostpath-driver
minikube addons enable volumesnapshots
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass csi-hostpath-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Deploy the Postgres Stateful Set

```
kubectl apply -f configmap.yaml
kubectl apply -f pvc.yaml
kubectl apply -f service.yaml
kubectl apply -f statefulset.yaml
```

Insert sample data into the Postgres DB.

Log into the postgres container first( password is 'admin' - as configured in the configmap.yaml)

```
kubectl exec -it postgres-0 --  psql -h localhost -U admin --password -d postgresdb -W
```

Copy and paste the below SQL statements to create some sample data and exit the container.

```
CREATE TABLE links (
    id SERIAL PRIMARY KEY,
    url VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description VARCHAR (255),
    last_update DATE
);

INSERT INTO links (url, name)
VALUES('https://web3auth.io/','Web3 Wallet Infrastructure');
```
Validat if the data was successfully inserted

```
select * from links;
```

Create a VolumeSnapshot. Since, we have added some sample data in the above step, a snapshot should capture the state of the volume along with that data.
```
kubectl apply -f volume_snapshot.yaml
```

Create a new PVC from the snapshot

```
kubectl apply -f restore_volume_from_snapshot.yaml
```

Update the StatefulSet: Modify your StatefulSet definition to use the new PVC created from the snapshot as the storage. Replace line 31 with updated value of the PVC claim, i.e. from 'postgres-csi-pvc' to 'postgres-restored-csi-pvc'. 

```
kubectl apply -f statefulset.yaml
```

In order to validate the data, we need to log inside the Postgres container and list the tables and data inside it.

```
kubectl exec -it postgres-0 --  psql -h localhost -U admin --password -d postgresdb -W
```

Execute the below SQL queries to validate the data
```
\dt;
select * from links;
```
Voila, you were able to restored your Postgres data from the snapshot.

#### Additional Improvements
- We can automate the snapshotting of the volume using a K8s Cron Job. In an event of a crash or failure, we can always restore the data from the last snapshot.
- We can even use tool like Velero which enables a comprehensive backup and restore of entire K8s state and not just persistent volumes

