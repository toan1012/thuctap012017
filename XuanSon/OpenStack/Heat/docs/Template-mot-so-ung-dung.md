# Template 1 số ứng dụng

# MỤC LỤC

# 1.Wordpress
## 1.1.Cài database và wordpress trên 1 server
```
heat_template_version: 2017-09-01

description: Create a instance

parameters:
  public_network:
    type: string
    label: Public network name or ID
    description: Public network with floating IP addresses.
    default: "provider"
  cidr:
    type: string
    label: Network CIDR
    description: The CIDR of the private network.
    default: "10.1.0.0/24"
  dns:
    type: comma_delimited_list
    label: DNS nameservers
    description: Comma separated list of DNS nameservers for the private network.
    default: "8.8.8.8,8.8.4.4"
  image:
    type: string
    label: Image name or ID
    description: Image to be used for compute instance
    default: "ubuntu16.04"
  flavor:
    type: string
    label: Flavor
    description: Type of instance (flavor) to be used
    default: "medium"
  root_pass:
    type: string
    label: mysql root password
    description: mysql root password
    default: welcome
  db_name:
    type: string
    label: database name
    description: database name
    default: wordpress
  db_user:
    type: string
    label: database user
    description: database user
    default: wordpress_user
  db_pass:
    type: string
    label: database password
    description: database password
    default: welcome123

resources:
  instance:
    type: OS::Nova::Server
    properties:
      flavor: {get_param: flavor}
      image: {get_param: image}
      networks: 
        - "network": {get_resource: new_network}
      key_name: {get_resource: my_key}
      user_data_format: RAW
      user_data:
        str_replace:
          params:
            __ROOT_PASS__: {get_param: root_pass}
            __DB_NAME__: {get_param: db_name}
            __DB_USER___: {get_param: db_user}
            __DB_PASS__: {get_param: db_pass}
          template: |
            #!/bin/bash
            apt update -y && apt upgrade -y
            apt-get install apache2 -y
            rm /var/www/html/index.html
            #
            apt-get install debconf-utils -y
            debconf-set-selections <<< "mysql-server mysql-server/root_password password __ROOT_PASS__"
            debconf-set-selections <<< "mysql-server mysql-server/root_password_again password __ROOT_PASS__"
            apt-get install mysql-server php7.0-mysql -y
            apt-get install php7.0 libapache2-mod-php7.0 php7.0-mcrypt -y
            #
            cat << EOF | mysql -u root -p__ROOT_PASS__
            CREATE DATABASE __DB_NAME__;
            CREATE USER __DB_USER___@localhost IDENTIFIED BY '__DB_PASS__';
            GRANT ALL PRIVILEGES ON __DB_NAME__.* TO __DB_USER___@localhost;
            FLUSH PRIVILEGES;
            EOF
            #
            wget http://wordpress.org/latest.tar.gz
            tar xzvf latest.tar.gz
            apt-get install php7.0-gd
            #
            cp wordpress/wp-config-sample.php wordpress/wp-config.php
            cp -r wordpress/* /var/www/html
            #
            sed -i 's/database_name_here/__DB_NAME__/g' /var/www/html/wp-config.php
            sed -i 's/username_here/__DB_USER___/g' /var/www/html/wp-config.php
            sed -i 's/password_here/__DB_PASS__/g' /var/www/html/wp-config.php
            #
            chown -R www-data:www-data /var/www/html

  new_network:
    type: OS::Neutron::Net 
  new_subnet:
    type: OS::Neutron::Subnet 
    properties:
      network: {get_resource: new_network}
      cidr: {get_param: cidr}
      dns_nameservers: {get_param: dns}
      ip_version: 4
  new_router:   
    type: OS::Neutron::Router
    properties:
      external_gateway_info: {"network": {get_param: public_network}}
  subnet_router:
    type: OS::Neutron::RouterInterface
    properties:
      router: {get_resource: new_router}
      subnet: {get_resource: new_subnet}
  floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: {get_param: public_network}
  association_floating_ip:
    type: OS::Neutron::FloatingIPAssociation
    properties:
      floatingip_id: {get_resource: floating_ip}
      port_id: {get_attr: [instance, addresses, {get_resource: new_network}, 0, port]}
  my_key:
    type: OS::Nova::KeyPair
    properties:
      save_private_key: true
      name: "my_key"


outputs:
  instance_ip_floating:
    description: Floating Ip of instance
    value: {get_attr: [floating_ip , floating_ip_address]}
  instance_ip_private:
    description: Ip private of instance
    value: {get_attr: [instance , networks, {get_resource: new_network}, 0]}
  private_key:
    description: Private key
    value: {get_attr: [my_key, private_key]}
```


