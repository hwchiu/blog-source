---
title: Install GPU in GKE(Google Kubernetes Engine)
keywords: 'Linux,Ubuntu,K8S,Kubernetes,GPU'
date: 2018-07-14 17:41:45
tags:
	- GPU
	- GKE
	- Kubernetes
description:
---

This post is about what problems I met when I want to enable a GPU in the GKE gluster.

After referring to the official website, I still met problems and could not run the GPU in the cluster.

When I want to enable GPUs in my GKE cluster, I try to google the `GKE GPU` first and I found many posts about this requirements.

The first website is the official post about GPUs in the GKE and you can found it here.
https://cloud.google.com/kubernetes-engine/docs/concepts/gpus

The workflow about deploying GPUs in the GKE is easy.
1. Request GPUs from the GCP control panel.
2. Create a GPU node pool and attach to your GKE cluster
3. Install the NVIDIA device drviers
4. Config the Pod to use the GPU devices.

<!--more-->
The first two steps are not difficult and you can follow those command to work well.
But for the third steps. we need to install the nvidia drivers in the kubernetes.

In the above link, it says use the following command to deploy a daemonset to install the nvidia driver
```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/stable/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```
But in the other post of the official [kubernetes ](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/). It says you should use the following command to install the driver
```
# Install NVIDIA drivers on Container-Optimized OS:
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/k8s-1.9/daemonset.yaml

# Install NVIDIA drivers on Ubuntu (experimental):
kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/k8s-1.9/nvidia-driver-installer/ubuntu/daemonset.yaml

# Install the device plugin:
kubectl create -f https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.9/cluster/addons/device-plugins/nvidia-gpu/daemonset.yaml
```

In the first approach, we only need to install the `daemonset` for dirver install but we need to add additional `daemonset` for device plugin in the second approach.

