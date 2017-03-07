---
title: K2 Helm Charts
permalink: k2-helm-charts.html
toc: false
author: red
---
CNCT creates [Helm charts](https://helm.sh/) that can be useful for deploying a K2 cluster with the functionality your application requires.

These charts are automatically linted and published with Jenkins CI to the CNCT Helm repo: [atlas.cnct.io](http://atlas.cnct.io/). Source for these charts is available on GitHub at https://github.com/samsung-cnct/k2-charts.

To install a chart with a K2 template yaml:

```
clusterServices:
  repos:
    -
      name: atlas
      url: http://atlas.cnct.io
  services:
    -
      name: kafka
      repo: atlas
      chart: kafka
      version: 0.1.0
```

To use Helm locally in you cluster to install charts run a command similar to:
```
docker run -v ~/:/root -it --rm=true -e HELM_HOME=/root/.kraken/<yourcluster>/.helm -e KUBECONFIG=/root/.kraken/<yourcluster>/admin.kubeconfig quay.io/samsung_cnct/k2:latest helm list
```
To use Helm with a locally installed kubectl:
```
export KUBECONFIG=~/.kraken/<yourcluster>/admin.kubeconfig
helm list --home ~/.kraken/<yourcluster>/.helm
```
