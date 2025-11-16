# Quickstart

## Setting up Kubernetes with k3s

Review the installation requirements for k3s [here](https://docs.k3s.io/installation/requirements)

Install `k3s` with the quickstart script.

```bash
curl -sfL https://get.k3s.io | sh -
```

This will setup the key tools for managing containers such as

- containerd runtime
- ctr
- crictl

### Install CNI Plugins

```bash
export CNI_PLUGIN_VERSION="1.7.1"
wget https://github.com/containernetworking/plugins/releases/download/v$CNI_PLUGIN_VERSION/cni-plugins-linux-amd64-v$CNI_PLUGIN_VERSION.tgz
# Extract into /opt/cni/bin
sudo mkdir -p /opt/cni/bin
sudo tar -zxvf cni-plugins-linux-amd64-v$CNI_PLUGIN_VERSION.tgz -C /opt/cni/bin/
```

### Install nerdctl

Nerdctl is a useful CLI that has the same interface as `docker` and `docker compose` for containerd runtimes.

```bash
export NERDCTL_VERSION="2.2.0"
wget https://github.com/containerd/nerdctl/releases/download/v$NERDCTL_VERSION/nerdctl-$NERDCTL_VERSION-linux-amd64.tar.gz
tar -zxvf nerdctl-$NERDCTL_VERSION-linux-amd64.tar.gz -C $HOME/.local
```


## Setting up NVIDIA drivers and CUDA

1. Purge previous versions of NVIDIA
    ```bash
    sudo apt remove --purge nvidia*
    sudo apt autoremove
    sudo apt clean
    sudo apt-get update -y
    ```

2. Install linux-headers

    ```bash
    sudo apt-get install linux-headers-$(uname -r) -y
    ```

3. Install required packages

    ```bash
    sudo apt-get install git make gcc gcc-14 build-essential curl gnupg2 -y
    ```

4. Install CUDA Keyring

    ```bash
    export NV_DISTRO="ubuntu2404"
    export NV_ARCH="x86_64"

    wget https://developer.download.nvidia.com/compute/cuda/repos/$NV_DISTRO/$NV_ARCH/cuda-keyring_1.1-1_all.deb
    sudo dpkg -i cuda-keyring_1.1-1_all.deb

    sudo apt-get update -y
    ```

5. Install NVIDIA open driver and CUDA drivers.

    ```bash
    sudo apt-get install nvidia-open cuda-drivers -y
    ```

    !!! note
        Verify that the `nvidia-smi` CLI is working with `sudo nvidia-smi`. Use `sudo nvidia-smi -L` to list all GPU devices.

6. Install Cuda Toolkit and Nvidia GDS

    ```bash
    sudo apt-get install cuda-toolkit -y
    sudo apt-get install nvidia-gds -y
    ```
    !!! note
        Reboot your system after this.

7. Post Installation Steps

    ```bash
    export PATH="/usr/local/cuda-13/bin:${PATH:+:${PATH}}"
    export LD_LIBRARY_PATH="/usr/local/cuda-13/lib64:${LB_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"
    ```

    !!!note
       Verify that `nvcc` is available. Use `nvcc --version`

## Setting up NVIDIA container dependencies

1. Configure the production repository

    ```bash
    curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
    && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    ```

2. Update packages list from repo

    ```bash
    sudo apt-get update -y
    ```

3. Install NVIDIA Container Toolkit Packages

    ```bash
    export NVIDIA_CONTAINER_TOOLKIT_VERSION="1.18.0-1"
    sudo apt-get install -y \
        nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
        nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
        libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
        libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
    ```
4. Configure containerd

    ```bash
    sudo nvidia-ctk runtime configure --runtime=containerd --config /var/lib/rancher/k3s/agent/etc/containerd/config.toml
    # Restart containerd
    sudo systemctl restart containerd
    ```

5. Post-install steps

    Check that `nvidia-container-cli` is working.
    ```bash
    nvidia-container-cli --help 
    ```
    Check that `nvidia-ctk` is working.
    ```bash
    nvidia-ctk --version
    ```

6. Run a test GPU workload with nerdctl

    ```bash
    export NV_CTR_IMAGE="docker.io/nvidia/cuda:13.0.2-base-ubuntu24.04"
    sudo ctr image pull $NV_CTR_IMAGE
    sudo ctr run --rm \
      --runc-binary=/usr/bin/nvidia-container-runtime \
      --env NVIDIA_VISIBLE_DEVICES=all \
      -t $NV_CTR_IMAGE \
      nvcudatest \
      nvidia-smi
    ```

## Setup Kubernetes Device Plugin

This allows Kubernetes to expose NVIDIA GPU as resources in Kubernetes.

1. Create Nvidia Runtime Class

    ```bash
    tee nvidia-runtime-class.yaml <<EOF
    apiVersion: node.k8s.io/v1
    kind: RuntimeClass
    metadata:
      name: nvidia
    handler: nvidia
    EOF
    ```

    ```bash
    sudo k3s kubectl apply -f nvidia-runtime-class.yaml
    ```

2. Install the plugin with Helm

    ```bash
    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia \
    && helm repo update
    
    sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade -i --wait nvidiagpu \
     -n gpu-operator --create-namespace \
     --values gpu-operator.yaml \
      nvidia/gpu-operator
    ```

3. Post installation

    Verify that you can view GPUs present on the node, look for `nvidia.com/gpu.present=true` when describing node.

    ```bash
    sudo k3s kubectl describe node <NODE NAME> | grep "nvidia.com/gpu.present=true"
    ```

    Test that we can run a sample container that uses GPU

    ```bash
    tee test-gpu-pod.yaml <<EOF
    # test-gpu-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
     name: gpu-pod
    spec:
     restartPolicy: OnFailure
     runtimeClassName: nvidia
     containers:
       - name: cuda-container
         image: nvidia/cuda:12.8.1-base-ubuntu24.04
         command: ["nvidia-smi"]
         resources:
           limits:
             nvidia.com/gpu: 1 # requesting 1 GPU
    EOF
    ```

    Start it with

    ```bash
    sudo k3s kubectl apply -f test-gpu-pod.yaml
    ```

    We can check its logs after it has finished deploying

    ```bash
    sudo k3s kubectl logs gpu-pod

    ### SAMPLE OUTPUT
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 570.172.08             Driver Version: 570.172.08     CUDA Version: 12.8     |
    |-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |                                                                                                 
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  NVIDIA GeForce RTX 5090        On  |   00000000:01:00.0  On |                  N/A |
    |  0%   50C    P5             34W /  600W |    1390MiB /  32607MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+
                                                                                             
    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |  No running processes found                                                             |
    +-----------------------------------------------------------------------------------------+
    ```

## Monitoring with Grafana / Prometheus

We will use Prometheus and Grafana to monitor GPU usage metrics in the cluster. We can also use it to scrape metrics from vLLM down the road.

1. Add the Prometheus Helm Chart Repo

    ```bash
    sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && sudo helm repo update
    ```

2. Search for available versions

    ```bash
    sudo helm search repo kube-prometheus
    ```

3. Template out base values for configuration

    ```bash
    sudo helm inspect values prometheus-community/kube-prometheus-stack > kpm-values.yaml
    ```

4. Change the `serviceMonitorSelectorNilUsesHelmValues` option to `false` in the `kpm-values.yaml` file

    ```bash
    serviceMonitorSelectorNilUsesHelmValues: false
    ```

5. Add scrape configs to allow Prometheus to scrape from gpu-operator metrics endpoint in the `kpm-values.yaml` file

    ```bash
    #
    additionalScrapeConfigs:
    - job_name: gpu-metrics
      scrape_interval: 1s
      metrics_path: /metrics
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - gpu-operator
      relabel_configs:
      - source_labels: [__meta_kubernetes_endpoints_name]
        action: drop
        regex: .*-node-feature-discovery-master
      - source_labels: [__meta_kubernetes_pod_node_name]
        action: replace
        target_label: kubernetes_node
    ```

6. Install the `kube-prometheus` operator via Helm

    ```bash
    sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade -i kpm prometheus-community/kube-prometheus-stack --create-namespace --namespace kpm --values kpm-values.yaml
    ```

7. Retrieve the admin password for Grafana

    ```bash
    sudo k3s kubectl --namespace kpm get secrets kpm-grafana -o jsonpath="{.data.admin-password}" | base64 -d
    ```

8. In a new terminal / tmux window, port-forward the grafana service

    ```bash
    sudo k3s kubectl port-forward -n kpm svc/kpm-grafana 10100:80
    ```

    You can login to grafana dashboard at `http://127.0.0.1:10100`

### Setup GPU Monitoring with DCGM

1. Add the helm repo for DCGM exporter

    ```bash
    sudo helm repo add gpu-helm-charts \
      https://nvidia.github.io/dcgm-exporter/helm-charts \
      && sudo helm repo update
    ```

2. Install the DCGM Exporter

    ```bash
    sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade -i dcgm gpu-helm-charts/dcgm-exporter --namespace kpm
    ```
## References

- k8s-device-plugin: [link](https://github.com/NVIDIA/k8s-device-plugin/)
- Setting up monitoring with Grafana / Prometheus: [link](https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/kube-prometheus.html)

## Todo

Maybe use this Helm chart for GPU metrics: https://artifacthub.io/packages/helm/utkuozdemir/nvidia-gpu-exporter
