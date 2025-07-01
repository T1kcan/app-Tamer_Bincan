
# Grafana Stack Installation:
```bash
kubectl create namespace meta && kubectl create namespace prod
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
helm repo update
git clone https://github.com/T1kcan/alloy-scenarios.git
cd alloy-scenarios/k8s/logs
helm install --values killercoda/loki-values.yml loki grafana/loki -n meta
helm install --values grafana-values.yml grafana grafana/grafana --namespace meta
helm install --values ./k8s-monitoring-values.yml k8s grafana/k8s-monitoring -n meta
kubectl get pod -n meta -w
```
Once all pods are up and running visit grafana UI by following command:
```bash
export POD_NAME=$(kubectl get pods --namespace meta -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}") && kubectl --namespace meta port-forward $POD_NAME 3000 --address 0.0.0.0
```
If you happen to visit alloy metrics ui port-forward alloy pod and visit ui:
```bash
# export POD_NAME=$(kubectl get pods --namespace meta -l "app.kubernetes.io/name=alloy-logs,app.kubernetes.io/instance=k8s" -o jsonpath="{.items[0].metadata.name}") && kubectl --namespace meta port-forward $POD_NAME 12345 --address 0.0.0.0
kubectl get pod -n meta
kubectl --namespace meta port-forward k8s-alloy-logs-4zg2p 12345 --address 0.0.0.0
```


Keda:
- server-pod.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
spec:
  selector:
    matchLabels:
      run: server
  replicas: 1
  template:
    metadata:
      labels:
        run: server
    spec:
      containers:
        - name: server
          image: iosifkoen/keda_server_tutorial:latest
          ports:
            - containerPort: 8080
              protocol: TCP
```
```bash
kubectl apply -f server-pod.yaml
```

```bash
kubectl expose pod server xxx --name=exposed-server-pod --port=8080 --type=NodePort
kubectl expose pod server-5458d48554-7wc74 --name=exposed-server-pod --port=8080 --type=NodePort
kubectl get svc
server-654c779b59-tbcfq   NodePort    10.97.207.159   <none>        8080:31860/TCP   5s
```
Deploy Keda:
https://keda.sh/docs/2.4/deploy/
```bash
kubectl create namespace keda
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace

kubectl get pod -l app=keda-operator -n keda
kubectl logs pod/<The name you have copied> -n keda
kubectl logs pod/keda-operator-96579d64c-mmwpb -n keda
```
Config our Prometheus 
- prometheus.yaml
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'flask_server'
    static_configs:
      - targets: ['<server-pod-ip>:<node-port>']
    metrics_path: '/metrics'
    honor_labels: true
```
```bash
sudo apt install net-tools
docker run -d --name prometheus -p 9090:9090 -v $(pwd)/prometheus.yaml:/etc/prometheus/prometheus.yml prom/prometheus
```

- keda-scaledobject.yaml
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: flask-server-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: server
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://<prometheus-host-ip>:9090
      metricName: flask_server_request_rate
      query: flask_http_request_total
      threshold: '10'
```
```bash
kubectl apply -f keda-scaledobject.yaml
kubectl get scaledobjects
```

- k6 Testing Tool Configuration
```bash
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

- k6-testing-script.js 
```yaml
import http from 'k6/http';
import { sleep } from 'k6';

export let options = {
  vus: 16,
  duration: '300s',
};

export default function () {
  http.get('http://<your-service-url>/get_url'); // replace with your service URL.
  sleep(1);
}
```
```bash
k6 run k6-testing-script.js
```
Observe metrics on Prometheus:9090

Query: flask_http_request_total

Graph

## Grafana Alerting


## Create a contact point

Use `Webhook.site` to quickly set up that test endpoint. This way we can make sure that our alert is actually sending a notification somewhere.

In your browser, sign in to your Grafana Cloud account.

To log in, navigate to http://localhost:3000, where Grafana is running.

In another tab, go to `Webhook.site`.

Copy Your unique URL.

Your webhook endpoint is now waiting for the first request.

Next, let's configure a contact point in Grafana's Alerting UI to send notifications to our webhook endpoint.

Return to Grafana. In Grafana's sidebar, hover over the Alerting (bell) icon and then click Contact points.

Click + Create contact point.

In Name, write Webhook.

In Integration, choose Webhook.

In URL, paste the endpoint to your webhook endpoint.

Click Test, and then click Send test notification to send a test alert to your webhook endpoint.

Navigate back to Webhook.site. On the left side, there's now a POST / entry. Click it to see what information Grafana sent.

Return to Grafana and click Save contact point.

We have created a dummy Webhook endpoint and created a new Alerting contact point in Grafana. Now, we can create an alert rule and link it to this new integration.

