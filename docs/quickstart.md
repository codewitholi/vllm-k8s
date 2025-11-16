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
    export PATH="/usr/local/cuda-12/bin:${PATH:+:${PATH}}"
    export LD_LIBRARY_PATH="/usr/local/cuda-12/lib64:${LB_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"
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
    sudo nvidia-ctk runtime configure --runtime=containerd
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
    export NV_CTR_IMAGE="docker.io/nvidia/cuda:12.8.1-base-ubuntu24.04"
    sudo ctr image pull $NV_CTR_IMAGE
    sudo ctr run --rm \
      --runc-binary=/usr/bin/nvidia-container-runtime \
      --env NVIDIA_VISIBLE_DEVICES=all \
      -t $NV_CTR_IMAGE \
      nvcudatest \
      nvidia-smi
    ```
