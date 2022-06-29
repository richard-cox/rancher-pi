# Versions / Restrictions
- https://rancher.com/docs/rancher/v2.6/en/cluster-provisioning/node-requirements/
- https://www.suse.com/assets/EN-2.6.5SupportMatrix-300422-0116-26.pdf
- https://rancher.com/docs/rancher/v2.6/en/installation/requirements/#k3s-specific-requirements / https://rancher.com/docs/rancher/v2.6/en/installation/requirements/#k3s-kubernetes
- https://rancher.com/docs/rancher/v2.6/en/cluster-provisioning/production/recommended-architecture/

# Core Setup
## OS
Ubuntu Server 20.04.4 LTS
- NOTE - Why not PI? Common, well updated (packages) OS. Only missing out on optimised GUI (not required)
- https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#4-boot-ubuntu-server
  - Pi imager to prep card
- Static ip 
  - Via router, not Os (https://pimylifeup.com/ubuntu-20-04-static-ip-address/)

# Pi's
- Case - https://thepihut.com/collections/raspberry-pi-cases/products/8-slot-cloudlet-cluster-case
- Power
  - NOTE - Originally wanted to use POE Hats to reduce cable mess, however due to issues below went with a power adapter per pi
    - the new PI POE Hats don't really leave much space in the case (would lose half the slots)
    - the screws that everything came with were too short to cover the hat and fix to the case's panels
- Network
  - Will use a mixture of cable and wifi. Ethernet for local resources, wi-fi on a guest network for external resources


## Layout / HW / SW
- Internal (wired)
  - Pi1 (8GB - 64GB)
    - Reserved 192.168.86.20
    - SW 
      - Rancher Manager  
  - Pi2 (4GB - 64GB)
    - Reserved 192.168.86.26
    - hostnamectl set-hostname pi2
    - SW
      - Kube Cluster (internal) - K3S Node - Server
  - Pi3 (4GB - 64GB)
    - K3S (worker)
    - Reserved 192.168.86.28
    - hostnamectl set-hostname pi3
    - SW
      - Kube Cluster (internal) - K3S Node - Agent
      - Pihole
      - Plex(?)
  - ?Pi4 (8GB - 128GB)
    - K3S (worker)
- External (wifi - guest network)
  - Pi5 (4GB - 64GB)
  - Pi6 (4GB - 64GB)
  
# Setup Pi 1 (Internal - Rancher Manager)
- Install K3S
  - https://rancher.com/docs/k3s/latest/en/installation/install-options/ / https://rancher.com/docs/rancher/v2.6/en/installation/resources/k8s-tutorials/ha-with-external-db/
   “ Failed to find memory cgroup, you may need to add "cgroup_memory=1 cgroup_enable=memory" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi)”
https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap

  - NOTE - args wise i had trouble with the config file, so args at command line
  - NOTE - `K3S_KUBECONFIG_MODE` never seemed to work for me, had to manually set file permisions (`sudo chmod 644 /etc/rancher/k3s/k3s.yaml`)
  - NOTE - This fails on Pi / Ubuntu Server with below, but the recommended fix isn't correct
    - `Failed to find memory cgroup, you may need to add "cgroup_memory=1 cgroup_enable=memory" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi)`
    - Instead `sudo nano /boot/firmware/cmdline.txt` `cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1` see https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap
  - NOTE - i had to use tls-sans as domain used by rancher (`<ip>.nip.io`) otherwise the rancher cert was crap)
- Install Helm
  - curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  - chmod 700 get_helm.sh
  - ./get_helm.sh
- Install K9S (useful when no rancher)
  - wget https://github.com/derailed/k9s/releases/download/v0.25.18/k9s_Linux_arm64.tar.gz
  - tar -xf k9s_Linux_arm64.tar.gz
  - sudo mv k9s /user/bin
- Install Rancher
  - https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/ 
  - NOTE - Certificates
    - My aim was to bring up rancher with a working certificate, for a long time this seemed to not work but didn't bork the install (the cert used was the default untrusted traefik)
  - NOTE - Certificates - letsencrypt
    - I tried for a long time to get letsencrypt working in my internal (192.168.) context to get a nice trusted cert (and to try and resolve a different issue). In summary
      - Domain name - cannot use `<internal ip>.nip.io` with letsencrypt (gave `no valid A records found for` & `no valid AAAA records found for` errors)
      - Domain name - cannot use `<my custom domain name>` pointing to `<internal ip>` with letsencrypt (gave `SERVFAIL looking up AAAA for <domain> - the domain's nameservers may be malfunctioning`)
    - In the end I just reverted back to using the default rancher cert way (created by cert-manager using their issuer)
  - NOTE - Certificates - rancher 
    - Even this method had issues, the cert-manager resources would show as failed until ~10-15 minutes. After that restarting all deployments seemed to work (cert used by rancher was issued by `dynamiclistener-ca` with correct `subject alternative name / SAN` of the rancher domain)
  - NOTE - `helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=<pi internal ip>.nip.io --set replicas=1`
- Install Longhorn
  - Note - change `Lonhorn Storage Class Settings`/ `Storage Class Retain Policy` to `Retain` to work better with `Rancher Backup`
- Install Rancher Backup
  - Use the longhorn storage class
