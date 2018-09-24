---
title: Install the kubernetes as single node in Fedora 28
keywords: 'Kubernetes,Install,Fedora,kubeadm'
abbrlink: 3041
date: 2018-09-24 00:31:10
tags:
  - Kubernetes
  - Fedora
  - Linux
description: To setup a kubernetes, there're so many approaches, including what infrstructure(VM/Container/Native) about the kubernetes and what tooles(kubeadm,kubespray,minikube). I choose use the kubeadm to install a native kubernetes in my laptop(Fedora) and you will see the whole setps to install all requirements, including the docker/kubernetes.

---

# Preface
For some reasons, I had swithced the my laptop from the MAC Book Pro to another laptop with the Fedora 28.

In order to play the kubernetes with anything I want, I need to install the kubernetes in the Fedora 28.

There're two approachs I can operate the kubernetes in my laptop.
1. Create a Ubuntu VM in the fedora 28 and install the kubernetes in that Ubuntu, just like how I play the kubernetes in the previous MAC OSX environment.
2. Install the kubernetes into the native Fedora 28 OS, it seems ok since there're many posts about Fedora and kubernetes in the internet.

So, I choose the second approach to install the kubernetes because it's new to me and I want to git it a try.
That's why this post exist and I will note every commands I used to install the kubernetes.

# Installation Choices
There're so many ways to install the kubernetes into a target system.
1. Use the kubernetes hard-way to install all components, including etcd,kubelet, kube-apiserver.
2. Use the minikube to install a single-node kubernetes(by default).
    - In fact, the minikube will lunch a VM to run a kubernetes but you can change to container-only method.
3. Use the ansible-liked method to help you install all necessary process/daemon we need.
4. Use the very simple official tool kubeadm to setup a kubernetes.

Just considered what subject I want to study in the kubernetes, I think the `kubeadm` is enough and I will choose it for the following tutorial.

# Environment
First, we need to prepare the container environmnet for the kubernetes, and I choose to use the docker for my kubernetes.

If you use the default repository to search the docker, the version is about 13.1 and that it not what we want.

You can use the following command to check the latest docker version in the remote repository. Besides, the command `sudo dnf list --showduplicates docker\*` to show all version of the docker.

```bash=
ahwchiu➜~» sudo dnf list docker\*                                                                                                                   [14:00:16]
Last metadata expiration check: 13:41:45 ago on Sun 23 Sep 2018 12:18:32 AM CST.                                                                             
Installed Packages
docker.x86_64                                                                  2:1.13.1-61.git9cb56fd.fc28                                            @updates
docker-common.x86_64                                                           2:1.13.1-61.git9cb56fd.fc28                                            @updates
docker-rhel-push-plugin.x86_64                                                 2:1.13.1-61.git9cb56fd.fc28                                            @updates
Available Packages
docker-anaconda-addon.x86_64                                                   0.4-7.fc28                                                             fedora 
docker-client-java.noarch                                                      6.2.5-7.fc28                                                           fedora 
docker-compose.noarch                                                          1.20.1-1.fc28                                                          fedora 
docker-devel.noarch                                                            2:1.13.1-61.git9cb56fd.fc28                                            updates
docker-distribution.x86_64                                                     2.6.2-7.git48294d9.fc28                                                fedora 
docker-fish-completion.x86_64                                                  2:1.13.1-61.git9cb56fd.fc28                                            updates
docker-latest.x86_64                                                           2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-devel.noarch                                                     2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-fish-completion.x86_64                                           2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-logrotate.x86_64                                                 2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-rhsubscription.x86_64                                            2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-unit-test.x86_64                                                 2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-v1.10-migrator.x86_64                                            2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-vim.x86_64                                                       2:1.13-36.git27e468e.fc28                                              fedora 
docker-latest-zsh-completion.x86_64                                            2:1.13-36.git27e468e.fc28                                              fedora 
docker-logrotate.x86_64                                                        2:1.13.1-61.git9cb56fd.fc28                                            updates
docker-lvm-plugin.x86_64                                                       2:1.13.1-61.git9cb56fd.fc28                                            updates
docker-novolume-plugin.x86_64                                                  2:1.13.1-61.git9cb56fd.fc28                                            updates
docker-unit-test.x86_64                                                        2:1.13.1-61.git9cb56fd.fc28                                            updates
docker-vim.x86_64                                                              2:1.13.1-61.git9cb56fd.fc28                                            updates
```

