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

### Debian box

At first I was planning on using the latest Debian box available. That is, [debian/stretch64](https://app.vagrantup.com/debian/boxes/stretch64) , but it seems that it needs a newer version of Vagrant to work fine.
I settled for stability and because of that I used [debian/jessie64](https://app.vagrantup.com/debian/boxes/jessie64), which offers Debian 8.9 with a deserved reputation of stability.

### Networking

I've configured both machines with an interface in a privat network `192.168.5.0/24` .
They can see each other, and I have established a port forwarding in the Rails machine, so we can check the app from our browser outside:

    host.vm.network :forwarded_port, guest: 3000, guest_ip: "192.168.5.12", host: 3000, protocol: "tcp"

### Provisioning: Ansible

### db1 host
tell about soft installed and users
connectivity

### front1 host
ruby version. rails. gems.
ports mapped

### Sample application
what is done by the app 

### Application autodeployment

### Proposed solution to replace the application version.
