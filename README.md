# GPSS Accelerator

This is an accelerator that can be used to set up GPSS server and gpsscli client as a RabbitMQ connector for [Greenplum](https://docs.vmware.com/en/VMware-Tanzu-Greenplum-Streaming-Server/1.9/greenplum-streaming-server/GUID-rabbitmq-loading.html).

* Install App Accelerator: (see https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-cert-mgr-contour-fcd-install-cert-mgr.html)
```
tanzu package available list accelerator.apps.tanzu.vmware.com --namespace tap-install
tanzu package install accelerator -p accelerator.apps.tanzu.vmware.com -v 1.0.1 -n tap-install -f resources/app-accelerator-values.yaml
Verify that package is running: tanzu package installed get accelerator -n tap-install
Get the IP address for the App Accelerator API: kubectl get service -n accelerator-system
```

Publish Accelerators:
```
tanzu plugin install --local <path-to-tanzu-cli> all
tanzu acc create gpss-rabbit --git-repository https://github.com/agapebondservant/gpss-rabbit-docker.git --git-branch main
```

## Contents
1. [Deploy GPSS connector on vanilla K8s](#k8s)

### Deploy GPSS on vanilla K8s<a name="k8s"/>

#### Before you begin (one time setup):
1. Create an environment file `.env` (use `.env-sample` as a template), then run:
```
source .env
export GPSS_NAMESPACE=gpss # or preferred namespace
```

2. Update template files:
```
for orig in `find . -name "*.in.*" -type f`; do
  target=$(echo $orig | sed 's/\.in//')
  envsubst < $orig > $target
  grep -qxF $target .gitignore || echo $target >> .gitignore
done
```

#### Deploy GPSS Server
1. Build Docker image which launches a GPSS Server instance (if it has not already been built):
```
cd resources/server
docker build -t $DATA_E2E_REGISTRY_USERNAME/gpss-rabbit .
docker push $DATA_E2E_REGISTRY_USERNAME/gpss-rabbit
cd -
```

2. Deploy GPSS Server instance:
```
kubectl create ns $GPSS_NAMESPACE || true
ytt -f resources/server/gpss-server.yaml -v registry_username=$DATA_E2E_REGISTRY_USERNAME | kubectl apply -n $GPSS_NAMESPACE -f -
watch kubectl get all -n $GPSS_NAMESPACE
```

3. To troubleshoot, view logs:
```
kubectl logs -l gpss-app=rabbitmq -n $GPSS_NAMESPACE
```
Or:
```
kubectl describe pod -l gpss-app=rabbitmq -n $GPSS_NAMESPACE
```

To undeploy:
```
kubectl delete all --all -n $GPSS_NAMESPACE
```

#### Deploy GPSS Client
1. Create target table in Greenplum instance:
```
export DATA_E2E_ML_TRAINING_DB_PASSWORD=<enter password>
export PSQL_CONNECT_STR=postgresql://${DATA_E2E_ML_TRAINING_DB_USERNAME}:${DATA_E2E_ML_TRAINING_DB_PASSWORD}@${DATA_E2E_ML_TRAINING_DB_HOST}:${DATA_E2E_ML_TRAINING_DB_PORT}/${DATA_E2E_ML_TRAINING_DB_DATABASE}?sslmode=require
psql ${PSQL_CONNECT_STR} -f resources/client/create_gp_target_tbl.sql
```

2. Generate and upload the gpss-client.yaml file:
```
ytt -f resources/client/gpss-client.yaml \
    -f resources/client/values.yaml \
    -v rabbit_server=${DATA_E2E_GPSS_RABBIT_USER}:${DATA_E2E_GPSS_RABBIT_PWD}@$(kubectl get svc ${DATA_E2E_GPSS_RABBIT_SVC} -n ${DATA_E2E_GPSS_RABBIT_NS} -o jsonpath="{.status.loadBalancer.ingress[0].hostname}"):5672 \
    -v gp_password=${DATA_E2E_ML_TRAINING_DB_PASSWORD} > resources/client/gpss-client-updated.yaml
kubectl cp resources/client/gpss-client-updated.yaml $GPSS_NAMESPACE/$(kubectl get pod -l gpss-app=rabbitmq -n $GPSS_NAMESPACE -o custom-columns=":metadata.name" --no-headers):/tmp
```

3. Initiate the gsscli load job from RabbitMQ to Greenplum:
```
kubectl exec -it $(kubectl get pod -oname -l gpss-app=rabbitmq -n $GPSS_NAMESPACE) -n $GPSS_NAMESPACE -- sh
source /usr/local/gpss-1.9.0/gpss_path.sh
gpsscli load /tmp/gpss-client-updated.yaml 
```

4. To troubleshoot, view the most recent logs:
```
cat $GPSS_HOME/gpsslogs/$(ls -Art $GPSS_HOME/gpsslogs | tail -n 1)
```