For that difference of two approachs I found the answear [here](https://github.com/GoogleCloudPlatform/container-engine-accelerators/tree/master/cmd/nvidia_gpu)

>In GKE, from 1.9 onwards, this daemonset is automatically deployed as an addon. Note that daemonset pods are only scheduled on nodes with accelerators attached, they are not scheduled on nodes that don't have any accelerators attached.

So we only need to manually deploy the daemonset to install the `NVIDIA drivers` in the GKE which k8s version >= 1.9

Now, let's deploy the daemonset to install the NVIDIA driver and you can find the yaml [here](https://github.com/GoogleCloudPlatform/container-engine-accelerators/tree/master/nvidia-driver-installer). Don't forget to change the branch to fit your kubernetes version and find the daemonset for your operating system.

The daemonset is be deployed in the `kube-system` namespace and you can use the `kubectl get -n kube-system pods` to watch the status.

After the status is `running`, we can start to deploy your own Pod with the GPU now.

From the example in the website.
```
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-gpu-container
    resources:
      limits:
       nvidia.com/gpu: 2
```
You only indicate how many GPU the Pod needs in the yaml files but I found my GPU application can't run successfully.

The log shows like `"libcuda.so.1 file" is not found.` and I did indeed can't find those libraries in my Pod.

The problem is the following setence which is posted [here](https://cloud.google.com/kubernetes-engine/docs/concepts/gpus#gpu_pool)


>CUDA® is NVIDIA's parallel computing platform and programming model for GPUs. The NVIDIA device drivers you install in your cluster include the CUDA libraries.

>CUDA libraries and debug utilities are made available inside the container at /usr/local/nvidia/lib64 and /usr/local/nvidia/bin, respectively.

>CUDA applications running in Pods consuming NVIDIA GPUs need to dynamically discover CUDA libraries. This requires including /usr/local/nvidia/lib64 in the LD_LIBRARY_PATH environment variable.

According to that clue, I think we need to find the libraries in the `/usr/local/nvidia/lib64` and set the `LD_LIBRARY_PATH` to link those libraries automatically.

I just wondering how the `daemonset` install the `NVIDIA drivers` for each GPU node and where did the `NVIDIA CUDA files` be installed?

Here is the yaml file for `NVIDIA driver`


```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-driver-installer
  namespace: kube-system
  labels:
    k8s-app: nvidia-driver-installer
spec:
  selector:
    matchLabels:
      k8s-app: nvidia-driver-installer
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: nvidia-driver-installer
        k8s-app: nvidia-driver-installer
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/gke-accelerator
                operator: Exists
      tolerations:
      - key: "nvidia.com/gpu"
        effect: "NoSchedule"
        operator: "Exists"
      volumes:
      - name: dev
        hostPath:
          path: /dev
      - name: nvidia-install-dir-host
        hostPath:
          path: /home/kubernetes/bin/nvidia
      - name: root-mount
        hostPath:
          path: /
      initContainers:
      - image: gcr.io/google-containers/ubuntu-nvidia-driver-installer@sha256:eea7309dc4fa4a5c9d716157e74b90826e0a853aa26c7219db4710ddcd1ad8bc
        name: nvidia-driver-installer
        resources:
          requests:
            cpu: 0.15
        securityContext:
          privileged: true
        env:
          - name: NVIDIA_INSTALL_DIR_HOST
            value: /home/kubernetes/bin/nvidia
          - name: NVIDIA_INSTALL_DIR_CONTAINER
            value: /usr/local/nvidia
          - name: ROOT_MOUNT_DIR
            value: /root
        volumeMounts:
        - name: nvidia-install-dir-host
          mountPath: /usr/local/nvidia
        - name: dev
          mountPath: /dev
        - name: root-mount
          mountPath: /root
      containers:
      - image: "gcr.io/google-containers/pause:2.0"
        name: pause
```

We can found the daemonset use the `init-container` to run the nvidia-driver-installer image and use the `hostpath` to mount the volume into the container.
The path in the host is `/home/kubernetes/bin/nvidia` and in the container is `/usr/local/nvidia`.

We can use the following command to see the logs of the init-container in the `nvidia-install-driver`
```bash
kubectl logs -n kube-system -l name=nvidia-driver-installer -c nvidia-driver-installer
```

The log looks like below and we can found the `nvidia-smi` works well.

```
+ COS_DOWNLOAD_GCS=https://storage.googleapis.com/cos-tools
+ COS_KERNEL_SRC_GIT=https://chromium.googlesource.com/chromiumos/third_party/kernel
+ COS_KERNEL_SRC_ARCHIVE=kernel-src.tar.gz
+ TOOLCHAIN_URL_FILENAME=toolchain_url
+ CHROMIUMOS_SDK_GCS=https://storage.googleapis.com/chromiumos-sdk
+ ROOT_OS_RELEASE=/root/etc/os-release
+ KERNEL_SRC_DIR=/build/usr/src/linux
+ NVIDIA_DRIVER_VERSION=384.111
+ NVIDIA_DRIVER_DOWNLOAD_URL_DEFAULT=https://us.download.nvidia.com/tesla/384.111/NVIDIA-Linux-x86_64-384.111.run
+ NVIDIA_DRIVER_DOWNLOAD_URL=https://us.download.nvidia.com/tesla/384.111/NVIDIA-Linux-x86_64-384.111.run
+ NVIDIA_INSTALL_DIR_HOST=/home/kubernetes/bin/nvidia
+ NVIDIA_INSTALL_DIR_CONTAINER=/usr/local/nvidia
++ basename https://us.download.nvidia.com/tesla/384.111/NVIDIA-Linux-x86_64-384.111.run
+ NVIDIA_INSTALLER_RUNFILE=NVIDIA-Linux-x86_64-384.111.run
+ ROOT_MOUNT_DIR=/root
+ CACHE_FILE=/usr/local/nvidia/.cache
+ set +x
[INFO    2018-07-14 14:57:09 UTC] Running on COS build id 10323.85.0
[INFO    2018-07-14 14:57:09 UTC] Checking if third party kernel modules can be installed
[INFO    2018-07-14 14:57:09 UTC] Checking cached version
[INFO    2018-07-14 14:57:09 UTC] Found existing driver installation for image version 10323.85.0           and driver version 384.111.
[INFO    2018-07-14 14:57:09 UTC] Configuring cached driver installation
[INFO    2018-07-14 14:57:09 UTC] Updating container's ld cache
[INFO    2018-07-14 14:57:14 UTC] Verifying Nvidia installation
Sat Jul 14 14:57:15 2018
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.111                Driver Version: 384.111                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla K80           Off  | 00000000:00:04.0 Off |                    0 |
| N/A   31C    P0    62W / 149W |      0MiB / 11439MiB |    100%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
[INFO    2018-07-14 14:57:16 UTC] Found cached version, NOT building the drivers.
[INFO    2018-07-14 14:57:16 UTC] Updating host's ld cache
```


Now, we know that daemonset will install the `NVIDIA driver` files into the `/usr/local/nvidia` and that path is mapping to the path `/home/kubernetes/bin/nvidia` in the host.

I think we should mount the `/home/kubernetes/bin/nvidia` into our Pod and the above Pod yaml will become like this.


```
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-gpu-container
    resources:
      limits:
       nvidia.com/gpu: 2
    volumeMounts:
    - mountPath: /usr/local/bin/nvidia
      name: nvidia-debug-tools
    - mountPath: /usr/local/nvidia/lib64
      name: nvidia-libraries
  volumes:
  - hostPath:
      path: /home/kubernetes/bin/nvidia/bin
      type: ""
    name: nvidia-debug-tools
  - hostPath:
      path: /home/kubernetes/bin/nvidia/lib64
      type: ""
    name: nvidia-libraries
```

Now, I can find all libraries about `NVIDIA drivers` in my Pod but the `nvidia smi` can't work correctly.
It shows the error message about
>"Failed to initialize NVML: Unknown Error"

I use the `strace` to debug and found that we don't have the permission to read the `/dev/nvidiactl `.
Just add the `privileged=true` in the yaml file and everything works well now.!

```
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: my-gpu-container
    resources:
      limits:
       nvidia.com/gpu: 2
    volumeMounts:
    - mountPath: /usr/local/bin/nvidia
      name: nvidia-debug-tools
    - mountPath: /usr/local/nvidia/lib64
      name: nvidia-libraries
    securityContext:
      privileged: true
  volumes:
  - hostPath:
      path: /home/kubernetes/bin/nvidia/bin
      type: ""
    name: nvidia-debug-tools
  - hostPath:
      path: /home/kubernetes/bin/nvidia/lib64
      type: ""
    name: nvidia-libraries
```          

Summary.
1. We only deploy the `daemonset` to install the `NVIDIA driver` in the GKE if the version >= 1.9
2. We need to mount the NVIDIA Cuda libraries into your Pod via `Hostpath`
3. We also need the `privileged=true` to open the `/dev/nvidiactl`.
