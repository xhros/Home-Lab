<h1>Setting up the ceph dashboard</h1>

This is my current environment: <br/>
Proxmox 8.4.1 <br/>
Ceph 19.2.1

This is a work in progress. I am continuing to update as I mess around with some of the more advanced features.

**Warning:** You will run into potential rabbit holes while setting this up. Those will be marked as such, so don't go chasing those unless you want to spend additional hours of your time on something that has no impact on getting the dashboard up and running.

Before we start, let's get some environmental variables setup for each manager node. What?! But why?!?! Honestly, they are easier to work with because global settings are applied to all nodes. This causes an IP conflict and the dashboard crashes on the slave nodes. This puts the cluster health in a persistent warning state, even after the issue is fixed. Basic troubleshooting steps and how to clear those are provided below. 

Do you like to live dangerously? Skip to the installation section and use the default global level settings. 

The dashboard module needs to be installed on every manager node, so each node will needs environmental variables setup. Make sure to update the IP address and server name to match the manager nodes details. Rabbit hole #1: You cannot use ports 443 or 80 since they are already allocated for use with [NGINX Proxy Manager](https://pve.proxmox.com/wiki/Web_Interface_Via_Nginx_Proxy). The dashboard uses the default ports of 8443 (secure) and 8080 (insecure), but you can change them to whatever you want. You really don't need to setup or mess with the insecure port, but why not have full control over it if it's there by default.

<h2>Section 1: Environmental Variables</h2>

<h4>Major benefits for using environmental variables</h4>
You will continue to run commands as you are working through the dashboard. It gets annoying having to update the commands to match each of the nodes any time you want to run a command you copied over. With environmental variables, you can set the variable and never have to reference an IP address, or node name. This is universal and can be used for anything running on the host, not just the dashboard.
<br/>
<br/>
Examples: These three commands:

```bash
journalctl -xeu ceph-mgr@node1
journalctl -xeu ceph-mgr@node2
journalctl -xeu ceph-mgr@node3
```

Can be replaced with this single command: 

```bash
journalctl -xeu ceph-mgr@$name
```

<h3>Let's get back to it</h3>


These commands need to be run on every manager node. Make sure to update the IP address and server name to match your setup. Unless you are running a different shell, you only need to worry about the bash commands since Proxmox uses Debian. You only need to run the commands to add the environmental variables for the shell that you are using. Move on to the the module install once you run those.

<h4>bash: ~/.bashrc</h4>

```bash
echo "export server_addr=[node IP]" >> ~/.bashrc
echo "export server_port=8080" >> ~/.bashrc
echo "export ssl_server_port=8443" >> ~/.bashrc
echo "export name=[serverName]" >> ~/.bashrc
source ~/.bashrc
```

<h4>zsh: ~/.zshrc</h4>

```bash
echo "export server_addr=[node IP]" >> ~/.zshrc
echo "export server_port=8080" >> ~/.zshrc
echo "export ssl_server_port=8443" >> ~/.zshrc
echo "export name=[serverName]" >> ~/.zshrc
source ~/.zshrc
```

<h4>oh-my-zsh: ~/.zshrc
  
```bash
echo "export server_addr=[node IP]" >> ~/.zshrc
echo "export server_port=8080" >> ~/.zshrc
echo "export ssl_server_port=8443" >> ~/.zshrc
echo "export name=[serverName]" >> ~/.zshrc
omz reload
```

**YOU MUST RELOAD YOUR SHELL FOR THE ENVIRONMENTAL VARIABLES TO TAKE EFFECT** 
<br/>
If you are having issues, run 'cat ~/.bashrc' for bash or 'cat ~/.zshrc' for zsh. You should see them just above the prompt since they are added to the bottom. If they are there and correct, just exit the shell and log back in, depending on how you access it.


<h2>Section 2: Installing the dashboard module</h2>

<h4>2a: Installing the module - Run on all nodes</h4>

