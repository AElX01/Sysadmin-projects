-  *Set up three virtual machines, in this case, we'll use an Ubuntu Linux Server and two Windows 11 machines (I already have them installed on the system).*
- In this small project, I'll set up three VMs (two windows machines and one Ubuntu Server). The ubuntu server will run a docker container with port forwarding for ssh and apache services. The container will also run with limited resources as the Ubuntu server doesn't have too much hardware assigned to operate with.
- Physical equipment I'll use for this lab:
	- AMD Ryzen 5 5600H (laptop)
	- 32Gb RAM
	- 500Gb SSD
	- 

##### Improve VMs performance for the set-up
**VMs set up:**
- 4Gb ram for Windows machines
- 1GB ram for Linux server
- 4 cores (enable Intel VT-x or AMD-V option to improve performance and security as it will separate VMs via hardware and execute some tasks on hardware). Reserve 2 cores for the Linux server
- Reserve space as you wish
- Host-Only network adapter for Windows VMs, create two network adapters for the Linux server ensuring one of them is set as NAT to have internet access and download necessary packages 


##### Static IP assignment

| Windows host 1     | 192.168.100.252     |
| ------------------ | ------------------- |
| **Windows host 2** | **192.168.100.253** |
| **Linux server**   | **192.168.100.254** |

**Windows hosts**
1. On the Powershell, list all the network adapters available on the host, in my case, I only have one:
```powershell
Get-NetAdapter #list all network adapters
Get-NetAdapter | Select ifIndex #we can obtain the index of the network adapter we'll configure
```
	
	![image](https://github.com/AElX01/Sysadmin-projects/blob/5c66597a269ca16477ab6f3d95545071caba1f76/Images/Pasted%20image%2020240517151825.png)
	Remember we have the Host-Only network adapter enabled for our windows VMs, according to the "ipconfig" command, we can see DHCP is enabled but failed as our IP is an APIPA IP.
	
	![screenshot](screenshot.png)


2. **Disable DHCP in the interface you'll configure**
```powershell
Set-NetIPINterface -InterfaceIndex {index} -Dhcp Disabled
```
Check if it was satisfactory disabled by:
```bash
Get-NetIPConfiguration -InterfaceIndex {index} -Detailed
```

![screenshot](screenshot.png)

As we can see, dhcp ipv4 is disabled, thus we can assign now a static IP.


3. **Assign static ipv4**
```powershell
New-NetIPAddress -InterfaceIndex 15 -IpAddress 192.168.100.x -PrefixLength 24
```

![screenshot](screenshot.png)

![screenshot](screenshot.png)


4. **Ensure communication between machines**
Take into account that the Windows firewall blocks ICMP packets, before trying to ping a host, make sure you configure the firewall to allow ICMP packets from the inbound rules.


Host 1
![screenshot](screenshot.png)

Host 2
![screenshot](screenshot.png)



#### Linux server

**Install necessary packages**
Docker:
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update


sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```


**Static IP assignment**
```bash
ip a
```

![screenshot](screenshot.png)

- Turning on the interface and assign the static ip as follows:
```bash
sudo ifconfig ens37 up #in case is down or something
sudo ip addr add 192.168.100.254/24 dev ens37
```

#### Docker container

Dockerfile configuration
```dockerfile
FROM ubuntu:24.04

RUN apt update && \
	apt-install -y \
	net-tools \
	iputils-ping \
	curl \
	git \
	vim \
	apache2 \
	openssh-server \
	&& \
	rm -rf /var/lib/apt/lists

RUN useradd webuser && \
	useradd adminuser && \
	echo "webuser:web123" | chpasswd && \
	echo "adminuser:admin123" | chpasswd && 

RUN groupadd webgroup && \
	groupadd nfsgroup &&

RUN chown -R webuser:webgroup /var/www/html && \
	usermod -aG sudo adminuser

EXPOSE 22 80

CMD service ssh start && /usr/sbin/apache2ctl -D FOREGROUND
```


Build the image and run the container
```bash
docker build -t webserver /path/to/dockerfile
docker run -dti -p 8022:22 -p 8080:80 --name MyWebServer --memory "512m" --cpus="1" webserver #create docker container with port forwarding for http and ssh services, we also limit resources for the container
```


With all of this configurations, we should be able to access to the web resources on the container:

**Web access trough port 8080**
![screenshot](screenshot.png)

**SSH access to the machine**
![screenshot](screenshot.png)

We can check we are on the container by executing the "ifconfig" command:
![screenshot](screenshot.png)


*With this small project, you can familiarize a bit more with Linux and Windows systems as well as develop more confidence when navigating on a terminal*

