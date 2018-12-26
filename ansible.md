# Ansible Course

### Configuration

In order to configure Ansible, we must create the following configuration file:  
```
ansible.cfg
```
When we run Ansible commands, it will look for this file in the following order:
* `ANSIBLE_CONFIG` (env variable)
* `ansible.cfg` (current directory)
* `~/.ansible.cfg` (in the home dir)
* `/etc/ansible/ansible.cfg`  

The inner structure of this file should look like this:
```
[default]
inventory=./hosts-dev
remote_user = ec2-user
private_key_file=/home/rlope/.ssh/ansible-course-key-pair.pem
host_key_checking = False
```
The `inventory` configuration, tells ansible to where to look for the inventory file and some other configuration.

`remote_user` tells ansible the user to used for sshing into the servers.  
`private_key_file` specifies the path to the private key.  
`host_key_checking` set this to “False” if you want to avoid host key checking by the underlying tools Ansible uses to connect to the host.  

### Commands

```
$ ansible -i hosts-dev --list-hosts all
```
This command will look at inventory file hosts-dev and print the entire list of hosts managed by Ansible.

```
$ ansible -i hosts-dev --list-hosts "*"
```
This command works exactly as the `all`, but it uses wildcards.  

```
$ ansible -i hosts-dev --list-hosts "app*"
```
Will look for all hosts starting with `app`
```
$ ansible -i hosts-dev --list-hosts webservers:loadbalancers
```
Will look for all hosts in groups `webservers` and `loadbalancers`
```
$ ansible -i hosts-dev --list-hosts \!control
```
Will look for all hosts except by `control`.

### Playbooks

Playbooks are an ordered lists of plays that can run tasks for configuration and orchestration. These plays allows us to run commands on a group or subset of servers within our inventory.  

The playbook needs to be writen in a logical step-by-step, like this:
* update all packages/services
* install missing packages
* then, configure the necessary packages
* finally, ensure all configuration made are correct.  

Playbooks are writen in YAML format, they are composed of one or more `plays` in a `list`. In order to run a playbook, you should issue the following command:
```
$ ansible-playbook playbook-name.yml
```
Sample playbook struture:
```
# ping.yml
---
  - hosts: all
    tasks: 
      - name: Ping all servers 
        action: ping 
```
`- hosts: all` enlist the hosts which this play should be applied
`tasks:` enlist all the tasks performed by this playbook  
`- name: Ping all servers` give a name in order to be easier to read and to refer to it  
`action: ping` the task itself, inhere we've set an action, but it might be a module instead.  

### Handlers

Very useful when we need to restart a service after making changes to its configuration. Ansible is smart enough to identify if there were any changes made in order to avoid restarting services when it's indeed not needed.

In order to create a handler, we should set in the playbook, at same level as `tasks` another key word: `handlers`. The handler has a name and in general it is making use of `services`. Then, we have to use `notify` in the task where we want to trigger the handler. We make use of handlers `name` to trigger it.

```
# setup-app.yml

---
  - hosts: webservers
    become: true
    tasks:
      - name: Upload application file
        copy:
          src: ../index.php
          dest: /var/www/html
          mode: 0755
      - name: Configure php.ini file
        lineinfile:
          path: /etc/php.ini
          regexp: ^short_open_tag
          line: 'short_open_tag=On'
        notify: restart apache
      
    handlers:
      - name: restart apache
        service: name=httpd state=restarted
```

### Using Variables

During TASK [Gathering Facts] step, these variables become populated. It gathers useful facts about our hosts and can be used in our playbooks.  
Use the `status` module to see all of the facts gathered during TASK [Gathering Facts] step. Use Jinja2 templating to evaluate those expressions.

Ansible also gives us the ability to create local variables to be used in our playbooks:
* Create playbook variables using `vars` to create key/value pair and dictionary/map of variables;
* Create variables file and import into our playbook;

Ansible gives us the ability to register variables from tasks that run to get information about its execution:
#### Using register
Create variables from information returned from tasks ran using `register`. We can call registered variables for late use.

#### Using debug
Very useful for debuging playbooks and also to get more information about values assigned to variables during the playbook execution.  

