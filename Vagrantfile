# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = "bento/centos-7.2"
  # config.vm.network "private_network", ip: "10.0.2.15"


  #config.vm.box_check_update = true
  config.vbguest.auto_update = false
  #config.vbguest.auto_reboot = true

  # workaround the vagrant 1.8.5 bug
  config.ssh.insert_key = false

  # change memory size
  config.vm.provider "virtualbox" do |v|
    v.memory = 6000 # 虚拟机的内存大小为 6000MB，即 6GB。
    v.cpus = 2 # 虚拟机可用的 CPU 核心数量为 2 个。
    v.name = "oracle12c-vagrant"
    v.customize ["modifyvm", :id, "--cableconnected1", "on"] # 确保虚拟机的网络适配器处于连接状态。
    v.customize ["modifyvm", :id, "--ioapic", "on"] # 将虚拟机的 IO APIC（高级可编程中断控制器）设置为 "on"，以改善虚拟机的性能。
    v.customize ["modifyvm", :id, "--cpuexecutioncap", "100"] # 设置了 CPU 执行能力限制为 100%，这意味着虚拟机可以充分利用物理主机的 CPU 资源，没有限制。
  end

  # Oracle port forwarding
  config.vm.network "forwarded_port", guest: 1521, host: 1521

  # Provision everything on the first run
  # config.vm.provision "shell", path: "scripts/install.sh"

  config.vm.synced_folder ".", "/vagrant"
  # config.vm.synced_folder "./oradata", "/vagrant/oradata", owner: "oracle", group: "oinstall"

  # if Vagrant.has_plugin?("vagrant-proxyconf")
  #   config.proxy.http     = "http://10.0.2.2:7890"
  #   config.proxy.https    = "http://10.0.2.2:7890"
  #   config.proxy.no_proxy = "localhost,127.0.0.1,mirrors.aliyun.com"
  # end
end
