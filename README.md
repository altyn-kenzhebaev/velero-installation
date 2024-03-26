# Velero installation on GCP
## Links for additional information about velero
* [Velero on GCP GKE Cluster] (https://gist.github.com/rcompos/adc4f0dd00e37df023fd78c5db7965ef)
* [Velero plugin for GCP] (https://github.com/vmware-tanzu/velero-plugin-for-gcp#setup)
* [gke-casa] (https://github.com/yongkanghe/gke-casa/tree/main)

## Installation
```
BUCKET=altyn-backup-test01
gsutil mb gs://$BUCKET/
```

```
gcloud config list
```

```
PROJECT_ID=$(gcloud config get-value project)
```

```
gcloud iam service-accounts create velero --display-name "Velero service account"
```

```
gcloud iam service-accounts list
```

```
SERVICE_ACCOUNT_EMAIL=$(gcloud iam service-accounts list --filter="displayName:Velero service account" --format 'value(email)')
```

```
ROLE_PERMISSIONS=(
    compute.disks.get
    compute.disks.create
    compute.disks.createSnapshot
    compute.snapshots.get
    compute.snapshots.create
    compute.snapshots.useReadOnly
    compute.snapshots.delete
    compute.zones.get
)
```

```
gcloud iam roles create velero.server --project $PROJECT_ID --title "Velero Server" --permissions "$(IFS=","; echo "${ROLE_PERMISSIONS[*]}")"
```

```
gcloud projects add-iam-policy-binding $PROJECT_ID --member serviceAccount:$SERVICE_ACCOUNT_EMAIL --role projects/$PROJECT_ID/roles/velero.server
```

```
gsutil iam ch serviceAccount:$SERVICE_ACCOUNT_EMAIL:objectAdmin gs://${BUCKET}
```

```
gcloud iam service-accounts keys create credentials-velero --iam-account $SERVICE_ACCOUNT_EMAIL
```

For linux:
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
```
For Mac:
```
brew install velero
```

```
velero install \
--provider gcp \
--plugins velero/velero-plugin-for-gcp:v1.6.0 \
--bucket $BUCKET \
--secret-file ./credentials-velero \
--use-volume-snapshots=true
```

## Creation of test Deployment
```
kubectl apply -f base-grafana.yaml
```

```
kubectl -n grafana-example get deployments
```

```
kubectl get pv -n grafana-example
```

## Test Backup Creation
```
velero backup create grafana-backup --selector app=grafana
velero backup describe grafana-backup
```

## Test Schedule Creation
```
velero schedule create grafana-weekly --schedule="0 1 */7 * *" --selector app=grafana
velero schedule get
```

## Restore from Backup
```
velero restore create --from-backup grafana-backup
velero restore get
```

## Clean Up
```
velero backup delete grafana-backup
kubectl delete -f base-grafana.yaml
```

## Uninstall
```
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero
```
