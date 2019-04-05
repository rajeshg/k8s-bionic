Vagrant.configure("2") do |config|
  config.vm.provider :virtualbox do |v|
    v.memory = 2048
    v.cpus = 2
  end

  config.vm.provision :shell, privileged: true, inline: $install_common_tools

  config.vm.define :master do |master|
    master.vm.box = "ubuntu/bionic64"
    master.vm.hostname = "master"
    master.vm.network :private_network, ip: "10.0.0.10"
    master.vm.provision :shell, privileged: false, inline: $provision_master_node
	master.vm.synced_folder "D:/sde", "/vagrant_sde"
  end

  %w{worker1 worker2}.each_with_index do |name, i|
    config.vm.define name do |worker|
      worker.vm.box = "ubuntu/bionic64"
      worker.vm.hostname = name
	  worker.vm.synced_folder "D:/sde", "/vagrant_sde"
      worker.vm.network :private_network, ip: "10.0.0.#{i + 11}"
      worker.vm.provision :shell, privileged: false, inline: <<-SHELL
sudo /vagrant/join.sh
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.#{i + 11}"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL
    end
  end

  config.vm.provision "shell", inline: $install_multicast
end


$install_common_tools = <<-SCRIPT
echo "----------------------------------------------------"
echo "             Installing common tools                "
echo "----------------------------------------------------"
# disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Install kubeadm, kubectl and kubelet
echo "Installing docker ....."
export DEBIAN_FRONTEND=noninteractive
apt-get -qq update
apt-get -qq install -y docker.io apt-transport-https curl
# Setup daemon with cgroupdriver
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl enable docker.service

echo "Installing k8s ....."

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get -qq update
apt-get -qq install -y kubelet kubeadm kubectl
echo "source <(kubectl completion bash)" >> ~/.bashrc
SCRIPT

$provision_master_node = <<-SHELL
echo "----------------------------------------------------"
echo "             Provisioning master node               "
echo "----------------------------------------------------"
OUTPUT_FILE=/vagrant/join.sh
rm -rf $OUTPUT_FILE

echo "Initializing cluster with kubeadm & Setting up calico network for the cluster  ..."

# Refer https://docs.projectcalico.org/v3.6/getting-started/kubernetes/
# Start cluster
sudo kubeadm init --apiserver-advertise-address=10.0.0.10 --pod-network-cidr=192.168.0.0/16
sudo kubeadm token create --print-join-command > ${OUTPUT_FILE}
chmod +x $OUTPUT_FILE

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Fix kubelet IP
echo 'Environment="KUBELET_EXTRA_ARGS=--node-ip=10.0.0.10"' | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Configure Calico
kubectl apply -f https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

sudo systemctl daemon-reload
sudo systemctl restart kubelet
SHELL

$install_multicast = <<-SHELL
apt-get -qq install -y avahi-daemon libnss-mdns
SHELL