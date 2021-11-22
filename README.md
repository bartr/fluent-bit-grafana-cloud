# Fluent Bit - Azure Log Analytics

![License](https://img.shields.io/badge/license-MIT-green.svg)
[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa.svg)](code_of_conduct.md)


This is an end-to-end walkthrough of setting up Fluent Bit for log forwarding from a Kubernetes cluster to [Grafana Cloud (Loki)](https://grafana.com/)

The sample application generates JSON logs. Normal logs are written to stdout. Error logs are written to stderr.

```json

{"date":"2020-12-28T21:19:06.1347849Z","statusCode":200,"path":"/log/app","duration":78,"value":"HWIkixicjA"}
{"date":"2020-12-28T21:19:06.1846021Z","statusCode":400,"path":"/log/app","duration":9,"message":"Invalid paramater: cMwyFA"}
{"date":"2020-12-28T21:19:06.1444807Z","statusCode":500,"path":"/log/app","duration":266,"message":"Server error 9750"}

```

## Prerequisites

- Knowledge of `Azure`, `Kubernetes`, `Fluent Bit` and `kubectl`
- Bash shell (tested on GitHub Codespaces, Mac, Ubuntu, WSL2)
- Kubernetes cluster
- kubectl with access to the Kubernetes cluster
- An account on `Grafana Cloud` (a [free account](https://grafana.com/get/?plcmt=graf-nav-menu&cta=create-free-account) will work)

## Clone this repo

```bash

git clone https://github.com/bartr/fluent-bit-grafana-cloud
cd fluent-bit-grafana-cloud/fluentbit

```

## Create Kubernetes Dependencies

> For safety, we use the `log-test` namespace
>
> Most deployments use the `logging` namespace by convention

```bash

# create namespace, service account, cluster role, cluster role binding
kubectl apply -f account.yaml

```

## Create Kubernetes Secrets

```bash

# delete secrets (if exist)
#    you can safely ignore a not found error
kubectl delete secret fluentbit-secrets -n log-test

# add Log Analytics secrets
kubectl create secret generic fluentbit-secrets -n log-test \
  --from-literal=WorkspaceId=$(az monitor log-analytics workspace show -g $LogAppRG -n $LogAppName --query customerId -o tsv) \
  --from-literal=SharedKey=$(az monitor log-analytics workspace get-shared-keys -g $LogAppRG -n $LogAppName --query primarySharedKey -o tsv)

# verify the secrets are set properly (base 64 encoded)
kubectl get secret fluentbit-secrets -n log-test -o jsonpath='{.data}'

```

## Deploy to Kubernetes

### Update Config (if required)

> The config works with `containerd` or `cri-o` runtimes
>
> config.yaml must be changed to work with `dockerd` or `dockershim`
>
> More details [here](https://github.com/microsoft/fluentbit-containerd-cri-o-json-log)

Check the Kubernetes runtime

```bash

kubectl describe node | grep "Container Runtime Version:"

```

If the result shows `dockerd` or `dockershim` edit config.yaml

- Replace
  - config.yaml
    - input-kubernetes.conf
      - `Parser  cri`
- with
  - `Parser docker`

```bash

# apply the fluentbit config
kubectl apply -f config.yaml

# start fluentbit daemonset
kubectl apply -f fluentbit-daemonset.yaml

# check daemonset until fluent-bit is running
kubectl get daemonset -n log-test

# check fluent-bit logs
kubectl logs -l k8s-app=fluent-bit-logging -n log-test

# run log app - this will generate 5 log entries
kubectl apply -f logapp.yaml

# check logapp pod
# logapp will run and then exit with Completed state
kubectl get pods

# check logs
kubectl logs logapp

# check fluent-bit logs
kubectl logs -l k8s-app=fluent-bit-logging -n log-test

# looking for a line like:
#   [2021/02/02 21:54:19] [ info] [output:azure:azure.0]

# check Log Analytics for your data on the Azure portal
# this can take 10-15 minutes initially

# generate more logs
kubectl delete -f logapp.yaml
kubectl apply -f logapp.yaml

# check Grafana Cloud for your data
# this should only take a few seconds

# delete logapp
kubectl delete -f logapp.yaml

# check daemonset
kubectl get daemonset -n log-test

# Result - fluent-bit daemonset is still running

```

### Cleaning up

```bash

kubectl delete secret fluentbit-secrets -n log-test
kubectl delete -f fluentbit-daemonset.yaml
kubectl delete -f config.yaml
kubectl delete -f account.yaml

# verify everything is deleted
#    Error from server (NotFound)
kubectl get ns log-test

```

## Contributing

This project welcomes contributions and suggestions and has adopted the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct.html).

For more information see the [Code of Conduct FAQ](https://www.contributor-covenant.org/faq).

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Any use of third-party trademarks or logos are subject to those third-party's policies.
