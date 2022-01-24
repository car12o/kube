# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'
set = YAML.load_file 'env.yml'

IMAGE_NAME = "generic/ubuntu2004"
MASTER_IP = set["MASTER_IP"]
NETWORK_CIDR = set["NETWORK_CIDR"]
ARR = MASTER_IP.split(".")
NODES_BASE_IP = "%s.%s.%s." % ARR
NODES_INDEX_IP = ARR[3].to_i
NODES = set["NODES"]
VM_MEMORY = set["VM_MEMORY"]
VM_CPU = set["VM_CPU"]

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox" do |v|
    v.memory = VM_MEMORY
    v.cpus = VM_CPU
  end

  # MASTER
  config.vm.define "k8s-master", primary: true do |master|
    HOSTNAME="k8s-master"
    master.vm.box = IMAGE_NAME
    master.vm.network "private_network", ip: MASTER_IP
    master.vm.hostname = HOSTNAME
    master.vm.synced_folder "./", "/vagrant/"
    master.vm.provision "setup", type: "shell", run: "once", args: [MASTER_IP,HOSTNAME,NETWORK_CIDR], inline: <<-SHELL
      apt-get update
      apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

      wget -qO - https://download.docker.com/linux/ubuntu/gpg | apt-key add -
      apt-add-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
      apt-get update
      apt-get install -y docker-ce docker-ce-cli containerd.io
      usermod -aG docker vagrant

      sed -i '/ swap / s/^/# /' /etc/fstab
      swapoff -a

      wget -qO - https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
      apt-add-repository "deb https://apt.kubernetes.io/ kubernetes-xenial main"
      apt-get update
      apt-get install -y kubelet kubeadm kubectl
      sh -c "echo 'KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs --node-ip=$1' >> /etc/default/kubelet"
      systemctl restart kubelet

      kubeadm init --apiserver-advertise-address="$1" --apiserver-cert-extra-sans="$1"  --node-name $2 --pod-network-cidr=$3
      sudo --user=vagrant mkdir -p /home/vagrant/.kube
      cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
      chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

      export KUBECONFIG=/etc/kubernetes/admin.conf
      kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

      sh -c "kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh"
      chmod +x /etc/kubeadm_join_cmd.sh
    SHELL

  master.vm.provision "ingress", type: "shell", run: "never", privileged: false, args: ["#{MASTER_IP}/24"], inline: <<-SHELL
      sudo wget -qO - https://baltocdn.com/helm/signing.asc | sudo apt-key add -
      sudo apt-add-repository "deb https://baltocdn.com/helm/stable/debian/ all main"
      sudo apt-get update
      sudo apt-get install -y helm

      helm repo add metallb https://metallb.github.io/metallb
      helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
      helm repo update

      cp /vagrant/metallb-values.yml ~/.kube/
      sed -i 's|{ADDRESS}|'$1'|g' ~/.kube/metallb-values.yml

      helm install --create-namespace -n metallb metallb metallb/metallb -f ~/.kube/metallb-values.yml
      helm install --create-namespace -n ingress-nginx ingress-nginx ingress-nginx/ingress-nginx
    SHELL
  end

  # NODES
  (1..NODES).each do |i|
    config.vm.define "node-#{i}" do |node|
        IP="#{NODES_BASE_IP}#{i + NODES_INDEX_IP}"
        HOSTNAME="node-#{i}"
        node.vm.box = IMAGE_NAME
        node.vm.network  "private_network", ip: IP
        node.vm.hostname = HOSTNAME
        node.vm.provision "setup", type: "shell", run: "once", args: [IP,MASTER_IP], inline: <<-SHELL
          apt-get update
          apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

          wget -qO - https://download.docker.com/linux/ubuntu/gpg | apt-key add -
          apt-add-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
          apt-get update
          apt-get install -y docker-ce docker-ce-cli containerd.io
          usermod -aG docker vagrant

          sed -i '/ swap / s/^/# /' /etc/fstab
          swapoff -a

          wget -qO - https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
          apt-add-repository "deb https://apt.kubernetes.io/ kubernetes-xenial main"
          apt-get update
          apt-get install -y kubelet kubeadm kubectl
          sh -c "echo 'KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs --node-ip=$1' >> /etc/default/kubelet"
          systemctl restart kubelet

          apt-get install -y sshpass
          sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@$2:/etc/kubeadm_join_cmd.sh .
          sh ./kubeadm_join_cmd.sh
        SHELL
    end
  end
end
