SUMMARY

This is a guide for setting up rootfull Podman containers for MYSQL / MARIADB Cent OS test beds.
Example uses 1 MySQL and 1 Mariadb setup.

------------------------------------------------------------------------------------
ENVIRONMENT

Cent OS: CentOS Linux release 8.2.2004 (Core)
Docker MySQL image: reducible/mysql:5.0.95
Docker MARIADB image: mariadb:10.2
Podman version: 1.6.4
User: root (IMPORTANT as we are using rootfull containers, which assigns IP to containers)

------------------------------------------------------------------------------------
PREPARATION

Install podman
> yum install podman -y

------------------------------------------------------------------------------------
MYSQL

Create persistent volume
> podman volume create mysql-volume

Create network interface
> podman network create docker-mysql

Check gateway
> vi /etc/cni/net.d/docker-mysql.conflist

In my case, it is having properties of
> "gateway": "10.89.0.1"

Pull container image and run container with gateway IP and port assigned
> podman run --name=mysql --network=docker-mysql -p 10.89.0.1:3306:3306 -v mysql-volume:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e TZ=Asia/Singapore -d reducible/mysql:5.0.95

*Left 3306 means exposed port of container to server. Right 3306 is the internal port used by MYSQL

------------------------------------------------------------------------------------
MARIADB

Create persistent volume
> podman volume create mariadb-volume

Create network interface
> podman network create docker-mariadb

Check gateway
> vi /etc/cni/net.d/docker-mariadb.conflist

In my case, it is having properties of
> "gateway": "10.89.1.1"

Pull container image and run container with gateway IP and port assigned
> podman run --name=mariadb --network=docker-mariadb -p 10.89.1.1:3306:3306 -v mariadb-volume:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -e TZ=Asia/Singapore -d mariadb:10.2

------------------------------------------------------------------------------------
PORT FORWARDING TO ALLOW EXTERNAL HOSTS TO CONNECT PRIVATE NETWORK OF DB

View iptables
> iptables -L --line-number

View iptables NAT configuration
> iptables -t nat -L -n --line-number | less

> iptables -t nat -v -L PREROUTING -n --line-number

> iptables -t nat -S PREROUTING

MYSQL port forwarding, set container IP as forward target, when incoming request is 3306
> iptables -A PREROUTING -t nat -i enp1s1 -p tcp --dport 3306 -j DNAT --to $(podman inspect -f "{{.NetworkSettings.IPAddress}}" mysql):3306

MARIADB port forwarding, set container IP as forward target, when incoming request is 3307
> iptables -A PREROUTING -t nat -i enp1s1 -p tcp --dport 3307 -j DNAT --to $(podman inspect -f "{{.NetworkSettings.IPAddress}}" mariadb):3306

Delete specific PREROUTING rule
> iptables -t nat -D PREROUTING <chainRuleNumber>

------------------------------------------------------------------------------------
MYSQLDUMP commands

Export mysqldump from container

> podman exec -it mysql mysqldump -uroot -p prbt > test-backup-$(date +%F).sql

Import mysqldump to container

> mysql --host=10.88.0.1 --port=3306 -u root -p prbt < test-insert.sql


*If host CentOS version do not support MySQL / MariaDB client, eg: Centos 8 does not support MYSQLv5.0.95 client, spawn an interactive shell

Copy file into container
> podman cp legacy_20200813.sql mysql:/

Opens interactive shell
> podman exec -it mysql /bin/bash

Run like normal
> mysql -u root -p < legacy_20200813.sql

------------------------------------------------------------------------------------
MISC

List all active containers
> podman ps

List all volumes
> podman volume ls

Get volume details
> podman inspect <first4letters of volume>

Remove all unused local volumes
> podman volume prune

Start container with running setttings
> podman start <containername>

Stop container
> podman stop <containername>

Remove container with running setttings
> podman rm -f <containername>

Opens an interactive bash shell for the container
> podman exec -it <containername> /bin/bash

Runs a one line command on the container
> podman exec -it <containername> <anyCommand>

List all images
> podman images -a 

Remove image
> podman rmi <IMAGEID>

Remove all unused images
> podman rmi -a (Deletes all images)

Get IP of container
> podman inspect -f "{{.NetworkSettings.IPAddress}}" <containername>

In case cannot "podman rm <containerName>", "Error: error creating container storage: the container name "xxx" is already in use "
> podman rm --storage <containerName>

View podman container volume usage
> docker system df --verbose


Alias podman to docker to emulate docker commands (Optional)
> vi ~/.bashrc

> alias docker="podman"


------------------------------------------------------------------------------------
REFERENCES

What is podman
https://developers.redhat.com/blog/2019/02/21/podman-and-buildah-for-docker-users/

Ancient 5.0.95 MySQL docker image
https://github.com/drpauldixon/docker-mysql-5.0

Detailed resources
https://stackoverflow.com/questions/60558185/is-it-possible-to-run-two-containers-in-same-port-in-same-pod-by-podman
https://towardsdatascience.com/connect-to-mysql-running-in-docker-container-from-a-local-machine-6d996c574e55
https://medium.com/@johnnyfekete/multiple-mysql-containers-in-docker-9dc5c01f7f7
https://stackoverflow.com/questions/30133664/how-do-you-list-volumes-in-docker-containers
https://www.redhat.com/sysadmin/container-networking-podman
https://www.systutorials.com/port-forwarding-using-iptables/
https://www.reddit.com/r/podman/comments/czypyp/container_questions_disk_space/
https://stackoverflow.com/questions/49959601/configure-time-zone-to-mysql-docker-container
