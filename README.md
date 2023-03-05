<p align="center">
  <img src="./rabbitmq_logo_strap.png" alt="Logo">
  
</p>
<img src="https://futurumresearch.com/wp-content/uploads/2020/01/aws-logo.png" alt="AWS Logo" width="500"/>

<link rel="stylesheet" type="text/css" href="./style.css">



# **How to Set Up a 3-Node Unmanaged Cluster of RabbitMQ on AWS**

RabbitMQ is a popular open-source message broker that supports various protocols and platforms. It can be used to implement reliable and scalable distributed systems that communicate through messages.

This walkthrough explains how to set up a 3-node unmanaged cluster of RabbitMQ on AWS with 1 master and 2 worker nodes. In an unmanaged cluster, nodes are manually configured and managed without using any tools or frameworks.

For this lab environment, we will use t2.micro instances, which are sufficient for our purposes. However, you may choose different instance types depending on your needs and budget.

## **Prerequisites**

Before starting the setup, ensure you have the following:

- An AWS account with access to EC2 service
- A key pair for SSH access to the instances
- A security group that allows inbound traffic on all ports (for simplicity)

## **Step 2: Configure Hostnames and Hosts File**

To enable communication between nodes by name instead of IP address, we need to configure the hostnames and hosts file of each instance. Follow these steps:

1. SSH into each instance using your key pair and username ubuntu. For example:
    
    ```
    ssh -i <key_pair.pem> ubuntu@<public_ip>
    
    ```
    
    Use terminator or any other terminal emulator for better usability.
    
2. In each node, type:
    
    ```
    sudo nano /etc/hosts
    
    ```
    
    Enter the IP address and hostname of all nodes below 127.0.0.1 localhost line. For example:
    
    ```
    127.0.0.1 localhost
    18.222.111.222 rabbit-master
    3.144.222.111 rabbit-worker1
    18.188.333.111 rabbit-worker2
    
    ```
    
    Save and exit by pressing Ctrl+O followed by Ctrl+X.
    
3. Use the command below on each node respectively:
    
    ```
    sudo hostnamectl set-hostname <name_of_node> --static
    
    ```
    
    For example:
    
    ```
    sudo hostnamectl set-hostname rabbit-master --static
    
    ```
    
    On the first node,
    
    ```
    sudo hostnamectl set-hostname rabbit-worker1 --static
    
    ```
    
    On the second node,
    
    And so on.
    
4. Reboot each system by typing:
    
    ```
    sudo reboot
    
    ```
    
    SSH into each instance again after they are back online. Notice the change in the prompt which might look something like ubuntu@rabbit-master: or whatever you have set it to.
    

## **Step 3: Install RabbitMQ Server**