```
---
  vars: 
    path_to_app: "/var/www/html"

  tasks:
    - name: List directory content
      command: ls -la {{ path_to_app }}
      register: dir_content
    - name: Debug directory contents
      debug:
        msg: "{{ dir_content }}"
```
Use the following command to get all the information gathering during `Gathering Facts` step:

```
$ ansible -m setup ip/name_of_host
```
### Roles

Ansible provides a framework that makes each part of the variables, tasks, templates, and modules fully independent.
* Group tasks together in a way that it is self contained;
* Clean and pre-defined folder structure;
* Break up the configurations into files;
* Reuse the code by others who need similar configuration;
* Easy to modify and reduces sintax errors.

Ex of directory structure:
```
setup.yml
roles/
    webservers/
        tasks/
            - main.yml
        vars/
            - main.yml
        handlers/
            - main.yml
```

How to create the folder structure using Ansible command:
```
$ ansible-galaxy webservers init
```
Ansible will create almost all of the folders, but they are optional. You can remove it if not needed.  

Following the example above, a simple role would look like this:
```
# setup.yml
---
  - hosts: webservers
    become: True
    roles:
      - webservers
```
```
# roles/webservers/tasks/main.yml
- name: Upload application file
    copy:
        src: ../index.php
        dest: "{{path_to_app}}"
        mode: 0755

    - name: Create info page
    copy:
        dest: "{{path_to_app}}/info.php"
        content: "<h1> Info about the server {{ ansible_hostname }} </h1>"
    - name: Configure php.ini file
    lineinfile:
        path: /etc/php.ini
        regexp: ^short_open_tag
        line: 'short_open_tag=On'
    notify: restart apache
```
```
# roles/webservers/handlers/main.yml
- name: restart apache
    service: name=httpd state=restarted
```
```
# roles/webservers/vars/main.yml
path_to_app: /var/www/html
```

### Dry Run
A.K.A Check Mode: Reports changes tha Ansible would have to make on the end hosts rather than applying the changes.

```
$ ansible-playbook setup.yml --check
```

This is very useful for production environments, instead of applying the changes, it would only test them before applying.

### Tags

Assigning tags to specific tasks in playbooks allows us to only call certain tasks even in a very long playbook. Use the `tags` property on the same level as the `name` property to assign the tag.  

Its possible to assign many tags to a single task, and also assign the same tag to many tasks.  

To run only the tasks assigned to a specific tag:
```
$ ansible-playbook setup.yml --tags create
```

### Ansible Vault
Vault is a way to keep sensitive information in encrypted files, rather than plain text, in your playbooks.

Its very useful in such situations:
* Keep passwords, keys and sensitive variables in encrypted files;
* You can share the vault files in your source control system;
* Vault can encrypt pretty much any data structure file used by Ansible;
* Password protected and the default cipher is AES.

Create encrypted data file
```
$ ansible-vault create secret-variables.yml
```
In order to use any data encrypted by Vault, its necessary to run the ansible command with the following argument:
```
$ ansible-playbook setup.yml --ask-vault-pass
```

If you need to edit this file, issue the following command:
```
$ ansible-vault edit secret-variables.yml
```
Then, type the password defined for this file, then you'll be able to changes its content.

Another way to encrypt data is piping a stdout file with ansible-vault command, like this:
```
$ cat file.txt | ansible-vault encrypt_string
```

### Prompt

Useful when you need to request user input before proceeding with some ansible tasks.
Check the example:
```
# setup.yml
---
  vars_prompt:
    - name: "upload_var"
      prompt: "Upload index.php?"
  tasks:
    - name: Upload application file
      copy:
        src: ../index.php
        dest: /var/www/html
      when: upload_var == "yes"
      tags: upload
```
### Resources

[Ansible Inventory File](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)  
[Ansible Patterns](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html)  
[Installing the Control Machine](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-the-control-machine)  
[Working with Dynamic Inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html)  
[Ansible Configuration Settings](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings)  
[Working with Conditionals](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html)
[Handlers](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html#handlers-running-operations-on-change)  
[Import a Playbook](https://docs.ansible.com/ansible/latest/modules/import_playbook_module.html)  
[Using variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)  
[Tags](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html)  
[Prompt](https://docs.ansible.com/ansible/latest/user_guide/playbooks_prompts.html)  
[The When Statement](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html#the-when-statement)