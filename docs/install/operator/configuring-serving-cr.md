# Configuring the Knative Serving Operator custom resource

The Knative Serving Operator can be configured with the following options:

- [Version configuration](#version-configuration)
- [Knative Serving configuration by ConfigMap](#knative-serving-configuration-by-configmap)
- [Private repository and private secret](#private-repository-and-private-secrets)
- [SSL certificate for controller](#ssl-certificate-for-controller)
- [Replace the default istio-ingressgateway service](#replace-the-default-istio-ingressgateway-service)
- [Replace the knative-ingress-gateway gateway](#replace-the-knative-ingress-gateway-gateway)
- [Cluster local gateway](#configuration-of-cluster-local-gateway)
- [High availability](#high-availability)
- [System resource settings](#system-resource-settings)
- [Override system deployments](#override-system-deployments)

## Version configuration

Cluster administrators can install a specific version of Knative Serving by using the `spec.version` field.

For example, if you want to install Knative Serving v0.23.0, you can apply the following `KnativeServing` custom resource:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  version: 0.23.0
```

If `spec.version` is not specified, the Knative Operator installs the latest available version of Knative Serving. If users specify an invalid or unavailable version, the Knative Operator will do nothing. The Knative Operator always includes the latest 3 minor release versions. For example, if the current version of the Knative Operator is v0.24.0, the earliest version of Knative Serving available through the Operator is v0.22.0.

If Knative Serving is already managed by the Operator, updating the `spec.version` field in the `KnativeServing` resource
enables upgrading or downgrading the Knative Serving version, without needing to change the Operator.

!!! important
    The Knative Operator only permits upgrades or downgrades by one minor release version at a time. For example, if the current Knative Serving deployment is version v0.22.0, you must upgrade to v0.23.0 before upgrading to v0.24.0.

## Knative Serving configuration by ConfigMap

The Operator manages the Knative Serving installation. It overwrites any updates to ConfigMaps which are used to configure Knative Serving.
The KnativeServing custom resource (CR) allows you to set values for these ConfigMaps by using the Operator.
Knative Serving has multiple ConfigMaps that are named with the prefix `config-`.
The `spec.config` in the KnativeServing CR has one `<name>` entry for each ConfigMap, named `config-<name>`, with a value which will be used for the ConfigMap `data`.

In the [setup a custom domain example](../../serving/using-a-custom-domain.md), you can see the content of the ConfigMap
`config-domain` is:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
data:
  example.org: |
    selector:
      app: prod
  example.com: ""
```

Using the operator, specify the ConfigMap `config-domain` using the operator CR:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  config:
    domain:
      example.org: |
        selector:
          app: prod
      example.com: ""
```

You can apply values to multiple ConfigMaps. This example sets `stable-window` to 60s in `config-autoscaler` as well as specifying `config-domain`:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  config:
    domain:
      example.org: |
        selector:
          app: prod
      example.com: ""
    autoscaler:
      stable-window: "60s"
```

All the ConfigMaps are created in the same namespace as the operator CR. You can use the operator CR as the
unique entry point to edit all of them.

## Private repository and private secrets

You can use the `spec.registry` section of the operator CR to change the image references to point to a private registry or [specify imagePullSecrets](https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod):

- `default`: this field defines a image reference template for all Knative images. The format
is `example-registry.io/custom/path/${NAME}:{CUSTOM-TAG}`. If you use the same tag for all your images, the only difference is the image name. `${NAME}` is
a pre-defined variable in the operator corresponding to the container name. If you name the images in your private repo to align with the container names (
`activator`, `autoscaler`, `controller`, `webhook`, `autoscaler-hpa`, `net-istio-controller`, and `queue-proxy`), the `default` argument should be sufficient.

- `override`: a map from container name to the full registry
location. This section is only needed when the registry images do not match the common naming format. For containers whose name matches a key, the value is used in preference to the image name calculated by `default`. If a container's name does not match a key in `override`, the template in `default` is used.

- `imagePullSecrets`: a list of Secret names used when pulling Knative container images. The Secrets
must be created in the same namespace as the Knative Serving Deployments. See
[deploying images from a private container registry](../../serving/deploying-from-private-registry.md) for configuration details.


### Download images in a predefined format without secrets:

This example shows how you can define custom image links that can be defined in the CR using the simplified format
`docker.io/knative-images/${NAME}:{CUSTOM-TAG}`.

In the following example:

- the custom tag `v0.13.0` is used for all images
- all image links are accessible without using secrets
- images are pushed as `docker.io/knative-images/${NAME}:{CUSTOM-TAG}`

First, you need to make sure your images pushed to the following image tags:

| Container | Docker Image |
|----|----|
|`activator` | `docker.io/knative-images/activator:v0.13.0` |
| `autoscaler` | `docker.io/knative-images/autoscaler:v0.13.0` |
| `controller` | `docker.io/knative-images/controller:v0.13.0` |
| `webhook` | `docker.io/knative-images/webhook:v0.13.0` |
| `autoscaler-hpa` | `docker.io/knative-images/autoscaler-hpa:v0.13.0` |
| `net-istio-controller` | `docker.io/knative-images/net-istio-controller:v0.13.0` |
| `queue-proxy` | `docker.io/knative-images/queue-proxy:v0.13.0` |

Then, you need to define your operator CR with following content:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  registry:
    default: docker.io/knative-images/${NAME}:v0.13.0
```

### Download images individually without secrets

If your custom image links are not defined in a uniform format by default, you will need to individually include each
link in the CR.

For example, to given the following images:

| Container | Docker Image |
|----|----|
| `activator` | `docker.io/knative-images-repo1/activator:v0.13.0` |
| `autoscaler` | `docker.io/knative-images-repo2/autoscaler:v0.13.0` |
| `controller` | `docker.io/knative-images-repo3/controller:v0.13.0` |
| `webhook` | `docker.io/knative-images-repo4/webhook:v0.13.0` |
| `autoscaler-hpa` | `docker.io/knative-images-repo5/autoscaler-hpa:v0.13.0` |
| `net-istio-controller` | `docker.io/knative-images-repo6/prefix-net-istio-controller:v0.13.0` |
| `net-istio-webhook` | `docker.io/knative-images-repo6/net-istio-webhooko:v0.13.0` |
| `queue-proxy` | `docker.io/knative-images-repo7/queue-proxy-suffix:v0.13.0` |

The Operator CR should be modified to include the full list:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  registry:
    override:
      activator: docker.io/knative-images-repo1/activator:v0.13.0
      autoscaler: docker.io/knative-images-repo2/autoscaler:v0.13.0
      controller: docker.io/knative-images-repo3/controller:v0.13.0
      webhook: docker.io/knative-images-repo4/webhook:v0.13.0
      autoscaler-hpa: docker.io/knative-images-repo5/autoscaler-hpa:v0.13.0
      net-istio-controller: docker.io/knative-images-repo6/prefix-net-istio-controller:v0.13.0
      net-istio-webhook/webhook: docker.io/knative-images-repo6/net-istio-webhook:v0.13.0
      queue-proxy: docker.io/knative-images-repo7/queue-proxy-suffix:v0.13.0
```

!!! note
    If the container name is not unique across all Deployments, DaemonSets and Jobs, you can prefix the container name with the parent container name and a slash. For example, `istio-webhook/webhook`.

### Download images with secrets

If your image repository requires private secrets for
access, include the `imagePullSecrets` attribute.

This example uses a secret named `regcred`. You must create your own private secrets if these are required:

- [From existing docker credentials](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials)
- [From command line for docker credentials](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line)
- [Create your own secret](https://kubernetes.io/docs/concepts/configuration/secret/#creating-your-own-secrets)

After you create this secret, edit the Operator CR by appending the following content:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  registry:
    ...
    imagePullSecrets:
      - name: regcred
```

The field `imagePullSecrets` expects a list of secrets. You can add multiple secrets to access the images as follows:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  registry:
    ...
    imagePullSecrets:
      - name: regcred
      - name: regcred-2
      ...
```

## SSL certificate for controller

To [enable tag to digest resolution](../../serving/tag-resolution.md), the Knative Serving controller needs to access the container registry.
To allow the controller to trust a self-signed registry cert, you can use the Operator to specify the certificate using a ConfigMap or Secret.

Specify the following fields in `spec.controller-custom-certs` to select a custom registry certificate:

- `name`: the name of the ConfigMap or Secret.
- `type`: either the string "ConfigMap" or "Secret".

If you create a ConfigMap named `testCert` containing the certificate, change your CR:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  controller-custom-certs:
    name: testCert
    type: ConfigMap
```

## Replace the default istio-ingressgateway-service

To set up a custom ingress gateway, follow [**Step 1: Create Gateway Service and Deployment Instance**](../../serving/setting-up-custom-ingress-gateway.md#step-1-create-the-gateway-service-and-deployment-instance).

### Step 2: Update the Knative gateway

Update `spec.ingress.istio.knative-ingress-gateway` to select the labels of the new ingress gateway:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  ingress:
    istio:
      enabled: true
      knative-ingress-gateway:
        selector:
          istio: ingressgateway
```

### Step 3: Update Gateway ConfigMap

Additionally, you will need to update the Istio ConfigMap:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  ingress:
    istio:
      enabled: true
      knative-ingress-gateway:
        selector:
          istio: ingressgateway
  config:
    istio:
      gateway.knative-serving.knative-ingress-gateway: "custom-ingressgateway.custom-ns.svc.cluster.local"
```

The key in `spec.config.istio` is in the format of `gateway.<gateway_namespace>.<gateway_name>`.

## Replace the knative-ingress-gateway gateway

To create the ingress gateway, follow [**Step 1: Create the Gateway**](../../serving/setting-up-custom-ingress-gateway.md#step-1-create-the-gateway).

### Step 2: Update Gateway ConfigMap

You will need to update the Istio ConfigMap:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  config:
    istio:
      gateway.custom-ns.knative-custom-gateway: "istio-ingressgateway.istio-system.svc.cluster.local"
```

The key in `spec.config.istio` is in the format of `gateway.<gateway_namespace>.<gateway_name>`.

## Configuration of cluster local gateway

Update `spec.ingress.istio.knative-local-gateway` to select the labels of the new cluster-local ingress gateway:

### Default local gateway name:

Go through the [installing Istio](../serving/installing-istio.md#installing-istio-without-sidecar-injection) guide to use local cluster gateway,
if you use the default gateway called `knative-local-gateway`.

### Non-default local gateway name:

If you create custom local gateway with a name other than `knative-local-gateway`, update `config.istio` and the
`knative-local-gateway` selector:

This example shows a service and deployment `knative-local-gateway` in the namespace `istio-system`, with the
label `custom: custom-local-gw`:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  ingress:
    istio:
      enabled: true
      knative-local-gateway:
        selector:
          custom: custom-local-gateway
  config:
    istio:
      local-gateway.knative-serving.knative-local-gateway: "custom-local-gateway.istio-system.svc.cluster.local"
```

## High availability

By default, Knative Serving runs a single instance of each deployment. The `spec.high-availability` field allows you to configure the number of replicas for all deployments managed by the operator.

The following configuration specifies a replica count of 3 for the deployments:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  high-availability:
    replicas: 3
```

The `replicas` field also configures the `HorizontalPodAutoscaler` resources based on the `spec.high-availability`. Let's say the operator includes the following HorizontalPodAutoscaler:

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  ...
spec:
  minReplicas: 3
  maxReplicas: 5
```

If you configure `replicas: 2`, which is less than `minReplicas`, the operator transforms `minReplicas` to `1`.

If you configure `replicas: 6`, which is more than `maxReplicas`, the operator transforms `maxReplicas` to `maxReplicas + (replicas - minReplicas)` which is `8`.

## System Resource Settings

The operator custom resource allows you to configure system resources for the Knative system containers.
Requests and limits can be configured for the following containers: `activator`, `autoscaler`, `controller`, `webhook`, `autoscaler-hpa`,
`net-istio-controller` and `queue-proxy`.

!!! info
    If multiple deployments share the same container name, the configuration in `spec.resources` for that certain container will apply to all the deployments.

To override resource settings for a specific container, create an entry in the `spec.resources` list with the container name and the [Kubernetes resource settings](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container).

For example, the following KnativeServing resource configures the `activator` to request 0.3 CPU and 100MB of RAM, and sets hard limits of 1 CPU, 250MB RAM, and 4GB of local storage:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  resources:
  - container: activator
    requests:
      cpu: 300m
      memory: 100Mi
    limits:
      cpu: 1000m
      memory: 250Mi
      ephemeral-storage: 4Gi
```

If you would like to add another container `autoscaler` with the same configuration, you need to change your CR as follows:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  resources:
  - container: activator
    requests:
      cpu: 300m
      memory: 100Mi
    limits:
      cpu: 1000m
      memory: 250Mi
      ephemeral-storage: 4Gi
  - container: autoscaler
    requests:
      cpu: 300m
      memory: 100Mi
    limits:
      cpu: 1000m
      memory: 250Mi
      ephemeral-storage: 4Gi
```

## Override system deployments

If you would like to override some configurations for a specific deployment, you can override the configuration by using `spec.deployments` in CR.
Currently `replicas`, `labels`, `annotations` and `nodeSelector` are supported.

### Override replicas, labels and annotations

The following KnativeServing resource overrides the `webhook` deployment to have `3` Replicas, the label `mylabel: foo`, and the annotation `myannotataions: bar`,
while other system deployments have `2` Replicas by using `spec.high-availability`.

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  high-availability:
    replicas: 2
  deployments:
  - name: webhook
    replicas: 3
    labels:
      mylabel: foo
    annotations:
      myannotataions: bar
```

!!! note
    The KnativeServing resource `label` and `annotation` settings override the webhook's labels and annotations for both Deployments and Pods.

### Override the nodeSelector

The following KnativeServing resource overrides the `webhook` deployment to use the `disktype: hdd` nodeSelector:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  deployments:
  - name: webhook
    nodeSelector:
      disktype: hdd
```

### Override the tolerations

The KnativeServing resource is able to override tolerations for the Knative Serving deployment resources.
For example, if you would like to add the following tolerations

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

to the deployment `activator`, you need to change your KnativeServing CR as below:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  deployments:
  - name: activator
    tolerations:
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoSchedule"
```

### Override the affinity

The KnativeServing resource is able to override the affinity, including nodeAffinity, podAffinity, and podAntiAffinity,
for the Knative Serving deployment resources. For example, if you would like to add the following nodeAffinity

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      preference:
        matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

to the deployment `activator`, you need to change your KnativeServing CR as below:

```yaml
apiVersion: operator.knative.dev/v1alpha1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  deployments:
  - name: activator
    affinity:
      nodeAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
            - key: disktype
              operator: In
              values:
              - ssd
```