```bash
apt install ceph-mgr-dashboard -y
```

<h4>Create a password file for the dashboard user</h4>
You can place the password file wherever you like. It is best to place it in the etc/pve/ceph folder since those are replicated to all nodes. You only need to run this once.</h4> Use whatever text editor you prefer. Normally you run chmod 600 on the password file, but since we are placing it in the /etc/pve/ceph folder, Proxmox handles the permissions.

```bash
nano /etc/pve/ceph/dashboard-pw
```

<h4>Creating a ceph dashboard admin user</h4>
Syntax: ceph dashboard ac-user-create [user] -i [/etc/pve/ceph/dashboard-pw] [role]

```bash
ceph dashboard ac-user-create admin -i /etc/pve/ceph/dashboard-pw administrator
```

<h2>Section 3a: Generate a self signed certificate - quick and dirty method</h2>

```bash
ceph dashboard create-self-signed-cert
```

<H2>Section 4a: Setting up the Dashboard Configs - Node Level</H2>
Must be performed on all nodes. Since we have setup environmental variables, just copy and paste the following lines on each node.

```bash
ceph config set mgr mgr/dashboard/$name/server_addr $server_addr
ceph config set mgr mgr/dashboard/$name/server_port $server_port
ceph config set mgr mgr/dashboard/$name/ssl_server_port $ssl_server_port
```

<h3>Checking the dashboard configs - Node Level </h3>
Note: You can copy and paste all three lines at the same time and they will spit out in the same order.

```bash
ceph config get mgr mgr/dashboard/$name/server_addr
ceph config get mgr mgr/dashboard/$name/server_port
ceph config get mgr mgr/dashboard/$name/ssl_server_port
```

<H2>Section 4b: Setting up the Dashboard Configs - Global Level</H2>
Technically, this isn't even needed since the dashboard uses the default configs. You can check the configs and adjust as needed. This is only performed on the master node since it's global. Note: Using :: as the server address binds to all IPv4 and IPv6 addresses.

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
You might get a warning that the dashboard is already disabled, but running it anyways in case it was enabled.

<h2> Accessing the dashboard </h2>
There are two ways to access the dashboard. Using the IP, or using the FQDN. From testing, I could navigate to the dashboard login page using the FQDN, but it would not acceept the password. However, if I navigated to the login page using the IP address, it would accept the password. I will journey down the rabbit hole to get the FQDN to work. 
<br/>
<br/>
Secure: https://[IP or FQDN]:8443 
<br/>
Insecure: http://[IP or FQDN]:8080

<h2>Troubleshooting</h2>
<h4>Ceph Crash Logs</h4>
This is at a global level since all crash logs are stored in the same place. No need to perform this on every node. Once all crash logs are handled and removed, the cluster health will return to normal.
<br/>
<br/>

Show a list of all crash logs
```bash
ceph crash ls
```

View a crash log
```bash
ceph crash info [CRASH_ID]
```

Remove a crash log
```bash 
ceph crash rm [CRASH_ID]
```

Restart the dashboard module on the node causing a crash once you've addressed it.
```bash
ceph mgr module disable dashboard && ceph mgr module enable dashboard
```

Rinse and repeat until no crashes are reported. You can use crash logs to find any other errors that need addressing as well. This will only show for items that have caused a crash, so other errors might not be shown since it may not have caused a crash. Address any dashboard errors first because you might see other crashes caused by the dashboard. This way you can rule out the dashboard as being the cause.

<h3>Manager Node Logs</h3>
**Warning!** Rabbit hole #2 through 23: You will see a lot of 'has missing NOTIFY_TYPES member' warnings when using the manager node logs. These have no bearing on the dashboard, so they can safely be ignored. There should be a set time that will automatically clear crash logs, but it's good practice to address them and remove them manually while troubleshooting. This will allow you to verify that all items have been addressed. 

```bash
journalctl -xeu ceph-mgr@$name
```
