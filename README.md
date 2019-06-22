# Setup Saltstack in KVM VM using Vagrant

Instructions
============

Run the following commands in a terminal. Git, KVM, Plugins and Vagrant must
already be installed.

    git clone https://github.com/coolpalani/vagrant-on-fedora27

if you're not installed vagrant or kvm then you can follow the instructions and install the tools.


    git clone https://github.com/coolpalani/saltstack-vagrant
    vagrant up

This will download an CentOS Linux 7/x86_64 Vagrant images and spin up two VMs.

    salt-master
    salt-minion

Login on Salt Master instance as root user and run the bootstrap script to install the salt-master component. So from master terminal run:


    cd ~/saltstack-vagrant/salt-master && vagrant ssh
	sudo su - 
	curl -L https://bootstrap.saltstack.com -o install_salt
	chmod +x install_salt
	./install_salt -M -N

The last command will only install the Salt Master component.

Let’s take a look at the command options:

    -M → install salt-master
    -N → do not install salt-minion

Now, open a new terminal on your host which we will call "minion terminal" and login to the Salt Minion instance to install the minion component:


    cd ~/saltstack-vagrant/salt-master && vagrant ssh
	sudo su - 
	curl -L https://bootstrap.saltstack.com -o install_salt
	chmod +x install_salt
	./install_salt -A 192.168.122.2 -i webserv

The last command will install only the Salt Minion component:

    -A → Pass the salt-master DNS name or IP. In our case, it’s the IP that we’ve assigned previously, during the creation of the Salt Master vagrant instance.
    -i → set the minion id

Now we have to add that minion to the master, so switch to the master terminal and run these commands:


	salt-key

	Output:

	Accepted Keys:
	Denied Keys:
	Unaccepted Keys:
	webserv
	Rejected Keys:

Salt Master has received the minion request, so we can add the minion by running:


	salt-key -a webserv -y

That’s it! Now we can run some test commands from master terminal to verify that it works:


	salt webserv test.ping


Output:

	webserv:True

States

The core of the Salt State system is the SLS, or SaLt State file, which is built on top of modules. The SLS file is a representation of the state in which a system should be in and is set up to contain this data in a simple format. By default, Salt represents the SLS data in YAML (YAML Ain’t Markup Language), a human-readable data serialization language.

By default, Salt State files are stored in the filesystem on path /srv/salt. 
For instance, we can create a state file named install_network_packages.sls in the /srv/salt . So from master terminal run:


	mkdir /srv/salt
	cat <<EOF > /srv/salt/install_network_packages.sls
	install_network_packages:
	  pkg.installed:
	    - pkgs:
	      - wget
	      - httpd
	EOF

That state makes sure that listed packages are installed on systems. Now, to apply that state we need to run the state.apply module passing the state file name as argument without the extension:


	salt webserv state.apply install_network_packages

Grains

Salt comes with an interface to derive information about the underlying system. This is called the Grains interface. Grains are collected for the operating system, domain name, IP address, kernel, OS type, memory, and many other system properties.

These grains are also expandable to include other bits of static information that you’d like to have assigned to a minion such as a custom role. You can use these grains to retrieve information from minions dynamically.

For instance, to retrieve all grains from systems we can run the following command from master terminal:


	salt webserv grains.items

You can also use grains in a Salt State file. You can access it via grains[‘key’]:

For example, let’s create a state file with the above content and save it in /srv/salt/install_apache.sls:


	cat <<EOF > /srv/salt/install_apache.sls
	Install apache:
	  pkg.installed:
	    {% if grains['os_family'] == 'RedHat' %}
	    - name: httpd
	    {% elif grains['os_family'] == 'Debian' %}
	    - name: apache2
	    {% endif %}
	EOF


	salt webserv state.apply install_apache

In the state above, grains are used to choose which apache package should be installed based on the target Operating System.

Pillar

The Pillar system lets you define secure data (variables) that is ‘assigned’ to one or more minions, using targets. You can store values such as ports, file paths, configuration parameters, and passwords.

Salt pillar uses a Top file to match Salt pillar data to Salt minions.

For example, we can assign the pillar defined in a file to our webserv minion. So from the master terminal, create the /srv/pillar directory, and then create a new file called top.sls in the new pillar directory:


	mkdir /srv/pillar
	cat <<EOF > /srv/pillar/top.sls
	base:
	  'webserv':
	    - default_users
	EOF

Next, create a file named default_users.sls in the same pillar directory:


	cat <<EOF > /srv/pillar/default_users.sls
	users:
	  john: /bin/bash
	  peter: /bin/shell
	  jacob: /bin/shell
	  andrew: /bin/bash
	EOF

This file contains a simple data structure, which is a list of users with each one assigned value.

Now, in order to retrieve the pillar data we need to refresh pillars by running:

	salt '*' saltutil.refresh_pillar

and then we can get that data by calling the execution module:

	
	salt webserv pillar.get users

	Output:

	webserv:
	    ----------
	    andrew:
	        /bin/bash
	    jacob:
	        /bin/shell
	    john:
	        /bin/bash
	    peter:
	        /bin/shell

The more powerful aspect of this component is that Salt pillar keys are available in a dictionary, in Salt states.

