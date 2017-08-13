# P161Hi
P161 technical test.

## Abstract
This Vagrant project creates a PoC about a RoR platform made up by two Linux hosts (database and Rails frontend). We are free to choose the architecture, to certain extent.

## Problem description / requirements

We would like to configure a new setup for a rails app. We need a Vagrant file and related code to create a two hosts environment. App has to be working after all deployment tasks.

We must use default rails app (rails appname) to generate it. A simple message has to be shown. It has to use Ruby 2.3.1

The deployment has to be made from the outside. That is, the application is not generated in the same server as it is deployed. Feel free to choose a deployment / distribution method (tar, git, package, ...). The same is required for the server that will serve the rails app.

Using any database is fine, but no SQLite file. The database service is expected to be on a separate server.

The two hosts will need, thus, connectivity. Database has to be accesible.

App server needs to run after the database in some way.

Vagrant and Virtualbox are mandatory. Running `vagrant up` and checking the message in the app at the end of the deployment is the goal.

We are free to use any CM tool.

Plan a redeployment strategy.

## Proposed solution

As I said, we're using Vagrant with VirtualBox provider.
There will be two Debian 8 hosts.
The first one running a MySQL database, listening the public interface.
The second one will run a Ruby on Rails stack and could access the database services on the former.

### Vagrant

For this project, I've used the following software versions. It's expected the project works with newer ones.

    $ vagrant -v
    Vagrant 1.8.1

    $ vboxmanage -v
    5.0.40_Ubuntur115130

    $ ansible --version
    ansible 2.0.0.2
      config file = /etc/ansible/ansible.cfg
      configured module search path = Default w/o overrides

Each Virtualbox VM is provisioned with as many CPU cores as possible. That is, with a maximum as the host machine.

### Debian box

At first I was planning on using the latest Debian box available. That is, [debian/stretch64](https://app.vagrantup.com/debian/boxes/stretch64) , but it seems that it needs a newer version of Vagrant to work fine.
I settled for stability and because of that I used [debian/jessie64](https://app.vagrantup.com/debian/boxes/jessie64), which offers Debian 8.9 with a deserved reputation of stability.

Whichever Debian version I choose, I have the certainty that I can trust in its tight and reliable dependency system to, in turn, having every dependency of a single page just met with an install command.

### Networking

I've configured both machines with an interface in a privat network `192.168.5.0/24` .
They can see each other, and I have established a port forwarding in the Rails machine, so we can check the app from our browser outside:

    host.vm.network :forwarded_port, guest: 3000, guest_ip: "192.168.5.12", host: 3000, protocol: "tcp"

### Provisioning: Ansible

I've chosen Ansible as my configuration management tool because:

- Its provisioner integrates [perfectly with Vagrant](https://www.vagrantup.com/docs/provisioning/ansible.html)
- Ansible has a lot of modules and even after that, I can use `command` or `shell` to do additional tasks, having it all together as a whole.
- So then, I can use only one provisioner to do it all
- Is agentless, so we need no other infrastructure to provision this project. In contrast with the Puppet provisioner, you can do it agent or agentless, but to me makes more sense to use Puppet with a Puppet master, when needed.

### db1 host

As I've been told, MySQL host is the fist one to start up.
Once it's running, with Ansible we apply a role to it.
This role can be applied to any number of hosts.

For the purpose of this PoC, I've ommited the customization of some variables and things like that, because we'll only need one host.

- Installation of MySQL server and python-mysqldb (needed by Ansible on the client side).
- MySQL listener configuration and restart trigger
- Sample MySQL user created so it can access this DB from other hosts

### front1 host

We've been told that the Rails host should meet some requirements:

- Ruby 2.3.1
- Rails installed
- As a consequence, every dependency, being a .deb package or a gem, should be installed previous to the sample application startup.

My chosen Vagrant box packs a Ruby package whose version doesn't meet this requirements, `2.1.5+deb8u2` .

As I'm unfamiliar to Rails for the time being, I've read some about the [alternatives](https://noteits.net/2016/06/10/installing-ruby-2-3-1-on-ubuntu/)
It seems that Ruby 2.3.1 is a pretty popular version, and fortunately there's plenty of literature about methods to deploy this exact version.

Basically, for me there's three ways to do it:

- `rbenv` . Having a Debian package is my preferred way to do it, but it seems that the packed version of the tool for Debian 8.9 can't meet the installation of Ruby 2.3.1 (only older versions). I'll try to avoid compilation processes in a machine that is supposed to be a production environment.
- `rvm` is a famous tool to manage Ruby versions. Again, I prefer to use the fewer compilation processes, so I ended following [these instructions](https://rvm.io/rvm/install#ubuntu) to install a .deb package that meets my expectations,  __so this will be my choice__.
It is done through a PPA repository, added by Ansible during the provisioning.
- Source code compilation. As the less desired option, was discarded.

Once `rvm` is installed, there are additional tasks that install the version 2.3.1 specified by var `ruby_vers` . Can be changed to test different environments.
With a working Ruby environment, I've made a list of the required gems, begining with `rails` and including some gems needed by our sample application.

Now we must download an initial version of a cooked app from somewhere (using Github repo as my personal artifact repository)
We can use any other, like Nexus or Artifactory.

### Sample application
I've followed [this quick tutorial](http://iridakos.com/2013/11/24/saying-hello-world-with-ruby-on-rails.html) to build a sample application.

Steps:

- `rails new sampleapp --database=mysql`
- `cd sampleapp`
- edited config/database.yml with our information about host db1
- `rails generate controller pages`
- edited `app/controllers/pages_controller.rb` to say "Hi P161"
- Created the file `app/views/pages/home.html.erb`
- edited `config/routes`

I added a script to perform certain tasks with `bundle` after we uncompress the sampleapp.tgz downloaded file.

### Application autodeployment

I wrapped both the sampleapp Rails application and a script to perform as much tasks as we want, versioned for each tar file.

user chosen: ruby
method employed.
autodownload

autodeploy scripts explained
locally installed 'bundle'
startup scripts

### Proposed solution to replace the application version.

change variables to do it. redownload. label every version.

### Final notes

deployment notes. balance between what Ansible do and what the deployment script do

different methods of replacing app

development stack: ubuntu 16.04 + versions of vagrant + virtualbox + git + editor + ...