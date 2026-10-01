# Cluster k3s with cloned RHEL VMs — step by step

(This is what I use as my setup for the ArgoCD GitOps lab and honestly most, if not all of my labs that use kubernetes)

**Plan:** 1 VM as a server (control plane) + 2 or more cloned VMs as workers. You can starts with just 1 worker and add more later, the process is the same.

---

## Step 0 — Before cloning: set the trap

When you clone a VM, the copy is born with:

- the **same hostname** (or some random hostname),
- sometimes even **the same IP** (or an ip that will change)

If we don't fix this in every VM, you're going to have nodes that maybe one day don't even have an IP, and fixing it is a mess. We're going to fix this in Step 2, before you install anything kubernetes related, and before we even SSH into it. These couple of minutes can save you hours of troubleshooting later (based on my sad experience)
Just wanted to add that I started with 2GB of memory and then added more memory to the control plane (server). This is why in the images you can see that the two workers have less memory than the control plane, other that that, they're clones of each other.

![instana-lab-server](https://github.com/karenmontesca/k3-lab-setup/blob/main/img/labgitserver-clone.png)

![instana-lab-worker#](https://github.com/karenmontesca/k3-lab-setup/blob/main/img/labgitworker-clone.png)

---

## Step 1 — Clone the VM base

From the hypervisor (vSphere, Proxmox, virt-manager, lo que uses), clone your base VM as many times as nodes you want (I create a VM, the install RHEL in minimal mode, after I make sure it works and i'm signed in and all, I clone). For example:

- `instana-lab-server` (will be the control plane)
- `instana-lab-worker1`
- `instana-lab-worker2`

Let's turn them on

---

## Step 2 — In each clone that we just turned on, let's fix the identity and give it a fixed IP

Connect to it (DO NOT SSH INTO IT JUST YET, because we still have to fix the IP address):

```bash
# 1. Give it a unique hostname
sudo hostnamectl set-hostname instana-lab-worker1

# Look for the Ip address
ip a

# Take note of the ip address

# 2. Assign the same IP address as static and add the mask /XX (example below)
nmcli connection modify ens160 ipv4.addresses 192.168.189.131/24

# 3. Configure the gateway detected
nmcli connection modify ens160 ipv4.gateway 192.168.189.2

# 4. Assign the DNS servers (your gateway and the google public dns)
nmcli connection modify ens160 ipv4.dns "192.168.189.2 8.8.8.8"

# 5. Change DHCP to manual
nmcli connection modify ens160 ipv4.method manual

# 6. Apply the changes to the interface
nmcli connection up ens160

# 7. Reboot
sudo systemctl reboot

```

Repeat this in every cloned VM (changing hostname and IP address accordingly of course).
Now you can finally SSH into the VMs with your favorite SSH of choice. I've been using MobaXterm recently, it's pretty good, you can even create a folder for your sessions (ssh conections)

---

## Step 3 — Verify conectivity between the VMs

If you want to simplify any possible conectivity problem just:

```bash
sudo systemctl stop firewalld
```
```bash
sudo systemctl status firewalld
```
If not, we can open only certain ports:
From the worker to the server: 

```bash
ping 192.168.1.50
```

k3s needs these ports to be open **between** the VMs (check `firewalld`, in RHEL):

```bash
# In the server:
sudo firewall-cmd --permanent --add-port=6443/tcp     # API Kubernetes
sudo firewall-cmd --permanent --add-port=10250/tcp    # kubelet
sudo firewall-cmd --permanent --add-port=8472/udp     # internal network
sudo firewall-cmd --reload

# In each worker:
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --reload
```

---

## Step 4 — Install k3s in the server

For exaple in `instana-lab-server`:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verify it:

```bash
sudo k3s kubectl get nodes
```

You should see it as `Ready`.

---

## Step 5 — Get the token to join the worker nodes

Still in the server:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy the complete value (probably starts something like `K10a3b...`).

---

## Step 6 — Join each worker to the main cluster

In `instana-lab-worker1` (and then you repeat in worker2, worker3, etc.):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<ip-of-the-control-plane-node>:6443 K3S_TOKEN=<the-token-you-copied> sh -
```

This installs the k3 agent in worker mode and connects in automatically to the server (control plane)

---

## Step 7 — Verify that all the nodes show up

Go back to the main server (control plane):

```bash
sudo k3s kubectl get nodes
```

You should be seeing something like this:

```
NAME                   STATUS   ROLES                  AGE   VERSION
instana-lab-server     Ready    control-plane,master   10m   v1.30.x+k3s1
instana-lab-worker1    Ready    <none>                 2m    v1.30.x+k3s1
instana-lab-worker2    Ready    <none>                 1m    v1.30.x+k3s1
```

If one of the workers doesn't show up, chek the logs in it:

```bash
sudo journalctl -u k3s-agent -f
```

90% of the times, the problem is firewall related or that the token/IP of the server is wrong.

---


And that's it, now you can proceed.