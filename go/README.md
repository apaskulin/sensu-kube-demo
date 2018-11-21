Tutorial: Monitoring with Sensu Go, Kubernetes, Prometheus, InfluxDB, and Grafana.
------

In this tutorial, we'll deploy a dummy app with Kubernetes and monitor it with Sensu.

The dummy app has three endpoints: `/` returns the local hostname, `/healthz` returns the boolean app health state, and `/metrics` returns app Prometheus metric data.

- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Monitoring an app](#monitoring-an-app)
	- [Create a Sensu pipeline to Slack](#create-a-sensu-pipeline-to-slack)
	- [Create a Sensu service check to monitor the status of the dummy app](#create-a-sensu-service-check-to-monitor-the-status-of-the-dummy-app)
- [Collecting app metrics](#collecting-app-metrics)
	- [Create a Sensu pipeline to InfluxDB](#create-a-sensu-pipeline-to-influxdb)
	- [Create a Sensu service check to collect Prometheus metrics](#create-a-sensu-service-check-to-collect-prometheus-metrics)
	- [Use Grafana to visualize metrics](#use-grafana-to-visualize-metrics)
- [Collecting Kubernetes metrics](#collecting-kubernetes-metrics)
- [Next steps](#next-steps)

## Prerequisites

**1. [Install Docker for Mac (Edge)](https://store.docker.com/editions/community/docker-ce-desktop-mac).**

**2. Enable Kubernetes (in the Docker for Mac preferences).**

<img src="https://github.com/sensu/sensu-kube-demo/raw/master/images/docker-kubernetes.png" width="400">

**3. Clone this repo.**

```
$ git clone git@github.com:sensu/sensu-kube-demo.git

$ cd sensu-kube-demo
```

**4. Deploy the [Kubernetes NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx).**

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

Then use the modified "ingress-nginx" Kubernetes Service definition (works with Docker for Mac).

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
```

**5. Open your `/etc/hosts` file and add the following hostnames.**

```
$ sudo vi /etc/hosts

127.0.0.1       sensu.local webui.sensu.local sensu-enterprise.local dashboard.sensu-enterprise.local
127.0.0.1       influxdb.local grafana.local dummy.local
```

**6. Create Kubernetes Ingress Resources.**

```
$ kubectl create -f go/ingress-nginx/ingress/sensu-go.yaml
```

**7. Deploy kube-state-metrics.**

```
$ git clone git@github.com:kubernetes/kube-state-metrics.git

$ kubectl apply -f kube-state-metrics/kubernetes
```

## Setup

**1. Install the Sensu backend.**

```
$ kubectl create -f go/deploy/sensu-backend.yaml
```

**2. Install two instances of a dummy app behind a load balancer, each with a Sensu agent sidecar.**

```
$ kubectl apply -f go/deploy/dummy.yaml

$ kubectl apply -f go/deploy/dummy.sensu.yaml
```

We can check to see if the dummy app is working using:

```
$ curl -i http://dummy.local
```

A `200` response indicates that the dummy app is working correctly.

**3. Install InfluxDB and Grafana with Sensu agent sidecars.**

Create a Kubernetes ConfigMap for InfluxDB and Grafana configurations

```
$ kubectl create configmap influxdb-config --from-file go/configmaps/influxdb.conf

$ kubectl create configmap grafana-provisioning-datasources --from-file=./go/configmaps/grafana-provisioning-datasources.yaml

$ kubectl create configmap grafana-provisioning-dashboards --from-file=./go/configmaps/grafana-provisioning-dashboards.yaml
```

Deploy InfluxDB and Grafana with Sensu agent sidecars.

```
$ kubectl create -f go/deploy/influxdb.sensu.yaml

$ kubectl create -f go/deploy/grafana.sensu.yaml
```

Let's see the pods we have set up so far.

```
$ kubectl get pods
```

You should see two dummy app pods, a Sensu backend pod, a Grafana pod, and an InfluxDB pod:

```
NAME                             READY     STATUS    RESTARTS   AGE
dummy-6c57b8f868-ft5dz           2/2       Running   0          5d
dummy-6c57b8f868-m24hw           2/2       Running   0          5d
grafana-5b88f8df8d-vgjtm         2/2       Running   0          5d
influxdb-78d64bcfd9-8km56        2/2       Running   0          5d
sensu-backend-79bb558646-trl5q   1/1       Running   0          5d
```

**4. Install sensuctl.**

Jump over to the [sensuctl installation guide](https://docs.sensu.io/sensu-go/5.0/installation/install-sensu/#install-sensuctl), and follow the instructions to install sensuctl on Windows, macOS, or Linux.

**5. Configure sensuctl.**

Before we can start using sensuctl, we'll need to configure it.

```
$ sensuctl configure
? Sensu Backend URL: http://127.0.0.1:8080
? Username: admin
? Password: P@ssw0rd!
? Namespace: default
? Preferred output format: tabular
```

## Monitoring an app

Now that we're logged in to sensuctl, let's take a look at what we're monitoring.

```
$ sensuctl entity list
```

We can see the Sensu agents we have installed on the dummp app, InfluxDB, and Grafana pods with their last seen timestamp.

```
ID               Class    OS                   Subscriptions                           Last Seen            
─────────────────────────── ─────── ─────── ─────────────────────────────────────────── ─────────────────────────────── 
dummy-6c57b8f868-ft5dz      agent   linux   dummy,entity:dummy-6c57b8f868-ft5dz         2018-11-20 18:43:15 -0800 PST  
dummy-6c57b8f868-m24hw      agent   linux   dummy,entity:dummy-6c57b8f868-m24hw         2018-11-20 18:43:15 -0800 PST  
grafana-5b88f8df8d-vgjtm    agent   linux   grafana,entity:grafana-5b88f8df8d-vgjtm     2018-11-20 18:43:14 -0800 PST  
influxdb-78d64bcfd9-8km56   agent   linux   influxdb,entity:influxdb-78d64bcfd9-8km56   2018-11-20 18:43:12 -0800 PST  
```

### Create a Sensu pipeline to Slack
Let’s say we want to receive a Slack alert if the dummy app returns an unhealthy response.
We can create a Sensu pipeline to send alerts to a #demo channel in Slack using the [Sensu Slack plugin](https://github.com/sensu/sensu-slack-handler).
Sensu Plugins are open-source collections of Sensu building blocks shared by the Sensu Community.

**1. To allow the Sensu agents to use this plugin, we'll use sensuctl to create a Sensu asset.**

```
$ sensuctl create -f go/config/assets/slack-handler.json
```

**2. Get your Slack webhook URL and add it to `go/config/handlers/slack.json`.**

If you're already an admin of a Slack, visit `https://YOUR WORKSPACE NAME HERE.slack.com/services/new/incoming-webhook` and follow the steps to add the Incoming WebHooks integration and save the settings.
(If you're not yet a Slack admin, start [here](https://slack.com/get-started#create) to create a new workspace.)
After saving, you'll see your webhook URL under Integration Settings.

Open `go/config/handlers/slack.json` and replace `SECRET` in the following line with your Slack workspace webhook URL:

```
"command": "slack-handler --channel '#demo' --timeout 20 --username 'sensu' --webhook-url 'SECRET'",
```

So it looks something like:

```
"command": "slack-handler --channel '#demo' --timeout 20 --username 'sensu' --webhook-url 'https://hooks.slack.com/services/XXXXXXXXXXXXXXXX'",
```

**3. Create a handler to send alert events to Slack that uses the `slack-handler` asset.**

```
$ sensuctl create -f go/config/handlers/slack.json
```

### Create a Sensu service check to monitor the status of the dummy app

To automatically monitor the status of the dummy app, we'll create an asset that lets the Sensu agents use a [Sensu HTTP plugin](https://github.com/portertech/sensu-plugins-go).

**1. Create the `check-plugins` asset.**

```
$ sensuctl create -f go/config/assets/check-plugins.json
```

**2. Now we can create a check to monitor the status of the dummy app that uses the `check-plugins` asset and the Slack pipeline.**

```
$ sensuctl create -f go/config/checks/dummy-app-health.json
```

**3. With the automated alert workflow in place, we can see the resulting events in the Sensu dashboard.**

At http://webui.sensu.local/signin, sign in with your sensuctl username and password (`admin` and `P@ssw0rd!`).

**4. Toggle the health of the dummy app to simulate a failure.**

```
$ curl -iXPOST http://dummy.local/healthz
```

We should now be able to see a critical alert in the [Sensu dashboard](http://webui.sensu.local/events) as well as by using sensuctl:

```
$ sensuctl event list
```

You should also see an alert in Slack.

Post to `/healthz` again to clear the alert and return all Sensu entities to a healthy state.

```
$ curl -iXPOST http://dummy.local/healthz
```

## Collecting app metrics

### Create a Sensu pipeline to InfluxDB

We can use Sensu to create automated workflows to monitor the dummy app.
The dummy app generates prometheus metrics, so let's create a Sensu pipeline to send those metrics to InfluxDB.
To send metrics to InfluxDB, we'll use the [Sensu InfluxDB plugin](https://github.com/sensu/sensu-influxdb-handler).

**1. To allow the Sensu agents to use this plugin, we'll use sensuctl to create a Sensu asset.**

```
$ sensuctl create -f go/config/assets/influxdb-handler.json
```

**2. Now we can create a handler to send metric events to InfluxDB that uses the `influxdb-handler` asset.**

```
$ sensuctl create -f go/config/handlers/influxdb.json
```

### Create a Sensu service check to collect Prometheus metrics

To automatically collect the Prometheus metrics from the dummy app, we'll create an asset that lets the Sensu agents use the [Sensu Prometheus plugin](https://github.com/sensu/sensu-prometheus-collector).

**1. Create the `prometheus-collector` asset.**

```
$ sensuctl create -f go/config/assets/prometheus-collector.json
```

**2. Now we can create a check to collect Prometheus metrics that uses the `prometheus-collector` asset.**

```
$ sensuctl create -f go/config/checks/dummy-app-prometheus.json
```

### Use Grafana to visualize metrics

To see the metrics we're collecting from the dummy app, log into [Grafana](http://grafana.local/login) with the username `admin` and password `password`.

Create a new dashboard using the InfluxDB datasource to live metrics from the dummy app.

## Collecting Kubernetes metrics
Now that we have a pipeline set up to send metrics, we can easily create a check that collects Prometheus metrics from Kubernetes and connect it to the pipeline.

Create a check to collect Prometheus metrics from Kubernetes that uses the `prometheus-collector` asset and the `influxdb` handler.

```
$ sensuctl create -f go/config/checks/kube-state-prometheus.json
```

You should now be able to access Kubernetes metric data in [Grafana](http://grafana.local) and see metric events in the [Sensu dashboard](http://webui.sensu.local/events).

## Next steps

For more information about monitoring with Sensu, check out the following resources:

- [Reducing alert fatigue with Sensu filters](https://docs.sensu.io/sensu-go/latest/guides/reduce-alert-fatigue/)
- [Aggregating StatD metrics with Sensu](https://docs.sensu.io/sensu-go/latest/guides/aggregate-metrics-statsd/)
- [Aggregating Nagios metrics with Sensu](https://docs.sensu.io/sensu-go/latest/guides/extract-metrics-with-checks/)
