# How to deploy a Flask application with uWSGI and Nginx on Ubuntu 20.4 using an Ansible playbook

This project has the following directory structure:

	web-ansible/
		hosts
		playbook/main.yml
		roles/web/
		    files/
			classynails.service
			.flaskenv
		    handlers/
			 main.yml
		    tasks/
			 main.yml
		    templates/
			 main.ini.j2
			 nginx.j2
		web-cre.yml

# Set up the project

## Create a host file: (hosts)
    [www]
    192.168.1.115 ansible_user=ubuntu ansible_connection=ssh ansible_become_pass="{{wwwrootpass}}"

## Create an encrypted file to store credentials: (web-cre.yml)
    gituser: ***
    gitpass: ***
    wwwrootpass: ***
> **Note:** To create an encrypted file with ansible-vault, use this command: **ansible-vault create --vault-id webcre@prompt web-cre.yml**
> Because I use two encrypted files in this project, I tag each file with a **vault-id** to distinguish between individual vault password. In this case, the ID **webcre** identifies the file **web-cre.yml** and the keyword **prompt** indicates that it will prompt for the password for the **webcre** vault ID. 

## Create a playbook to call the role: (playbook/main.yml)

     ---
     - hosts: www
       vars_files:
          - /home/thang/web-ansible/web-cre.yml
       vars:
          - username: ubuntu
          - destdir: /home/{{username}}/classy-nails-website
          - requirement: /home/{{username}}/classy-nails-website/requirements.txt
          - env: /home/{{username}}/classy-nails-website/venv
          - app_name: classynails
       roles:
        - role: '/home/thang/web-ansible/roles/web'
 
 ## Role
**roles/web/tasks/main.yml**

	---
    - name: Update all packages
	  apt:
		update_cache: yes
     	upgrade: dist
		autoclean: yes
		autoremove: yes
	  become: true
	  become_user: root
	- name: Install git
	  apt:
		name: git
		state: latest
	  become: true
	  become_user: root
	- name: Check if the directory exists in the machine
	  stat:
		path: "{{destdir}}"
	  register: dir
	- name: Clone the repository if the directory doesn't exist in the machine
	  git:
		repo: "https://{{gituser}}:{{gitpass}}@github.com/tommypnguyen/classy-nails-website.git"
		dest: "{{destdir}}"
	  when: dir.stat.exists == false
	- name: Pull from the repository on github if the directory exists 
	  shell:
		cmd: "git pull"
		chdir: "{{destdir}}"
	  when: dir.stat.exists == true
	- name: Install necessary packages
	  apt:
		name: ['python3-pip', 'python3-dev', 'build-essential', 'libssl-dev', 'libffi-dev', 'python3-setuptools', 'python3-venv', 'nginx', 'virtualenv', 'python3-dotenv']
		state: latest
	  become: true
	  become_user: root
	- name: Install flask and uwsgi with pip in the virtual environment
	  pip:
		name: ['uwsgi', 'flask']
		virtualenv: "{{env}}"
		chdir: "{{destdir}}"
		virtualenv_command: python3 -m venv venv
	- name: Install packages from requirement.txt with pip in the virtual environment
	  pip:
		requirements: "{{requirement}}"
		virtualenv: "{{env}}"
		chdir: "{{destdir}}"
		state: latest
	- name: Create .flaskenv file
	  file:
		path: "{{destdir}}/.flaskenv"
		state: touch
		mode: 0755
	- name: Update file .flaskenv
	  copy:
		src: ".flaskenv"
     		dest: "{{destdir}}/.flaskenv"
		remote_src: no
	- name: Write to file main.ini
	  template:
		src: "main.ini.j2"
		dest: "{{destdir}}/main.ini"
		force: no
	- name: Create a systemd unit file
	  copy:
	    	src: "{{app_name}}.service"
		dest: "/etc/systemd/system/{{app_name}}.service"
		remote_src: no
		force: no
	  become: true
	  become_user: root
	  notify: Start and enable service classynails
	- name: Configuring nginx to proxy requests
	  template:
		src: "nginx.j2"
		dest: "/etc/nginx/sites-available/{{app_name}}"
		force: no
	  become: true
	  become_user: root
	- name: Remove default nginx site config
	  file:
		path: "/etc/nginx/sites-enabled/default"
		state: absent
	  become: true
	  become_user: root
	- name: Enable the nginx server block config
	  file:
		src: "/etc/nginx/sites-available/{{app_name}}"
		dest: "/etc/nginx/sites-enabled/{{app_name}}"
	    	state: link
	    	force: yes
	  become: true
	  become_user: root
	  notify: restart nginx service
	- name: Allow all incoming http
	  iptables:
		chain: INPUT
		ctstate: NEW,ESTABLISHED
		jump: ACCEPT
		protocol: tcp
		destination_port: "80"
	  become: true
	  become_user: root
	- name: Allow all outcomming http
	  iptables:
		chain: OUTPUT
	    	ctstate: ESTABLISHED
	    	jump: ACCEPT
		protocol: tcp
		source_port: "80"
	  become: true
	  become_user: root 

**roles/web/files/classynails.service**
	
	[Unit]
	Description=uWSGI instance to serve classynails
	After=network.target
	
	[Service]
	User=ubuntu
	Group=www-data
	WorkingDirectory=/home/ubuntu/classy-nails-website
	Environment="PATH=/home/ubuntu/classy-nails-website/venv/bin"
	ExecStart=/home/ubuntu/classy-nails-website/venv/bin/uwsgi --ini main.ini
	
	[Install]
	WantedBy=multi-user.target

**roles/web/files/.flaskenv**

	FLASK_APP=main.py
	FLASK_ENV=production
	INSTAGRAM_TOKEN= **insert your token here**

- I encrypted this file using Ansible vault because it contains the instagram token. The ID **flaskenv** identifies 
the file **.flaskenv** and the keyword **prompt** indicates that it will prompt for the password for the **flaskenv** vault ID. 

**roles/web/templates/main.ini.j2**
	[uwsgi]
	module = wsgi:app

	master = true
	processes = 5

	socket = main.sock
	chmod-socket = 660
	vacuum = true

	die-on-term = true

	logto = /home/{{username}}/classy-nails-website/main.log

**roles/web/templates/nginx.j2**

	server {
	listen 80;

	location / {
	include uwsgi_params;
	uwsgi_pass unix:{{destdir}}/main.sock;
	}}

**playbook/main.yml**

	---
	- hosts: www
	  vars_files:
		- /home/thang/web-ansible/web-cre.yml
	  vars:
		- username: ubuntu
		- destdir: /home/{{username}}/classy-nails-website
		- requirement: /home/{{username}}/classy-nails-website/requirements.txt
		- env: /home/{{username}}/classy-nails-website/venv
		- app_name: classynails
	  roles:
		- role: '/home/thang/web-ansible/roles/web'

# To run this playbook, use this command:

ansible-playbook -i hosts --vault-id webcre@prompt --vault-id flaskenv@prompt playbook/main.yml

**Note:** You should make sure that you are in the **web-ansible** directory before running the command above. 
