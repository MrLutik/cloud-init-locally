# Creating a template from cloud init image

OS: Arch

To simplify and speed up the process of VMs deployment on local machine I will use cloud init image. We can find them all around the place. For my tasks I need to use ubuntu 20.04 server.

Source of the image can be found: [cloud-images](https://cloud-images.ubuntu.com/)

Dependencies:
* libvirt
* qemu
* virt-manager
* bridge-utils
* cloud-utils

```bash
sudo pacman -Syy
sudo pacman -S qemu libvirt virt-manager virt-viewer bridge-utils cloud-utils curl
sudo usermod -aG libvirt $(whoami)
sudo systemctl enable --now libvirtd
```

#### Downloading image

The standard path for libvirt to store the images is `/var/lib/libvirt/images`.

Downloading latest focal version of ubuntu.
```bash
sudo curl -L https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img -o /var/lib/libvirt/images/focal-server-cloudimg-amd64.img
```

#### Adding ssh keys

While configuring cloud-init we will need to provide ssh public keys

```bash
ssh-keygen -t rsa -b 4096 -C "yourmail@mail.com"
```

Save this file in `~/.ssh/ci_ubuntu`, where ci is for cloud init and ubuntu for user name

#### Creating cloud init configuring

1. Create a directory `cloud-init-config` at `/var/lib/libvirt/images`

2. Enter the directory and create 2 files:
    * cloud-config
    ```bash
    ( echo "#cloud-config
users:
  - name: ubuntu
    gecos: Ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    shell: /bin/bash
    ssh-authorized-keys:
      - $(cat <PATH_TO_YOUR_PUB_KEY>.pub)

# Enable password authentication with the SSH daemon
ssh_pwauth: True

# Set the default password for the user (it's recommended to use SSH keys instead)
password: yourpassword
chpasswd: { expire: False }
write_files:
  - path: /etc/netplan/01-netcfg.yaml
    content: |
      network:
        version: 2
        ethernets:
          all:
            match:
              name: \"en*\"
            dhcp4: true" ) > cloud-config
    ```
    * user-data
    ```bash
    touch user-data
    ```

3. The last one - use cloud-localds to create iso image. Run in `cloud-init-config` directory
    ```bash
    cloud-localds ubuntu-cloud-init.iso cloud-config user-data
    ```
#### Creating VM which we will use as template
Using virt-manager:
1. Create new virtual machine -> Import existing disk image -> <Choose your cloud init image here and set the operating system name> -> Set your resources -> Set check box "Customize configuration before install" and select network (NAT)
2. Add hardware -> Device type: CDROM -> Select or create custom image: `/var/lib/libvirt/images/cloud-init-config/ubuntu-cloud-init.iso`
3. Boot options -> Select CDROM and move it on top
4. Begin installation

#### Init ssh to template
1. Get your template IP.
```bash
sudo virsh list --all
sudo virsh domifaddr <your_template_vm_name>
```
2. SSH to your template using cloud init credentials:
```bash
ssh -i ~/.ssh/ci_ubuntu ubuntu@YOUR_VM_IP_ADDRESS
```
3. Update, install qemu agent 
```bash
sudo agt-get udpate && apt-get install qemu-guest-agent -y
sudo systemctl enable qemu-guest-agent --now
```
4. Clean your image
```
sudo cloud-init clean
```
5. Delete your machid-id data 
``bash
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
```
6. Shutdown
```bash
sudo shutdown now
```

#### Tweaking permissions
While it is a template - you can still run it which will lead to mailfunction.
To prevent this - we will set privilages to 400

```bash
sudo chmod 400 /var/lib/libvirt/images/focal-server-cloudimg-amd64.img
```





