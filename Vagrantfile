# -*- mode: ruby -*-
# vi: set ft=ruby :
VAGRANTFILE_API_VERSION = '2' unless defined? VAGRANTFILE_API_VERSION

# Absolute paths on the host machine.
host_solrvm_dir = File.dirname(File.expand_path(__FILE__))
host_project_dir = ENV['SOLRVM_PROJECT_ROOT'] || host_solrvm_dir
host_config_dir = ENV['SOLRVM_CONFIG_DIR'] ? "#{host_project_dir}/#{ENV['SOLRVM_CONFIG_DIR']}" : host_project_dir

# Absolute paths on the guest machine.
guest_project_dir = '/vagrant'
guest_solrvm_dir = ENV['SOLRVM_DIR'] ? "/vagrant/#{ENV['SOLRVM_DIR']}" : guest_project_dir
guest_config_dir = ENV['SOLRVM_CONFIG_DIR'] ? "/vagrant/#{ENV['SOLRVM_CONFIG_DIR']}" : guest_project_dir

# Cross-platform way of finding an executable in the $PATH.
def which(cmd)
  exts = ENV['PATHEXT'] ? ENV['PATHEXT'].split(';') : ['']
  ENV['PATH'].split(File::PATH_SEPARATOR).each do |path|
    exts.each do |ext|
      exe = File.join(path, "#{cmd}#{ext}")
      return exe if File.executable?(exe) && !File.directory?(exe)
    end
  end
  nil
end

def walk(obj, &fn)
  if obj.is_a?(Array)
    obj.map { |value| walk(value, &fn) }
  elsif obj.is_a?(Hash)
    obj.each_pair { |key, value| obj[key] = walk(value, &fn) }
  else
    obj = yield(obj)
  end
end

require 'yaml'
vconfig = YAML.load_file("#{host_solrvm_dir}/default.config.yml")
# Use optional config.yml and local.config.yml for configuration overrides.
['config.yml', 'local.config.yml'].each do |config_file|
  if File.exist?("#{host_config_dir}/#{config_file}")
    vconfig.merge!(YAML.load_file("#{host_config_dir}/#{config_file}"))
  end
end

# Replace jinja variables in config.
vconfig = walk(vconfig) do |value|
  while value.is_a?(String) && value.match(/{{ .* }}/)
    value = value.gsub(/{{ (.*?) }}/) { vconfig[Regexp.last_match(1)] }
  end
  value
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Networking configuration.
    config.vm.hostname = vconfig['vagrant_hostname']
    if vconfig['vagrant_ip'] == '0.0.0.0' && Vagrant.has_plugin?('vagrant-auto_network')
      config.vm.network :private_network, ip: vconfig['vagrant_ip'], auto_network: true
    else
      config.vm.network :private_network, ip: vconfig['vagrant_ip']
    end

    if !vconfig['vagrant_public_ip'].empty? && vconfig['vagrant_public_ip'] == '0.0.0.0'
      config.vm.network :public_network
    elsif !vconfig['vagrant_public_ip'].empty?
      config.vm.network :public_network, ip: vconfig['vagrant_public_ip']
    end

    # SSH options.
    config.ssh.insert_key = false
    config.ssh.forward_agent = true

    # Forward solr port.
    config.vm.network :forwarded_port, guest: vconfig['solr_port'], host: vconfig['solr_port']

    # Vagrant box.
    config.vm.box = vconfig['vagrant_box']

  # VirtualBox.
  config.vm.provider :virtualbox do |v|
    v.linked_clone = true if Vagrant::VERSION =~ /^1.8/
    v.name = vconfig['vagrant_hostname']
    v.memory = vconfig['vagrant_memory']
    v.cpus = vconfig['vagrant_cpus']
    v.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
    v.customize ['modifyvm', :id, '--ioapic', 'on']
  end

    # Provisioning.
    # Use ansible if it's installed, ansible_local if not or if forced.
    if which('ansible-playbook') && !vconfig['force_ansible_local']
      config.vm.provision 'ansible' do |ansible|
        ansible.playbook = "#{host_solrvm_dir}/provisioning/playbook.yml"
        ansible.extra_vars = {
          config_dir: host_config_dir
        }
      end
    else
      config.vm.provision 'ansible_local' do |ansible|
        ansible.playbook = "#{guest_solrvm_dir}/provisioning/playbook.yml"
        ansible.extra_vars = {
          config_dir: guest_config_dir
        }
      end
    end

    # Set the name of the VM. See: http://stackoverflow.com/a/17864388/100134
      config.vm.define vconfig['vagrant_machine_name']
end
