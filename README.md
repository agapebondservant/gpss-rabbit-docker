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

### Install GPSS on vanilla K8s<a name="k8s"/>

#### Before you begin (one time setup):
1. Create an environment file `.env` (use `.env-sample` as a template), then run:
```
source .env
```
