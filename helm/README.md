# Helm Charts

Each helm chart that you can use has the following keys and you need to set them. The `cluster.provider` is used as a key for the various cloud features enabled. Also you only need to specify one cloud provider, **not** both if deploying to cloud. As of writing this doc, AWS and Azure are fully supported.

```bash
# dict with what features and the env you're deploying to
cluster:
  provider: local  # choose from: local | aws | azure
  cloudNativeServices: false # set to true to use Cloud Native Services (SecretsManager and IAM for AWS; KeyVault & Managed Identities for Azure)

aws:
  # the aws cli commands uses the name 'quorum-node-secrets-sa' so only change this if you altered the name
  serviceAccountName: quorum-node-secrets-sa
  # the region you are deploying to
  region: ap-southeast-2

azure:
  # the script/bootstrap.sh uses the name 'quorum-pod-identity' so only change this if you altered the name
  identityName: quorum-pod-identity
  # the clientId of the user assigned managed identity created in the template
  identityClientId: azure-clientId
  keyvaultName: azure-keyvault
  # the tenant ID of the key vault
  tenantId: azure-tenantId
  # the subscription ID to use - this needs to be set explictly when using multi tenancy
  subscriptionId: azure-subscriptionId

```

## Usage

Verify kubectl is connected to Minikube with: (please use the latest version of kubectl)

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.1", GitCommit:"4485c6f18cee9a5d3c3b4e523bd27972b1b53892", GitTreeState:"clean", BuildDate:"2019-07-18T09:18:22Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:32:14Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

### _Spin up ELK for logs: (Optional but recommended)_

**NOTE:** this uses charts from Elastic - please configure this as per your requirements and policies

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
# if on cloud
helm install elasticsearch --version 7.17.1 elastic/elasticsearch --namespace teku --create-namespace --values ./values/elasticsearch.yml
# if local - set the replicas to 1
helm install elasticsearch --version 7.17.1 elastic/elasticsearch --namespace teku --create-namespace --values ./values/elasticsearch.yml --set replicas=1 --set minimumMasterNodes=1
helm install kibana --version 7.17.1 elastic/kibana --namespace teku --values ./values/kibana.yml
helm install filebeat --version 7.17.1 elastic/filebeat  --namespace teku --values ./values/filebeat.yml
```

Please also deploy the ingress (below) and the ingress rules to access kibana on path `http://<INGRESS_IP>/kibana`.
Alternatively configure the kibana ingress settings in the [values.yml](./values/kibana.yml)

Once you have kibana open, create a `filebeat` index pattern and logs should be available. Please configure this as
per your requirements and policies

### _Spin up prometheus-stack for metrics: (Optional but recommended)_

**NOTE:** this uses charts from prometheus-community - please configure this as per your requirements and policies

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
# NOTE: please refer to values/monitoring.yml to configure the alerts per your requirements ie slack, email etc
helm install monitoring prometheus-community/kube-prometheus-stack --version 39.10.0 --namespace=teku --create-namespace --values ./values/monitoring.yml --wait
kubectl --namespace teku create -f  ./values/monitoring/
```

Additionally, you will need to deploy a separate ingress which will serve external facing services like the explorer and monitoring endpoints

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install teku-monitoring-ingress ingress-nginx/ingress-nginx \
    --namespace teku \
    --set controller.ingressClassResource.name="monitoring-nginx" \
    --set controller.ingressClassResource.controllerValue="k8s.io/monitoring-ingress-nginx" \
    --set controller.replicaCount=1 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.externalTrafficPolicy=Local

kubectl apply -f ../ingress/ingress-rules-monitoring.yml
```

Once complete, view the IP address listed under the `Ingress` section if you're using the Kubernetes Dashboard
or on the command line `kubectl -n quorum get services quorum-monitoring-ingress-ingress-nginx-controller`.

You can then access Grafana on: 
```bash
# For Teku's grafana address:
http://<INGRESS_IP>/d/DzqnL9oGk/teku-overview?orgId=1&refresh=10s

# For Besu's grafana address:
http://<INGRESS_IP>/d/XE4V0WGZz/besu-overview?orgId=1&refresh=10s

# For Geths's grafana address:
http://<INGRESS_IP>/d/3UR2LdFGAEUzuJ2KJCzs/geth-overview?orgId=1&refresh=10s
```

You can access Kibana on:
```bash
http://<INGRESS_IP>/kibana
```

```bash
helm dependency update ./charts/teku
helm template reader ./charts/teku --namespace teku --create-namespace --values ./values/teku_besu.yml
```

### _Validator Keys_
If you are deploying with validator keys, please put the relevant encrypted json keys and passwords as txt files in the [validator](../helm/charts/teku/validator/) folder and set the `.Values.node.teku.validators.enabled` value

### Questions
- keys path vs validators keys path (this is mounted read only)
- startup command
- is the normal process bulkload & then start or is this wrapped up in the one subcommand?
- eth2. config docs?