- Create a routine Rancher Backup to run monthly
- Install Rancher Monitoring
  - NOTE - unable to to this, Pi only has 4CPU - `This chart requires 4.5 CPU cores, but the cluster only has 3.8 available.`

TODO: 
Problems in paralelt - cert & import cluster??

# Setup Pi 2 & 3 (Internal - Cluster 1 Master & Slave)
- Attempt #1 - Via Rancher Manager's 'Create Custom' feature
  - https://rancher.com/docs/rancher/v2.5/en/cluster-provisioning/rke-clusters/custom-nodes/#2-create-the-custom-cluster
  - NOTE - I got stuck at this point by `insecure` issues related to how rancher was brought up (see cert manager)
  - NOTE - Latest supported 1.27 K3S is `v1.23.7-k3s1`, however the associated agent installer does not support ARM (support was added to `v1.23.8-k3s1` released 2 hours before this was discovered, so should be covered in next rancher release)
    - `error executing instruction 0: no child with platform linux/arm64 in index rancher/system-agent-installer-k3s:v1.23.7-k3s1: failed to get image index.docker.io/rancher/system-agent-installer-k3s:v1.23.7-k3s1"`
- Attempt #2 - Via vanilla K3S cluster and Rancher Manager's `Import Cluster` feature
  - https://360techexplorer.com/install-k3s-on-raspberry-pi/
  - Pi2 (master) - `sudo curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="0644" INSTALL_K3S_VERSION="v1.23.7+k3s1" K3S_NODE_NAME="pi2" INSTALL_K3S_EXEC="--node-ip 192.168.86.26 --node-external-ip 192.168.86.26" sh -`
    - NOTE - After a lot of issues discovered both `--node-ip` and `--node-external-ip` had to be set
  - Pi3 (agent) - `sudo curl -sfL https://get.k3s.io | K3S_TOKEN="` + `<snip>::node:<snip>` + `" K3S_URL="https://192.168.86.26:6443" K3S_NODE_NAME="pi3" INSTALL_K3S_VERSION="v1.23.7+k3s1" sh -`
    - NOTE - All docs say that the `K3S_TOKEN` comes from the master node via `sudo cat /var/lib/rancher/k3s/server/node-token`. For me this was in the format of `<snip 1>::server:<snip 2>` and did NOT work
      - Errors
        - Agent - `Waiting to retrieve agent configuration; server is not ready: failed to retrieve configuration from server: https://127.0.0.1:6444/v1-k3s/config: 401 Unauthorized`
        - Server - `Failed to authenticate request from 192.168.86.28:55928: invalid username/password combination`
      - Fix
       - To get this working I had to use `<snip 1>::node:<node pswd from /var/lib/rancher/k3s/server/cred/passwd>` as the token instead
       - The node should then register and show up in master `kubectl get nodes`
    - NOTE - The newly registered node didn't have a role, so manually assigned one with `kubectl label nodes pi3 kubernetes.io/role=worker`. This can be done when installing the agent via `--node-label 'node_type=worker'`

# Apps Apps Apps
## Pihole - https://pi-hole.net/
### Install
- https://www.technicallywizardry.com/pihole-docker-kubernetes/
- https://uthark.github.io/post/2021-10-06-running-pihole-kubernetes/
- https://subtlepseudonym.medium.com/pi-hole-on-kubernetes-87fc8cdeeb2e (see gotcha at the end RE services)
  - ```
    As I mentioned earlier, we’re using a basic Service definition with an explicitly defined external IP address. The downside to this is that all requests reflected in the pihole dashboard will be forwarded from the kube-dns cluster internal IP address.
    ```
- https://dev.to/ivanmoreno/how-to-deploy-pihole-and-wireguard-on-kubernetes-using-a-recursive-dns-4l9 (complicated)
- https://cdcloudlogix.com/pihole-docker-and-kubernetes-simple-guide/

## To think about
unbound (eh Unbound  
Pihole remote app (ios only)
https://github.com/pucherot/Pi.Alert
vps (with pihole?)
DO monitor
Plex
MiniDLNA (vs plex?)
dnsmasq??

# Useful
## Commands
- Pi
  - `cat /sys/class/thermal/thermal_zone0/temp`
- K3S
  - used by cert manager - `kubectl -n kube-system get secret k3s-serving -o yaml`, should include `tls-san`
  - `sudo cat /var/lib/rancher/k3s/server/cred/passwd` / `sudo cat /var/lib/rancher/k3s/server/node-token`
  - `kubectl label nodes pi3 kubernetes.io/role=worker`
  - server
    - `systemctl status k3s` / `journalctl -u k3s -f`
    - `systemctl status rancher-system-agent` / `journalctl -u rancher-system-agent -f`
    - `k3suninstall.sh`
  - agent 
    - `systemctl status k3s-agent` / `journalctl -u k3s-agent -f`
    - `k3s-agent-uninstall.sh`


# Links
## Cert Rotation
- Should be automatically managed by cert-manager?
- certificates / rotation (upstream/local cluster)
- https://rancher.com/docs/rancher/v2.6/en/installation/resources/update-rancher-cert/
- https://www.suse.com/c/rancher_blog/manual-rotation-of-certificates-in-rancher-kubernetes-clusters/
- https://jmrobles.medium.com/fix-rancher-ssl-certificate-aaa9cb7cc7de


# Scrap


