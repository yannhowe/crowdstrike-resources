# Deploying Falcon Sensor for Container Based Workloads

There are a number of ways to collect telemetry from containerized workloads depending on the underlying platform.

You will need to 
1. Select a deployment approach
2. Retrieve the appropriate Falcon Sensor to your own private registry
3. Deploy the sensor

## Prerequisites

Cloud Workload Protection subscription

# Selecting a Deployment Approach

Generally:

**If access to the worker nodes running the containers is available**, deploy the **Falcon Sensor for Linux** on the worker node via DaemonSet or via installing the DEB/RPM package to capture all telemetry from the worker node and the containers running on it.

**If using a managed service similar to Fargate with no access to worker nodes** you will have to deploy the **Falcon Container Sensor for Linux** on every container or as a sidecar in every pod.

# Retrieving the Falcon Sensor Image

To download the Falcon sensor for Linux image you must have a Cloud Workload Protection subscription.

## Prep Work

### Create an API client key

Create a client ID with the required API scope to download the image from the CrowdStrike repository.

- In the Falcon console, go to Support and resources > Resources and tools > API clients and keys.
- Click Add new API client. 
- On the Add new API client dialog, enter a client name and description. 
- Scroll to Falcon Images Download API scope and select Read. 
- Scroll to Sensor Download API scope and select Read
- Click Add. 
- From the API client created dialog, copy the client ID <YOUR_FALCON_CLIENT_ID> and secret <YOUR_FALCON_CLIENT_SECRET> to a password management or secret management service.

Define them in your command line variables

```
export FALCON_CLIENT_ID=<YOUR_FALCON_CLIENT_ID>
export FALCON_CLIENT_SECRET=<YOUR_FALCON_CLIENT_SECRET>
```

### Get your CID with checksum

Your cid with checksum <YOUR_CID_W_CHECKSUM> in the form XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-YY, can be found in the Falcon console Sensor Downloads page.

Define it in your command line variables

```
export FALCON_CID=<YOUR_CID_W_CHECKSUM>
```

### Get your Falcon Cloud Region Details

Depending on your CrowdStrike cloud, define your variables using the values from the following table

If you’re on cloud | YOUR_CLOUD      | YOUR_CLOUD_TAG | YOUR_REGISTRY |
| --- | --- | --- | --- |
| US-1      | api.crowdstrike.com | us-1 | registry.crowdstrike.com |
| US-2      | api.us-2.crowdstrike.com | us-2 | registry.crowdstrike.com |
| EU-1      | api.eu-1.crowdstrike.com | eu-1 | registry.crowdstrike.com |
| GOV-1      | api.laggar.gcw | us-gov-1 | registry.laggar.gcw |


```
export FALCON_CLOUD_API=<YOUR_CLOUD>
export FALCON_REGION=<YOUR_CLOUD_TAG>
export FALCON_CONTAINER_REGISTRY=<YOUR_REGISTRY>
```

## Retrieve the Falcon Sensor Image Locally

You can then obtain the Falcon sensor via
- convenience script or
- command line

### Convenience Script

The convenience script is an open source tool available at Github.

The script requires the following commands to be installed:
- curl
- docker (default), or podman, or skopeo
- CrowdStrike API Client created with the following scope assigned
  - Falcon Images Download (read)
  - Sensor Download (read)

If you are using docker, make sure that docker is running locally.

Download the convenience script and make it executable

```
curl https://raw.githubusercontent.com/yannhowe/falcon-scripts/main/bash/containers/falcon-container-sensor-pull/falcon-container-sensor-pull.sh --output falcon-container-sensor-pull.sh
chmod +x falcon-container-sensor-pull.sh
```

As the variables are already set in the prep work section, you can simply run the following command with the appropriate options set.

This will retrieve the Falcon Sensor for Linux image for deploying as a daemonset.
```
./falcon-container-sensor-pull.sh \
--cid ${FALCON_CID} \
--client-id ${FALCON_CLIENT_ID} \
--client-secret ${FALCON_CLIENT_SECRET} \
--region ${FALCON_REGION} \
--node
```

This will retrieve the Falcon Container Sensor for Linux for deploying as a sidecar.
```
./falcon-container-sensor-pull.sh \
--cid ${FALCON_CID} \
--client-id ${FALCON_CLIENT_ID} \
--client-secret ${FALCON_CLIENT_SECRET} \
--region ${FALCON_REGION}
```


