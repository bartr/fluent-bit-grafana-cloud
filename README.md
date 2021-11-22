# Fluent Bit - Grafana Cloud

![License](https://img.shields.io/badge/license-MIT-green.svg)
[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa.svg)](code_of_conduct.md)

This is an end-to-end walkthrough of setting up Fluent Bit for log forwarding from a Kubernetes cluster (k3d) to [Grafana Cloud (Loki)](https://grafana.com/)

The sample application generates JSON logs. Normal logs are written to stdout. Error logs are written to stderr.

This sample will work with other Kubernetes clusters but may require minor changes. This sample assumes a single node cluster for simplicity.

```json

{"date":"2020-12-28T21:19:06.1347849Z","statusCode":200,"path":"/log/app","duration":78,"value":"HWIkixicjA"}
{"date":"2020-12-28T21:19:06.1846021Z","statusCode":400,"path":"/log/app","duration":9,"message":"Invalid paramater: cMwyFA"}
{"date":"2020-12-28T21:19:06.1444807Z","statusCode":500,"path":"/log/app","duration":266,"message":"Server error 9750"}

```

## Prerequisites

- Basic knowledge of `Kubernetes` and `kubectl`
- Bash shell (tested on GitHub Codespaces, Mac, Ubuntu, WSL2)
- `GitHub Codespaces` or `k3d cluster` with `kubectl` access
- An account on `Grafana Cloud` (a [free account](https://grafana.com/get/?plcmt=graf-nav-menu&cta=create-free-account) will work)

## Fork this repo

- Fork this repo

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
  - Make sure to select your fork of this repo
- Click `Add Secret`

## Create a Codespace

- Navigate to your fork of this repo
- Click `Code`
  - Click `New Codespace`
  - Select cores
  - Click `Create Codespace`

> GitHub Codespaces is still in preview for individual users
>
> If you do not have access, you can use your dev workstation

## Set Environment Variables

- If you are not using Codespaces

  ```bash

  export GC_PAT=yourGrafanaCloudPAT

  ```

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

  ```bash

  # uncomment this line if not using Codespaces
  ### Warning: this will store your PAT in clear text
  # echo "export GC_PAT=$GC_PAT" >> ~/.zshrc

  echo "export GC_LOKI_USER=$GC_LOKI_USER" >> ~/.zshrc
  echo "export GC_USER=$GC_USER" >> ~/.zshrc

  cat ~/.zshrc

  ```

## Create k3d Cluster (if required)

```bash

# change to the deploy directory
cd deploy

k3d cluster create --config k3d.yaml --k3s-server-arg "--no-deploy=traefik" --k3s-server-arg "--no-deploy=servicelb"

```

## Create Kubernetes Secrets

```bash

# create namespace if needed
kubectl create namespace log-test

# add Loki secret
### Note: verify the URL on your Grafana Cloud / Loki config page
kubectl create secret generic fluent-bit-secrets -n log-test \
  --from-literal=LokiUrl=https://$GC_LOKI_USER:$GC_PAT@logs-prod-us-central1.grafana.net/loki/api/v1/push

# verify the secrets are set properly (base 64 encoded)
kubectl get secret fluent-bit-secrets -n log-test -o jsonpath='{.data}'

```

## Deploy to Kubernetes

```bash

# apply the fluentbit config
kubectl apply -f fluentbit.yaml

# check fluent-bit pod until running
kubectl get pods -n log-test

# check fluent-bit logs
kubectl logs -n log-test fluent-bit

# run log app - this will generate 5 log entries
kubectl apply -f logapp.yaml

# check logs for 5 lines of output
kubectl logs logapp

# delete logapp
kubectl delete -f logapp.yaml

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

## Deploy LogApp in a loop

```bash

kubectl apply -f run-in-loop.yaml

kubectl logs logapp-loop

```

### Validate logs

- Click `Run Query` and watch the logs get added every 2-3 seconds

## Changes to Fluent Bit

- All changes to forward logs to Grafana Cloud (Loki) are in `deploy/fluentbit.yaml`

- Use grafana/fluent-bit image
- Use LokiUrl secret
- Beginning at line 156

  ```yaml

    containers:
    - name: fluent-bit
      imagePullPolicy: Always
      image: grafana/fluent-bit-plugin-loki:latest
      env:
        - name: LOKI_URL
          valueFrom:
            secretKeyRef:
              name: fluent-bit-secrets
              key: LokiUrl

  ```

- Configure grafana-loki as output
  - Url is the environment variable from the secret
  - Only match logapp*.* logs
  - Add the `job` label
  - `Lift` fields from JSON log body to top level fields to make querying easier / faster
  - Remove the lifted fields from the JSON log
- Beginning at line 59

  ```yaml

  output.conf: |
    [OUTPUT]
        Name              grafana-loki
        Url               ${LOKI_URL}
        Match             kube.var.log.containers.logapp*.*
        Labels            { job="log-app" }
        LabelKeys         date,statusCode,duration,message,stream
        RemoveKeys        date,statusCode,duration,message,stream

  ```

## Cleaning up

```bash

# delete log app
kubectl delete -f run-in-loop.yaml

# delete the log-test namespace
kubectl delete namespace log-test

# verify everything is deleted
kubectl get ns

```

## Contributing

This project welcomes contributions and suggestions and has adopted the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct.html).

For more information see the [Code of Conduct FAQ](https://www.contributor-covenant.org/faq).

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Any use of third-party trademarks or logos are subject to those third-party's policies.
