# SUSE CaaS Platform Grafana Monitoring Dashboards

Example files for SUSE CaaS Platform Grafana monitoring dashboards.

- [Monitor Whole Cluster](#monitor-whole-cluster)
- [Monitor All Nodes](#monitor-all-nodes)
- [Monitor All Pods](#monitor-all-pods)
- [Monitor All Namespaces](#monitor-all-namespaces)
- [Monitor ETCD Cluster](#monitor-etcd-cluster)
- [Monitor Certificates Status](#monitor-certificates-status)

## Monitor Whole Cluster

```
kubectl apply -f https://raw.githubusercontent.com/SUSE/caasp-monitoring/master/grafana-dashboards-caasp-cluster.yaml
```

## Monitor All Nodes

```
kubectl apply -f https://raw.githubusercontent.com/SUSE/caasp-monitoring/master/grafana-dashboards-caasp-nodes.yaml
```

## Monitor All Pods

```
kubectl apply -f https://raw.githubusercontent.com/SUSE/caasp-monitoring/master/grafana-dashboards-caasp-pods.yaml
```

## Monitor All Namespaces

```
kubectl apply -f https://raw.githubusercontent.com/SUSE/caasp-monitoring/master/grafana-dashboards-caasp-namespaces.yaml
```

## Monitor ETCD Cluster

Apply [etcd-cluster.yaml](grafana-dashboards-caasp-etcd-cluster.yaml) to monitor the etcd cluster.

```
kubectl apply -f https://raw.githubusercontent.com/SUSE/caasp-monitoring/master/grafana-dashboards-caasp-etcd-cluster.yaml
```

Then we need to manually setup extra configuration to Prometheus server. The etcd server expose metrics on `/metrics` endpoint, the Prometheus jobs does not scrapes it by default. We need to add a new job to Prometheus configmap to scapes metrics from the etcd cluster. Also since the etcd cluster run in https, we need the etcd client certificate in order to access the `/metrics` endpoint.

1. At one of the Kubernetes control plane node, create the etcd client certificate to secret `etcd-certs` in monitoring namespace.
    ```
    kubectl --kubeconfig=/etc/kubernetes/admin.conf -n monitoring create secret generic etcd-certs --from-file=/etc/kubernetes/pki/etcd/ca.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key
    ```

2. Get all the etcd cluster nodes' in-cluster IP address.
    ```
    kubectl get pods -n kube-system -l component=etcd -o wide

    NAME           READY   STATUS    RESTARTS   AGE   IP             NODE      NOMINATED NODE   READINESS GATES
    etcd-master0   1/1     Running   0          42h   192.168.0.32   master0   <none>           <none>
    etcd-master1   1/1     Running   0          42h   192.168.0.17   master1   <none>           <none>
    etcd-master2   1/1     Running   0          42h   192.168.0.5    master2   <none>           <none>
    ```

3. Edit the Prometheus server deployment, add additional mount `etcd-certs` which mount secret `etcd-certs` to path `/etc/secrets`.
    ```
    kubectl edit -n monitoring deployment prometheus-server
    ```

    ```yaml
      volumeMounts:
      - mountPath: /etc/secrets
        name: etcd-certs
        readOnly: true

    ...

    volumes:
    - name: etcd-certs
      secret:
        secretName: etcd-certs
    ```

4. Add a new job for etcd cluster, change the targets IP address(es) as your output in step 2.
    ```
    kubectl edit -n monitoring configmap prometheus-server
    ```

    ```yaml
    scrape_configs:
    - job_name: etcd
      static_configs:
      - targets: ['192.168.0.32:2379','192.168.0.17:2379','192.168.0.5:2379']
      scheme: https
      tls_config:
        ca_file: /etc/secrets/ca.crt
        cert_file: /etc/secrets/healthcheck-client.crt
        key_file: /etc/secrets/healthcheck-client.key
    ```

5. After saving the prometheus server configmap, the prometheus server would auto reload and apply new setting.

## Monitor Certificates Status

```
kubectl apply -f https://raw.githubusercontent.com/SUSE/caasp-monitoring/master/grafana-dashboards-caasp-certificates.yaml
```
