<h1>Setting up the ceph dashboard</h1>

**Warning:** You will run into potential rabbit holes while setting this up. Those will be marked as such, so don't go chasing those unless you want to spend additional hours of your time on something that has no impact on getting the dashboard up and running.

Before we start, let's get some environmental variables setup for each manager node. What?! But why?!?! Honestly, they are easier to work with because global settings are applied to all nodes. This causes an IP conflict and the dashboard crashes on the slave nodes. This puts the cluster health in a persistent warning state, even after the issue is fixed. Basic troubleshooting steps and how to clear those are provided below. 

Do you like to live dangerously? Skip to the installation section and use the default global level settings. 

This dashboard needs to be installed on every manager node, so each node will needs environmental variables setup. Make sure to update the IP address and server name to match the manager nodes details. Rabbit hole #1: You cannot use ports 443 or 80 since they are already allocated for use with [NGINX Proxy Manager](https://pve.proxmox.com/wiki/Web_Interface_Via_Nginx_Proxy). The dashboard uses the default ports of 8443 (secure) and 8080 (insecure), but you can change them to whatever you want. You really don't need to setup or mess with the insecure port, but why not have full control over it if it's there by default.

<h2>Section 1: Environmental Variables</h2>
These commands need to be run on every manager node. Make sure to update the IP address and server name to match your setup. Unless you are running a different shell, you only need to worry about the bash command since Proxmox uses Debian. 

<h4>bash: ~/.bashrc</h4>

```bash
echo "export server_addr=[node IP]" >> ~/.bashrc
echo "export server_port=8080" >> ~/.bashrc
echo "export ssl_server_port=8443" >> ~/.bashrc
echo "export name=[serverName]" >> ~/.bashrc
source ~/.bashrc
```
<h4>zsh: ~/.zshrc - Skip this part if you are only using bash. If you are using oh-my-zsh, replace 'source ~/.zshrc' with 'omz reload'</h4>

```bash
echo "export server_addr=[node IP]" >> ~/.zshrc
echo "export server_port=8080" >> ~/.zshrc
echo "export ssl_server_port=8443" >> ~/.zshrc
echo "export name=[serverName]" >> ~/.zshrc
source ~/.zshrc
```

<h2>Section 2: Installing the dashboard module</h2>
Run all section 2 commands on all manager node

<h4>2a: Installing the module</h4>

```bash
apt install ceph-mgr-dashboard -y
```

<h4>create a password file for the dashboard user. This can be placed wherever</h4>

```bash
nano [/path/to/pw/file]
chmod 600 [/path/to/pw/file]
```
<h4>Create ceph dashboard admin user</h4>
Syntax: ceph dashboard ac-user-create [user] -i [/path/to/pw/file] [role]

```bash
ceph dashboard ac-user-create admin -i /path/to/pw/file administrator
```

<h2>Section 3: Generate a dashboard certificate - quick and dirty method using a self-signed certificate</h2>

```bash
openssl req -newkey rsa:4096 -nodes -x509 \
-keyout /etc/ceph/dashboard-key.pem -out /etc/ceph/dashboard-crt.pem -sha512 \
-days 3650 -subj "/CN=IT/O=ceph-mgr-dashboard" -utf8
```

<h4>Pointing the dashboard to use the self signed certs</h4>
<h4>Syntax: ceph dashboard ac-user-create [user] -i [/path/to/pw/file] [role]</h4>

```bash
ceph config-key set mgr/dashboard/key -i /etc/ceph/dashboard-key.pem
ceph config-key set mgr/dashboard/crt -i /etc/ceph/dashboard-crt.pem
```
<h3>Do you own a domain and use it internally, and want to generate a CA signed certificate?</h3>
Stay tuned for a future update on this section.

<H2>Setting up the Dashboard Configs - Node Level</H2>
Since we already setup the environmental variables, just copy and paste the following lines.

```bash
ceph config set mgr mgr/dashboard/$name/server_addr $server_addr
ceph config set mgr mgr/dashboard/$name/server_port $server_port
ceph config set mgr mgr/dashboard/$name/ssl_server_port $ssl_server_port
```

<h3>Checking the dashboard configs - Node Level </h3>

```bash
ceph config get mgr mgr/dashboard/$name/server_addr
ceph config get mgr mgr/dashboard/$name/server_port
ceph config get mgr mgr/dashboard/$name/ssl_server_port
```

<H2>Setting up the Dashboard Configs - Globally</H2>
We will be using the default ports. Change these to fit your needs. Using :: as the server address binds to all IPv4 and IPv6 addresses.

```bash
ceph config set mgr mgr/dashboard/server_addr [::]
ceph config set mgr mgr/dashboard/server_port [8080]
ceph config set mgr mgr/dashboard/ssl_server_port [8443]
```

<h3>Checking the dashboard configs - Globally </h3>

```bash
ceph config get mgr mgr/dashboard/server_addr
ceph config get mgr mgr/dashboard/server_port
ceph config get mgr mgr/dashboard/ssl_server_port
```
<h2>Starting the dashboard</h2>

```bash
ceph mgr module disable dashboard && ceph mgr module enable dashboard
```

<h2> Accessing the dashboard </h2>
There are two ways to access the dashboard. Using the IP, or using the FQDN. From testing, I could navigate to the dashboard login page using the FQDN, but it would not acceept the password. However, if I navigated to the login page using the IP address, it would accept the password. I will journey down the rabbit hole to get the FQDN to work. 

Secure: https://[IP or FQDN]:8443 
Insecure: http://[IP or FQDN]:8080

<h2>Troubleshooting</h2>
<h4>Ceph Crash Logs</h4>
This is at a global level since all crash logs are stored in the same place. No need to perform this on every node. Once all crash logs are handled and removed, the cluster health will return to normal.

```bash
# Show a list of all crash logs
ceph crash ls

# View a crash log
ceph crash info [CRASH_ID]

# Remove a crash log
ceph crash rm [CRASH_ID]

# Restart the dashboard module on all manager nodes to verify that all items were corrected
ceph mgr module disable dashboard && ceph mgr module enable dashboard

# This method also applies to any other errors that need addressing, not just the dashboard
# Any errors that haven't been addressed or missed will return after restarting the dashboard module
# Rinse and repeat until the dashboard health returns to normal
```

<h4>Manager Node Logs</h4>
**Warning!** Rabbit hole #2 through 23: You will see a lot of 'has missing NOTIFY_TYPES member' warnings when using the manager node logs. These have no bearing on the dashboard, so they can safely be ignored. All crash logs must be cleared before the dashboard returns to normal. There is a set time that these will be cleared, but it's good practice to fix them and remove them manually while troubleshooting. This will allow you to verify that all items are addressed. 

```bash
journalctl -xeu ceph-mgr@[serverName]
```
