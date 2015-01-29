### Vagrant
Vagrant project is useful for having a generic way to build/start VMs from the command line, because while there can be more than one **"providers"** of VMs there are certainly common characteristics when describing a VM like memory size, network setup, shared folders, forwarded ports, etc. What do we mean by a **provider** for a Vagrant VM? Well you are probably familiar with **Virtualbox**(which happens to be the default provider in Vagrant) or **VmWare**. Those are just two of the possible providers for Vagrant VMs and you can pass into the **Vagrant** file custom configuration properties for each provider. 
Things has be evolving though and a Vagrant "template" for a VM can even be used to build and deploy instances on cloud providers like Digital Ocean.


I'm planning to use Vagrant to start a VM with a bare Ubuntu image, and then use Ansible on this fresh image to install and configure jenkins along with the plugins and dependencies(jdk).


Download and install [Virtualbox]() and [Vagrant](https://www.vagrantup.com/downloads.html).
The **Vagrantfile** describing my config looks like:
```
box      = 'ubuntu/trusty64'
url      = 'https://atlas.hashicorp.com/ubuntu/boxes/trusty64'
hostname = 'jenkins'
ip       = '192.168.5.99'
ram      = '512'

Vagrant::Config.run do |config|
  config.vm.box = box
  config.vm.box_url = url
  config.vm.host_name = hostname
  config.vm.network :hostonly, ip 

  # Set the Timezone to something useful
  config.vm.provision :shell, :inline => "echo \"Europe/Bucharest\" | sudo tee /etc/timezone && dpkg-reconfigure --frontend noninteractive tzdata"

  config.vm.customize [
    'modifyvm', :id,
    '--name', hostname,
    '--memory', ram
  ]
end

Vagrant.configure("2") do |config|
  config.vm.network "forwarded_port", guest: 8080, host: 9090
end
```
You can save it, and we can start the machine with **vagrant up** in the same directory.

It will take some time first as Vagrant pulls the image from the repository. Other runs will take less time.
To get into the instance you can run **vagrant ssh**

### Ansible
This work was based upon [ansible-jenkins repo](https://github.com/ICTO/ansible-jenkins)
Ansible is a great tool for provisioning - meaning the automation of installing/configuring stuff through scripts-. Some other provisioning tools you might have heard of: **Puppet**, **Chef**, **Salt**. 
But I rather like Ansible because it can perform the installation without the need to have something particular(like a specific agent running) other than a working ssh connection to the target fleet of machines which we want to set up. 
Also the syntax is quite simple with a low learning curve (which apparently puppet and chef do not have). 


Ansible is written in python so you need to have it instaled along with pip(the python package manager). After that it’s as simple as running **sudo pip install ansible**.

Simplistically put, Ansible through a **playbook** file defines a series of tasks to be run on specific hosts or a defined group of hosts. For ex given the [db servers], [tomcat servers] group you could build your playbook as to execute first some tasks on [db] servers then continue with tasks destined for [tomcat] group,then other tasks back to [db], etc.

Ansible helps with **idempotent operations** on hosts. This is a fancy way to say it could rerun your tasks(installation scripts) again on the target hosts(useful if for some reason parts of the scripts have failed) and Ansible will know to skip the parts that were executed succesfully. 

This is possible because Ansible can check back to see if a linux package is installed, a service is already running, file exists, etc. (Ofcourse  Ansible cannot always know by itself if it should consider an action succesfull or not so that it should not be run again - think of an execution of a .sh file that does a whole bunch of stuff-, you need to place a hint like "creates: /path/to/database" for ansible to check and not run them again.

Playbooks are run with an **inventory** which is simply a file into which a set of hosts and groups are defined something like:
```
machine1 ansible_ssh_port=2222 ansible_ssh_host=192.168.1.10
machine2 ansible_ssh_port=2222 ansible_ssh_host=192.168.1.11

[db]
machine1

[tomcat]
machine2
```

Since we need to our built vagrant machine already has the "vagrant" user with sudo rights created by default and vagrant has saved the ssh private key on the host. We just need to define our single machine in the inventory file we're going to call 'ansible.hosts' and run our playbook 'jenkins.yml' through the **ansible-playbook** command:

```bash
ansible-playbook -i ansible.hosts --private-key=./.vagrant/machines/default/virtualbox/private_key -u vagrant jenkins.yml
```

### Roles
It's a good ideea to introduce the concept of **Roles** because they are a good way to organize things as you can define and combine roles together: something like role "common", "tomcat", "mysql" and combine them 'common + mysql' to run on the [db] servers group of host and 'common + tomcat' on [tomcat]. 

Roles use a standard file structure, the file **main.yml** being the entrypoint for each folder.Not all forders are required to exist.
```
├── ansible.host <-- inventory file- the hosts/groups are defined
├── host_vars
│   └── jenkins <-- variables specific to each host
├── jenkins.yml
├── roles
│   ├── jdk8
│   │   ├── files
│   │   │   └── webupd8.key.asc
│   │   ├── tasks
│   │   │   ├── main.yml     #<-- this is executed 
│   │   │   └── webupd8.yml  #<-- this can be reference with 'include' command
│   │   └── vars
│   │       └── main.yml
│   └── jenkins
│       ├── defaults        #<-- lowest priority for variables shadowing
│       │   └── main.yml 
│       ├── files #<-- files referenced in for use with the copy resource
│       │   └── jenkins-ci.org.key 
│       ├── handlers	    #<--handlers are triggered on events
│       │   └── main.yml 
│       ├── tasks
│       │   ├── dependencies_deb.yml
│       │   ├── main.yml    #<-- this one gets executed
│       │   ├── plugins.yml #<-- this can be reference with 'import' command
│       │   └── repo.yml
│       ├── templates
│       │   └── hudson.tasks.Mailer.xml.j2 #<-- templates end in .j2
│       └── vars
│           └── main.yml    #<-- variables associated with this role
```

### Tasks
The **/tasks** directory is the directory where the actions are defined. Some examples of tasks
```yaml
# Add Jenkins repository
- name: Add Jenkins repository
  sudo: yes
  apt_repository: repo='{{ jenkins.deb.repo }}' state=present update_cache=yes
  
# Get latest Jenkins update file
- name: Get Jenkins updates
  sudo: yes
  get_url: url=http://updates.jenkins-ci.org/update-center.json dest={{ jenkins.updates_dest }} thirsty=yes mode=0440
  register: jenkins_updates
  
# Jenkins Update-center
- name: Update-center Jenkins
  sudo: yes
  action: "shell cat {{ jenkins.updates_dest }} | sed '1d;$d' | curl -X POST -H 'Accept: application/json' -d @- http://localhost:8080/updateCenter/byId/default/postBack"
  when: jenkins_updates.changed
  notify:
    - 'Restart Jenkins'
```
Task are executed in order they appear. You can specify also the user under which the tasks will run with **remote_user: yourname**.
In the example the tasks required sudo rights.


There is also the **'include'** directive to better organize and import other .yml files.

### Variables 
The **{{ jenkins.deb.repo }}** construct means a placeholder for the **jenkins.deb.repo** variable.
Variables are of course defined in the **/vars** folder inside the role. The **main.yml** looks like:
```
---
plugins:
   - 'ldap'
   - 'github'
   - 'translation'
jenkins_dest: /opt/jenkins
jenkins:
  deb:
    repo: 'deb http://pkg.jenkins-ci.org/debian binary/' # Jenkins repository
    dependencies: # Jenkins dependencies
      - 'git'
      - 'curl'
  cli_dest: '{{ jenkins_dest }}/jenkins-cli.jar' # Jenkins CLI destination
```
Indentations is important this is how the variable 'jenkins.deb.repo' is defined.

Variable values **can be overridden per specific host or groups of hosts** when the script runs against that host. This is done by creating a file with the name of the host in the **host_vars/**(or **group_vars/** if you plan on defining for groups) directory.

Variables passed into the command line have highest precedence. It's usefull if you plan to use Jenkins with ansible so we can pass into the installation scripts reference build id. 

#### Conditional execution of tasks
Tasks can be skipped if the **when** condition is false. The condition can simply be a variable to be defined like 'when: email is defined' or a condition on the result of a previous command which can be stored for reference down the line through the **register** keyword.
Lets look at an example where the registered variable 'plugins_installed' is evaluated through 
**plugins_installed.stdout.find('{{ item }}') == -1** - here the **'item' variable is injected as part of a loop iteration**-.

```yml
- name: List plugins
  shell: java -jar {{ jenkins.cli_dest }} -s http://localhost:8080 list-plugins | cut -f 1 -d ' '
  when: plugins is defined ##plugins is a variable defined in the upper example
  register: plugins_installed

# Install/update Jenkins plugins
- name: Install/update plugins
  command: java -jar {{ jenkins.cli_dest }} -s http://localhost:8080 install-plugin {{ item }}
  when: plugins_installed.changed and plugins_installed.stdout.find('{{ item }}') == -1
  with_items: plugins
  notify:
    - 'Restart Jenkins'
```

### Ansible Loops
In the above example I've also introduced **a loop** in ansible considering the fact that the 'plugins' variable is a list(defined in the variables ex). 
Notice the injection of the current iteration value through the **'{{item}}'** value and the conditional **when plugins_installed.stdout.find('{{ item }}') == -1** that will make the task execute only when the plugin item is not found in the output of the command that listed the plugins.


Also of note is the **notify** directive, which is like an emitter of events for which you can bind a set of actions whenever that event is triggered. That is the role of the **handlers**
 
### Handlers
Handlers are listening for events whenever one is triggered(notified) it executes. The following command will restart jenkins whenever the 'Restart Jenkins' is emitted.
```
- name: Restart Jenkins
  command: java -jar {{ jenkins.cli_dest }} -s http://localhost:8080 safe-restart
```

### Templates
Templates are great for generating and customization of configuration files. They use the jinja2 templating language for python. Needless to say that they are stored into the **/templates** directory inside the role.
```
- name: Configure Jenkins E-mail
  sudo: yes
  when: email is defined
  template: src=hudson.tasks.Mailer.xml.j2 dest={{ jenkins_lib }}/hudson.tasks.Mailer.xml owner=jenkins group=jenkins mode=0644
```

### Jenkins
Let's review the whole steps:

If you've checked out my github [repo](https://github.com/balamaci/jenkins-ansible) you can **check what plugins it will install** in the **/host_vars** folder in the **'jenkins'** file.

Running **vagrant up** will use the provided **Vagrant** file.

After the machine is up, it's just a matter of invoking ansible playbook command to do the jenkins installation using the inventory file and providing as user the 'vagrant' user(which is automatically created and added to the list of SUDOERS by vagrant when it built the image).
The playbook file jenkins.yml just combines the jdk + jenkins role

```bash
ansible-playbook -i ansible.hosts --private-key=./.vagrant/machines/default/virtualbox/private_key -u vagrant jenkins.yml
```


To install the plugins, the ansible script is using the [jenkins cli](https://wiki.jenkins-ci.org/display/JENKINS/Jenkins+CLI) which is supplied into the jenkins installation.

Hopefully after, will have a working Jenkins servers with the plugins of your choice available at http://localhost:9090(9090 port forwarding setup in the Vagrant file)