For the latest docker version, we should install the docker-ce (docker community edition) in the system and we can find the detail tutorail in the [official document](https://docs.docker.com/install/linux/docker-ce/fedora/).

Remember, you need to remove all installed docker before you install the docker-ce and you can use the following command to remove the existing docker packages.
```bash=
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

First, we need to add additional repo about docker-ce into the dnf system.
```bash=
sudo dnf config-manager \
    --add-repo \
    https://download.docker.com/linux/fedora/docker-ce.repo
```

Then, we can install the docker-ce and activate it by systemd system.
```bash=
sudo dnf -y install docker-ce
sudo docker version
sudo systemctl enable docker
sudo systemctl start docker
```

In addition to docker, there're some issues that we need to solve for the kubernetes.
1. firewalld, we must sure the default firewall won't block the port of 6443/10250.
2. it's still not support to run kubeadm with the swap. we need to disalbe it.

```bash=
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo swapoff -a && sudo sysctl -w vm.swappiness=0
```


# Tools
After installing the docker-ce as the container environment for kubernetes, the next parts is the tools of kubernetes.

Just like the `docker-ce`, we can't find those tools in the default repo, hence, we need to add additional repo to dnf system now.


Before we use the `kubeadm` to install the kubernetes, we have to install three packages we need. the `kubectl`, `kubelet` and `kubeadm`.

Since there're not any existing repo file about kubernetes for fnd/yum system, we need to create it by ourself.

```bash=
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```

If you want to install the specific version of kubernetes tools, use the `sudo dnf list --showduplicates kube\*` to see all verions. Otherwise, use the following comand to install the latest verion of kubernetes.


We need to use the `--disableexcludes` to disable the exclude list of kubernetes, since all tools are in that kubernetes repo.
After installing that, we also need to enable the systemd service for kubelet.

```bash=
sudo dnf install -f kubelet kubectl kubeadm --disableexcludes=kubernetes
sudo kubectl version
sudo kubeadm version
sudo kubelet --version
sudo systemctl enable kubelet
```
      
   
# kubernetes
Now, I will ues the `kubeadm` to install the kubernetes environment now.
The important thing is that you need to take care the parameter of your `kubeadm` process, if you choose the `flannel` as your CNI(Containre Network Interface). You use pass the `--pod-network-cidr=10.244.0.0/16` and the `flannel` can use that to choose the IP address range of each node.


This init procedure takes time to install whole kubernetes, the most part of that is used to download the dokcer image for all kubernetes components.
```bash=
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
[init] using Kubernetes version: v1.11.3
[preflight] running pre-flight checks
I0923 23:45:54.958338   16222 kernel_validator.go:81] Validating kernel version
I0923 23:45:54.958582   16222 kernel_validator.go:96] Validating kernel config
        [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.06.1-ce. Max validated version: 1
7.03
[preflight/images] Pulling images required for setting up a Kubernetes cluster
[preflight/images] This might take a minute or two, depending on the speed of your internet connection                                                       
[preflight/images] You can also perform this action in beforehand using 'kubeadm config images pull'                                                         
[kubelet] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"                                                           
[kubelet] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[preflight] Activating the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [localhost.localdomain kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.s
vc.cluster.local] and IPs [10.96.0.1 10.0.4.63]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [localhost.localdomain localhost] and IPs [127.0.0.1 ::1]                                    
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [localhost.localdomain localhost] and IPs [10.0.4.63 127.0.0.1 ::1]                            
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"                                                                                     
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"                                                                                   
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"                                                                        
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"                                                                                 
[controlplane] wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"                                     
[controlplane] wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"                   
[controlplane] wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"                                     
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"                                                          
[init] waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests"                                        
[init] this might take a minute or longer if the control plane images have to be pulled                                                                                                                            
[apiclient] All control plane components are healthy after 38.003505 seconds                                                                                 
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace                                                   
[kubelet] Creating a ConfigMap "kubelet-config-1.11" in namespace kube-system with the configuration for the kubelets in the cluster                         
[markmaster] Marking the node localhost.localdomain as master by adding the label "node-role.kubernetes.io/master=''"                                        
[markmaster] Marking the node localhost.localdomain as master by adding the taints [node-role.kubernetes.io/master:NoSchedule]                               
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "localhost.localdomain" as an annotation                  
[bootstraptoken] using token: 9s11hf.6ooyk6587vqeirn7

[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials               
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token                            
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster                                         
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace                                                                        
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.0.4.63:6443 --token 9s11hf.6ooyk6587vqeirn7 --discovery-token-ca-cert-hash sha256:961df828697d1b003a8205af51912ae05ea06e79576121ae25d0a5835ffb0e6d
```

Now, follow the instructions to setup the kubeconfig for your regular user.

```bash=
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

With that kubeconfig, we can use the `kubectl` to control the kubernetes now and you can use the `kubectl get nodes` to see the status of the node.

Since we have not installed the CNI in the cluster, the status is `NotReday` and it will become the `Ready` once you install any CNI into it.
```bash=
hwchiu➜~» kubectl get nodes                                                                                                                        [23:50:27]
NAME                    STATUS     ROLES     AGE       VERSION
localhost.localdomain   NotReady   master    3m        v1.11.3

hwchiu➜~» kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml                                 [23:50:39]
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds created

hwchiu➜~» kubectl get nodes                                                                                                                        [23:51:30]
NAME                    STATUS    ROLES     AGE       VERSION
localhost.localdomain   Ready     master    5m        v1.11.3
```

# Deploy Pod
Unfortunately, we still can't run a Pod successfully, the reason is that the `master` node doesn't be allowed to deploy any Pods if we use the `kubeadm` to setup the kubernetes cluster.
In my condition, I have only one node and it must be master node, that's why any user-defined Pod will be pending forever.

The mechanism about that limitation is the `taint` and we can use the following command to remove the taint rules of all master node.
```bash=
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Now, we can deploy a Pod and enjoy your kubernetes cluster.
```bsah=
kubectl run test --image=hwchiu/netutils
kubectl get pods -o wide -w
```

# Helm Chart (Optional)
There's a package system like the `deb` or something else in the kubernetes ecosystem, and it's called  `Helm Chart`.
You can use that to install some predefined packages which contains all resources you need, such as the deployment,service,configmap,RBAC and others into your kubernetes cluster.

Again, it's optional tool and you can install it if you need it.

You can follow the [official document](https://docs.helm.sh/using_helm/) to install the `helm chart` but you will meet some problems.

I will note those problems and share how I solve those problems.

## Install
Since we are use the `Fedora` now, the `helm` has not been takend by any dnf repos, we need to download the binary from the official website and you can use the following script to download the binary automatically.
```
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

You will be informed to use the `helm init` to initial the helm environment in your kubernetes cluster after executing the `get_helm.sh`, **but don't do that now.**

Since we use the `kuberadm` to install the kubernetes and it use the `RBAC` mode as default. We need to add another parameter for the `RBAC` mode.

You need to prepare the RBAC config for your `helm chart` service before you using it to install any packages.
And you can find the [whole tutoral here](https://docs.helm.sh/using_helm/#role-based-access-control)

After create the `RBAC` config, use the follwing command to init your `helm`
```bash=
helm init --service-account tiller
```

# Test
Now, using the following command to install the nginx ingress package by `helm`.

```bash=
helm install --name my-release stable/nginx-ingress 
```

Use the `kubectl` to see all the kubernetes resource we installed by the helm
```bash=
kubectl get all | grep my-release
```

