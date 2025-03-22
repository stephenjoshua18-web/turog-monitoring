Kubernetes Logging Stack with Grafana, Loki & Promtail on AWS
This project was done to set up a logging and monitoring system using Kubernetes. The task was to run Grafana, Loki , and Promtail, ingest logs from any tool on kubernetes node, and create a custom dashboard on Grafana.
promtail runs as a daemonset on node, so it sends all pods logs running on nodes to loki gateway.


What I Used:
AWS EC2 Instance – t3.medium (2 vCPU, 4GB RAM)
Minikube – to run Kubernetes locally on the EC2 instance
Docker – for container runtime
Helm – to install Loki, Promtail, and Grafana
Ubuntu – as the base OS for EC2


Steps I Took
1. Set up EC2 & Minikube
Launched a t3.medium EC2 instance
Installed Docker and Minikube
Started Minikube with enough resources for the stack
2. Installed Helm Charts
I added the Grafana Helm repo and updated it:


helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

Loki Setup
Loki was installed with custom values:


helm show values grafana/loki > loki.yaml  I tweaked the values to match my system resources
helm install loki grafana/loki -n loki --create-namespace -f loki.yaml
Loki Components that got deployed:

loki: Core pod for ingestion, distribution, and storage
loki-canary: Health check pod that sends test logs
loki-chunks-cache: Caches recent logs for faster queries
loki-gateway: Acts as an NGINX ingress
loki-results-cache: Speeds up repeat queries


Promtail Setup
Used Promtail to send logs from all pods running on kubernetes cluster to Loki:
since loki is deployed as a daemon set, it scarpes config from /var/log/containers/*.log


helm show values grafana/promtail > promtail.yaml
# Updated the push URL to http://loki-gateway/loki/api/v1/push since loki and promtail are deployed on same namespace
helm install promtail grafana/promtail -n loki -f promtail.yaml


Grafana Setup
Installed Grafana using Helm in a separate namespace:


helm install grafana grafana/grafana -n grafana --create-namespace -f grafana.yaml
Grafana Access:
Default username: admin
Password (retrieve it using):

kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode
Since persistence wasn’t set (due to limited resources), the password resets on pod delete/redeploy.


Exposing Grafana
I used LoadBalancer service type, so Minikube gave it a virtual IP.
Created a Minikube tunnel so the LoadBalancer IP works.
Port-forwarded Grafana’s service to 8080:80 with this:

kubectl port-forward --address=0.0.0.0 svc/grafana 8080:80 -n grafana

Then I created systemd services for grafana svc port forward:
- sudo nano /etc/systemd/system/grafana-portforward.service
- then i added the file 
- sudo systemctl daemon-reexec
- sudo systemctl daemon-reload
- sudo systemctl enable grafana-portforward
- sudo systemctl start grafana-portforward
- sudo systemctl status grafana-portforward







Running the Minikube tunnel on system boot (same systemd link sudo nano /etc/systemd/system/minikube-tunnel.service
)

Running the port-forward command automatically
So even after stopping or restarting the EC2 instance, Grafana becomes accessible again at:


http://EC2_PUBLIC_IP:8080/login

Because Loki is resource-hungry, the dashboard loads very very slowly on this instance (t3.medium). But everything works.


Custom Dashboard
Loki was added already as the datasource in the values file for grafana

- Final Result
- Kubernetes running via Minikube 
- Loki collecting logs
- Promtail forwarding kubernetes logs
- Grafana showing logs in a custom dashboard
- Setup doesn't break across restarts using systemd