For example, create the /srv/salt/users directory and then a file called init.sls in that directory:


	mkdir /srv/salt/users
	cat <<EOF > /srv/salt/users/init.sls
	{% for user, shell in pillar.get('users', {}).items() %}
	Create the user {{user}}:
	  user.present:
	    - name: {{user}}
	    - shell: {{shell}}
	{% endfor %}
	EOF

So, we can execute that state by running:

salt webserv state.apply users

    Note: As you know, /srv/salt/users is a directory. In this case, Salt will check if a init.sls file exists in that directory and if found runs the content of that init.sls state.

The above state defines a "for cycle" that iterate on "users" pillar and create a user with correspondent shell associated for each iteration.

Beacons

Beacons let you use the event system to monitor processes continually. So when monitored activity occurs in a system process, an event is sent on the event bus.

Let’s enable your first beacon by creating the configuration file /etc/salt/minion.d/beacons.conf from the minion terminal:


	cat <<EOF > /etc/salt/minion.d/beacons.conf
	beacons:
	  service:
	    - services:
	        httpd:
	          onchangeonly: True
	          delay: 5
	EOF

then restart the minion service:


	systemctl restart salt-minion

The above example will fire an event 5 seconds after the state of httpd service changes.

In order to check the configured monitoring events in real-time, Salt provides a command that displays them as they are received on the Salt Event Bus.

So check them by running, from the master terminal:


	salt-run state.event pretty=True

Next, switch on the minion terminal and run:
	
	systemctl restart httpd

After 5 seconds you should see on the master terminal, a similar event fired on the Salt Event Bus:


	salt/beacon/webserv/service/httpd	{
	    "_stamp": "2019-06-22T07:12:50.679443", 
	    "httpd": {
	        "running": true
	    }, 
	    "id": "webserv", 
	    "service_name": "httpd"
	}

Then press Ctrl+c on the master terminal to exit the routine.

So, we have a way to monitor our services but now we want to do something more than that. For example, you may wish to send alerts or run a remediation flow. To accomplish this task we can use the Salt Reactor System.

Reactor

Reactor system gives Salt the ability to trigger actions in response to an event. It’s an interface that watches Salt’s event bus for event tags that match a given pattern, then runs one or more actions in response.

Let’s begin by configuring a simple reactor state, based on the previously configured beacon.

On the master terminal create the /srv/salt/reactor directory:


	mkdir /srv/salt/reactor

Next, create a file in that directory named restart_httpd.sls:

	cat <<EOF > /srv/salt/reactor/restart_httpd.sls
	{%- if data[data['service_name']]['running'] == False %}
	start {{data['service_name']}}:
	  local.service.start:
	  - tgt: {{data['id']}}
	  - args:
	    - name: {{data['service_name']}}
	{%- endif %}
	EOF

Then, we need to tag it with your beacon in the master config file /etc/salt/master.d/reactor.conf:


	cat <<EOF > /etc/salt/master.d/reactor.conf
	reactor:
	  - 'salt/beacon/*/service/*':
	    - salt://reactor/restart_httpd.sls
	EOF

Now restart the salt-master service to reload the configurations.


	systemctl restart salt-master

and watch again the Event Bus:


	salt-run state.event pretty=True

Ok, on minion terminal let’s try to stop the httpd service again and wait for 5 seconds:


	systemctl stop httpd

You should see on master terminal a series of events that represent our newly configured remediation flow.

Next, try to check the status of httpd service:


	systemctl status httpd

The service is up again!

Highstate

In order to manage groups of machines, an administrator needs to be able to create roles for those groups. In Salt, the file which contains a mapping between groups of machines on a network and their configuration roles is called top file.

Let’s check the Salt root directory tree:


	/srv/salt
	├── install_apache.sls
	├── install_network_packages.sls
	├── reactor
	│   └── restart_httpd.sls
	└── users
	    └── init.sls

Alright, from master terminal, let’s create the top.sls file in the /srv/salt directory and map some state files to some minion targets:


	cat <<EOF > /srv/salt/top.sls
	base:
	  # All minions get the following two state files applied
	  '*':
	    - install_network_packages
	    - install_apache# Minions that have a grain set indicating that they are
	  # running the Debian operating system will have the state file 
	  # called 'users' applied.
	  'os_family:Debian':
	    - match: grain
	    - users
	EOF

and run:


	salt webserv state.apply

	salt webserv state.apply install_apache
webserv:
----------
          ID: Install apache
    Function: pkg.installed
        Name: httpd
      Result: True
     Comment: The following packages were installed/updated: httpd
     Started: 07:12:03.170142
    Duration: 17704.395 ms
     Changes:   
              ----------
              httpd:
                  ----------
                  new:
                      2.4.6-89.el7.centos
                  old:

This action is referred to as a "highstate". When this function is executed, a minion will download the top file and attempt to match the expressions within it. When the minion does match an expression the modules listed for it will be downloaded, compiled, and executed.

Orchestrate

While the execution of states or highstate is perfect when you want to ensure that the minion is configured in the way you want, sometimes you want to configure a set of minions all at once.

For example, if you want to set up a load balancer in front of a cluster of web servers you can ensure the web servers are set up first, and then the configuration is applied consistently to the load balancer.
