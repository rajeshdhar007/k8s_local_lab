# Kubernetes Local Lab 🚥

## Pre-requisites:
You need to have the following tools installed on your local machine:
- Virtual Box
  - https://www.virtualbox.org/wiki/Downloads
- Vagrant
  - https://www.vagrantup.com/

This local lab setup assumes you have machine powerful enough to host 3 VMs with 2 CPUs and 2 GB of memory each
- 1 Kubernetes Master (kubemaster)
- 2 Kubernetes Nodes (kubenode0#)

The IP of ```kubemaster``` is ```192.168.56.2```. Ofcourse, all these can be changed by manipulating the ```Vagrantfile```

# Steps:
1. Initialize the Infrastructure
    ```$ vagrant up```

2. Run the following command to see the status of the 3 VMs provisioned on VirtualBox

   ```
   $ vagrant status
   Current machine states:
   kubemaster                running (virtualbox)
   kubenode01                running (virtualbox)
   kubenode02                running (virtualbox)
   ```
3. You may log in to any of the server with the following command:
   ```
   vagrant ssh <servername>
   ex:
   vagrant ssh kubemaster
   ```

4. On all three servers (kubemaster, kubenode01 and kubenode02):
   - ### Install Container Runtime:
        We will use Docker for our container runtime 🐳:
        Ref: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker

        ```
        # (Install Docker CE)
        ## Set up the repository:
        ### Install packages to allow apt to use a repository over HTTPS
        sudo apt-get update && sudo apt-get install -y \
            apt-transport-https ca-certificates curl software-properties-common gnupg2

        # Add Docker's official GPG key:
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key --keyring /etc/apt/trusted.gpg.d/docker.gpg add -

        # Add the Docker apt repository:
        sudo add-apt-repository \
            "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
            $(lsb_release -cs) \
            stable"

        # Install Docker CE
        sudo apt-get update && sudo apt-get install -y \
            containerd.io=1.2.13-2 \
            docker-ce=5:19.03.11~3-0~ubuntu-$(lsb_release -cs) \
            docker-ce-cli=5:19.03.11~3-0~ubuntu-$(lsb_release -cs)

        ## Create /etc/docker
        sudo mkdir /etc/docker

        # Set up the Docker daemon
        cat <<EOF | sudo tee /etc/docker/daemon.json
        {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2"
        }
        EOF

        # Create /etc/systemd/system/docker.service.d
        sudo mkdir -p /etc/systemd/system/docker.service.d

        # Restart Docker
        sudo systemctl daemon-reload
        sudo systemctl restart docker

        # Docker service to start on boot
        sudo systemctl enable docker

        # Use docker without sudo
        # Ref: https://docs.docker.com/engine/install/linux-postinstall/
        sudo groupadd docker
        sudo usermod -aG docker $USER

        ```
    - ### Installing kubeadm, kubelet and kubectl
        ```
        # Letting iptables see bridged traffic
        lsmod | grep br_netfilter
        # If you get an output, you're good. Else run the command below:
        sudo modprobe br_netfilter

        # Install kubeadm, kubelet and kubectl
        sudo apt-get update && sudo apt-get install -y apt-transport-https curl
        curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
        cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
        deb https://apt.kubernetes.io/ kubernetes-xenial main
        EOF
        sudo apt-get update
        sudo apt-get install -y kubelet kubeadm kubectl
        sudo apt-mark hold kubelet kubeadm kubectl

        # Letting iptables see bridged traffic
        cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
        br_netfilter
        EOF

        cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        EOF
        sudo sysctl --system
        ```
5. Initialize the kubemaster node (NOT on all servers)
    ```
    kubeadm init --pod-network-cidr <network_not_currently_in_use> \
        --apiserver-advertise-address=<ip-address>

    ex:
    kubeadm init --pod-network-cidr 10.244.0.0/16 \
        --apiserver-advertise-address=192.168.56.2

    # To start using your cluster, you need to run the following as a regular user:
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    # At this point, you may be able to run kubectl commands.
    # However, you may not see other 2 nodes since they have not joined the cluster
    # Make sure you take a note of the kubeadm join command that we need to run on the nodes to join the cluster
    ex: kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>


    # Install the Network CNI Addon
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"    
    ```
6. Run the join command on the nodes (kubenode01 and kubenode02) to join the cluster.
    ```
    kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

    # If you missed making a note of the kubeadm join command along with the token,
    # you can list them or recreate one:
    kubeadm token list

    -OR-
    # This will create a new token and print the new command to run on the nodes to join as well.
    kubeadm token create --print-join-command

    ```
7. To test, from the kubemaster, deploy an image
   ```
   kubectl run nginx --image=nginx

   kubectl get pods

   kubectl get nodes
   ```

## Congratulations!! 😊