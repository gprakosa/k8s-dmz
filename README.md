# DMZ-ing Your Kubernetes!

## 1. Use-Case Requirement

Your company requires to separate its datacenter into two different networks for security reason:

1. **Server Farm** where is all your default infrastructures and backends reside, and
2. **DMZ** or (Demilitarized Zone) where is the frontends exposed and more likely published also to the Internet. 

## 2. Infrastructure Preparation

You might then start do some research and decide to built a homelab. I've made my own using a Windows 10 PC (RAM 32GB) with Hyper-V enabled by create 3 following VMs:

VM     | RAM | OS             | Kubernetes |
------ | --- | -------------- | ---------- |
router | 1GB | Ubuntu 20.04.6 | N/A        |
master | 4GB | Ubuntu 20.04.6 | v1.28.2    |
worker | 2GB | Ubuntu 20.04.6 | v1.28.2    |

with its network configuration as follow:

VM     | eth0                        | eth1        | eth2                                 |
------ | --------------------------- | ----------- | ------------------------------------ |
router | 10.81.56.1                  | 10.81.57.1  | DHCP (bridge from PC, default route) |
master | 10.81.56.11 (default route) | 10.81.57.11 | N/A                                  |
worker | 10.81.56.12 (default route) | 10.81.57.12 | N/A                                  |

In summary:
1. Router will act as gateway for traffic forwarding between cluster, PC, and the Internet.
2. Network 10.81.56.0/24 will simulate Server Farm network, and
3. Network 10.81.57.0/24 will simulate DMZ network.

### Infra Setup

These activities are consider as common things that might includes:

+ Set IP address (mandatory)
+ Set SSH Pubkey 
+ Set Aliases `~/.bash_aliases`
+ Set Bypass SUDO
+ Set Hostname
+ Set Timezone
+ Set Firewall (advance)

### Traffic Forward Setup

#### Router VM

> Enable IP Forward
```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
sysctl net.ipv4.ip_forward
```

> Iptables
```bash
iptables -A FORWARD -j ACCEPT
iptables -t nat -s 10.81.56.0/24 -A POSTROUTING -j MASQUERADE
iptables -t nat -s 10.81.57.0/24 -A POSTROUTING -j MASQUERADE
apt-get install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```

#### Windows PC

> Static Route
```bash
arp -a
route add 10.81.0.0 mask 255.255.0.0 <router-eth2-ipaddress-got-from-arp-a>
```

## 3. Cluster Configuration

### Initialize the Cluster

You can use anything, but I prefer [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) as per Kubernetes official toolbox.

### LoadBalancing the Bare-Metal

As our infra are not in the cloud, we need [Metallb](https://metallb.universe.tf/installation/) to present LoadBalancer into the game. 

```bash
k apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

Once installation completed, lets configure it so any service with `LoadBalancer` type created will be assigned with our pre-configured Layer2 IP addresses pool depend on label/selector we set, in our case is whether server farm or dmz.

```bash
k apply -f 01/yaml/01_metallb_config.yaml
```

Lets test it by create some nginx web server deployment:

```bash
k apply -f 01/yaml/deploy/nginx/
```

Verify the **EXTERNAL-IP** using:

```bash
k get svc
```

You might notice that `nginx-1, nginx-3, and nginx-5` are in DMZ network, while `nginx-2 and nginx-4` are in Server Farm network. Verify them all using curl command or a browser directly from PC to the all EXTERNAL-IPs. Great! Lets cleanup: 

```bash
kdf -f 01/yaml/deploy/nginx/
```

> alias kdf='kubectl delete --force --grace-period 0'

### Multiple Ingress Controller

#### Create the default Ingress Controller

So, instead of using LoadBalancer for all of our services which can cause our IP addresss pool exhausted, we'll use Ingress to expose our services. We need [Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start) to be installed: 

```bash
k apply -f 02/yaml/02_ingress_controller-1_deploy.yaml
```

Actually the service type of the controller itself is also set to LoadBalancer, so we need to set the label match with our MetalLB setup. But that's it, we only need to load-balance the ingress controller! 

Make sure the controller pod is up and running:

```bash
k -n ingress-nginx get po
```

Lets test it by create a httpd web server deployment that will reside on Server Farm network. This is done by set the `ingressClassName` match to the controller on the ingress:

```bash
k apply -f 02/yaml/deploy/httpd/
```

Verify the **ADDRESS** using:
```bash
k get ingress -w
```

> :hourglass_flowing_sand: Plase note that it might require several time for the ADDRESS to be available! 

#### Create the second Ingress Controller

To make DMZ also available via ingress (on the different network for sure), lets create another ingress controller:

```bash
k apply -f 02/yaml/02_ingress_controller-2_deploy.yaml
```

Make sure the controller pod is up and running:

```bash
k -n ingress-nginx get po
```

Lets test it by create some nginx web server deployment:

```bash
k apply -f 02/yaml/deploy/nginx
```

Verify the **ADDRESS** again using:
```bash
k get ingress -w
```

> :hourglass_flowing_sand: Plase note that it might require several time for the ADDRESS to be available!

Once all ingresses ADDRESS are available lets then create DNS records for them. You might just create it on the master node at `/etc/hosts` then curl it, or if you want to use browser you also need to create it on the PC at `C:\Windows\System32\drivers\etc\hosts`:

```bash
10.81.56.200 demo.localdev.me
10.81.56.200 farm.localdev.me
10.81.57.200 dmz.localdev.me
```

Great! Lets cleanup:

```bash
kdf -R -f .
kdf -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

> :hourglass_flowing_sand: Plase note that `ingress-nginx` namespace might require several time to be fully complete terminated!
