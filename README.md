# Velero Search and Practice Post demo code

## KIND k8s cluster

1. Create KIND k8s cluster
    ```bash
    kind create cluster --name rocketchat-migrate-demo --config kind-config.yaml
    ```
1. setup metallb
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml
    kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml

    kubectl apply -f metallb-config.yaml
    ```
1. setup `rocketchat-migrate-demo-control-plane` node volume mount
    - 連線至 `rocketchat-migrate-demo-control-plane` node
      ```bash
      docker exec -it rocketchat-migrate-demo-control-plane bash
      ```
    - 在 `rocketchat-migrate-demo-control-plane` node 建立後緒所需的磁碟區
      ```bash
      for vol in pv1; do mkdir /mnt/disks/$vol; mount -t tmpfs $vol /mnt/disks/$vol; done
      ```
1. create `local-storage` storageClass
    ```bash
    kubectl apply -f local-pv-storage.yaml
    ```
1. creat `local-persistent-storage` local persistent volume
    ```bash    
    kubectl apply -f local-pv.yaml
    ```
1. create Minio Server
    ```bash
    helm install minio --create-namespace --namespace minio -f minio-values.yaml bitnami/minio
    ```
1. create rocketchat workload
    ```bash
    helm install rocketchat --namespace rocketchat --create-namespace -f rocketchat.yaml stable/rocketchat
    ```
1. setup velero and restic
    ```bash
    helm install velero vmware-tanzu/velero --namespace velero --create-namespace -f velero-values.yaml
    ```
1. annotate pod
    ```bash
    kubectl annotate pod -n rocketchat --selector=release=rocketchat,app=mongodb backup.velero.io/backup-volumes=datadir --overwrite
    ```
1. backup rocketchat
    ```
    velero backup create rocketchat-backup --include-namespaces rocketchat
    ```

## s3cmd download bacup

```bash
cat <<EOF > ~/.s3cfg
host_base = <kind-minio-server>:9000
host_bucket = <kind-minio-server>:9000
bucket_location = default
use_https = False
access_key =  minio
secret_key = minio123
signature_v2 = False
EOF

mkdir backup
s3cmd -p sync s3://velero/ backup/
```

## GKE cluster

1. create GKE cluster
1. setup Minio Server
    ```bash
    helm install minio --create-namespace --namespace minio -f minio-values.yaml bitnami/minio
    ```
2. upload local backup to Minio server
    ```bash
    s3cmd -p sync backup/ s3://velero/
    ```
3. setup Velero & Restic
    ```bash
    MINIO_SVC=$(kubectl -n minio get svc minio -o=jsonpath="{.status.loadBalancer.ingress[0].ip}")
    PUBLIC_URL=http://$MINIO_SVC:9000
    echo $PUBLIC_URL
    helm install velero vmware-tanzu/velero --namespace velero --create-namespace --set configuration.backupStorageLocation.config.publicUrl=$PUBLIC_URL -f velero-values.yaml
    ```
4. setup `change-storage-class-config.yaml` for StorageClass mapping
    ```bash
    kubectl apply -f change-storage-class-config.yaml -n velero
    ```
5. restore velero backup
    ```bash
    velero restore create --from-backup rocketchat-backup
    ```