# AWS EKS Monitoring Manifests

This directory deploys Kubernetes-side monitoring agents into an AWS EKS cluster while keeping the central monitoring services on the existing Monitoring EC2 instance.

## Architecture

**Existing Monitoring EC2**
- Grafana
- Prometheus
- Loki
- VictoriaMetrics

**Inside EKS**
- Node Exporter: DaemonSet, one Pod per eligible worker node.
- Grafana Alloy: DaemonSet, collects container logs and forwards them to central Loki.
- kube-state-metrics: Deployment, exposes Kubernetes object-state metrics.
- vmagent: Deployment, scrapes EKS metrics and remote-writes them to central VictoriaMetrics.
- cAdvisor: use the kubelet's built-in `/metrics/cadvisor` endpoint; no separate cAdvisor Pod is deployed.

The EKS control plane is AWS-managed. Monitor its available metrics and logs through CloudWatch; do not install Node Exporter on the control plane.

---

## 1. Prerequisites

Run the following commands from an administrator workstation or an EC2 administration host with network access to the EKS cluster.

Install and configure:
- AWS CLI v2
- `kubectl` compatible with your EKS Kubernetes version
- IAM permissions to describe the EKS cluster and authenticate to it
- Kubernetes RBAC permissions to create Namespaces, DaemonSets, Deployments, Services, ConfigMaps, ServiceAccounts, ClusterRoles and ClusterRoleBindings

Check the tools:

```bash
aws --version
kubectl version --client
```

Configure AWS credentials if they are not already configured:

```bash
aws configure
```