Read the instructions [here](https://github.com/CrowdStrike/falcon-scripts/tree/main/bash/containers/falcon-container-sensor-pull) for further details and options available.

### Manually via Command Line

You’ll need to have the following commands available on your command line
- curl 
- jq
- docker, podman, or skopeo
- Get private CrowdStrike registry credentials from API

Get OAuth2 token to interact with the CrowdStrike API
```
export FALCON_CS_API_TOKEN=$(curl \
  --data "client_id=${FALCON_CLIENT_ID}&client_secret=${FALCON_CLIENT_SECRET}"\
  --request POST \
  --silent \
  https://${FALCON_CLOUD_API}/oauth2/token | jq -cr '.access_token | values')
```

Get CrowdStrike registry username and password
```
export FALCON_ART_USERNAME="fc-$(echo ${FALCON_CID} | awk '{ print tolower($0) }' | cut -d'-' -f1)"
export FALCON_ART_PASSWORD=$(curl -X GET -H "authorization: Bearer ${FALCON_CS_API_TOKEN}" https://${FALCON_CLOUD_API}/container-security/entities/image-registry-credentials/v1 | jq -cr '.resources[].token | values')
```
Get container image tags for latest sensor images
Obtain a token to interact with the CrowdStrike private registry
```
export REGISTRY_BEARER=$(curl -X GET -s -u "${FALCON_ART_USERNAME}:${FALCON_ART_PASSWORD}" "https://${FALCON_CONTAINER_REGISTRY}/v2/token?=fc-${CID}&scope=repository:falcon-sensor/${FALCON_REGION}/release/falcon-sensor:pull&service=${FALCON_CONTAINER_REGISTRY}" | jq -r '.token')
```
Select the image required
Set this if getting the falcon-sensor tag for daemonset deployment
```
export SENSORTYPE=falcon-sensor
```

Set this if getting the falcon-container tag for sidecar deployment
```
export SENSORTYPE=falcon-container
```
Fetch the latest tag
```
export FALCON_SENSOR_IMAGE_REPO="${FALCON_CONTAINER_REGISTRY}/${SENSORTYPE}/${FALCON_REGION}/release/${SENSORTYPE}"
export FALCON_SENSOR_IMAGE_TAG=$(curl -X GET -s -H "authorization: Bearer ${REGISTRY_BEARER}" "https://${FALCON_CONTAINER_REGISTRY}/v2/${SENSORTYPE}/${FALCON_REGION}/release/falcon-sensor/tags/list" | jq -r '.tags[-1]')
```
Move the container images to your internal registry

Set variables for your internal registry at <YOUR_INTERNAL_REPOSITORY> and repositories and ensure they exist and have the correct credentials to access them
```
export MY_INTERNAL_CONTAINER_REGISTRY=<YOUR_INTERNAL_REPOSITORY>
export 
MY_INTERNAL_SENSOR_IMAGE_REPO="${MY_INTERNAL_CONTAINER_REGISTRY}/${SENSORTYPE}"
```

Login to Docker Registries
```
# Login to crowdstrike registry
echo $FALCON_ART_PASSWORD | docker login -u $FALCON_ART_USERNAME --password-stdin ${FALCON_CONTAINER_REGISTRY}

# Login to your internal registry
docker login ${MY_INTERNAL_CONTAINER_REGISTRY}
```

Using Docker to move images to local registry
```
# Pull latest falcon-sensor image for daemonset deployment
docker pull ${FALCON_SENSOR_IMAGE_REPO}:${FALCON_SENSOR_IMAGE_TAG}
 
# Tag the images to point to your registry
docker tag ${FALCON_SENSOR_IMAGE_REPO}:${FALCON_SENSOR_IMAGE_TAG} ${MY_INTERNAL_SENSOR_IMAGE_REPO}:${FALCON_SENSOR_IMAGE_TAG}
 
# push the images to your registry

docker push ${MY_INTERNAL_SENSOR_IMAGE_REPO}:${FALCON_SENSOR_IMAGE_TAG}
```

Using Skopeo to move images to local registry
```
skopeo copy docker://${FALCON_SENSOR_IMAGE_REPO}:${FALCON_SENSOR_IMAGE_TAG} docker://${MY_INTERNAL_SENSOR_IMAGE_REPO}:${FALCON_SENSOR_IMAGE_TAG}
```

# Deploying the Falcon Sensor

Further documentation is available to deploy the falcon sensor:

Kubernetes (as a Pod Sidecar or DaemonSet)
- Using Helm - [https://github.com/CrowdStrike/falcon-helm](https://github.com/CrowdStrike/falcon-helm)
- Using Operator - [https://github.com/CrowdStrike/falcon-operator](https://github.com/CrowdStrike/falcon-operator)

ECS Fargate (as a Container Sidecar via ECS Task Patching)
- Sidecar - [https://github.com/TomRyan-321/crowdstrike-ecs-fargate-pipepline-demo](https://github.com/TomRyan-321/crowdstrike-ecs-fargate-pipepline-demo)