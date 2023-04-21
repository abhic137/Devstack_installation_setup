# Devstack_installation_setup



Using yoga release with following reference: Tacker is the  https://docs.openstack.org/tacker/yoga/install/devstack.html 

as regular user
sudo useradd -s /bin/bash -d /opt/stack -m stack
sudo chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo -u stack -i

as user stack
sudo mkdir stack
sudo chmod 777 stack
cd stack
git clone https://opendev.org/openstack/devstack --branch stable/yoga
cd devstack/


vi local.conf; # details in cat local.conf below

cat local.conf

[[local|localrc]]
HOST_IP=172.27.5.38

ADMIN_PASSWORD=devstack
MYSQL_PASSWORD=devstack
RABBIT_PASSWORD=devstack
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=devstack

PIP_USE_MIRRORS=False
USE_GET_PIP=1

LOGFILE=$DEST/logs/stack.sh.log
VERBOSE=True
ENABLE_DEBUG_LOG_LEVEL=True
ENABLE_VERBOSE_LOG_LEVEL=True

# Neutron ML2 with OpenVSwitch
Q_PLUGIN=ml2
Q_AGENT=ovn

# Disable security groups
LIBVIRT_FIREWALL_DRIVER=nova.virt.firewall.NoopFirewallDriver

# Enable neutron, heat, networking-sfc, barbican and mistral
enable_plugin neutron https://opendev.org/openstack/neutron stable/yoga
enable_plugin heat https://opendev.org/openstack/heat stable/yoga
enable_plugin networking-sfc https://opendev.org/openstack/networking-sfc stable/yoga
enable_plugin barbican https://opendev.org/openstack/barbican stable/yoga
enable_plugin mistral https://opendev.org/openstack/mistral stable/yoga

# Ceilometer
#CEILOMETER_PIPELINE_INTERVAL=300
enable_plugin ceilometer https://opendev.org/openstack/ceilometer stable/yoga
enable_plugin aodh https://opendev.org/openstack/aodh stable/yoga

# Blazar
enable_plugin blazar https://github.com/openstack/blazar.git stable/yoga

# Fenix
enable_plugin fenix https://opendev.org/x/fenix.git master

# Tacker
enable_plugin tacker https://opendev.org/openstack/tacker stable/yoga

enable_service n-novnc
enable_service n-cauth

disable_service tempest

# Enable kuryr-kubernetes, docker, octavia
KUBERNETES_VIM=True
enable_plugin kuryr-kubernetes https://opendev.org/openstack/kuryr-kubernetes stable/yoga
enable_plugin octavia https://opendev.org/openstack/octavia stable/yoga
enable_plugin devstack-plugin-container https://opendev.org/openstack/devstack-plugin-container stable/yoga
#KURYR_K8S_CLUSTER_IP_RANGE="10.0.0.0/24"

enable_service kubernetes-master
enable_service kuryr-kubernetes
enable_service kuryr-daemon

[[post-config|/etc/neutron/dhcp_agent.ini]]
[DEFAULT]
enable_isolated_metadata = True

[[post-config|$OCTAVIA_CONF]]
[controller_worker]
amp_active_retries=9999




Following is to start the stack – repeat only after clean+ unstack+ reboot
 ./stack.sh
 ./clean.sh
 ./unstack.sh
sudo reboot now


First download an image it should be .img file



in order to customise the image: (source: https://stackoverflow.com/questions/61591885/how-do-i-set-a-custom-password-with-cloud-init-on-ubuntu-20-04)
=======

rm -f vm_0001-focal-server-cloudimg-amd64.qcow2
qemu-img create -f qcow2 -F qcow2 -b focal-server-cloudimg-amd64.img  vm_0001-focal-server-cloudimg-amd64.qcow2 20G
qemu-img info vm_0001-focal-server-cloudimg-amd64.qcow2
VM_NAME="ubuntu-20-cloud-image"
USERNAME="programster"
PASSWORD="thisok"
echo "#cloud-config
system_info:
  default_user:
    name: $USERNAME
    home: /home/$USERNAME

password: $PASSWORD
chpasswd: { expire: False }
hostname: $VM_NAME

# configure sshd to allow users logging in using password 
# rather than just keys
ssh_pwauth: True
" | sudo tee user-data
cloud-localds ./cidata.iso user-data
qemu-system-x86_64 -m 2048 -smp 4 -hda ./vm_0001-focal-server-cloudimg-amd64.qcow2 \
      -cdrom ./cidata.iso -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 -nographic
==========

your image will be in .qcow2 format with new name
(source:https://docs.openstack.org/image-guide/convert-images.html )
=> we have to convert it to the .img file  

qemu-img convert -f qcow2 -O raw imgname.qcow2 imgname.img

==========

select the newly converted file in the openstack image as QEMU format
now create an instace

make sure to create a router which connects to public network aswell as the network in which the instance is in.



Now open the instance and go the netplan and adjust the netplan setting in the instance.

cd etc/netplan
sudo nano 50-cloud-init.yaml


sudo netplan apply



now go to /etc/resolv.conf
and add the line nameserver 8.8.8.8 at the end and save the file

inorder to access the openstack dahboard from any pc from the same network we have to diable the ufw firewall or add exceptions for port 80 and 433 


commands:
sudo ufw status
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status



now enjoy the internet.


Inorder to ping the instance from the bear metal add the route entry:

route
sudo ip route add 10.10.1.0/24 via 172.24.4.44
route

10.10.1.0/24 is the network of the instances and 172.24.4.44 is the ip of the router
you can verify with the ping command.


Inorder to ping the instance from another pc in the same network add the route entry by keeping openstack host as a gateway:
sudo ip route add 10.10.1.0/24 via 192.168.138.131








