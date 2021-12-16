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
  - Automated deployment with Ansible
  - Monitoring Web Applications with Beats
   
### Diagram and Description of Network Topology

![STUDENT TODO: Update image file path](Images/ELK_Stack_Diagram.png)  


The main purpose of this network is to simulate a network hosting vulnerable web applications in order to learn how ELK can be used to monitor traffic to those web applications. 

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
allow web traffic from the administrator's IP to the ELK server to view the aggregated log data in **Kibana**. Access via SSH was also permitted on the ELK server. Public SSH keys from the administrator account were added to increase security around SSH access. Finally, a **Peering** was created between the two networks in order to share traffic.


## Automated Deployment and Configuration Using Ansible
For this network, the Ansible container was downloaded to the jump box to serve as a provisioner for the network. **Containers** are simplified virtual machines that are dedicated to one task. Multiple containers can share the resources and operating system of a single virtual machine. These lightweight virtual machines contained on a single virtual machine on a network saves system resources and reduces costs. **Ansible playbooks** use YML (YAML Ain't Markup Language) files in order to create a series of commands to be executed on multiple servers. This enables rapid deployment and configuration of many servers simultaneously, reducing errors, creating uniformity and lowering administration costs. Finally, we utilized **Docker** a common container manager app, to create and manage our containers. 

The commands to install Docker on the jump box and then use Docker to create the Ansible container:

1. SSH into the jump box from a terminal using the admin and SSH key configured on the jump box: ```ssh RedAdmin@104.42.221.76```

2. Install Docker: `sudo apt update` then `sudo apt install docker.io`

3. Check Docker status: `sudo systemctl status docker` or Start Docker status: `sudo systemctl start docker`

4. Pull the Ansible container: `sudo docker pull cyberxsecurity/ansible`

5. Launch the Ansible container: `docker run -ti cyberxsecurity/ansible:latest bash` 

Once the Ansible container is deployed on the jump box you can SSH to the jump box and connect to the container with the following commands:
1. List all containers: `docker container list -a` the Ansible container will be given a randomized name. 

2. Start the container: ```bash $ sudo docker start container_name``` Linux will confirm the name of the container started.

3. Connect to the container using attach: ```bash $ sudo docker attach container_name``` Linux will show you have connected by giving root access prompt. 

Before running Ansible playbooks to deploy the web application to the web servers, the Ansible host file and the ansible.cfg file must be updated. The hosts file will list the IP address Ansinble will reference in deployment. The configuration file, ansible.cfg, is where the system administrator login credentials will be configured. 

Run `cd /etc/ansible` and then `ls` to show all the files:

    ```bash
    root@containerID:~# cd /etc/ansible/
    root@containerID:/etc/ansible# ls
    ansible.cfg  hosts
    ```
Use nano to open the `ansible.cfg` file: `root@containerID:/etc/ansible# nano ansible.cfg`

- This setting  is to be changed is the `remote_user`. The default admin user name is "root". Replace this with the admin user name used to set up the VMs (RedAdmin). 

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

Next use nano to edit the IP addresses in the hosts file: `root@containerID:/etc/ansible# nano hosts`

Hosts can be grouped together under headers using brackets: `[webservers]` or `[databases]` or `[workstations]`

- Uncomment the `[webservers]` header line and add the IP addresses of all of the web servers: `10.0.0.5, 10.0.0.6, 10.0.0.7`

Ansible works by creating a Python script and then running that script on the target using that machine's installation of Python. Historically, Ansible has had issues determining which version of Python to use on the target machine. This is resolved by forcing ansible to use Python 3 using a configuration in the hosts file:

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

The next task is to create an Ansible playbook that installs Docker and configures all of the virtual web servers with the DVWA web app. This is done by using nano to create a YAML file in the ansible directory in the Ansible container:
 ```bash
  root@container_ID:~# nano /etc/ansible/playbook-name.yml
  ```
This is the Ansible playbook code used to install Docker and create the DVWA containers:

```bash
---
- name: configure web vms with docker
  hosts: webservers
  become: true
  tasks: 

  - name: docker.io 
    apt:
      update_cache: yes
      name: docker.io
      state: present

  - name: install pip3  
    apt: 
      name: python3-pip
      state: present

  - name: Install Python Docker Module
    pip:
      name: docker
      state: present

  - name: download and launch our DVWA web container
    docker_container: 
      name: dvwa
      image: cyberxsecurity/dvwa
      state: started
      restart_policy: always
      published_ports: 80:80   

  - name: enable docker service 
    systemd: 
     name: docker
     enabled: yes
   ```
   
 The final step is to run the playbook using the ansible-playbook command: ```bash ansible-playbook playbook-name.yml```
 Running this playbook executes all of the commands in the playbook on each server identified in the hosts file in the ansible directory.
 Upon successful completion, the output appears as follows:
 
 ```bash
    root@container_ID:~# ansible-playbook /etc/ansible/playbook-name.yml

    PLAY [Config Web VM with Docker] ***************************************************************

    TASK [Gathering Facts] *************************************************************************
    ok: [10.0.0.5]
    ok: [10.0.0.6]
    ok: [10.0.0.7]
   
    TASK [docker.io] *******************************************************************************
    [WARNING]: Updating cache and auto-installing missing dependency: python-apt
    changed: [10.0.0.5]
    changed: [10.0.0.6]
    changed: [10.0.0.7]

    TASK [Install pip3] *****************************************************************************
    changed: [10.0.0.5]
    changed: [10.0.0.6]
    changed: [10.0.0.7]

    TASK [Install Docker python module] ************************************************************
    changed: [10.0.0.5]
    changed: [10.0.0.6]
    changed: [10.0.0.7]

    TASK [download and launch a docker web container] **********************************************
    changed: [10.0.0.5]
    changed: [10.0.0.6]
    changed: [10.0.0.7]

    PLAY RECAP *************************************************************************************
    10.0.0.5                   : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
    10.0.0.6                   : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
    10.0.0.7                   : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
  ```
 
Now you can use the IP address of the administrators machine that has access in the RedTeamSecurityGroup to test the deployment. Because the web servers hosting the DVWA do not have a public IP and are behind a load balancer, the web app is accessed using the IP of the load balancer. 

![DVWA Landing Page](Images/DVWA_landing.jpg)

The login credentials for the DVWA are ```admin:password``` 
Upon successful login, the user will see a page similar to the one below. Click the ![DB button](Images/DVWA_CreateDB.jpg) to set up the DVWA web application. 

![set up page](Images/DVWA_DB_Setup.jpg)


## ELK Server Configuration

Previous sections covered ELK Server creation and configuration in Azure. This section will cover deploying the ELK Server image using Ansible. 

The ELK Stack is an open-source software platform that includes three powerful tools: Elasticsearch, Logstash, and Kibana. 
 - Logstash aggregates data from multiple sources, generally log data, and sends it to Elasticsearch. 
 - Elasticsearch is a search and analytics engine used as centeralized data pool.
 - Kibana creates visulizations of that data in multiple views, graphs and charts. 

Security analysts can utilize an integrated ELK server to:
 - Monitor traffic to web servers to detect exploits and identify vulnerabilities
 - Detect changes to the file systems of the VMs on the network using Filebeats
 - Watch system metrics such as CPU usage, attempted SSH logins, and `sudo` escalation failures using Metricbeats

The steps to connect to the Ansible container described above are repeated to navigate to the ansible directory. Additionally, the host file must be updated with the IP address of the ELK server. Separate from the ```[webservers]``` header in the hosts file, an ```[ELK]``` section is created:

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

#green.example.com
#blue.example.com
#192.168.100.1
#192.168.100.10

# Ex 2: A collection of hosts belonging to the 'webservers' group

[webservers]
10.0.0.5 ansible_python_interpreter=/usr/bin/python3
10.0.0.6 ansible_python_interpreter=/usr/bin/python3
10.0.0.7 ansible_python_interpreter=/usr/bin/python3

[ELK]
10.1.0.4 ansible_python_interpreter=/usr/bin/python3

#beta.example.org
#192.168.1.100
#192.168.1.110

# If you have multiple hosts following a pattern you can specify
# them like this:

#www[001:006].example.com

# Ex 3: A collection of database servers in the 'dbservers' group

#[dbservers]
#
#db01.intranet.mydomain.net
#db02.intranet.mydomain.net
#10.25.1.56
#10.25.1.57

# Here's another example of host ranges, this time there are no
# leading 0s:

#db-[99:101]-node.example.com
```

The next step is to create a separate playbook to deploy Docker and configure the ELK server with an Ansible playbook. Following the same instructions for creating the YAML file above, the following is the Ansible playbook for creating the container on the ELK server:

```yml
---
- name: Config elk VM with Docker
  hosts: ELK
  remote_user: RedAdmin
  become: true
  tasks:

  - name: Use more memory
    sysctl:
      name: vm.max_map_count
      value: '262144'
      state: present
      reload: yes

      
  - name: docker.io
    apt:
      update_cache: yes
      name: docker.io
      state: present

  - name: install pip3
    apt:
      name: python3-pip
      state: present

  - name: Install Python Docker Module
    pip:
      name: docker
      state: present
 
  - name: download and launch our elk container
    docker_container:
      name: elk
      image: sebp/elk:761
      state: started
      restart_policy: always
      published_ports: 5601:5601,9200:9200,5044:5044

  - name: enable docker service
    systemd:
     name: docker
     enabled: yes 
 ```

The modules for installing Docker and Python were the same as the previous playbook, however there were three additional modules in this playbook. First, we configured this playbook to only run the on the IP addresses added to the ELK section of the hosts file. The same remote user was used so we could secure the ELK server with the same public SSH key used for securing the web servers. This allows the administrator to SSH into every server from the Ansible container. 

```yml
- name: Config elk VM with Docker
  hosts: ELK
  remote_user: RedAdmin
  become: true
  tasks:
```

The next additional module of the playbook is a system requirement for running the ELK container. More info [at the `elk-docker` documentation](https://elk-docker.readthedocs.io/#prerequisites). Memory must be increased as follows:

```yml
- name: Use more memory
    sysctl:
      name: vm.max_map_count
      value: '262144'
      state: present
      reload: yes
```

Lastly, the ELK Docker container configuration must be included. The sebp/elk:761 Docker image provides a web interface to interact with Elasticsearch, Logstash, and Kibana. The published ports are what will be used to access the web interfaces. See below:

```yml
 - name: download and launch our elk container
    docker_container:
      name: elk
      image: sebp/elk:761
      state: started
      restart_policy: always
      published_ports: 5601:5601,9200:9200,5044:5044
 ```

Finally, the Ansible playbook must be run to complete the deployment. ```root@containerID:/etc/ansible/files# ansible-playbook install-elk.yml```

The output should verify successful deployement to the IP address of the ELK server configured in the hosts file:

```bash
[WARNING]: ansible.utils.display.initialize_locale has not been called, this may result in incorrectly calculated text
widths that can cause Display to print incorrect line lengths

PLAY [Config elk VM with Docker] ***************************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [10.1.0.4]

TASK [Use more memory] *************************************************************************************************
ok: [10.1.0.4]

TASK [docker.io] *******************************************************************************************************
ok: [10.1.0.4]

TASK [install pip3] ****************************************************************************************************
ok: [10.1.0.4]

TASK [Install Python Docker Module] ************************************************************************************
ok: [10.1.0.4]

TASK [download and launch our elk container] ***************************************************************************
[DEPRECATION WARNING]: The container_default_behavior option will change its default value from "compatibility" to
"no_defaults" in community.docker 2.0.0. To remove this warning, please specify an explicit value for it now. This
feature will be removed from community.docker in version 2.0.0. Deprecation warnings can be disabled by setting
deprecation_warnings=False in ansible.cfg.
ok: [10.1.0.4]

TASK [enable docker service] *******************************************************************************************
ok: [10.1.0.4]

PLAY RECAP *************************************************************************************************************
10.1.0.4                   : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```


### Monitoring Web Applications with Beats

The ELK server is configured to monitor Web1, Web2 and Web3 using tools in the ELK Stack. The next step is to use Ansible to deploy **Beats** to all three web servers.
Beats are lightweight data shippers that take data from various sources and sends them to Elasticsearch. Examples of data shipped for monitoring:
 - Contents of a log file
 - Metrics on a system
 - Network activity

The following beats are installed on the web servers in this network:

- **Filebeat**: Filebeat detects changes to the filesystem. Typically, it is used to collect and parse system logs.
- **Metricbeat**: Metricbeat detects changes in system metrics, such as CPU/RAM usage. Additionally, it will detect SSH login attempts, and failed `sudo` escalations.

**Filebeats**

The first step is to install Filebeats on both web servers using Ansible. The instructions are the same as mentioned previously. First create the Ansible playbook, then run the playbook to deploy. The Filebeat playbook:

```yml
---

- name: Installing and Launch Filebeat
  hosts: webservers
  become: yes
  tasks:
   
  - name: Download filebeat .deb file
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.4.0-amd64.deb

  - name: Install filebeat .deb
    command: dpkg -i filebeat-7.4.0-amd64.deb

  - name: Drop in filebeat.yml
    copy:
      src: /etc/ansible/files/filebeat-config.yml
      dest: /etc/filebeat/filebeat.yml

  - name: Enable and Configure System Module
    command: filebeat modules enable system

  - name: Setup filebeat
    command: filebeat setup
   
  - name: Start filebeat service
    command: service filebeat start

  - name: Enable service filebeat on boot
    systemd:
      name: filebeat
      enabled: yes
 ```

The first module downloads the Debian Linux sytem file (.deb) for Filebeats using the curl command. The second installs that downloaded file. See below: 

```yml
 - name: Download filebeat .deb file
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.4.0-amd64.deb

  - name: Install filebeat .deb
    command: dpkg -i filebeat-7.4.0-amd64.deb
```

The next module references the Filebeat configuration file. In the configuration file is where the IP address is updated for the ELK server. For this network the following configuration file was downloaded using the curl command:

`curl https://gist.githubusercontent.com/slape/5cc350109583af6cbe577bbcc0710c93/raw/eca603b72586fbe148c11f9c87bf96a63cb25760/Filebeat > /etc/ansible/files/filebeat-config.yml`

For reference: https://github.com/landindonner/ELK_Stack_Project/blob/main/Playbooks/filebeat-config.yml 

On line 1105 and 1805 is where the IP addresses are configured for the ELK server. With this configuration file updated and stored in the referenced location, the following Filebeat playbook module will run correctly:

```yml
 - name: Drop in filebeat.yml
    copy:
      src: /etc/ansible/files/filebeat-config.yml
      dest: /etc/filebeat/filebeat.yml
 ```

The remaining modules are commands to enable, setup, start Filebeats and enable Filebeats everytime the servers are booted. 

```yml
 - name: Enable and Configure System Module
    command: filebeat modules enable system

  - name: Setup filebeat
    command: filebeat setup
   
  - name: Start filebeat service
    command: service filebeat start

  - name: Enable service filebeat on boot
    systemd:
      name: filebeat
      enabled: yes
 ```

To deploy Filebeat to the servers, run the Ansible playbook using the ansible playbook command: ```root@containerID:/etc/ansible/files# ansible-playbook filebeat_playbook.yml```

If run correctly the out put will appear as follows:

```yml
[WARNING]: ansible.utils.display.initialize_locale has not been called, this may result in incorrectly calculated text
widths that can cause Display to print incorrect line lengths

PLAY [Installing and Launch Filebeat] **********************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [10.0.0.7]
ok: [10.0.0.5]
ok: [10.0.0.6]

TASK [Download filebeat .deb file] *************************************************************************************
changed: [10.0.0.7]
changed: [10.0.0.6]
changed: [10.0.0.5]

TASK [Install filebeat .deb] *******************************************************************************************
changed: [10.0.0.7]
changed: [10.0.0.6]
changed: [10.0.0.5]

TASK [Drop in filebeat.yml] ********************************************************************************************
ok: [10.0.0.5]
ok: [10.0.0.6]
changed: [10.0.0.7]

TASK [Enable and Configure System Module] ******************************************************************************
changed: [10.0.0.5]
changed: [10.0.0.7]
changed: [10.0.0.6]

TASK [Setup filebeat] **************************************************************************************************
changed: [10.0.0.5]
changed: [10.0.0.6]
changed: [10.0.0.7]

TASK [Start filebeat service] ******************************************************************************************
changed: [10.0.0.7]
changed: [10.0.0.5]
changed: [10.0.0.6]

TASK [Enable service filebeat on boot] *********************************************************************************
ok: [10.0.0.5]
ok: [10.0.0.6]
changed: [10.0.0.7]

PLAY RECAP *************************************************************************************************************
10.0.0.5                   : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
10.0.0.6                   : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
10.0.0.7                   : ok=8    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

To verify Filebeat was correctly installed on the web servers use the following steps:
  - Navigate to the ELK server using the IP address through port 5601: http://104.42.221.76:5601/app/kibana
   
  - On the home page, click Add log data button in the Observability section to navigate to the Add Data to Kibana page: 
   
  ![add log button](Images/KibanaAddLog.jpg)
  
  - On the Add Data to Kibana page, under the Logs tab, click the Systems logs link in the Systems log box
  
  ![sys logs](Images/KibanaSysLogs.jpg)
  
  - On the Systems logs page, scroll to the bottom to step 5 Module Status and click the Check Data button. If successful this result is displayed:
  
  ![Data received](Images/KibannaDataReceived.jpg)
  
Next click the System Log Dashboard button to view the web app activity of Web1, Web2 and Web3 through Kibana's interface and visualizations:

![dashboard](Images/KibanaSysLogDash.jpg)
  
This completes the successful deployment of ELK Stack with Filebeat data shipper. 

**Metricbeats**

To install Metricbeats for additional data analytics, follow the same procedure as described above for Filebeat. The differnces are described below:

1. Metricbeat configuration file - for this implementation we downloaded the following configuration file in the `/etc/ansible/files/` directory:
    
`curl https://gist.githubusercontent.com/slape/58541585cc1886d2e26cd8be557ce04c/raw/0ce2c7e744c54513616966affb5e9d96f5e12f73/metricbeat`

The configuration file is too large to post here. For reference: https://github.com/landindonner/ELK_Stack_Project/blob/main/Playbooks/metricbeat-config.yml

On line 62, update the IP address of the host to the IP address of the ELK Server:

```yml

#============================== Kibana =====================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:
  host: "10.1.0.4:5601"
```

2. Follow the steps for creating and running the Ansible playbook as described above in the Filebeat section. Replace instances of "Filebeat" with "Metricbeat" in the playbook.

3. When verifying the successful deployment use the following steps:
  
  - Navigate to the ELK server using the IP address through port 5601: http://104.42.221.76:5601/app/kibana
   
  - On the home page, click the Add metric data button in the Observability section to navigate to the Add Data to Kibana page: 
   
  ![add metrics button](Images/KibanaAddMetricData.jpg)
  
  - On the Add Data to Kibana page, under the Metrics tab click the Docker metrics link in the Docker metrics box:
  
  ![docker metrics](Images/KibanaDockerMetrics.jpg)
  
  - On the Docker metrics page, scroll to the bottom to step 5 'Module Status' and click the Check Data button. If successful this result is displayed:
  
  ![Data received](Images/KibanaMetricDockerDataRecv.jpg)
  
  - Next click the Docker metrics dashboard button to view the web app system metrics of Web1, Web2 and Web3 through Kibana's interface and visualizations:

![dashboard](Images/KibanaDockerMetricDash.jpg)

If metrics from all three web servers are displayed, then Metricbeat was successfully deployed. 

Navigate back to the DVWA web interface at http://20.121.37.42/. Now the security analyst can interact with the DVWA and view log and metric data in Kibana: http://104.42.182.122:5601/ 





**Congratulations** - this completes the full deployment of the ELK Stack monitoring project!










