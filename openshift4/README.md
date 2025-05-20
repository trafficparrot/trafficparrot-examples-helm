# Traffic Parrot OpenShift 4 Deployment

This directory contains Helm charts and Docker build configurations for deploying Traffic Parrot and Traffic Parrot License Server on OpenShift 4.

## Contents

- `trafficparrot-chart/` - Helm chart for deploying Traffic Parrot
- `trafficparrot-image/` - Docker build configuration for Traffic Parrot
- `licenseserver-chart/` - Helm chart for deploying Traffic Parrot License Server
- `licenseserver-image/` - Docker build configuration for Traffic Parrot License Server

## Prerequisites

- OpenShift 4.x cluster
- OpenShift CLI (`oc`)
- Podman or Docker
- Helm
- Traffic Parrot license files and distribution packages

## Important Configuration Notes

### License Server Connectivity

When configuring Traffic Parrot to connect to the License Server:

1. For Traffic Parrot instances inside the same cluster:
   - You can use the internal service URL: `http://trafficparrot-license-usage.trafficparrot-license-usage.svc.cluster.local:8040`
   - This approach works well for instances deployed in the same cluster

2. For Traffic Parrot instances outside the cluster (e.g., developer laptops):
   - You must use the externally accessible Route URL
   - Example: `https://trafficparrot-license-usage-usage-trafficparrot-license-usage.apps.<cluster-domain>`
   - This URL should be used in the `trafficparrot.license.usage.server` property

### TLS Termination for License Server

The License Server usage port requires specific TLS termination settings:

- For the UI port (8050): Use `Edge` termination
- For the usage port (8040): Use `Edge` termination with `license.usage.port.ssl.enabled=false`

## Deployment Steps

### 1. Deploy Traffic Parrot License Server

1. Download Traffic Parrot License Server distribution and license:
   ```bash
   wget https://.../trafficparrot.usage.license -P ./licenseserver-image
   wget https://.../trafficparrot-license-usage-linux-x64-jre-X.Y.Z.zip -P ./licenseserver-image
   ```

2. Extract properties file:
   ```bash
   unzip -p ./licenseserver-image/trafficparrot-license-usage-*.zip '*/licenseusage.properties' > ./licenseserver-image/licenseusage.properties
   ```
   
   > **Note:** When using OpenShift Routes with Edge termination, you'll need to set `license.usage.port.ssl.enabled=false`. This can be done either by modifying the properties file directly or by using system property overrides at runtime.

3. Login to OpenShift:
   ```bash
   oc login -u <username> <openshift-api-url>
   ```

4. Create project and build Docker image:
   ```bash
   export APP_NAME=trafficparrot-license-usage
   export REGISTRY=<your-openshift-registry>
   export VERSION=1.16.10  # Set to the actual version of your Traffic Parrot License Server
   oc new-project ${APP_NAME} || oc project ${APP_NAME}
   
   podman login -u <username> -p $(oc whoami -t) ${REGISTRY} --tls-verify=false
   podman build \
     -t ${REGISTRY}/${APP_NAME}/${APP_NAME}:latest \
     -t ${REGISTRY}/${APP_NAME}/${APP_NAME}:${VERSION} \
     ./licenseserver-image
   
   podman push ${REGISTRY}/${APP_NAME}/${APP_NAME}:latest --tls-verify=false
   podman push ${REGISTRY}/${APP_NAME}/${APP_NAME}:${VERSION} --tls-verify=false
   oc set image-lookup ${APP_NAME} && oc set image-lookup
   ```
   
   > **Note:** It's recommended to use proper versioning for your Docker images rather than relying only on the 'latest' tag. This allows for easier rollbacks and clearer deployment history.

5. Deploy using Helm:
   ```bash
   export HELM_ARGUMENTS="./licenseserver-chart
   --namespace ${APP_NAME}
   --create-namespace
   --set appid=${APP_NAME}
   --set image=${APP_NAME}:${VERSION}
   --set uiport=8050
   --set uitermination=edge
   --set usageport=8040
   --set usagetermination=passthrough
   --set storage=1Gi
   --set storageclass=<your-storage-class>
   --set localdev=false"
   
   helm upgrade --install --wait trafficparrot-license-usage ${HELM_ARGUMENTS}
   ```

