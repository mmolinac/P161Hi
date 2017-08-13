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

As a side plot, I have to plan a redeployment strategy.

## Proposed solution

As I said, we're using Vagrant with VirtualBox provider.
There will be two Debian 8 hosts.
The first one running a MySQL database, listening the public interface.
The second one will run a Ruby on Rails stack and could access the database services on the former.

Way to use this project:

- Clone this repository onto your working environment:

    $ git clone https://github.com/mmolinac/P161Hi.git

- Start the project:

    $ cd P161Hi

    $ vagrant up

- After the deployment of the two VMs, you can use the app, pointing your web broser to `http://127.0.0.1:3000`

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

Each Virtualbox VM is provisioned automatically by Vagrant with as many CPU cores as possible. That is, with the maximum as the host machine allows.

### Debian box

At first I was planning on using the latest Debian box available. That is, [debian/stretch64](https://app.vagrantup.com/debian/boxes/stretch64) , but it seems that it needs a newer version of Vagrant to work fine.
I settled for stability and because of that I used [debian/jessie64](https://app.vagrantup.com/debian/boxes/jessie64), which offers Debian 8.9 with a deserved reputation of stability.

Whichever Debian version I choose, I have the certainty that I can trust in its tight and reliable dependency system to, in turn, having every dependency of a single page just met with an install command.

### Networking

I've configured both machines with an interface in a private network `192.168.5.0/24` .
They can see each other, and I have established a port forwarding in the Rails machine, so we can check the app from our browser outside:

    host.vm.network :forwarded_port, guest: 3000, guest_ip: "192.168.5.12", host: 3000, protocol: "tcp"

### Provisioning: Ansible

I've chosen Ansible as my configuration management tool because:

- Its provisioner integrates [perfectly with Vagrant](https://www.vagrantup.com/docs/provisioning/ansible.html)
- Ansible has a lot of modules and even after that, I can use `command` or `shell` to do additional tasks, having it all together as a whole.
- So then, I can use only one provisioner to do it all
- Is agentless, so we need no other infrastructure to provision this project. In contrast with the Puppet provisioner, you can do it agent or agentless, but to me makes more sense to use Puppet with a Puppet master, when needed.

### db1 host

As I've been told, MySQL host should be the fist one to start up.
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
- As a consequence, every dependency, being a .deb package or a gem, should be installed prior to the sample application startup.

Besides, we'll create a user to run the application, and will be specified as a setting inside the Ansible playbook. Defaults to `ruby`, password `ruby`

My chosen Vagrant box packs a Ruby package whose version doesn't meet this requirements, `2.1.5+deb8u2` .

As I'm unfamiliar to Rails for the time being, I've read some about the [alternatives](https://noteits.net/2016/06/10/installing-ruby-2-3-1-on-ubuntu/)
It seems that Ruby 2.3.1 is a pretty popular version, and fortunately there's plenty of literature about methods to deploy this exact version.

Basically, for me there's three ways to do it:

- `rbenv` . Having a Debian package is my preferred way to do it, but it seems that the packed version of the tool for Debian 8.9 can't meet the installation of Ruby 2.3.1 (only older versions). I'll try to avoid compilation processes in a machine that is supposed to be a production environment.
- `rvm` is a famous tool to manage Ruby versions. Again, I prefer to use the fewer compilation processes, so I ended following [these instructions](https://rvm.io/rvm/install#ubuntu) to install a .deb package that meets my expectations,  __so this will be my choice__.
It is done through a PPA repository, added by Ansible during the provisioning.
- Source code compilation. As the less desired option, was discarded.

Once `rvm` is installed, there are additional tasks that install the version 2.3.1 specified by var `ruby_vers` in the Ansible playbook. Can be changed to test different environments.
With a working Ruby environment, I've only installed gem `rails` system-wide.
The rest of the gem needed by the application will be installed inside the user's home folder.
Check __Application autodeployment__ to know how it's done.

Now we must download an initial version of a cooked app from somewhere (using my Github repo as my personal artifact repository)
We could have used any other, like Nexus or Artifactory.

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

__The sample application has been generated in my laptop, and will be in a remote repository before Vagrant starts__

I added a script to perform certain tasks with `bundle` after we uncompress the sampleapp.tgz downloaded file.

    #!/bin/sh
    # Autodeployment of Rails sampleapp
    #
    script_name=$0
    base=`dirname $(readlink -f $script_name)`
    cd ${base}/sampleapp
    echo "Bundle install ..."
    bundle install --path vendor/bundle
    install_err=$?
    echo "Database creation/migration ..."
    bundle exec rake db:create
    db_err=$?
    if [ $install_err -eq 0 -a $db_err -eq 0 ]; then
      echo "Deployment OK."
      cd $base
      touch ${script_name}.ready
    else
      echo "Deployment error."
      exit 2
    fi

The script is very simple, but it's tied to their app version, so every version can be installed from inside Ansible playbook or by hand, and leaves a trace for this exact version.
Either way, if the autodeployment process goes well, every subsequent Ansible provision won't try to redeploy.

### Application autodeployment

I wrapped both the sampleapp Rails application and a script to perform as much tasks as we want, versioned for each tar.gz file.

By default, the artifact whose version is chosen will be downloaded from a repository set in the Ansible playbook. Of course, the repository, the name and appliation version can be changed at the user's discretion.

The deployment of the artifact is mandatory only the first time, because a system service will be set on top of that.

Every ansible execution after that will not result in actions taken if there's a correct application running.

### Proposed solution to deploy another version.

I propose two different ways to replace the running application.

- Fist, if you change application version in file `playbooks/roles/rails-frontend/vars/main.yml`, and issue `vagrant provision`, it will download other version from the repository and run the autodeployment script if needed.

- Second way: you can do it by hand. I'll give you an example to deploy a version of the application that shows another message:

(start this from your user's shell)

    $ vagrant ssh front1

    vagrant@front1:~$ sudo systemctl stop sampleapp.service
    vagrant@front1:~$ sudo su - ruby
    ruby@front1:~$ wget -c https://github.com/mmolinac/P161Hi/raw/master/sampleapp-1.1.0.tgz

    ruby@front1:~$ tar xzf sampleapp-1.1.0.tgz 
    ruby@front1:~$ ./sampleapp-1.1.0.sh 
    Bundle install ...

    Bundle complete! 12 Gemfile dependencies, 47 gems now installed.
    Bundled gems are installed into ./vendor/bundle.
    Database creation/migration ...
    sampleapp_development already exists
    sampleapp_test already exists
    Deployment OK.
    ruby@front1:~$ exit
    logout
    vagrant@front1:~$ sudo systemctl start sampleapp.service

And that's it!

### Final notes

There are different approaches that can be reached, depending of the deployment tools we had and other technical choices already taken (stacks, coding conventions, team best-practices ...)
Besides of being partisan of a certain approach, I think this project is flexible enough to be taken as a start point for possible improvements and adaptations to a team with well established working agreements.

Regarding the autodeployment structure, I've chosen a certain packing method, but it's obvious that depending of the project, you would need to install more software system-wide, or include additional tasks in the deployment script. I tend to find a balance of where and how to do things when I take the requirements for the project.
For instance, one improvement would be to include the version of the app inside the app folder, somehow. This way, Ansible could check not only if there's one application, but if it's the required one, or download on top of that folder.

This project has been developed with this software stack:

OS: Ubuntu 16.04.3 LTS

Kernel: 4.4.0-91-generic #114-Ubuntu SMP

Vagrant: Vagrant 1.8.1 (Ubuntu repository)

Ansible: ansible 2.0.0.2 (Ubuntu repository)

Virtualbox: 5.0.40-dfsg-0ubuntu1.16.04.1

Git: 2.7.4-0ubuntu1.2

IDE: Visual Studio Code 1.15.0

