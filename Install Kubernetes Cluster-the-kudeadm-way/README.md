# Install Kubernetes Cluster-the kubeadm-way

1. Use `Vagrantfile` from the given repository https://github.com/kodekloudhub/certified-kubernetes-administrator-course to bring up the 1 master, 2 worker-cluster setup.

   ```sh
   vagrant up
   ```

   Once the setup is done, you should see the following when you `vagrant status` in the terminal.

   ```sh
   Current machine states:
   
   kubemaster                running (virtualbox)
   kubenode01                running (virtualbox)
   kubenode02                running (virtualbox)
   
   This environment represents multiple VMs. The VMs are all listed
   above with their current state. For more information about a specific
   VM, run `vagrant status NAME`
   ```

2. Login into separate sessions into the brought up VMs.

   ```sh
   vagrant ssh kubemaster
   ```

   ```sh
   vagrant ssh kubenode01
   ```

   ```sh
   vagrant ssh kubenode02
   ```

3. **On all the nodes, run the following commands:**

   ```sh
   sudo modprobe br_netfilter && lsmod | grep br_netfilter
   ```

   This should give you the following output:

   ```sh
   br_netfilter           24576  0
   bridge                155648  1 br_netfilter
   ```

   Further,

   ```sh
   sudo -i
   # letting iptables see bridged traffic
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   br_netfilter
   EOF
   
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   sudo sysctl --system
   
   ## installing container-runtime {DOCKER}
   # removing old versions (if any)
   sudo apt-get remove docker docker-engine docker.io containerd runc
   # setting up apt repository
   sudo apt-get update 
   sudo apt-get install \
       ca-certificates \
       curl \
       gnupg \
       lsb-release 
   curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
     $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   # installing docker runtime
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io
   # configuring docker daemon to use systemd for the management of the container's cgroups
   sudo mkdir /etc/docker
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
   # restart and enable on boot
   sudo systemctl enable docker
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   
   ## installing kubeadm, kubectl and kubelet
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
   sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
   echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   
   # since we are install docker we do not need cgroup driver
   ```

4. On master node, run 

   ```sh
   kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.56.2
   ```

   > `--control-plane-endpoint` should be outside networking solution CIDR.
   > `--apiserver-advertise-address` has IP of `enp0s8`. (Use `ifconfig enp0s8` and get `inet` field value.)

   Once the command is complete, you should get the following as the output.

   ```sh
   Your Kubernetes control-plane has initialized successfully!
   
   To start using your cluster, you need to run the following as a regular user:
   
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
   
   Alternatively, if you are the root user, you can run:
   
     export KUBECONFIG=/etc/kubernetes/admin.conf
   
   You should now deploy a pod network to the cluster.
   Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
     https://kubernetes.io/docs/concepts/cluster-administration/addons/
   
   Then you can join any number of worker nodes by running the following on each as root:
   
   kubeadm join 192.168.56.2:6443 --token oiaauw.qb1l8dgaknwk4b8j \
   	--discovery-token-ca-cert-hash sha256:1ce828ac52d733c603d3d265174bc1317bfafef51ec92f5b85e26300de7f3b60 
   ```

   **So on your master**, `logout` from `sudo` user access and run the following as a regular user.

   ```sh
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

   Now we install networking solution `Weave net` **on your master**.

   ```sh
   kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
   ```

   **Now on worker nodes,** run the following:

   ```sh
   kubeadm join 192.168.56.2:6443 --token oiaauw.qb1l8dgaknwk4b8j --discovery-token-ca-cert-hash sha256:1ce828ac52d733c603d3d265174bc1317bfafef51ec92f5b85e26300de7f3b60 
   ```

   **Now on master node**, run the following:

   ```sh
   watch kubectl get nodes
   # wait for all the nodes to come online
   ```

   

