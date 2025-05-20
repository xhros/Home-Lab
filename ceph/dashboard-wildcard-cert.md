<h2>Generating a Wildcard Cert</h2>
This is working under the assumption that you already have valid certificates for your Proxmox nodes. Not sure why, but Proxmox does not seem to use your existing certs for the ceph dashboard.

Let's pretend for a second that they did. That would be great, right? Well, it's a bit more complicated than that. Why is that? It's because it's a cluster with a master node. Two problems arise with this. 1) It uses a single certificate at a global level. Not sure if it's possible to set it per node since the command to apply a cert does't involve specifying a node like the option we had when setting up the dashboard. 2) Redirects. 

Let's say you are using valid certs for node#.yourDomain.com for each of your nodes, which node 1 being the master. If you navigate to either of the slave nodes ceph dashboard, it will redirect you to the master nodes dashboard. For some reason, it's not a true redirect. Your browser will show the master nodes address, but it still thinks your at node3. This will cause certificate errors for your nodes, not just the dashboard. Talk about a headache.

I learned this the hard way, and I'm sharing what I did to resolve this issue. Since proxmox does not generate wildcard certificates, we will use a third party tool called [acme.sh](https://github.com/acmesh-official/acme.sh). We are also using Let's Encrypt and Cloudflare. 

Store your Cloudflare token. One or the other, but not both (whichever shell you are using)

```bash
echo "export CF_Token="YourCloudflareToken"" >> ~/.bashrc
echo "export CF_Token="YourCloudflareToken"" >> ~/.zshrc
```

Reload your shell (whichever shell you are using)

```bash
source ~/.bashrc #bashrc
source ~/.zshrc #zsh
omz reload #oh-my-zsh
```

Install acme.sh

```bash
curl https://get.acme.sh | sh -s email=[your@email.com]
```

Set Let's Encrypt as default and generate wildcard domain certificate

```bash
acme.sh --set-default-ca --server letsencrypt
acme.sh --issue --dns dns_cf --keylength 4096 -d [domain.com] -d '*.[domain.com]'
```

Verify that the certificates are valid and that both outputs match.

```bash
openssl x509 -noout -modulus -in ~/.acme.sh/[domain.com]/fullchain.cer
openssl rsa -noout -modulus -in ~/.acme.sh/[domain.com]/[domain.com].key
```

Copy certs to Proxmox cluster ceph directory. The .cer file is copied over as .pem.

```bash
cp ~/.acme.sh/[domain.com]/fullchain.cer /etc/pve/ceph/[domain.com].pem
cp ~/.acme.sh/[domain.com]/[domain].key /etc/pve/ceph/[domain.com].key
```

Install the certs and restart the dashboard

```bash
ceph dashboard set-ssl-certificate -i /etc/pve/ceph/[domain.com].pem
ceph dashboard set-ssl-certificate-key -i /etc/pve/ceph/[domain.com].key
ceph mgr module disable dashboard && ceph mgr module enable dashboard
```
