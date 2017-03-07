---
title: Command-line wrapper for K2 (k2cli)
permalink: k2cli.html
toc: false
author: red
---
k2cli is a command-line interface for K2. It supports Kubernetes deployments to AWS, GKE, and other environments.

Using the k2cli tool you can perform tasks such as:  
- Create a cluster  
- Configure ssh for your cluster  
- List and install Helm charts  
- List nodes in your cluster  
- List installed applications  
- Destroy a running cluster

k2cli command-line options are documented here: [https://github.com/samsung-cnct/k2cli/blob/master/docs/k2cli.md](https://github.com/samsung-cnct/k2cli/blob/master/docs/k2cli.md).

## Installing k2cli

### Requirements
Docker must be installed on the machine where you run k2cli, and your user must have permissions to run it.

### Fetching the official build
The latest official build can be found here: [https://github.com/samsung-cnct/k2cli/releases](https://github.com/samsung-cnct/k2cli/releases). You should use the latest version unless you have a specific reason to use a different version.

### Building a configuration file
K2 uses a yaml configuration file for all aspects of the both the Kubernetes cluster and the infrastructure that is
running it. To build a generic AWS configuration file that has a large number of sensible defaults, you can run:
```
./k2cli generate ~/k2configs/
```
which will create a file at `~/k2configs/config.yaml`. There are three fields that need to be set before this file can
be used:

*  **Cluster name**  
All K2 clusters should have a unique name so their assets can be easily identified by humans in the
AWS console (no more than 13 characters). The cluster name is set in the `deployment.cluster` field.  This dotted notation refers to the hierarchical structure of a yaml file where cluster is a sub field of deployment. This is also the third line in the generated file.

*  **AWS access key**  
Your AWS access key is required for programmatic access to AWS. The field is named
`deployment.providerConfig.authentication.accessKey`. This can be either set to the literal value, or to an environment
variable that K2 will use.

*  **AWS secret key**  
This is your AWS secret key that is paired to the above access key. This field is named
`deployment.providerConfig.authentication.secretKey`. This can be either set to the actual value or a environment
variable and K2 will perform the replacement.This can be either set to the literal value, or to an environment
variable that K2 will use.

**Note**  
The `deployment.providerConfig.authentication.credentialsFile` field is present but is not yet fully implemented.
For more information see: [https://github.com/samsung-cnct/k2/issues/128](https://github.com/samsung-cnct/k2/issues/128)

When the required fields are set your configuration file is ready to go! The default file will create a production-ready
cluster with the following configuration:

Role | # | Type
--- | ---  | ---  
Primary etcd cluster | 5 | t2.small  
Events etcd cluster | 5 | t2.small  
Master nodes | 3 | m3.medium  
Cluster nodes | 3 | c4.large  
Special nodes | 2 | m3.medium

We have chosen this configuration based on our own, and other's, publicly available research. It creates an underpowered cluster
in terms of cluster nodes, but that's an easy setting to change. The important point is to ensure that the control plane is
production quality.

### Creating your first cluster
To create your first cluster, run the following command. (This assumes you have a configuration built as described above.)
```
./k2cli cluster up ~/k2configs/config.yaml
```
This will take anywhere from five to twenty minutes depending on how AWS is feeling when you execute this command. When
complete the cluster will exist in its own VPC and will be accesible via the `tool` subcommands. The output artifacts
will be stored in the default location: `~/.kraken/<cluster name>`.  

### Working with your cluster (using k2cli)
k2cli uses the K2 image (github.com/samsung_cnct/k2) for all of its operations. The K2 image ships with `kubectl` and `helm`
installed and through the `k2cli tool` subcommand you can access these tools. Using the `k2cli tool` subcommand helps ensure you are
using the correct version of the relevant CLI for your cluster.

`kubectl` [(http://kubernetes.io/docs/user-guide/kubectl-overview/)](http://kubernetes.io/docs/user-guide/kubectl-overview/) is a CLI for working with a Kubernetes cluster. It is
used for deploying application, checking system status and more. See the linked documentation for more details.

`helm` [(http://kubernetes.io/docs/user-guide/kubectl-overview/)](http://kubernetes.io/docs/user-guide/kubectl-overview/) is a CLI for deploying and packaging applications to deploy
to Kubernetes. See the linked documentation for more details.

#### Example usage - k2cli tool kubectl
To see all nodes in your Kubernetes cluster
```
./k2cli tool kubectl --config ~/k2configs/config.yaml get nodes
```

To see all installed application across all namespaces
```
./k2cli tool kubectl --config ~/k2configs/config.yaml -- get pods --all-namespaces
```

#### Example usage - k2cli tool helm
To list all installed charts
```
./k2cli tool helm --config ~/k2configs/config.yaml list
```

To install the Kafka chart maintained by Samsung CNCT.
```
./k2cli tool helm --config ~/k2configs/config.yaml install atlas/kafka
```

### Working with your cluster (using host installed tools)
The file that is required for both helm and kubectl to connect to, and interact with, your Kubernetes deployment is saved to your
local machine in the output directory. By default, this directory is `~/.kraken/<cluster name>/`. The filename is `admin.kubeconfig`.

#### Example usage - local kubectl
To list all nodes in your Kubernetes cluster
```
kubectl --kubeconfig=~/.kraken/<cluster name>/admin.kubeconfig --cluster=<cluster name> get nodes
```

To list all installed applications across all namespaces
```
kubectl --kubeconfig=~/.kraken/<cluster name>/admin.kubeconfig --cluster=<cluster name> get pods --all-namespaces
```

#### Example usage - local helm
Helm requires both the admin.kubeconfig file and the saved local helm state. The helm state directory is also in the output
directory

To list all installed charts
```
KUBECONFIG=~/.kraken/<cluster name>/admin.kubeconfig HELM_HOME=~/.kraken/<cluster name>/.helm helm list
```

To install the Kafka chart maintained by Samsung CNCT.
```
KUBECONFIG=~/.kraken/<cluster name>/admin.kubeconfig HELM_HOME=~/.kraken/<cluster name>/.helm helm install atlas/kafka
```

### Destroying the running cluster
While not something to be done in production, during development when you are done with your cluster (or with a quickstart) it's
best to clean up your resources. To destroy the running cluster from this guide, simply run:
```
./k2cli cluster down ~/k2configs/config.yaml
```

**Note:** if you have specified an '--output' directory during the creation command, make sure you specify it here or the cluster
will still be running!

## Note on using environment variables in yaml configuration

k2cli will automatically attempt to expand all ```$VARIABLE_NAME``` strings in your configuration file. It will pass the variable and value to the K2 Docker container, and mount the path (if it's a path to an existing file or folder) into the K2 Docker container.

### Environment variable expansion

For example, given a variable such as ```export MY_SERVICE_ACCOUNT_KEYFILE=/Users/kraken/.ssh/keyfile.json```, the configuration:

```
deployment:
  cluster: production-cluster
  keypair:
    -
      name: key
      publickeyFile:
      privatekeyFile:
      providerConfig:
        username:
        serviceAccount: "serviceaccount@project.iam.gserviceaccount.com"
        serviceAccountKeyFile: "$MY_SERVICE_ACCOUNT_KEYFILE"

...
```

will be expanded to:

```
deployment:
  cluster: production-cluster
  keypair:
    -
      name: key
      publickeyFile:
      privatekeyFile:
      providerConfig:
        username:
        serviceAccount: "serviceaccount@project.iam.gserviceaccount.com"
        serviceAccountKeyFile: "/Users/kraken/.ssh/keyfile.json"

...
```

and the K2 Docker container would get a ```/Users/kraken/.ssh/keyfile.json:/Users/kraken/.ssh/keyfile.json``` mount and a ```K2_SERVICE_ACCOUNT_KEYFILE=/Users/kraken/.ssh/keyfile.json``` environment variable

### Automatic binding of environment variables

Environment variables with a `K2` prefix can also bind automatically to configuration values.

For example, given that ```export K2_DEPLOYMENT_CLUSTER=production-cluster```, the configuration:

```
deployment:
  cluster: changeme
  keypair:
    -
      name: key
      publickeyFile:
      privatekeyFile:
      providerConfig:
        username:
        serviceAccount: "serviceaccount@project.iam.gserviceaccount.com"
        serviceAccountKeyFile: "/Users/kraken/.ssh/keyfile.json"

...
```

will be expanded to:

```
deployment:
  cluster: production-cluster
  keypair:
    -
      name: key
      publickeyFile:
      privatekeyFile:
      providerConfig:
        username:
        serviceAccount: "serviceaccount@project.iam.gserviceaccount.com"
        serviceAccountKeyFile: "/Users/kraken/.ssh/keyfile.json"

...
```

## How to build k2cli
k2cli is a Go project with vendored dependencies so building is a snap. Use the following commands to build k2cli.

```
git clone https://github.com/<your github account>/k2cli.git
cd k2cli
go build
```