### 2. Deploy Traffic Parrot

1. Download Traffic Parrot distribution and license:
   ```bash
   wget https://.../trafficparrot.license -P ./trafficparrot-image
   wget https://.../trafficparrot-linux-x64-jre-X.Y.Z.zip -P ./trafficparrot-image
   ```

2. Extract and configure properties:
   ```bash
   unzip -p ./trafficparrot-image/trafficparrot-*.zip '*/trafficparrot.properties' > ./trafficparrot-image/trafficparrot.properties
   
   # Configure license server URL
   export LICENSE_SERVER=https://<license-server-route>
   sed -i "s~^trafficparrot.license.usage.server=.*~trafficparrot.license.usage.server=${LICENSE_SERVER}~g" ./trafficparrot-image/trafficparrot.properties
   ```

3. Create project and build Docker image:
   ```bash
   export APP_NAME=trafficparrot
   export REGISTRY=<your-openshift-registry>
   export VERSION=5.53.0  # Set to the actual version of your Traffic Parrot
   oc new-project ${APP_NAME} || oc project ${APP_NAME}
   
   podman login -u <username> -p $(oc whoami -t) ${REGISTRY} --tls-verify=false
   podman build \
     -t ${REGISTRY}/${APP_NAME}/${APP_NAME}:latest \
     -t ${REGISTRY}/${APP_NAME}/${APP_NAME}:${VERSION} \
     ./trafficparrot-image
   
   podman push ${REGISTRY}/${APP_NAME}/${APP_NAME}:latest --tls-verify=false
   podman push ${REGISTRY}/${APP_NAME}/${APP_NAME}:${VERSION} --tls-verify=false
   oc set image-lookup ${APP_NAME} && oc set image-lookup
   ```

   > **Note:** It's recommended to use proper versioning for your Docker images rather than relying only on the 'latest' tag. This allows for easier rollbacks and clearer deployment history.

4. Deploy using Helm:
   ```bash
   export HELM_ARGUMENTS="./trafficparrot-chart
   --namespace ${APP_NAME}
   --create-namespace
   --set appid=${APP_NAME}
   --set image=${APP_NAME}:${VERSION}
   --set uiport=8080
   --set uitermination=edge
   --set httpvsport=8081
   --set grpcvsport=5552
   --set vstermination=passthrough"
   
   helm upgrade --install --wait trafficparrot ${HELM_ARGUMENTS}
   ```

## Accessing Traffic Parrot

After deployment, Traffic Parrot will be accessible via the routes created by the Helm chart, for example:

- Traffic Parrot UI: https://trafficparrot-ui-trafficparrot.apps.<cluster-domain>
- Traffic Parrot HTTP Virtual Service: https://trafficparrot-http-vs-trafficparrot.apps.<cluster-domain>
- Traffic Parrot gRPC Virtual Service: https://trafficparrot-grpc-vs-trafficparrot.apps.<cluster-domain>

## Accessing Traffic Parrot License Server

After deployment, Traffic Parrot License Server will be accessible via the routes created by the Helm chart, for example:

- License Server UI: https://trafficparrot-license-usage-ui-trafficparrot-license-usage.apps.<cluster-domain>
- License Server Usage Service: https://trafficparrot-license-usage-usage-trafficparrot-license-usage.apps.<cluster-domain>

## Overriding Properties at Runtime

Instead of modifying property files in the Docker image, you can override properties at runtime:

1. For Traffic Parrot, modify the start command in the Dockerfile:
   ```docker
   CMD ["./start-foreground.sh", "trafficparrot.license.usage.server=https://trafficparrot-license-usage-usage-trafficparrot-license-usage.apps.example.com"]
   ```

2. For License Server, modify the start command in the Dockerfile:
   ```docker
   CMD ["./start.sh", "license.usage.port.ssl.enabled=false"]
   ```

## Cleaning Up

To remove the deployments:

```bash
# Remove Traffic Parrot
helm uninstall trafficparrot -n trafficparrot
oc delete project trafficparrot

# Remove License Server
helm uninstall trafficparrot-license-usage -n trafficparrot-license-usage
oc delete project trafficparrot-license-usage
```