## Create an alert

1. In Grafana, navigate to Alerts & IRM > Alerting > Alert rules. Click on + New alert rule.

2. Enter alert rule name for your alert rule. Make it short and descriptive as this appears in your alert notification. For instance, database-metrics

### Define query and alert condition

In this section, we use the default options for Grafana-managed alert rule creation. The default options let us define the query, a expression (used to manipulate the data -- the WHEN field in the UI), and the condition that must be met for the alert to be triggered (in default mode is the threshold).

Grafana includes a test data source that creates simulated time series data. This data source is included in the demo environment for this tutorial. If you're working in Grafana Cloud or your own local Grafana instance, you can add the data source through the Connections menu.

Select the `Prometheus` data source from the drop-down menu.

In the Alert condition section:

Select `flask_http_request_total`

Keep Last as the value for the reducer function (WHEN ), and IS ABOVE 10 as the threshold value. This is the value above which the alert rule should trigger.
Click Preview alert rule condition to run the query.

It should return random time series data. The alert rule state should be `Firing` .


### Add folders and labels

In Folder, click + New folder and enter a name. For example: `metric-alerts` . This folder contains our alert rules.

### Set evaluation behavior

The alert rule evaluation defines the conditions under which an alert rule triggers, based on the following settings:

- Evaluation group: every alert rule is assigned to an evaluation group. You can assign the alert rule to an existing evaluation group or create a new one.

- Evaluation interval: determines how frequently the alert rule is checked. For instance, the evaluation may occur every 10s, 30s, 1m, 10m, etc.
Pending period: how long the condition must be met to trigger the alert rule.

- Keep firing for: defines how long an alert should remain in the Firing state after the alert condition stops being true. During this time, the alert enters a Recovering state, suppressing additional notifications but keeping the alert active. It helps prevent alert flapping, where alerts rapidly switch between firing and resolved due to noisy or unstable metrics.

To set up the evaluation:

1. In the Evaluation group and interval, enter a name. For example: `1m-evaluation` .

2. Choose an Evaluation interval (how often the alert are evaluated). For example, every `1m` (1 minute).

3. Set the pending period to, `0s` (zero seconds), so the alert rule fires the moment the condition is met.

4. Set Keep firing for to, `0s` , so the alert stops firing immediately after the condition is no longer true. Use this when you want alerts to be resolved as soon as the system is healthy again.

## Configure notifications

Choose the contact point where you want to receive your alert notifications.

1. Under Contact point, select Webhook from the drop-down menu.

2. Click Save rule and exit at the top right corner.

---

## Trigger and resolve an alert

Now that the alert rule has been configured, you should receive alert notifications in the contact point whenever alerts trigger and get resolved.

## Trigger an alert

Since the alert rule that you have created has been configured to always fire, once the evaluation interval has concluded, you should receive an alert notification in the Webhook endpoint.


The alert notification details show that the alert rule state is Firing , and it includes the value that made the rule trigger by exceeding the threshold of the alert rule condition. The notification also includes links to see the alert rule details, and another link to add a Silence to it.

---

## Resolve an alert

To see how a resolved alert notification looks like, you can modify the current alert rule threshold.

To edit the Alert rule:

1. Navigate to Alerting > Alert rules.

2. Click on the metric-alerts folder to display the alert that you created earlier

3. Click the edit button on the right hand side of the screen

4. Increment the Keep firing for Threshold expression to 1.

5. Click Save rule and exit.

By incrementing the threshold, the condition is no longer met, and after the evaluation interval has concluded (1 minute approx.), you should receive an alert notification with status “Resolved”.

---

## Step 1: Create a visualization to monitor metrics
To keep track of these metrics you can set up a visualization for CPU usage and memory consumption. This will make it easier to see how the system is performing.

The time-series visualization supports alert rules to provide more context in the form of annotations and alert rule state. Follow these steps to create a visualization to monitor the application’s metrics.

1. Log in to Grafana:

* Navigate to http://localhost:3000, where Grafana should be running.
* Username and password: admin

2. Create a time series panel:

* Navigate to Dashboards.
* Click + Create dashboard.
* Click + Add visualization.
* Select Prometheus as the data source (provided with the demo).
* Enter a title for your panel, e.g., CPU and Memory Usage.
3. Add queries for metrics:

Select `flask_http_request_total`
```
* Click Run queries.

This query should display the simulated CPU usage data for the prod environment.

4. Add memory usage query:

* Click + Add query.

* In the query area, paste the following PromQL query:
```text
Select `flask_http_request_total`
```

5. Click Save dashboard. Name it: flask-request-metrics .
