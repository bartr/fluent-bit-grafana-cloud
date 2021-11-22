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

- Knowledge of `Kubernetes` and `kubectl`
- Bash shell (tested on GitHub Codespaces, Mac, Ubuntu, WSL2)
- `GitHub Codespaces` or `k3d cluster` with `kubectl` access
- An account on `Grafana Cloud` (a [free account](https://grafana.com/get/?plcmt=graf-nav-menu&cta=create-free-account) will work)

## Fork this repo

- Fork this repo and open with `Codespaces` or clone the repo to your development machine

## Create k3d Cluster (if required)

```bash

# create the cluster and wait for ready
# default cluster name is k3d

k3d cluster create --registry-use k3d-registry.localhost:5500 --config k3d.yaml --k3s-server-arg "--no-deploy=traefik" --k3s-server-arg "--no-deploy=servicelb"

# wait for cluster to be ready
kubectl wait node --for condition=ready --all --timeout=60s

```

## Create Grafana Cloud Account

- Go to <https://grafana.com> and create a free account
- Click on `My Account`
  - You will get redirected to this URL <https://grafana.com/orgs/yourAccountNameHere>
- In the left nav bar, click on `API Keys` (under Security)
- Click on `+ Add API Key`
  - Name your API Key (i.e. yourName-publisher)
  - Select `MetricsPublisher` as the role
  - Click on `Create API Key`
  - Click on `Copy to Clipboard` and save wherever you save your PATs
    - WARNING - you will not be able to get back to this value!!!

## Add your PAT to Codespaces

- Open this link in a new browser tab <https://github.com/settings/codespaces>
- Click `New Secret`
- Enter `GC_PAT` as the name
- Paste the Grafana Cloud PAT you just created in the value
- Click on `Select repositories`
  - Select any repos you want to load this secret
- Click `Add Secret`

## Create a Codespace

- Navigate to your fork of this repo
- Click `Code`
  - Click `New Codespace`
  - Select cores
  - Click `Create Codespace`

## Set Environment Variables

- Export your Grafana Cloud user name

  ```bash

  export GC_USER=yourUserName

  ```

- Export Loki Tenant ID
  - From the `Grafana Cloud Portal`
    - <https://grafana.com/orgs/yourUser>
  - Click `Details` in the `Loki` section
    - Copy your `User` value
    - Export the value

    ```bash

    export GC_LOKI_USER=pasteValue

    ```

- Verify GC_* env vars are set

  ```bash

  env | grep GC_

  ```

  - Ouput should look like this

  ```text

  GC_PAT=xxxxxxxxxxxxxx==
  GC_LOKI_USER=######
  GC_USER=bartr

  ```

- Save values for future use (optional)

> Note this does not save GC_PAT for security reasons

  ```bash

  echo "export GC_LOKI_USER=$GC_LOKI_USER" >> ~/.zshrc
  echo "export GC_USER=$GC_USER" >> ~/.zshrc

  cat ~/.zshrc

  ```

## Create Kubernetes Secrets

```bash

# create namespace if needed
kubectl create namespace log-test

# delete secrets (if exist)
#    you can safely ignore a not found error
kubectl delete secret fluent-bit-secrets -n log-test

# add Log Analytics secret
kubectl create secret generic fluent-bit-secrets -n log-test \
  --from-literal=LokiUrl=https://$GC_LOKI_USER:$GC_PAT@logs-prod-us-central1.grafana.net/loki/api/v1/push

# verify the secrets are set properly (base 64 encoded)
kubectl get secret fluent-bit-secrets -n log-test -o jsonpath='{.data}'

```

## Deploy to Kubernetes

```bash

# apply the fluentbit config
kubectl apply -f config.yaml

# check fluent-bit pod until running
kubectl get pods -n log-test

# check fluent-bit logs
kubectl logs -n log-test fluent-bit

# run log app - this will generate 5 log entries
kubectl apply -f logapp.yaml

# check logapp pod
# logapp will run and then exit with Completed state
kubectl get pods

# check logs
kubectl logs logapp

# generate more logs
kubectl delete -f logapp.yaml
kubectl apply -f logapp.yaml

# check Grafana Cloud for your data
# this should only take a few seconds

# delete logapp
kubectl delete -f logapp.yaml

# check pod
kubectl get pods -n log-test

# Result - fluent-bit pod is still running

```

## Validate Logs

- Open your Grafana Cloud dashboard
  - Make sure to replace yourUser
    - <https://yourUser.grafana.net>
- Select the `Explore` tab from the left navigation menu
- Select Logs data from the `Explore` drop down at top left of panel
- Enter `{ job = "log-app" }` in the `Loki Query`
- Click `Run Query` or press `ctl + enter`
- Other queries
  - `{ job="log-app", stream="stdout" }`
  - `{ job="log-app", stream="stderr" }`
  - `{ job="log-app" } | statusCode=200`
  - `{ job="log-app" } | statusCode=400`
  - `{ job="log-app" } | statusCode=500`
  - `{ job="log-app" } | duration > 20`

### Cleaning up

```bash

kubectl delete namespace log-test

# verify everything is deleted
#    Error from server (NotFound)
kubectl get ns log-test

```

## Contributing

This project welcomes contributions and suggestions and has adopted the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct.html).

For more information see the [Code of Conduct FAQ](https://www.contributor-covenant.org/faq).

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Any use of third-party trademarks or logos are subject to those third-party's policies.
