
# Prerequisite
- kubectl
- helm

# Setup Prometheus with grafana

First we need to import the prometheus helm charts.
The `kube-prometheus-stack` chart includes grafana and will deploy it when we install it

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack

```

# Setup Loki
```
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack
```

# Add Loki as a data source

Now that our deployments are running, we need to open grafana's web interface. Since grafana's service is `ClusterIP`, we have to either add an `Ingress` for it, or simply port forward it as follows:
```
kubectl port-forward deployment/prometheus-grafana 3000
```

Go to your browser and put your machine's IP with port 3000 (ex: `localhost:3000`).

The default credentials are:
- username: admin
- password: prom-operator

If the credentials are wrong, we can decode them from the secrets file as follows:

Username:
```
kubectl get secret prometheus-grafana -o yaml | grep admin-user | awk '{print $2}' | base64 -D
```

Password:
```
kubectl get secret prometheus-grafana -o yaml | grep admin-password | awk '{print $2}' | base64 -D
```

Now that we are in, use the left panel to navigate to : `Configuration` -> `Data sources`.
In this page we need to add a new data source:
- choose loki in the logging section
- add the loki service url with port (ex: `http://loki:3100`)
- save and test

The test should be successful.

# Check container logs using loki
In the left menu again, go to `Explore`.
- Select `Loki` in the upper dropdown menu
- select `log browser`
- choose `container` label
- choose the container you wish to see logs of
- click on `show logs`

![Screen Shot 2022-06-10 at 10.33.11 AM.png](/.attachments/Screen%20Shot%202022-06-10%20at%2010.33.11%20AM-0d27ca20-e227-403d-9397-bf76ca1493a4.png)

Now we can see logs of any container in our cluster. We can use queries to only see relevant log data.

We can even use pre-defined dashboard from the [grafana website](https://grafana.com/grafana/dashboards/)

