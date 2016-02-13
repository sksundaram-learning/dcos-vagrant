# -*- mode: ruby -*-
# vi: set ft=ruby :

require "yaml"


## Config
##############################################

$user_config = {
  box:            ENV.fetch("DCOS_BOX", "mesosphere/dcos-centos-virtualbox"),
  box_url:        ENV.fetch("DCOS_BOX_URL", "https://downloads.mesosphere.com/dcos-vagrant/metadata.json"),
  box_version:    ENV.fetch("DCOS_BOX_VERSION", nil),

  vm_config_path:       ENV.fetch("DCOS_VM_CONFIG_PATH", "VagrantConfig.yaml"),
  config_path:          ENV.fetch("DCOS_CONFIG_PATH", "etc/1_master-config.json"),
  generate_config_path: ENV.fetch("DCOS_GENERATE_CONFIG_PATH", "dcos_generate_config.sh"),
  java_enabled:         ENV.fetch("DCOS_JAVA_ENABLED", "false"),
  private_registry:     ENV.fetch("DCOS_PRIVATE_REGISTRY", "false"),
}


## Config Validation
##############################################

if !File.file?($user_config[:vm_config_path])
  raise "vm config not found: DCOS_VM_CONFIG_PATH=#{$user_config[:vm_config_path]}"
end

if !File.file?($user_config[:generate_config_path])
  raise "dcos installer not found: DCOS_GENERATE_CONFIG_PATH=#{$user_config[:generate_config_path]}"
end

if !File.file?($user_config[:config_path])
  raise "dcos config not found: DCOS_CONFIG_PATH=#{$user_config[:config_path]}"
end


## VM Config
##############################################

$vagrant_config = YAML::load_file($user_config[:vm_config_path])


## VM Config Validation
##############################################

def master_ips()
  $vagrant_config.select{ |_, cfg| cfg["type"] == "master" }.map{ |_, cfg| cfg["ip"] }
end

if master_ips.empty?
  raise "vm config must contain at least one vm of type master"
end


## Provision Environment
##############################################

def vagrant_path(path)
  if ! /^\w*:\/\//.match(path)
    path = "file:///vagrant/" + path
  end
  return path
end

$provision_environment = {
  "DCOS_CONFIG_PATH" => vagrant_path($user_config[:config_path]),
  "DCOS_GENERATE_CONFIG_PATH" => vagrant_path($user_config[:generate_config_path]),
  "DCOS_JAVA_ENABLED" => $user_config[:java_enabled],
  "DCOS_PRIVATE_REGISTRY" => $user_config[:private_registry],
  "DCOS_MASTER_IPS" => master_ips.join(" "),
}

def provision_path(type)
  return "./provision/bin/#{type}.sh"
end


#### Setup & Provisioning
##############################################

Vagrant.configure(2) do |config|

  # configure the vagrant-hostmanager plugin
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.ignore_private_ip = false

  # configure the vagrant-vbguest plugin
  config.vbguest.auto_update = true

  $vagrant_config.each do |name,cfg|
    config.vm.define name do |vm_cfg|
      vm_cfg.vm.hostname = "#{name}.dcos"
      vm_cfg.vm.network "private_network", ip: cfg["ip"]

      # custom hostname aliases
      if cfg["aliases"]
        vm_cfg.hostmanager.aliases = %Q(#{cfg["aliases"].join(" ")})
      end

      # allow explicit nil values in the cfg to override the defaults
      vm_cfg.vm.box = cfg.fetch("box", $user_config[:box])
      vm_cfg.vm.box_url = cfg.fetch("box-url", $user_config[:box_url])
      vm_cfg.vm.box_version = cfg.fetch("box-version", $user_config[:box_version])

      vm_cfg.vm.provider "virtualbox" do |v|
        v.name = vm_cfg.vm.hostname
        v.cpus = cfg["cpus"] || 2
        v.memory = cfg["memory"] || 2048
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      end

      if cfg["forwards"]
        cfg["forwards"].each do |from,to|
          vm_config.vm.forward_port from, to
        end
      end

      vm_cfg.vm.provision "shell", name: "Certificate Authorities", path: provision_path("ca-certificates")
      if $user_config[:private_registry] == "true"
        vm_cfg.vm.provision "shell", name: "Private Docker Registry", path: provision_path("insecure-registry")
      end
      if cfg["type"]
        vm_cfg.vm.provision "shell", name: "DCOS #{cfg['type'].capitalize}", path: provision_path("type-#{cfg["type"]}"), env: $provision_environment
      end
    end
  end
end
