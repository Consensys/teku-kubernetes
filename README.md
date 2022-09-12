

# Teku-Kubernetes (k8s)

The following repo has example reference implementations of teku (as consensus layer) and besu/geth (as execution layer) using k8s. You will need the following tools to proceed:

- [Kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/)
- [Helm Diff plugin](https://github.com/databus23/helm-diff)

Verify kubectl is connected with (please use the latest version of kubectl)
```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.1", GitCommit:"4485c6f18cee9a5d3c3b4e523bd27972b1b53892", GitTreeState:"clean", BuildDate:"2019-07-18T09:18:22Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:32:14Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

Install helm & helm-diff:
Please note that the documentation and steps listed use *helm3*. The API has been updated so please take that into account if using an older version
```bash
$ helm plugin install https://github.com/databus23/helm-diff --version master
```

The current repo layout is:

```bash
  ├── ingress                       # ingress rules, hidden here for brevity
  │   └── ...                       
  ├── static                        # static assets
  ├── aws                           # aws specific artifacts
  │   ├── templates                 # aws templates to deploy resources ie cluster, secrets manager, IAM etc
  ├── azure                         # azure specific artifacts
  │   ├── arm                       # azure ARM templates to deploy resources ie cluster, keyvault, identity etc
  │   └── scripts                   # azure scripts to install CSI drivers on the AKS cluster and the like
  ├── helm                       
  │   ├── charts            
  │   │   ├── ...                   # helm charts, hidden here for brevity
  │   └── values            
  │       ├── ...                   # values.yml overrides for various node types

```

It is recommended you follow the approach of an override `values.yml` for your deployments and leave the default values.yml in the chart as is. The helm chart **will** use a load balancer for discovery, pleae set the `cluster` map with what features and the env you're deploying to:
```bash
cluster:
  provider: azure  # choose from: aws | azure
  cloudNativeServices: false # set to true to use Cloud Native Services (SecretsManager and IAM for AWS; KeyVault & Managed Identities for Azure)

```
Setting the `cluster.cloudNativeServices: true` will:
- store validator keys in KeyVault or Secrets Manager 
- make use of Managed Identities or IAMs for access

## Concepts:

#### Providers
If you are deploying to cloud, we support AWS and Azure at present. Please refer to the [Azure deployment documentation](./azure/README.md) or the [AWS deployment documentation](./aws/README.md).

| ⚠️ **Note**: As of writing this (12/9/22), only Azure works. We are awaiting AWS upgrading to K8S ver 1.24 - this allows loadbalancers to use TCP & UDP on the same port       |
| ---                                                                                                                                                                                                                                                                                                                                                                                |

#### Namespaces:
Currently we do **not** deploy anything in the `default` namespace and instead use the `teku` namespace. You can change this to suit your requirements
Namespaces are part of the setup and do not need to be created via kubectl prior to deploying. 

#### Data Volumes:
We use seperate data volumes to store the blockchain data, over the default of the host nodes. This is similar to using seperate volumes to store data when using docker containers natively or via docker-compose. This is done for a couple of reasons; firstly, containers are mortal and we don't want to store data on them, secondly, host nodes can fail and we would like the chain data to persist.  

Please ensure that you provide enough capacity for data storage for all nodes that are going to be on the cluster. Select the appropriate [type](https://kubernetes.io/docs/concepts/storage/volumes/) of persitent volume based on your cloud provider. In the templates, the size of the claims has been set small. If you have a differnt storage account than the one in the charts, please set that up in the storageClass. We recommend you grow the volume claim as required (this also lowers cost)

#### ELC and CLC configuration:
Configuration of client nodes can be done either via a single item inside a config map, as Environment Variables or as command line options. Please refer to the Configuration section of our documentation. 

#### RBAC:
We encourage the use of RBAC's for access to resources, ie. only a specific pod/statefulset is allowed to access a specific secret. If you need to specify a Kube config file to each pod please use the `KUBE_CONFIG_PATH` variable

#### Monitoring
As always please ensure you have sufficient monitoring and alerting setup.

Besu/Geth & Teku publish metrics to [Prometheus](https://prometheus.io/) and metrics can be configured using the [kubernetes scraper config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config).

All clients also have a custom Grafana [dashboards](https://grafana.com/orgs/consensys) to make monitoring of the nodes easier.

For ease of use, the kubectl & helm examples included have both installed and included as part of the setup. Please configure the kubernetes scraper and grafana security to suit your requirements, grafana supports multiple options that can be configured using env vars

#### Ingress Controllers:

If you require the use of ingress controllers for the RPC calls or the monitoring dashboards, we have provided [examples](./ingress) with rules that are configured to do so.

Please use these as a reference and develop solutions to match your network topology and requirements.

#### Logging
Node logs can be [configured](https://besu.hyperledger.org/en/latest/HowTo/Troubleshoot/Logging/#advanced-custom-logging) to suit your environment. For example, if you would like to log to file and then have parsed via logstash into an ELK cluster, please use the Elastic charts as well