## 1.2.Cài database và wordpress trên 2 server
```
heat_template_version: 2017-09-01

description: Create a instance

parameters:
  public_network:
    type: string
    label: Public network name or ID
    description: Public network with floating IP addresses.
    default: "provider"
  cidr:
    type: string
    label: Network CIDR
    description: The CIDR of the private network.
    default: "10.1.0.0/24"
  dns:
    type: comma_delimited_list
    label: DNS nameservers
    description: Comma separated list of DNS nameservers for the private network.
    default: "8.8.8.8,8.8.4.4"
  image:
    type: string
    label: Image name or ID
    description: Image to be used for compute instance
    default: "ubuntu16.04"
  flavor:
    type: string
    label: Flavor
    description: Type of instance (flavor) to be used
    default: "medium"
  root_pass:
    type: string
    label: mysql root password
    description: mysql root password
    default: welcome
  db_name:
    type: string
    label: database name
    description: database name
    default: wordpress
  db_user:
    type: string
    label: database user
    description: database user
    default: wordpress_user
  db_pass:
    type: string
    label: database password
    description: database password
    default: welcome123


resources:
  database_instance:
    type: OS::Nova::Server
    properties:
      flavor: {get_param: flavor}
      image: {get_param: image}
      networks: 
        - "network": {get_resource: new_network}
      key_name: {get_resource: my_key}
      user_data_format: RAW
      user_data:
        str_replace:
          params:
            __ROOT_PASS__: {get_param: root_pass}
            __DB_NAME__: {get_param: db_name}
            __DB_USER___: {get_param: db_user}
            __DB_PASS__: {get_param: db_pass}
          template: |
            #!/bin/bash
            apt update -y && apt upgrade -y
            apt-get install debconf-utils -y
            debconf-set-selections <<< "mysql-server mysql-server/root_password password __ROOT_PASS__"
            debconf-set-selections <<< "mysql-server mysql-server/root_password_again password __ROOT_PASS__"
            apt-get install mysql-server -y
            sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/mysql.conf.d/mysqld.cnf
            systemctl restart mysql
            #
            cat << EOF | mysql -u root -p__ROOT_PASS__
            CREATE DATABASE __DB_NAME__;
            CREATE USER __DB_USER___@'%' IDENTIFIED BY '__DB_PASS__';
            GRANT ALL PRIVILEGES ON __DB_NAME__.* TO __DB_USER___@'%';
            FLUSH PRIVILEGES;
            EOF
            #

  wordpress_instance:
    type: OS::Nova::Server
    properties:
      flavor: {get_param: flavor}
      image: {get_param: image}
      networks: 
        - "network": {get_resource: new_network}
      key_name: {get_resource: my_key}
      user_data_format: RAW
      user_data:
        str_replace:
          params:
            __DB_HOST__: {get_attr: [database_instance, networks, {get_resource: new_network}, 0]}
            __DB_NAME__: {get_param: db_name}
            __DB_USER___: {get_param: db_user}
            __DB_PASS__: {get_param: db_pass}
          template: |
            #!/bin/bash
            apt update -y && apt upgrade -y
            apt-get install apache2 -y
            rm /var/www/html/index.html
            #
            apt-get install php7.0-mysql -y
            apt-get install php7.0 libapache2-mod-php7.0 php7.0-mcrypt -y
            #
            wget http://wordpress.org/latest.tar.gz
            tar xzvf latest.tar.gz
            apt-get install php7.0-gd
            #
            cp wordpress/wp-config-sample.php wordpress/wp-config.php
            cp -r wordpress/* /var/www/html
            #
            sed -i 's/database_name_here/__DB_NAME__/g' /var/www/html/wp-config.php
            sed -i 's/username_here/__DB_USER___/g' /var/www/html/wp-config.php
            sed -i 's/password_here/__DB_PASS__/g' /var/www/html/wp-config.php
            sed -i 's/localhost/__DB_HOST__/g' /var/www/html/wp-config.php
            #
            chown -R www-data:www-data /var/www/html

  new_network:
    type: OS::Neutron::Net 
  new_subnet:
    type: OS::Neutron::Subnet 
    properties:
      network: {get_resource: new_network}
      cidr: {get_param: cidr}
      dns_nameservers: {get_param: dns}
      ip_version: 4
  new_router:   
    type: OS::Neutron::Router
    properties:
      external_gateway_info: {"network": {get_param: public_network}}
  subnet_router:
    type: OS::Neutron::RouterInterface
    properties:
      router: {get_resource: new_router}
      subnet: {get_resource: new_subnet}
  floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: {get_param: public_network}
  association_floating_ip:
    type: OS::Neutron::FloatingIPAssociation
    properties:
      floatingip_id: {get_resource: floating_ip}
      port_id: {get_attr: [wordpress_instance, addresses, {get_resource: new_network}, 0, port]}
  my_key:
    type: OS::Nova::KeyPair
    properties:
      save_private_key: true
      name: "my_key"


outputs:
  database_instance_name:
    description: Name of database instance
    value: {get_attr: [database_instance, name]}
  wordpress_instance_name:
    description: Name of wordpress instance
    value: {get_attr: [wordpress_instance, name]}
  database_instance_ip:
    description: IP of database instance
    value: {get_attr: [database_instance , networks, {get_resource: new_network}, 0]}
  wordpress_ip_floating:
    description: Floating Ip of wordpress instance
    value: {get_attr: [floating_ip , floating_ip_address]}
  wordpress_ip_private:
    description: Ip private of wordpress instance
    value: {get_attr: [wordpress_instance , networks, {get_resource: new_network}, 0]}
  private_key:
    description: Private key for database and wordpress instances
    value: {get_attr: [my_key, private_key]}
```