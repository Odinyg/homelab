# homelab


## Nvidia runtime for k3s
### For NVIDIA container toolkit configuration with k3s, you'll need to:
1. Install Nvidia driver,toolkit and nvidia-smi follow the nvidia documentation.
2. Configure the k3s containerd instead of the standalone containerd:
`sudo nvidia-ctk runtime configure --runtime=containerd --config=/var/lib/rancher/k3s/agent/etc/containerd/config.toml`
`sudo asystemctl restart k3s
#### Links:
https://github.com/NVIDIA/k8s-device-plugin
https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html
