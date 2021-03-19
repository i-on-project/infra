# Infrastructure Documentation

## Why Ansible

Currently there are several options for provisioning a VM such as Puppet or Chef but in the end Ansible was the winner.

Ansible provides an easy installation and configuration process compared to the alternatives. For example, Puppet and Chef require Ruby knowledge to configure the provisionment file whereas Ansible adheres to a simple YAML configuration, that is extensible through the use of python modules. 

Also, when provisioning several machines, Ansible allows to do that from the base machine image (as long as the machine has python installed, which is common nowadays), while the alternatives require an Agent to be installed, which adds time to the provisioning process.

## How to use Ansible

Ansible can be used through adhoc on the CLI, or through playbooks. Since we want to have a configuration file that describes the necessary steps to provision a machine, we're only going to be taking the playbooks approach.

### Inventory

Firstly, before configuring our playbook, we need to define what machines we do have. For that purpose we use an **inventory**. The inventory allows us to specify our deployment VM's by labels such as *web* or *database*. This inventory file is later used to specify where each play is going to run.

An example inventory file is presented as follows:

```toml
[web]
iondev.live

[database]
iondev.live

[all:children]
web
database
```

In this case we have three groups: *web*, *database* and *all*, the latter being a special group that includes every machine present in the first two groups. Each group has machines that are identified by its IP address, or by its DNS name (used for this example).

### Playbooks, Plays and Tasks

After creating our inventory we can begin our playbook creation. Before beginning the YAML file configuration, we should review some playbook concepts:

- Playbook: A set of plays
- Play: A set of tasks to be executed by one or more hosts
- Task: A module or command to be executed on the Play hosts

For example, if we want to specify a playbook that has a play that installs docker on every VM of our inventory we can do as follows:

```yml
# This is a new play
- name: Install Docker # The name of the play
  hosts: all # The hosts in which the play is going to run (check inventory file)
  become: true # We want to run this play with escalated permissions
  tasks: 
    # The set of tasks that this play is going to run in order
    - name: Update aptitude repositories # Name of the task
      shell: apt update # In this case, we execute a shell script that updates the local aptitude repository
    - name: Upgrade aptitude packages
      shell: apt upgrade -y
    - name: Install docker and docker-compose
      shell: apt install docker docker-compose -y # Here we install docker and docker-compose by using a shell script again
    - name: Check docker status # This task checks the docker daemon state
      # It uses the service module.
      service:
        name: docker # The service we want to check
        state: started # We want to check if it has started, and if not, it starts the service
        enabled: true # Checks if the service is enabled (if it starts on system start), and if not, enables the service
```

*More documentation on the service module is available [here](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html).*

If we want to have several plays we can continue specifying:

```yml
- name: Install Docker
  hosts: all
  become: true
  tasks: 
    - name: Update aptitude repositories
      shell: apt update
    - name: Upgrade aptitude packages
      shell: apt upgrade -y
    - name: Install docker and docker-compose
      shell: apt install docker docker-compose -y
    - name: Check docker status
      service:
        name: docker
        state: started
        enabled: true
# Our new play
- name: Install Postgres
  hosts: database # This play is only going to be executed in our database inventory group
  become: true
  tasks:
    - name: Install Postgres
      shell: apt install postgresql -y
```

### Relevant Modules

Sometimes we want to do operations over files, like copying a file from the host to the guests, or doing operations to the remote file system. To this end we can use the [copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html) and [file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html) Ansible modules, respectively.

For example, let's say we want to copy our nginx config:

```yml
- name: Install nginx and setup virtual host
  hosts: web
  become: true
  tasks: 
    # ...
    # Copy the virtual host file from our host machine to the guest VM
    - name: Copy nginx config
      copy: # We use the copy module
        src: nginx.conf # Path to the file in our host machine
        dest: /etc/nginx/sites-available/ # Where to copy the file to in the guest VM

    # Create a symlink to enable our virtual host
    - name: Enable nginx virtual host
      file: # Using the file module
        path: /etc/nginx/sites-enabled/ # The folder where our symlink should be
        src: /etc/nginx/sites-available/nginx.conf # The linked file
        state: link # Check that there's a link, otherwise it creates a new one (equivalent to ln -s src path)

    # Check or update nginx status
    - name: Check nginx status
      service:
        name: nginx

        state: reloaded
        # This state guarantees two things:
        #   - If nginx has not started, it starts it
        #   - If nginx is started, it reloads the servicem, loading the config

        enabled: true # Check if nginx is enabled
```

As we can see, we can execute these operations easily using modules.

### Variables

However, we are still using hardcoded strings such as the name of the local nginx configuration. If we want to copy other nginx configuration we have to change the playbook, which may not be ideal. To that end we have **Variables**.

So, for the aforementioned example, we can do something like: 

```yml
- name: Install nginx and setup virtual host
  hosts: web
  become: true
  # We define our variables in the var block
  vars:
    nginx_file: nginx.conf # In this case our variable is the virtual host config name
  tasks: 
    # ...
    - name: Copy nginx config
      copy:
        src: "{{ nginx_file }}" # We can use the variables with Mustache templating
        dest: /etc/nginx/sites-available/
    - name: Enable nginx virtual host
      file:
        path: /etc/nginx/sites-enabled/
        src: "/etc/nginx/sites-available/{{ nginx_file }}" # Here too :)
        state: link
    # ...
```

Allowing us to only have to change the file name on one place, instead of several.

But, what do we do if we have secrets? Secrets shouldn't be in the playbook! To that end we can specify a variable files, where our play is going to load variables from:

```yml
- name: Sets up some secret service
  hosts: web
  become: true
  # We define the necessary variable files in this block
  vars_files:
    - secrets.yml # This file has a varible "secret" defined
  tasks: 
    - name: Executes important command
      shell: "important-command --our-secret={{ secret }}" # We use our secret that was imported from the variables file
```

Furthermore, Ansible allows us to use Vaults: we can encrypt our variable files, providing an extra layer of security (or the only layer of security if the file is in source control), however, this feature was not explored as of yet, because it was deemed unnecessary for the time being.

### Running our playbook

With our playbook created and inventory defined we want to run it. To do that we need to have Ansible installed in our host machine. That can be done through `pip`, or through the package manager.

After installing Ansible we can execute our playbook in our VM's by using:

```sh
ansible-playbook playbook.yml -i inventory -u core --key-file=.ssh/private_key -K
```

Where:

- `-i inventory` specifies the inventory file
- `-u core` specifies the user where the playbook is going to be run
- `--key-file` specifies the private key path of the user
- `-K` allows to ask the become user (by default, root) password when a play needs escalation of priviledges (`become: true`)