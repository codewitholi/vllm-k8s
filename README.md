# vllm-k8s

This repository is a sample of how to create your own local LLM setup with vLLM and Openwebui.

## Usage

### Install Helm Chart for vllm and openwebui

Sample LLM configurations can be found in `helm-examples` folder.

**Installing**

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade -i cluster helm-charts/llm-k8s --values helm-examples/qwen3.yaml --namespace llm-cluster --create-namespace
```

**Uninstalling**

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm uninstall cluster
```

### Expose services

Grafana

```bash
sudo k3s kubectl port-forward -n kpm svc/kpm-grafana 10100:80
```


Openwebui

```bash
sudo k3s kubectl port-forward -n llm-cluster svc/cluster-llm-k8s-openwebui 9000:9000
```

## Developing

### Updating docs

```bash
mkdocs serve -a localhost:9002 -w docs --livereload
```
