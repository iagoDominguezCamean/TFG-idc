require_relative 'provisioning/vbox.rb'
VBoxUtils.check_version('7.0.6')
Vagrant.require_version ">= 2.3.4"

require 'yaml'
custom_settings = YAML.load_file('settings.yml')

class VagrantPlugins::ProviderVirtualBox::Action::Network
  def dhcp_server_matches_config?(dhcp_server, config)
    true
  end
end

# Hostnames for master and worker nodes
MASTER_HOSTNAME = "idc-tfg-k8s-master"
MASTER_SEC_HOSTNAME = "idc-tfg-k8s-sec-master"
WORKER_HOSTNAME = "idc-tfg-k8s-worker"
LB_HOSTNAME = "idc-tfg-lb"

# Cluster settings
MASTER_IP = custom_settings['master_ip']
MASTER_CORES = custom_settings['master_cores']
NUM_SEC_MASTERS = custom_settings['num_sec_masters']
NUM_WORKERS = custom_settings['num_workers']
WORKER_CORES = custom_settings['worker_cores']
MASTER_MEMORY = custom_settings['master_mem']
WORKER_MEMORY = custom_settings['worker_mem']
POD_NETWORK = custom_settings['pod_network']
CLUSTER_ENDPOINT = custom_settings['cluster_endpoint']
NFS_SERVER_IP = custom_settings['nfs_server_ip']
NFS_SERVER_NET = custom_settings['nfs_server_net']
NFS_SERVER_NAME = custom_settings['nfs_server_name']
NFS_SERVER_PATH = custom_settings['nfs_server_path']
CLUSTER_ENDPOINT_IP = custom_settings['cluster_endpoint_ip']
CLUSTER_ENDPOINT_NAME = custom_settings['cluster_endpoint_name']

require 'ipaddr'
CLUSTER_IP_ADDR = IPAddr.new MASTER_IP
CLUSTER_IP_ADDR = CLUSTER_IP_ADDR.succ


Vagrant.configure("2") do |config|
  
  config.vm.box = "generic/rocky8"
  config.vm.box_check_update = false
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

  # Configure hostmanager and vbguest plugins
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.vbguest.auto_update = false

  config.vm.define "master", primary: true do |master|
    master.vm.hostname = MASTER_HOSTNAME
    master.vm.network "private_network", ip: MASTER_IP
    
    master.vm.provider "virtualbox" do |prov|
	    prov.name = "TFG-#{master.vm.hostname}"
      prov.cpus = MASTER_CORES
      prov.memory = MASTER_MEMORY
	    prov.gui = false
      
      disk = ".vagrant\\machines\\master\\nfsdsk.vmdk"
      sataController = "SATA Controller"
      
      # Create the virtual disk if doesn't exist
      unless File.exist?(disk)
        prov.customize ["createmedium", "disk", "--filename", disk, "--format", "VMDK", "--size", 20480]
      end

      # Attach the virtual disk into the storage SAS controller
      prov.customize ["storageattach", :id, "--storagectl", sataController, "--port", 1, "--device", 0, "--type", "hdd", "--medium", disk]
    end

    # Install and setup K8s using ansible
    master.vm.provision "ansible_local", run: "once" do |ansible|
	    ansible.install = "true"
	    ansible.install_mode = "pip3"
	    ansible.playbook = "provisioning/playbook-main.yml"
      ansible.inventory_path = "ansible.inventory"
      ansible.limit = "all"
	    ansible.extra_vars = {
        master_ip: MASTER_IP,
        master_hostname: MASTER_HOSTNAME,
        pod_network: POD_NETWORK,
        cluster_endpoint: CLUSTER_ENDPOINT,
        nfs_server_ip: NFS_SERVER_IP,
        nfs_server_net: NFS_SERVER_NET,
        nfs_server_path: NFS_SERVER_PATH,
        nfs_server_name: NFS_SERVER_NAME,
        cluster_endpoint_ip: CLUSTER_ENDPOINT_IP,
        cluster_endpoint_name: CLUSTER_ENDPOINT_NAME,
      }
    end
    
  end
  # Secondary master nodes
  (2..NUM_SEC_MASTERS+1).each do |i|
    config.vm.define "master-#{i}" do |masters|
      masters.vm.hostname = "#{MASTER_HOSTNAME}-#{i}"
      IP_ADDR = CLUSTER_IP_ADDR.to_s
      CLUSTER_IP_ADDR = CLUSTER_IP_ADDR.succ
      masters.vm.network "private_network", ip: IP_ADDR

      masters.vm.provider "virtualbox" do |prov|
        prov.name = "TFG-#{masters.vm.hostname}"
        prov.cpus = MASTER_CORES
        prov.memory = MASTER_MEMORY
        prov.gui = false
      end
    end
  end

  # Worker nodes
  (1..NUM_WORKERS).each do |i|
    config.vm.define "worker-#{i}" do |worker|
	    worker.vm.hostname = "#{WORKER_HOSTNAME}-#{i}"
	    IP_ADDR = CLUSTER_IP_ADDR.to_s
      CLUSTER_IP_ADDR = CLUSTER_IP_ADDR.succ
      worker.vm.network "private_network", ip: IP_ADDR
        
      worker.vm.provider "virtualbox" do |prov|
        prov.name = "TFG-#{worker.vm.hostname}"
        prov.cpus = WORKER_CORES
        prov.memory = WORKER_MEMORY
        prov.gui = false
      end
    end
  end
  
  # Global provisioning bash script
  config.vm.provision "shell", run: "once", path: "provisioning/bootstrap.sh"

end
