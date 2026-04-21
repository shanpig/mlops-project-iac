# Monitoring, Autoscaling, and Alerting Demo (Exam-Ready)

This runbook shows exactly how to demonstrate:
- Infrastructure health monitoring
- Automated scaling to preserve health
- Alerting when the system degrades

## 1) Apply latest platform/pipeline config

```bash
cd /Users/shanpig/Desktop/projects/nyu/mlops/iac
ansible-playbook -i inventory.yml argocd/argocd_add_pipeline.yml --ask-vault-pass
```

Then verify resources:

```bash
kubectl -n proj10-platform get deploy,svc,hpa | egrep 'prometheus|grafana|alertmanager|ml-api|ml-serving'
```

Expected:
- `prometheus`, `grafana`, `alertmanager` Deployments/Services
- `ml-api-hpa`, `ml-serving-hpa`

## 2) Monitoring evidence (Grafana + Prometheus)

Open:
- Prometheus: `http://<external-ip>:9090`
- Grafana: `http://<external-ip>:3000`

Capture screenshots:
1. Prometheus `/targets` showing `ml-api`, `stream-consumer`, `pushgateway`, `prometheus` as `UP`.
2. Grafana dashboard list with `batch_pipeline`, `etl`, and `live_data`.
3. One active dashboard panel showing recent metrics.

## 3) Alerting evidence (degradation + alert fired)

Trigger a controlled failure:

```bash
kubectl -n proj10-platform scale deploy ml-api --replicas=0
```

Wait ~2-3 minutes, then check:

```bash
kubectl -n proj10-platform port-forward svc/prometheus 9090:9090
```

Open `http://127.0.0.1:9090/alerts` and capture screenshot of:
- `MlApiDown` firing.

Recover:

```bash
kubectl -n proj10-platform scale deploy ml-api --replicas=1
```

Capture screenshot showing alert resolved.

## 4) Autoscaling evidence (HPA)

Check HPA before load:

```bash
kubectl -n proj10-platform get hpa
```

Generate load (example against ml-serving):

```bash
kubectl -n proj10-platform run loadgen --rm -it --restart=Never --image=curlimages/curl -- sh -lc '
  i=0
  while [ $i -lt 2000 ]; do
    curl -s -o /dev/null http://ml-serving.proj10-platform.svc.cluster.local:8000/health || true
    i=$((i+1))
  done
'
```

Watch autoscaler:

```bash
kubectl -n proj10-platform get hpa ml-serving-hpa -w
kubectl -n proj10-platform get pods -l app=ml-serving -w
```

Capture screenshots:
1. HPA target CPU > threshold and replica increase.
2. Pod replica count increase.
3. Replica count scale down after load ends.

## 5) Suggested report wording

"We monitor platform and pipeline health using Prometheus and Grafana dashboards. We enforce automated horizontal scaling with HPA on critical services (`ml-api`, `ml-serving`) based on resource pressure. We configured alert rules in Prometheus and routed alerts to Alertmanager for degradation events (e.g., service down/scrape failure). During validation, we induced service failure and observed alert firing/resolution, and induced load to verify automatic scale-out and scale-in behavior."
