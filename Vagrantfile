VAGRANTFILE_API_VERSION = "2"
box      = 'ubuntu/trusty64'
url      = 'https://atlas.hashicorp.com/ubuntu/boxes/trusty64'
hostname = 'myprecisebox'
domain   = 'example.com'
ip       = '192.168.5.99'
ram      = '512'

Vagrant::Config.run do |config|
  config.vm.box = box
  config.vm.box_url = url
  config.vm.host_name = hostname + '.' + domain
  config.vm.network :hostonly, ip

  config.vm.customize [
    'modifyvm', :id,
    '--name', hostname,
    '--memory', ram
  ]
end

#Vagrant.configure("2") do |config|
#  config.ssh.private_key_path = "~/.ssh/id_rsa"
#  config.ssh.username = "sbalamaci"
#  config.ssh.forward_agent = true
#end
