# Cloud Pricing API

![Version: 0.5.0](https://img.shields.io/badge/Version-0.5.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: v0.3.3](https://img.shields.io/badge/AppVersion-v0.3.3-informational?style=flat-square)

A Helm chart for running the Infracost [Cloud Pricing API](https://github.com/infracost/cloud-pricing-api).

## TL;DR

Installing the chart will create three pods: PostgreSQL DB, Cloud Pricing API, and an init job that loads the pricing data. The init job will take a few minutes and exit after the logs show `Completed: downloading DB data` -- you should **wait** for that before running the Infracost CLI. A weekly cronjob is also created to update the prices. Our resource request/limit recommendations are commented-out in the [values.yaml](values.yaml) file per Helm best practices.

  ```sh
  helm repo add infracost https://infracost.github.io/helm-charts/
  helm repo update

  # Run `infracost register` to create an API key, this is used by the weekly job to download the latest cloud pricing data from us.
  helm install cloud-pricing-api infracost/cloud-pricing-api \
    --set infracostAPIKey="YOUR_INFRACOST_API_KEY_HERE" \
    --set postgresql.postgresqlPassword="STRONG_PASSWORD_HERE"
  ```

  We recommend you create an [ingress route](#install-in-aws-with-alb-ingress) so your Infracost CLI users can connect to your self-hosted Cloud Pricing API.

  Regardless of you using ingress or port-forward, the home page for the Cloud Pricing API, [**http://localhost:4000**](http://localhost:4000), shows if prices are up-to-date and some statistics.

Uninstalling the chart will not delete the PVC used by the PostgreSQL DB.

  ```sh
  helm uninstall cloud-pricing-api
  ```

## Configure CLI to use self-hosted Cloud Pricing API

The best way to get instructions for configuring Infracost to use the self-hosted Cloud Pricing API is to check the output at the end of the `helm install` step since this contains the exact commands you need to run. If these are not available, you can:

1. If you don't have ingress enabled you can port-forward the Cloud Pricing API to your local machine by doing this:
    ```sh
    export NAMESPACE=my-namespace
    echo "Your self-hosted Infracost API key is $(kubectl get secret --namespace $NAMESPACE cloud-pricing-api --template="{{ index .data \"self-hosted-infracost-api-key\" }}" | base64 -D)"
    export POD_NAME=$(kubectl get pods --namespace $NAMESPACE -l "app.kubernetes.io/name=cloud-pricing-api,app.kubernetes.io/instance=cloud-pricing-api" -o jsonpath="{.items[0].metadata.name}")
    export CONTAINER_PORT=$(kubectl get pod --namespace $NAMESPACE $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
    kubectl --namespace $NAMESPACE port-forward $POD_NAME 4000:$CONTAINER_PORT
    ```

2. When using the CLI locally, run the following two required commands to point your CLI to your self-hosted Cloud Pricing API. Your Infracost CLI users will use the API key to authenticate when calling your self-hosted Cloud Pricing API.
    ```sh
    infracost configure set pricing_api_endpoint http://localhost:4000
    infracost configure set api_key API_KEY_FROM_ABOVE

    infracost breakdown --path /path/to/code
    ```

3. In CI/CD systems, set the following two required environment variables:

    ```sh
    export INFRACOST_PRICING_API_ENDPOINT=http://endpoint
    export INFRACOST_API_KEY=API_KEY_FROM_ABOVE
    ```

## Prerequisites

* Kubernetes 1.12+ with Beta APIs enabled
* Helm >= 3.1.0
* PV provisioner support in the underlying infrastructure

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| https://charts.bitnami.com/bitnami | postgresql | 10.x.x |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| api.affinity | object | `{}` | API affinity |
| api.autoscaling.enabled | bool | `false` | Create a HorizontalPodAutoscaler for the API |
| api.autoscaling.maxReplicas | int | `10` | The maximum replicas for the API autoscaler |
| api.autoscaling.minReplicas | int | `1` | The minimum replicas for the API autoscaler |
| api.autoscaling.targetCPUUtilizationPercentage | int | `80` | The target CPU threshold for the API autoscaler |
| api.disableTelemetry | bool | `false` | Set this to true to opt-out of telemetry |
| api.extraContainers | list | `[]` | API extra sidecar containers |
| api.livenessProbe.enabled | bool | `true` | Enable the liveness probe |
| api.livenessProbe.failureThreshold | int | `3` | The liveness probe failure threshold |
| api.livenessProbe.initialDelaySeconds | int | `5` | The liveness probe initial delay seconds |
| api.livenessProbe.periodSeconds | int | `5` | The liveness probe period seconds |
| api.livenessProbe.successThreshold | int | `1` | The liveness probe success threshold |
| api.livenessProbe.timeoutSeconds | int | `2` | The liveness probe timeout seconds |
| api.logLevel | string | `"info"` | Set this to debug, info, warn or error |
| api.nodeSelector | object | `{}` | API node selector |
| api.readinessProbe.enabled | bool | `true` | Enable the readiness probe |
| api.readinessProbe.failureThreshold | int | `3` | The readiness probe failure threshold |
| api.readinessProbe.initialDelaySeconds | int | `5` | The readiness probe initial delay seconds |
| api.readinessProbe.periodSeconds | int | `5` | The readiness probe period seconds |
| api.readinessProbe.successThreshold | int | `1` | The readiness probe success threshold |
| api.readinessProbe.timeoutSeconds | int | `2` | The readiness probe timeout seconds |
| api.replicaCount | int | `1` | Replica count |
| api.resources | object | `{"limits":{"cpu":"1","memory":"512Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}` | API resource limits and requests, our request recommendations are based on minimal requirements and the limit recommendations are based on usage in a high-traffic production environment. If you are running on environments like Minikube you may wish to remove these recommendations. |
| api.selfHostedInfracostAPIKey | string | `""` | If you ever need to rotate the API key, you can simply update `self-hosted-infracost-api-key` in the `cloud-pricing-api` secret and restart the application. |
| api.tolerations | list | `[]` | API tolerations |
| fullnameOverride | string | `""` | Full name override for the deployed app |
| image.pullPolicy | string | `"Always"` | Image pull policy pullPolicy: IfNotPresent |
| image.repository | string | `"infracost/cloud-pricing-api"` | Cloud Pricing API image |
| image.tag | string | `""` | Overrides the image tag whose default is the chart appVersion. |
| imagePullSecrets | list | `[]` | Any image pull secrets |
| infracostAPIKey | string | `""` | Use the [Infracost CLI](https://github.com/infracost/infracost/blob/master/README.md#quick-start) `infracost register` command to get an API key so your self-hosted Cloud Pricing API can download the latest pricing data from us. |
| ingress.annotations | object | `{}` | Ingress annotation |
| ingress.className | string | `""` | Ingress class field that replace the kubernetes.io/ingress.class annotation starting at kubernetes 1.18 |
| ingress.enabled | bool | `false` | Enable the ingress controller resource |
| ingress.hosts[0].host | string | `"cloud-pricing-api.local"` | Host name |
| ingress.hosts[0].paths[0].path | string | `"/"` | Path for host |
| ingress.tls | list | `[]` | TLS configuration |
| job.affinity | object | `{}` | Job affinity |
| job.backoffLimit | int | `6` | Job backoff limit |
| job.extraContainers | list | `[]` | Job extra sidecar containers |
| job.failedJobsHistoryLimit | int | `5` | History limit for failed jobs |
| job.logLevel | string | `"info"` | Set this to debug, info, warn or error |
| job.nodeSelector | object | `{}` | Job node selector |
| job.resources | object | `{"limits":{"cpu":"200m","memory":"640Mi"},"requests":{"cpu":"50m","memory":"128Mi"}}` | Job resource limits and requests. If you are running on environments like Minikube you may wish to remove these recommendations. |
| job.runInitJob | bool | `true` | Run the job as a one-off on deploy |
| job.schedule | string | `"0 4 * * SUN"` | Job schedule |
| job.startingDeadlineSeconds | int | `3600` | Deadline seconds for the job starting |
| job.successfulJobsHistoryLimit | int | `5` | History limit for successful jobs |
| job.tolerations | list | `[]` | Job tolerations |
| job.ttlSecondsAfterFinished | int | `nil` | Marks the jobs as eligible for automatic cleanup after the job has finished and the TTL has expired, whether the job is successful or failed. This avoids the need to delete the init job before upgrading the Helm chart. |
| nameOverride | string | `""` | Name override for the deployed app |
| podAnnotations | object | `{}` | Any pod annotations |
| podSecurityContext | object | `{}` | The pod security context |
| postgresql.enabled | bool | `true` | Deploy PostgreSQL servers. See [below](#postgresql) for more details |
| postgresql.existingSecret | string | `""` | Use an existing secret with the PostgreSQL password |
| postgresql.external | object | `{}` | Details of external PostgreSQL server, such as AWS RDS, to use (assuming you've set postgresql.enabled to false) |
| postgresql.postgresqlDatabase | string | `"cloudpricingapi"` | Name of the PostgreSQL database |
| postgresql.postgresqlUsername | string | `"cloudpricingapi"` | Name of the PostgreSQL user |
| postgresql.usePasswordFile | bool | `false` | Have the secrets mounted as a file instead of env vars |
| securityContext | object | `{}` | The container security context |
| service.port | int | `80` | Kubernetes service port |
| service.type | string | `"ClusterIP"` | Kubernetes service type |
| serviceAccount.annotations | object | `{}` | Annotations to add to the service account |
| serviceAccount.create | bool | `true` | Specifies whether a service account should be created |
| serviceAccount.name | string | `""` | The name of the service account to use. If not set and create is true, a name is generated using the fullname template |

See the [values.yaml](values.yaml) file for parameters that our chart uses. The full list of parameters are in the [Bitnami PostgreSQL chart](https://github.com/bitnami/charts/blob/master/bitnami/postgresql/README.md); you can specify the values for this chart by prefixing them with `postgresql.`

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example:

```sh
helm install cloud-pricing-api infracost/cloud-pricing-api \
  --set api.selfHostedInfracostAPIKey=CUSTOM_API_KEY
```

Alternatively, a YAML file that specifies the values for the above parameters can be provided while installing the chart. For example:

```sh
helm install -f my-values.yaml cloud-pricing-api infracost/cloud-pricing-api
```

## PostgreSQL

By default, PostgreSQL is installed as part of the chart using the [Bitnami PostgreSQL chart](https://github.com/bitnami/charts/blob/master/bitnami/postgresql/README.md). You can specify the values for this chart by prefixing them with `postgresql.`. To avoid issues when upgrading this chart, provide `postgresql.postgresqlPassword` for subsequent installs and upgrades. This is due to an issue in the PostgreSQL chart where password will be overwritten with randomly generated passwords otherwise. See [here](https://github.com/helm/charts/tree/master/stable/postgresql#upgrade) for more detail.

To use an external PostgreSQL server (such as AWS RDS or Azure Database for PostgreSQL), set `postgresql.enabled` to `false` and set the `postgresql.external.*` values:
sh
```
helm install cloud-pricing-api infracost/cloud-pricing-api \
  --set infracostAPIKey="YOUR_INFRACOST_API_KEY_HERE" \
  --set postgresql.enabled="false" \
  --set postgresql.external.host="MY_HOST" \
  --set postgresql.external.port="MY_PORT" \
  --set postgresql.external.database="MY_DATABASE" \
  --set postgresql.external.user="MY_USER" \
  --set postgresql.external.password="MY_PASSWORD"
```

## Examples

### Install in AWS with ALB ingress

This is how the Infracost team deploys the Cloud Pricing API on our EKS cluster to test it.

```sh
export DOMAIN=cloud-pricing.api.dev.infracost.io
export CERTIFICATE_DOMAIN=*.api.dev.infracost.io
export CERTIFICATE_ARN=$(aws acm list-certificates --query 'CertificateSummaryList[].[CertificateArn,DomainName]' --output text | grep ${CERTIFICATE_DOMAIN} | cut -f1)

helm install cloud-pricing-api infracost/cloud-pricing-api \
  --set ingress.enabled=true \
  --set ingress.hosts\[0\].host=${DOMAIN} \
  --set ingress.hosts\[0\].paths\[0\].path=/\* \
  --set ingress.extraPaths\[0\].path=/\* \
  --set ingress.extraPaths\[0\].backend.serviceName=ssl-redirect \
  --set ingress.extraPaths\[0\].backend.servicePort=use-annotation \
  --set ingress.annotations."kubernetes\.io/ingress\.class"=alb \
  --set ingress.annotations."alb\.ingress\.kubernetes\.io/scheme"=internet-facing \
  --set ingress.annotations."alb\.ingress\.kubernetes\.io/target-type"=ip \
  --set ingress.annotations."alb\.ingress\.kubernetes\.io/certificate-arn"=${CERTIFICATE_ARN} \
  --set ingress.annotations."alb\.ingress\.kubernetes\.io/healthcheck-path"=/health
  --set-string ingress.annotations."alb\.ingress\.kubernetes\.io/listen-ports"="\[\{\"HTTP\": 80\}\, \{\"HTTPS\":443\}\]" \
  --set-string ingress.annotations."alb\.ingress\.kubernetes\.io/ssl-redirect"="\{\"Type\": \"redirect\"\, \"RedirectConfig\": \{ \"Protocol\": \"HTTPS\"\, \"Port\": \"443\"\, \"StatusCode\": \"HTTP_301\"\}\}"
```

## Upgrade to latest version

Use the following commands to upgrade to the latest released version of the Cloud Pricing API and Helm chart; you should pass-in any variables that you set during install with `--set`:
```
kubectl delete job -n my-namespace hosted-cloud-pricing-api-init-job

helm upgrade cloud-pricing-api infracost/cloud-pricing-api \
    --set infracostAPIKey="YOUR_INFRACOST_API_KEY_HERE" \
    --set postgresql.postgresqlPassword="STRONG_PASSWORD_HERE"
```

## Development

To install the chart from your local repository with the name `my-release`:

```sh
helm install my-release .
```

To uninstall `my-release` deployment:

```sh
helm uninstall my-release
```

To debug issues, such as `job-cron.yaml: error converting YAML to JSON: yaml: line X`, use the following:

```sh
helm template cloud-pricing-api charts/cloud-pricing-api/ --debug > out.yaml
cat out.yaml # look for issues in the YAML in the erroring resource
```
