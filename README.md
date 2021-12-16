# ELK Stack Deployment Project

The following is a complete walkthrough guide in how to deploy a simple virtual network with an ELK Stack server. 
The purpose of the project was to simulate a live environment with potentially vulnerable web applications. Additionally,
the project demonstrates how to use the ELK Stack to aggregate and view log data from multiple servers. 

Key highlights from the project included in this document:
- Diagram and Description of Network Topology
- Load Balancing
- Access Policies
- Automated Deployment and Configuration with Ansible
- ELK Stack Configuration
  - Monitored servers
  - Beats utilized
 
### Diagram and Description of Network Topology

![STUDENT TODO: Update image file path](Images/ELK_Stack_Diagram.png)  


The main purpose of this network is to simulate a network hosting vulnerable web applications in order to learn how ELK can be used to log traffic to those web applications. 

Table 1 - Summary of server names and details
| Name     |   Function  | IP Address |  Operating System  |
|----------|-------------|------------|--------------------|
| Jump Box | Provisioner | 10.0.0.4   | Linux Ubuntu 18.04 |
| Web1     | Web Server  | 10.0.0.5   | Linux Ubuntu 18.04 |
| Web2     | Web Server  | 10.0.0.6   | Linux Ubuntu 18.04 |
| Web3     | Web Server  | 10.0.0.7   | Linux Ubuntu 18.04 |
| ELK      | Monitoring  | 10.1.0.4   | Linux Ubuntu 18.04 |

## Load Balancing

In addition to the above, a **load balancer** was placed in front of the web servers. The load balancer's targets are organized into the following availability zones:

**Red Team Availability Set**: Web1-Web2-Web3

One of the most important aspects of securing a network is ensuring it's **Availability**. The availability of network resources is a key component of the CIA triad. Load balancing and redundancy are two ways to increase the uptime of network resources. In this network, several redundant web servers were created behind a load balancer. They were grouped together logically in a backend pool. This means if one of the web servers were taken down, another instance can immediately receive traffic from the load balancer. In addition to creating this concept of failover, the load balancer can spread the traffic to multiple servers should the network be flooded with traffic. This is especially useful in mitigating the risk of a distributed denial-of-service attack (DDOS).

## Access Policies
**Confidentiality** is another key component of the CIA triad. Confidentiality refers to the cybersecurity measures surrounding access controls to data and network resources. 
In Azure, this is accomplished with Network Security Groups. For this network most of the security controls for access to the jump box and the web servers were managed by RedTeamSecurityGroup. To harden the system a rule was created that only allowed web traffic to the jump box for system administration. The administrator's Public IP address 
was added to the rule to ensure only traffic from this IP address could access the jump box. Additionally, all of the machines in the network were configured to allow SSH access for network administration. We added security to this potential vulnerability by only allowing SSH from machines with allowed public SSH keys. Finally, we added a rule that only
SSH access from the jump box to the web servers was permitted. 

The ELK server was created in it's own virtual network. As a result, a separate security group was created to manage traffic to and from the ELK server. A rule was created to 
allow web traffic from the administrator's IP to the ELK server to view the aggregated log data in **Kibana**. Access via SSH was also permitted on the ELK server. Public SSH keys from the administrator account were added to increase security around SSH access. 


