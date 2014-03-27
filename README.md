#Readme
This ansible role deploys zabbix for Ubuntu 12.04 (tested on vagrant)

##Prerequisite
* Having ansible installed on your workstation. 
* *Optional* postgresql and mysql server (this playbook can install postgresql and mysql(experimental)
* *Optional* Use Ansible Zabbix module to dynamically add Zabbix host groups and hosts to your Zabbix server https://github.com/ahelal/ansible-zabbix_modules

##How to install
* Use github to clone/fork in your role directory
* ansible galaxy ```ansible-galaxy install adham.helal.zabbix_server```
* *Note:* if you intend to install database I would recommend to use one of the following
  * ```ansible-galaxy install Ansibles.postgresql```
  * ```ansible-galaxy install Ansibles.mysql```

## Design
With Zabbix it is easy to change from one design to another, at least in theory. I am not going to cover the design here only the basics.
Zabbix  has the following component
* Database (This playbook support postgresql and mysql(experimental) )
* Zabbix Server or Zabbix Proxy server
* Zabbix Frontend (Apache2 and php)
* Zabbix agent (Will be installed by default)
* SSH tunnel (to secure communication and to add authentication)
* Zabbix Java gateway (currently not supported)

### Types of zabbix Installation
1. Standalone
2. Distributed
3. HA (not supported) 


### Standalone
Simplest installation and is the default of this playbook. Deploy Zabbix server, Frontend and the DB one the same host. Most probably that can handle at least a couple of hundreds hosts if deployed on reasonable server. And I would recommand to play with that first

### Distributed
Deploy components on different hosts. A common design is to deploy Zabbix server on a host.  DB on another and fronetend on another. Then, if needed N proxy servers. Depending on your use you might want to deploy with different settings

By changing the following varaibles you can manage what to deploy on your host.

```zabbix_server_install : True``` Deploy  zabbix 'server' or ‘proxy‘

```zabbix_server_install_type : server``` Deploy  zabbix  either 'server' or ‘proxy‘

```zabbix_server_front_install : True``` Deploy frontend php and apache

```zabbix_server_db_install : True``` Deploy database



### HA 
This can't be covered as it needs very custom configuration for databases, load balancers and other tools depending on your design. If you venture to build such a system you most probably can build on top of this playbook also. Here are some resources that can help you.
* https://www.zabbix.org/wiki/Docs/howto/high_availability
* http://blog.zabbix.com/scalable-zabbix-lessons-on-hitting-9400-nvps/2615/


##Variables 
All default variables are located **defaults/main.yml**.

  - *zabbix_server_host:*  Your zabbix server  ```zabbix_server_host: "zabbix.example.com"```

  - *zabbix_server_name:* Zabbix server name  ```zabbix_server_name : "Zabbix Server"```
  
  - *zabbix_server_timezone:* Time zone  ```zabbix_server_timezone : ""America/Los_Angeles"``

  - *zabbix_server_db_type:* DB type either pgsql or mysql   ```zabbix_server_db_type : "pgsql"```
  
  - *zabbix_server_db_host:* DB hostname   ```zabbix_server_db_host : "localhost"```    
  
  - *zabbix_server_db_user:* DB username   ```zabbix_server_db_user : "zabbix"```

  - *zabbix_server_db_pass:* DB password  ```zabbix_server_db_pass : "password"```

  - *zabbix_server_db_name:* DB name ```zabbix_server_db_name : "zabbix"```
  
  - *zabbix_server_db_port:* DB port ```zabbix_server_db_port : "5432"```

##Zabbix over SSH (Optional)
By default Zabbix communication between agent and server is in plain text and no authentication. If monitoring over the internet you might want to use ssh tunneling.

Here is an example of how Zabbix agent over ssh will work
- A user will be created on the target host and key will be deployed that user has no terminal rights and can only bind to one port default port 10500
- Our Zabbix agent(s) will connect to **localhost** **10500** 
- A tunnel will be created so connection to that part will be tunneled to the server
- On the server a different port will be assigned to each host. So you can add hosts like localhost AssignedPort
- Reverse tunnel is used for active check

### Configuring 

For more details and other options look at 

1. defaults/main.yml tunnel section
2. templates/tunnel_mgt.j2

First you need to enable *zabbix_server_tunnel:*  ```zabbix_server_tunnel : True``` and assigned port for each host must be unique   this can be managed.

####1 **Statically**
In your hostvars for each host ```ZabbixSSH: 11212``` and make sure every port is unique
here is an example of tunnel creation task

```
##You must define a per host variable ZabbixSSH with the desired port
- name : tunnel_mgt | Loop over inventory and create assh connection 
  template   :
    src=templates/assh/tunnel_mgt.j2
    dest={{zabbix_server_tunnel_etc}}/{{hostvars[item]["inventory_hostname"]}}
    owner="{{zabbix_server_tunnel_user}}"
    mode=0644 
  with_items : groups['all']
  when       : "zabbix_server_tunnel == True"
  notify     : restart assh
  tags       : tunnel_mgt
```

####2 **Dynamically**
You can let the ```with_indexed_items``` loop and use that and add to base number. Have a look at templates/tunnel_mgt.j2

```
##A more dynamic way to do that is to use sequence
- name : tunnel_mgt | Loop over inventory and create assh connection
  template   :
    src=templates/assh/tunnel_mgt.j2
    dest={{zabbix_server_tunnel_etc}}/{{hostvars[item.1]["inventory_hostname"]}}
    owner="{{zabbix_server_tunnel_user}}"
    mode=0644 
  with_indexed_items: groups['all']
  when       : "zabbix_server_tunnel == True"
  notify     : restart assh
  tags       : tunnel_mgt
```

**Note: You would need to use the same code to create hosts in zabbix to match the ports same logic**

####3 Find another way and let me know :) 


##Configure
You can configure your variables in ansible with one of the following

 * Create a variable in host/group variables directory (recommend)
 * Editing var/main.yml
 * Run ansible-playbook with -e
 * Edit the default/main.yml (not recommended)

##Run
    
  ```ansible-playbook -l hostname zabbix_server_postgresql.yml```