Use an IAM identity that is authorized to access the EKS cluster. The identity must also be mapped/authorized through the EKS access configuration (EKS access entry or the cluster's configured authentication method).

## 2. Connect kubectl to EKS

Set your actual AWS Region and EKS cluster name:

```bash
export AWS_REGION="<your-aws-region>"
export EKS_CLUSTER_NAME="<your-eks-cluster-name>"

aws sts get-caller-identity

aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$EKS_CLUSTER_NAME"
```

Verify the connection and worker nodes:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A
```

For the example environment, confirm that the four expected worker nodes are `Ready`. EKS control-plane machines are AWS-managed and will not appear as ordinary worker nodes.

## 3. Configure the Monitoring EC2 destination

Find the **private IP address** of the existing Monitoring EC2 instance.

Replace the placeholder `MONITORING_EC2_PRIVATE_IP` in these files before applying them:

- `alloy/configmap.yaml`
  - Loki push URL: `http://MONITORING_EC2_PRIVATE_IP:3100/loki/api/v1/push`
- `vmagent/deployment.yaml`
  - VictoriaMetrics remote-write URL: `http://MONITORING_EC2_PRIVATE_IP:8428/api/v1/write`

For example, replace the placeholder with the Monitoring EC2 private IP, keeping the port and path unchanged.

Check network access:
- EKS worker nodes/Pods must be able to reach the Monitoring EC2 on TCP 3100 for Loki.
- EKS worker nodes/Pods must be able to reach the Monitoring EC2 on TCP 8428 for VictoriaMetrics remote write.
- Configure private routing, VPC connectivity and security groups accordingly.
- Allow inbound traffic only from the appropriate EKS security group or tightly scoped source; do not expose these ports publicly.
- Confirm that Loki and VictoriaMetrics are listening on interfaces reachable from EKS, not only on `127.0.0.1`.

If the cluster and Monitoring EC2 are in different VPCs, establish and verify the required private connectivity before deploying.

## 4. Review the manifests before production

Review the manifests for your EKS version, security requirements, node taints and Pod Security Admission policy.

Important:
- The Node Exporter manifest uses host-level mounts and host networking/ports. Review these permissions against your security policy.
- The Alloy manifest needs access to Kubernetes Pod metadata and log files. Validate its log collection configuration and required host log mounts for your chosen Alloy collection method. The supplied Alloy configuration is a starting point and must be tested against your cluster.
- The vmagent manifest scrapes kubelet `/metrics/cadvisor`. The supplied TLS setting `insecure_skip_verify: true` is only a bootstrap setting. For production, configure trusted kubelet certificate validation and remove that setting.
- Verify that the vmagent ServiceAccount has the required Kubernetes permissions and that EKS kubelet authorization permits the scrape.
- Pin image versions approved by your organization; do not use `latest` for production.
- The manifests are starter examples, not a substitute for testing in DEV/QA first.

## 5. Deploy the monitoring components

From the `target-server-setup-on-eks` directory, apply the files in this order.

### 5.1 Create the monitoring namespace

```bash
kubectl apply -f namespace/namespace.yaml
```

### 5.2 Deploy Node Exporter

```bash
kubectl apply -f node-exporter/
```

Node Exporter is deployed as a DaemonSet. On four eligible worker nodes, expect four Node Exporter Pods.

### 5.3 Deploy Grafana Alloy

```bash
kubectl apply -f alloy/rbac.yaml
kubectl apply -f alloy/configmap.yaml
kubectl apply -f alloy/daemonset.yaml
```

Alloy is deployed as a DaemonSet. It collects container logs on the nodes where it runs and sends them to the central Loki endpoint configured in `alloy/configmap.yaml`.

### 5.4 Deploy kube-state-metrics

```bash
kubectl apply -f kube-state-metrics/rbac.yaml
kubectl apply -f kube-state-metrics/deployment.yaml
kubectl apply -f kube-state-metrics/service.yaml
```

kube-state-metrics exposes Kubernetes object-state metrics through its ClusterIP Service.

### 5.5 Deploy vmagent

```bash
kubectl apply -f vmagent/rbac.yaml
kubectl apply -f vmagent/configmap.yaml
kubectl apply -f vmagent/deployment.yaml
kubectl apply -f vmagent/service.yaml
```

vmagent scrapes the targets configured in `vmagent/configmap.yaml` and sends samples to the central VictoriaMetrics remote-write endpoint.

## 6. Verify the deployment

Check DaemonSets:

```bash
kubectl get daemonsets -n monitoring
```

For four eligible worker nodes, expect:
- `node-exporter`: desired 4, ready 4
- `alloy`: desired 4, ready 4

Check Deployments:

```bash
kubectl get deployments -n monitoring
```

Expect:
- `kube-state-metrics`: 1/1 available
- `vmagent`: 1/1 available

Check all Pods and the nodes where they run:

```bash
kubectl get pods -n monitoring -o wide
```

Check Services:

```bash
kubectl get services -n monitoring
```

Inspect logs if something is not ready:

```bash
kubectl logs -n monitoring daemonset/node-exporter
kubectl logs -n monitoring daemonset/alloy
kubectl logs -n monitoring deployment/kube-state-metrics
kubectl logs -n monitoring deployment/vmagent
```

Describe a failing Pod:

```bash
kubectl describe pod -n monitoring <pod-name>
```

## 7. Validate data in the central monitoring stack

### VictoriaMetrics

Open the existing VictoriaMetrics UI:

```text
http://<MONITORING_EC2_PRIVATE_IP>:8428
```

Use the MetricsQL/Prometheus-compatible query interface to check metrics such as:

```promql
up
node_cpu_seconds_total
node_memory_MemAvailable_bytes
kube_pod_status_phase
container_cpu_usage_seconds_total
```

The exact available series depend on successful scrape configuration and the metrics exposed by your EKS version.

### Loki

In Grafana, select the existing Loki data source and check whether EKS container logs are arriving. Example LogQL query:

```logql
{namespace="default"}
```

The labels available depend on the Alloy configuration and the labels attached to discovered Pods.

### Grafana

Use the existing Grafana instance. Add or reuse:
- VictoriaMetrics as a Prometheus-compatible metrics data source.
- Loki as the log data source.

Create or import EKS dashboards only after confirming that the expected metrics and labels are present.

## 8. Enable EKS control-plane logging in CloudWatch

Control-plane logging is configured on the EKS cluster, separately from these Kubernetes manifests.

Example:

```bash
aws eks update-cluster-config \
  --region "$AWS_REGION" \
  --name "$EKS_CLUSTER_NAME" \
  --logging '{
    "clusterLogging": [{
      "types": [
        "api",
        "audit",
        "authenticator",
        "controllerManager",
        "scheduler"
      ],
      "enabled": true
    }]
  }'
```

Review CloudWatch Logs and the EKS cluster configuration after enabling. CloudWatch charges may apply.

## 9. Common troubleshooting commands

```bash
kubectl get pods -n monitoring -o wide
kubectl get events -n monitoring --sort-by=.lastTimestamp
kubectl describe daemonset node-exporter -n monitoring
kubectl describe daemonset alloy -n monitoring
kubectl describe deployment vmagent -n monitoring
kubectl logs -n monitoring deployment/vmagent
```

Common causes of problems:
- `ImagePullBackOff`: image registry access, image name/tag or node egress issue.
- `Pending`: node capacity, taints/tolerations, affinity or admission policy.
- Loki push errors: wrong private IP, port 3100 blocked, Loki listener or endpoint path issue.
- VictoriaMetrics remote-write errors: wrong private IP, port 8428 blocked or remote-write endpoint issue.
- Missing kubelet metrics: kubelet authorization, TLS trust, network access or RBAC issue.
- Missing logs: Alloy configuration, Pod log access, Kubernetes discovery permissions or Loki connectivity.

## 10. Rollback / remove the EKS monitoring components

Run from this directory:

```bash
kubectl delete -f vmagent/
kubectl delete -f kube-state-metrics/
kubectl delete -f alloy/daemonset.yaml
kubectl delete -f alloy/configmap.yaml
kubectl delete -f alloy/rbac.yaml
kubectl delete -f node-exporter/
kubectl delete -f namespace/namespace.yaml
```

Deleting the namespace removes remaining namespaced monitoring resources in that namespace. Review the namespace first and do not use it for unrelated workloads.

---

## Important architecture notes

- Keep the existing central Grafana, Prometheus, Loki and VictoriaMetrics Docker Compose deployment on Monitoring EC2.
- These EKS manifests deploy Kubernetes-side agents and vmagent; they do not deploy another Grafana, Loki or Prometheus server in EKS.
- Node Exporter and Alloy are DaemonSets, so Kubernetes schedules one Pod per eligible worker node.
- kube-state-metrics and vmagent are Deployments.
- Do not deploy a separate cAdvisor container for the normal EKS setup; collect cAdvisor metrics from kubelet.
- EKS control-plane machines are AWS-managed; use CloudWatch for the supported control-plane metrics and logs.