## Automated Deployment and Configuration Using Ansible
For this network, the Ansible container was downloaded to the jump box to serve as a provisioner for the network. **Containers** are simplified virtual machines that are dedicated to one task. Multiple containers can share the resources and operating system of a virtual machine. This way we could deploy apps and configure servers using Ansible playbooks. **Ansible playbooks** use YML (YAML Ain't Markup Language) files in order to create a series of commands to be executed on multiple servers. This automation tool reduced deployment and configuration time in this small set up of only four severs and would be extremely powerful in a network of hundreds or thousands of servers. Finally, we utilized **Docker** to create and manage our containers. 

The commands to install Docker and then use Docker to create the Ansible container:

1. Install Docker: `sudo apt update` then `sudo apt install docker.io`

2. Check Docker status: `sudo systemctl status docker` or Start Docker status: `sudo systemctl start docker`

3. Pull the Ansible container: `sudo docker pull cyberxsecurity/ansible`

4. Launch the Ansible container: `docker run -ti cyberxsecurity/ansible:latest bash` 

Once the Ansible container is deployed on the jump box you can SSH to the jump box and connect to the container with the following commands:
1. List all containers: `docker container list -a` the Ansible container will be given a randomized name. 

2. Start the container: ```bash $ sudo docker start container_name``` Linux will confirm the name of the container started.

3. Connect to the container using attach: ```bash $ sudo docker attach container_name``` Linux will show you have connected by giving root access prompt. 

The next step to take before running Ansible playbooks to deploy the web application to the web servers is to configure the Ansible host file and the ansible.cfg file. The hosts file is where to compile the list of servers Ansible will deploy to. The configuration file, ansible.cfg is where the system administrator login credentials will be configured. 

Run `cd /etc/ansible` and then `ls` to show all the files:

    ```bash
    root@containerID:~# cd /etc/ansible/
    root@containerID:/etc/ansible# ls
    ansible.cfg  hosts
    ```
Use Nano to open the `ansible.cfg` file: `root@containerID:/etc/ansible# nano ansible.cfg`

- This setting  is to be changed is the `remote_user`. The default admin user name is "root". Replace this with the admin user name used to set up the VMs. 

    ```bash
    # What flags to pass to sudo
    # WARNING: leaving out the defaults might create unexpected behaviors
    #sudo_flags = -H -S -n

    # SSH timeout
    #timeout = 10

    # default user to use for playbooks if user is not specified
    # (/usr/bin/ansible will use current user as default)
    remote_user = YOUR_USER_NAME

    # logging is off by default unless this path is defined
    # if so defined, consider logrotate
    #log_path = /var/log/ansible.log

    # default module name for /usr/bin/ansible
    #module_name = command
    ```

Next use Nano to edit the IP addresses in the hosts file: `root@containerID:/etc/ansible# nano hosts`

Hosts can be grouped together under headers using brackets: `[webservers]` or `[databases]` or `[workstations]`

- Uncomment the `[webservers]` header line and add the IP addresses of all of the web servers: `10.0.0.5, 10.0.0.6, 10.0.0.7`

Ansible works by creating a python script and then runs that script on the target machine using that machine's installation of Python. Typically, Ansible may have issues determining which python to use on the target machine, but this is solved by forcing ansible to use python 3.

- Add the line: `ansible_python_interpreter=/usr/bin/python3` besides each IP address.

 ```bash
    # This is the default ansible 'hosts' file.
    #
    # It should live in /etc/ansible/hosts
    #
    #   - Comments begin with the '#' character
    #   - Blank lines are ignored
    #   - Groups of hosts are delimited by [header] elements
    #   - You can enter hostnames or ip addresses
    #   - A hostname/ip can be a member of multiple groups
    # Ex 1: Ungrouped hosts, specify before any group headers.

    ## green.example.com
    ## blue.example.com
    ## 192.168.100.1
    ## 192.168.100.10

    # Ex 2: A collection of hosts belonging to the 'webservers' group

    [webservers]
    ## alpha.example.org
    ## beta.example.org
    ## 192.168.1.100
    ## 192.168.1.110
    10.0.0.5 ansible_python_interpreter=/usr/bin/python3
    10.0.0.6 ansible_python_interpreter=/usr/bin/python3
    10.0.0.7 ansible_python_interpreter=/usr/bin/python3
 ```

The next task is to create an Ansible playbook that installed Docker and configure all of the virtual web servers with the DVWA web app. This is done by using nano to create a yml file in the ansible directory in the Ansible container:
 ```bash
  root@container_ID:~# nano /etc/ansible/example-playbook.yml
  ```

## ELK Server Configuration

Integrating an ELK server allows users to easily monitor the vulnerable VMs for changes to the **file systems of the VMs on the network**, as well as watch **system metrics**, such as CPU usage; attempted SSH logins; `sudo` escalation failures; etc.

The ELK VM exposes an Elastic Stack instance. Docker is used to download and manage an ELK container.

Rather than configure ELK manually, we opted to develop a reusable Ansible Playbook to accomplish the task. This playbook is duplicated below.


To use this playbook, one must log into the Jump Box, then issue: `ansible-playbook install_elk.yml elk`. This runs the `install_elk.yml` playbook on the `elk` host.


Ansible was used to automate configuration of the ELK machine. No configuration was performed manually, which is advantageous because...

- _TODO: What is the main advantage of automating configuration with Ansible?_

The playbook implements the following tasks:
- _TODO: In 3-5 bullets, explain the steps of the ELK installation play. E.g., install Docker; download image; etc._
- ...
- ...


The following screenshot displays the result of running `docker ps` after successfully configuring the ELK instance.

- _TODO_: Update the image file path with the name of your screenshot of docker ps output:

  ![STUDENT TODO: Update image file path](Images/docker_ps_output.png)



The playbook is duplicated below.

```yaml
---
# install_elk.yml
- name: Configure Elk VM with Docker
  hosts: elkservers
  remote_user: elk
  become: true
  tasks:
    # Use apt module
    - name: Install docker.io
      apt:
        update_cache: yes
        name: docker.io
        state: present

      # Use apt module
    - name: Install pip3
      apt:
        force_apt_get: yes
        name: python3-pip
        state: present

      # Use pip module
    - name: Install Docker python module
      pip:
        name: docker
        state: present

      # Use command module
    - name: Increase virtual memory
      command: sysctl -w vm.max_map_count=262144

      # Use sysctl module
    - name: Use more memory
      sysctl:
        name: vm.max_map_count
        value: "262144"
        state: present
        reload: yes

      # Use docker_container module
    - name: download and launch a docker elk container
      docker_container:
        name: elk
        image: sebp/elk:761
        state: started
        restart_policy: always
        published_ports:
          - 5601:5601
          - 9200:9200
          - 5044:5044
```

### Target Machines & Beats
This ELK server is configured to monitor the DVWA 1 and DVWA 2 VMs, at `10.0.0.5` and `10.0.0.6`, respectively.

We have installed the following Beats on these machines:
- Filebeat
- Metricbeat
- Packetbeat

These Beats allow us to collect the following information from each machine:
- **Filebeat**: Filebeat detects changes to the filesystem. Specifically, we use it to collect Apache logs.
- **Metricbeat**: Metricbeat detects changes in system metrics, such as CPU usage. We use it to detect SSH login attempts, failed `sudo` escalations, and CPU/RAM statistics.
- **Packetbeat**: Packetbeat collects packets that pass through the NIC, similar to Wireshark. We use it to generate a trace of all activity that takes place on the network, in case later forensic analysis should be warranted.

The playbook below installs Metricbeat on the target hosts. The playbook for installing Filebeat is not included, but looks essentially identical — simply replace `metricbeat` with `filebeat`, and it will work as expected.

```yaml
---
- name: Install metric beat
  hosts: webservers
  become: true
  tasks:
    # Use command module
  - name: Download metricbeat
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.4.0-amd64.deb

    # Use command module
  - name: install metricbeat
    command: dpkg -i metricbeat-7.4.0-amd64.deb

    # Use copy module
  - name: drop in metricbeat config
    copy:
      src: /etc/ansible/files/metricbeat-config.yml
      dest: /etc/metricbeat/metricbeat.yml

    # Use command module
  - name: enable and configure docker module for metric beat
    command: metricbeat modules enable docker

    # Use command module
  - name: setup metric beat
    command: metricbeat setup

    # Use command module
  - name: start metric beat
    command: service metricbeat start
```

### Using the Playbooks
In order to use the playbooks, you will need to have an Ansible control node already configured. We use the **jump box** for this purpose.

To use the playbooks, we must perform the following steps:
- Copy the playbooks to the Ansible Control Node
- Run each playbook on the appropriate targets

The easiest way to copy the playbooks is to use Git:

```bash
$ cd /etc/ansible
$ mkdir files
# Clone Repository + IaC Files
$ git clone https://github.com/yourusername/project-1.git
# Move Playbooks and hosts file Into `/etc/ansible`
$ cp project-1/playbooks/* .
$ cp project-1/files/* ./files
```

This copies the playbook files to the correct place.

Next, you must create a `hosts` file to specify which VMs to run each playbook on. Run the commands below:

```bash
$ cd /etc/ansible
$ cat > hosts <<EOF
[webservers]
10.0.0.5
10.0.0.6

[elk]
10.0.0.8
EOF
```

After this, the commands below run the playbook:

 ```bash
 $ cd /etc/ansible
 $ ansible-playbook install_elk.yml elk
 $ ansible-playbook install_filebeat.yml webservers
 $ ansible-playbook install_metricbeat.yml webservers
 ```

To verify success, wait five minutes to give ELK time to start up. 

Then, run: `curl http://10.0.0.8:5601`. This is the address of Kibana. If the installation succeeded, this command should print HTML to the console.
