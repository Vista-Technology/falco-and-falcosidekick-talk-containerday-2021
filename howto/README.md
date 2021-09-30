# How to reproduce the demo

## Prerequisites
A simple Kubernetes cluster (at least v1.20), that's it! We will use Helm3 (for components installation) and kubectl. A NetworkPolicy-compatible CNI is required.

For example we have used a Minikube environment in a single Ubuntu VM. If you want to match exactly our steps use the following instructions.

```shell
$ sudo apt-get -y install build-essential linux-headers-$(uname -r) virtualbox docker.io
$ sudo systemctl enable --now docker
$ sudo usermod -aG docker ubuntu

$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube

$ curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
$ sudo apt-get install apt-transport-https --yes
$ echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
$ sudo apt-get update
$ sudo apt-get install helm

$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

$ minikube start --cpus=4 --memory=16384 --kubernetes-version=1.20.10 --driver=virtualbox --network-plugin=cni --cni=cilium
```

## Components installation
### Kubeless
```shell
$ export RELEASE=$(curl -s https://api.github.com/repos/kubeless/kubeless/releases/latest | grep tag_name | cut -d '"' -f 4)
$ kubectl create ns kubeless
$ kubectl create -f https://github.com/kubeless/kubeless/releases/download/$RELEASE/kubeless-$RELEASE.yaml
```
### Monitoring stack (Prometheus/Loki/Grafana)
```shell
$ kubectl create ns monitoring

$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm install prometheus prometheus-community/prometheus -n monitoring -f howto/falcosidekickExtraScrapeConfigs.yaml

$ helm repo add loki https://grafana.github.io/loki/charts
$ helm repo update
$ helm install grafana grafana/grafana -n monitoring

$ helm repo add loki https://grafana.github.io/loki/charts
$ helm repo update
$ helm install loki grafana/loki-stack -n monitoring
```

#### Configure and expose Grafana service
Get the default admin password
```shell
$ kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Expose the service in port forwarding
```shell
$ export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
$ kubectl --namespace monitoring port-forward --address 0.0.0.0 $POD_NAME 3000
```
You have to configure Grafana Datasource as follows:
 - LOKI: http://loki:3100
 - PROMETHEUS: http://prometheus-server:80

We have used a simple pre-built Grafana Dashboard, you can import it from https://grafana.com/grafana/dashboards/11914. You can use it to take some inspiration.

## Falco installation
```shell
$ kubectl create ns falco

$ helm repo add falcosecurity https://falcosecurity.github.io/charts
$ helm repo update

$ helm install falco falcosecurity/falco --set falco.jsonOutput=true --set falco.httpOutput.enabled=true --set falco.httpOutput.url=http://falcosidekick:2801 -f howto/falcoCustomRule.yaml -n falco
```
We have passed an external YAML file (with -f argument) to add our Custom Rule definition to the installation. This is the way we tell Falco to consider also our rule in the evaluation process.

We have also indicated the falcosidekick endpoint URL (we will install it right now)

## Falcosidekick installation
```shell
$ helm install falcosidekick falcosecurity/falcosidekick --set config.kubeless.namespace=kubeless --set config.kubeless.function=isolate-pod --set config.loki.hostport=http://loki.monitoring:3100 --set config.customfields="source:falco" --set webui.enabled=true --set config.slack.webhookurl=<INSERT HERE YOUR SLACK WEBHOOK ULR> -n falco
```
Let's analyze the arguments:
 - *config.kubeless.namespace* and *config.kubeless.function*: set the data to indicates the namespace where is deployed and the function name of serverless part
 - *config.loki.hostport*: set the Loki URL endpoint
 - *config.customfields*: add custom field to Loki log
 - *config.slack.webhookurl*: HTTP Slack webhook URL to POST to

## Kubeless function deployment
```shell
$ kubectl -n kubeless apply -f howto/kubelessFunctionDeploy.yaml
```

## Demo app deployment
We will deploy a Nginx POD inside a demo namespace
```shell
$ kubectl create -f howto/demoMaliciousAppDeployment.yaml
```

## BONUS: Falco Event Generator
If you want to generate some Falco random events please use this fantastic project: https://github.com/falcosecurity/event-generator.

We have use this repo to create some events to fill our monitoring dashboard.
```shell
$ git clone https://github.com/falcosecurity/event-generator.git
$ cd event-generator
$ kubectl apply -f deployment/run-as-job.yaml
```
