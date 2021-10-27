# Equinix Helm chart for the OpenTelemetry Collector

Based on [Honeycomb's Helm chart](https://github.com/honeycombio/helm-charts/tree/main/charts/opentelemetry-collector),
but significantly pared down.

## Using this chart

### Add the Collector chart to your app

Generate a tarball from this Helm chart and place it in the `charts/` directory of the target application's k8-site repository:

```sh
cd k8s-site-{appname}
mkdir charts
helm package ~/path/to/k8s-otel-collector --destination ./charts
```

### Set the OTLP endpoint

Your application's OpenTelemetry configuration looks for the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable to determine where to send data.
In this case the OTLP endpoint is the url of the collector deployed to that application's namespace.

For apps sending OTLP over HTTP (Ruby apps), use the HTTP endpoint:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT="http://opentelemetry-collector:55681"
```

For apps sending OTLP over gRPC (Go apps), use the gRPC endpoint:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT="opentelemetry-collector:4317
```

Depending on the app's Helm chart configuration, the environment variable may need to be set in different ways.
Most k8s-site-{appname} charts will set environment variables in `values.yaml` like so:

```yaml
{appname}:
  env:
    . . .
    OTEL_EXPORTER_OTLP_ENDPOINT: "opentelemetry-collector:4317"
```

If you're not sure where to add the environment variable, ask SRE (`#sre`) or the Delivery team (`#eng-k8s`) for help.

## Deploying the collector

### Honeycomb secret

Generate a new API key in Honeycomb following [these instructions in Confluence](https://packet.atlassian.net/wiki/spaces/SWE/pages/2794946582/Deploying+the+OpenTelemetry+Collector#Set-up-Honeycomb-secrets).
The API key name in Honeycomb should use the format `prod-{appname}`.

Push the key to keymaker following [these instructions on the delivery docs site](https://delivery-docs.metalkube.net/core_services/keymaker/?h=keymaker#add-secret-to-secret-store). A typical push will look like below. Replace the all caps values, the rest should be consistent across apps.

```yaml
apiVersion: keymaker.equinixmetal.com/v1
kind: ExternalSecretPush
metadata:
  name: honeycomb-secret
spec:
  backend: ssm
  environment: prod
  namespace: REPLACE_ME_NAMESPACE
  secrets:
    - key:   honeycomb-secret
      value: REPLACE_ME_HONEYCOMB_TOKEN
      version: v1
```

The final key path should look like `/prod/{appname}/honeycomb-secret/v1`, which will automatically get picked up by the ExternalSecretPull generated by the template in this chart.

### Syncing in Argo

For initial initial deployment and any changes to the OTLP endpoint, the app's pods will need to be restarted in order to pick up the new/updated `OTEL_EXPORTER_OTLP_ENDPOINT`.
For some configurations, Argo will restart the pods automatically.
For others, you may need to update the Helm chart so that Argo detects a change in the environment variables.
Reach out to SRE (`#sre`) or the Delivery team (`#eng-k8s`) for help with that.

## Troubleshooting

[Follow the instructions in these docs](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/troubleshooting.md)
to set the Collector's own logs to `DEBUG`.

## Architecture

By default, we're deploying one collector instance per application namespace.
