# Config Connector Test

## Setup

I followed the [Installing with the GKE add-on] documentation to create a cluster and deploy to
it.

### Creating a Kubernetes Cluster

Initialize your gcloud configuation and enable the necessary services in the project (I use
`direnv` to set the `CLOUDSDK_ACTIVE_CONFIG_NAME` environment variable to a configuration that
points at the project I'm using):

```bash
gcloud init
gcloud services enable \
  compute.googleapis.com \
  container.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com
```

Create a Kubernetes cluster to use for testing (I also had to bump by "In use addresses" quote for
the region I was creating the cluster in):

```bash
CLUSTER_NAME=config-connector-test
CHANNEL=regular
PROJECT_ID=jerome-config-connector-test
gcloud container clusters create $CLUSTER_NAME \
    --release-channel $CHANNEL \
    --addons ConfigConnector \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --logging=SYSTEM,WORKLOAD \
    --monitoring=SYSTEM
```

Create and configure an IAM service account for Config Connector:

```bash
SERVICE_ACCOUNT_NAME=config-connector
PROJECT_ID=jerome-config-connector-test
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/owner"
```

Create an IAM policy binding between the IAM service account and the predefined Kubernetes service
account that Config Connector runs:

```bash
SERVICE_ACCOUNT_NAME=config-connector
PROJECT_ID=jerome-config-connector-test
gcloud iam service-accounts add-iam-policy-binding \
${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
    --role="roles/iam.workloadIdentityUser"
```

## Configuring Config Connector

Install Config Connector on the cluster and then check that it is ready:

```bash
kubectl apply -f k8s/configconnector.yaml
kubectl wait -n cnrm-system \
      --for=condition=Ready pod --all
```

Create a namespace to use for testing:

```bash
kubectl apply -f k8s/namespace.yaml
```

Enable the Service Usage API as Config Connector uses it to enable service APIs:

```bash
gcloud services enable serviceusage.googleapis.com
```

## Example Usage

I followed the [Authenticating to Google Cloud with Workload Identity] documentation for this
example.

### Creating the Pub/Sub Resources

For our example, we'll create a Pub/Sub topic. Before we can do that though we need to enable the
Pub/Sub API:

```bash
kubectl apply -f k8s/enable-pubsub.yaml
```

Then we can create the Pub/Sub topic and wait for it to be ready:

```bash
kubectl apply -f k8s/pubsub-topic.yaml

TOPIC_NAME=echo
kubectl wait --for=condition=READY pubsubtopics $TOPIC_NAME
```

Then create a subscription on the topic:

```bash
kubectl apply -f k8s/pubsub-subscription.yaml
```

### Deploying the Application

Create a service account and deployment:

```bash
kubectl apply -f k8s/service-account.yaml
kubectl apply -f k8s/deployment.yaml
```

At this point the pod in the deployment should fail to start because it doesn't have permisson to
access the pubsub topic:

```console
$ k logs -l app=pubsub
    return api_call(*args)
  File "/usr/local/lib/python3.8/site-packages/google/gax/api_callable.py", line 376, in inner
    return a_func(*args, **kwargs)
  File "/usr/local/lib/python3.8/site-packages/google/gax/retry.py", line 125, in inner
    raise errors.RetryError(
google.gax.errors.RetryError: RetryError(Exception occurred in retry method that was not classified as transient, caused by <_InactiveRpcError of RPC that terminated with:
        status = StatusCode.PERMISSION_DENIED
        details = "User not authorized to perform this action."
        debug_error_string = "{"created":"@1640707902.738121863","description":"Error received from peer ipv4:173.194.210.95:443","file":"src/core/lib/surface/call.cc","file_line":1063,"grpc_message":"User not authorized to perform this action.","grpc_status":7}"
>)
```

### Configuring IAM permissions

Create an IAM service account for the application and grant it the `roles/pubsub.subscriber` role
for the topic we created:

```bash
kubectl apply -f k8s/iam-service-account.yaml
kubectl apply -f k8s/iam-policy-member.yaml
```

At this point the pods should be running successfully as they now have access to the topic:

```console
$ k logs -l app=pubsub
Pulling messages from Pub/Sub subscription...
```

[installing with the gke add-on]: https://cloud.google.com/config-connector/docs/how-to/install-upgrade-uninstall
[authenticating to google cloud with workload identity]: https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#authenticating_to