Now, we are ready to install RabbitMQ server on each node using this official guide: **[https://www.rabbitmq.com/install-debian.html](https://www.rabbitmq.com/install-debian.html)**

We will use this script to automate the installation process:

The user data script is:

```
#!/usr/bin/sh

sudo apt-get install curl gnupg apt-transport-https -y

## Team RabbitMQ's main signing key
curl -1sLf "<https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA>" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null
## Launchpad PPA that provides modern Erlang releases
curl -1sLf "<https://keyserver.ubuntu.com/pks/lookup?op=get&search=0xf77f1eda57ebb1cc>" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/net.launchpad.ppa.rabbitmq.erlang.gpg > /dev/null
## PackageCloud RabbitMQ repository
curl -1sLf "<https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey>" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/io.packagecloud.rabbitmq.gpg > /dev/null

## Add apt repositories maintained by Team RabbitMQ
sudo tee /etc/apt/sources.list.d/rabbitmq.list <<EOF
## Provides modern Erlang/OTP releases
##
## "bionic" as distribution name should work for any reasonably recent Ubuntu or Debian release.
## See the release to distribution mapping table in RabbitMQ doc guides to learn more.
deb [signed-by=/usr/share/keyrings/net.launchpad.ppa.rabbitmq.erlang.gpg] <http://ppa.launchpad.net/rabbitmq/rabbitmq-erlang/ubuntu> bionic main
deb-src [signed-by=/usr/share/keyrings/net.launchpad.ppa.rabbitmq.erlang.gpg] <http://ppa.launchpad.net/rabbitmq/rabbitmq-erlang/ubuntu> bionic main

## Provides RabbitMQ
##
## "bionic" as distribution name should work for any reasonably recent Ubuntu or Debian release.
## See the release to distribution mapping table in RabbitMQ doc guides to learn more.
deb [signed-by=/usr/share/keyrings/io.packagecloud.rabbitmq.gpg] <https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/> bionic main
deb-src [signed-by=/usr/share/keyrings/io.packagecloud.rabbitmq.gpg] <https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/> bionic main
EOF

## Update package indices
sudo apt-get update -y

## Install Erlang packages
sudo apt-get install -y erlang-base \\
                        erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \\
                        erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \\
                        erlang-runtime-tools erlang-snmp erlang-ssl \\
                        erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl

## Install rabbitmq-server and its dependencies
sudo apt-get install rabbitmq-server -y --fix-missing

```

Follow these steps:

1. Check the version of rabbitmqctl on each node to see if it is working correctly:
    
    ```
    sudo rabbitmqctl --version
    
    ```
    
2. Start, enable and check the status of rabbitmq-server service on each node:
    
    ```
    sudo systemctl start rabbitmq-server
    sudo systemctl enable rabbitmq-server
    sudo systemctl status rabbitmq-server
    
    ```
    
3. Copy the Erlang cookie of the master node (node1) to all other nodes. The Erlang cookie is a file that contains a secret token that allows nodes to communicate with each other. You can find it in `/var/lib/rabbitmq/.erlang.cookie`. On node1, run:
    
    ```
    sudo cat /var/lib/rabbitmq/.erlang.cookie
    
    ```
    
    Copy the output and paste it on node2 and node3 by running:
    
    ```
    sudo su
    echo "copied_cookie_of_master_node" > /var/lib/rabbitmq/.erlang.cookie
    cat /var/lib/rabbitmq/.erlang.cookie #to verify if the changes have been made
    exit #to exit from root user
    
    ```
    
4. Restart rabbitmq-server service on node2 and node3:
    
    ```
    sudo systemctl restart rabbitmq-server
    
    ```
    
5. Stop the RabbitMQ application on node2 and node3:
    
    ```
    sudo rabbitmqctl stop_app
    
    ```
    
6. Join node2 and node3 to the cluster by running:
    
    ```
    sudo rabbitmqctl join_cluster rabbit@node1
    #replace node1 with your master node hostname
    
    ```
    
7. Start the RabbitMQ application on node2 and node3:
    
    ```
    sudo rabbitmqctl start_app
    
    ```
    
8. On node1, check the cluster status by running:
    
    ```
    sudo rabbitmqctl cluster_status
    
    ```
    
    
    
9. Add a new admin user to your RabbitMQ cluster by running:
    
    ```
    sudo rabbitmqctl add_user name password
    sudo rabbitmqctl set_user_tags name administrator
    rabbitmqctl set_permissions -p / name ".*" ".*" ".*"
    
    ```
    
    Replace name and password with your desired username and password.
    
10. Delete the default guest user by running:
    
    ```
    sudo rabbitmqctl delete_user guest
    
    ```
    
11. Check all users by running:
    
    ```
    sudo rabbitmqctl list_users
    
    ```
    
12. Set a policy to replicate all queues across all nodes by running:
    
    ```
    sudo rabbitmqctl set_policy ha-all "" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
    
    ```
    
13. Check all policies by running:
    
    ```
    sudo rabbitmqctl list_policies
    
    ```
    
14. Enable the RabbitMQ management plugin on all nodes by running:
    
    ```
    sudo rabbitmq-plugins enable rabbitmq_management
    
    ```
    
15. Allow TCP traffic on port 15672 (the default port for management web interface) by running:
    
    ```
    sudo ufw allow proto tcp from any to any port 15672
    
    ```
    
16. Access the management web interface at **[http://node1:15672](http://node1:15672/)** (replace **`node1`** with your master node hostname) and sign in using your username and password.
    

