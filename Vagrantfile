# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  $provisioning_nova_docker = <<EOF
#
# Disable Network Manager
#
systemctl mask NetworkManager
systemctl stop NetworkManager
cat > /etc/sysconfig/network-scripts/ifcfg-enp0s8 << MIDO_EOF
DEVICE=enp0s8
BOOTPROTO=dhcp
NM_CONTROLLED=no
MIDO_EOF
cat > /etc/sysconfig/network-scripts/ifcfg-enp0s9 << MIDO_EOF
DEVICE=enp0s9
BOOTPROTO=dhcp
NM_CONTROLLED=no
MIDO_EOF
systemctl enable network.service
systemctl start network.service


#
# Repos
#

# packstack
yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-juno/rdo-release-juno-1.noarch.rpm
yum install -y epel-release

# Midonet
cat >> /etc/yum.repos.d/midonet.repo << EOF_MIDO
[midonet]
name=MidoNet
baseurl=http://repo.midonet.org/midonet/v2015.01/RHEL/7/stable/
enabled=1
gpgcheck=1
gpgkey=http://repo.midonet.org/RPM-GPG-KEY-midokura

[midonet-openstack-integration]
name=MidoNet OpenStack Integration
baseurl=http://repo.midonet.org/openstack-juno/RHEL/7/testing/
enabled=1
gpgcheck=1
gpgkey=http://repo.midonet.org/RPM-GPG-KEY-midokura

[midonet-misc]
name=MidoNet 3rd Party Tools and Libraries
baseurl=http://repo.midonet.org/misc/RHEL/7/misc/
enabled=1
gpgcheck=1
gpgkey=http://repo.midonet.org/RPM-GPG-KEY-midokura
EOF_MIDO

# Cassandra
cat >> /etc/yum.repos.d/cassandra.repo << EOF_MIDO
[datastax]
name= DataStax Repo for Apache Cassandra
baseurl=http://rpm.datastax.com/community
enabled=1
gpgcheck=0
EOF_MIDO


#
# Updating and installing dependencies
#

yum update -y

# Tools
yum install -y augeas crudini screen


#
# Packstack
#

IP=192.168.124.185
yum install -y openstack-packstack
packstack --install-hosts=$IP \
 --nagios-install=n \
 --os-ceilometer-install=n \
 --os-cinder-install=y \
 --os-glance-install=y \
 --os-heat-install=n \
 --os-horizon-install=y \
 --os-nova-install=y \
 --provision-demo=n

rc=$?; if [[ $rc != 0 ]]; then exit $rc; fi

#
# Disable unneeded remaining services
#
yum remove -y openstack-neutron-openvswitch
systemctl stop openvswitch
systemctl mask openvswitch
systemctl stop neutron-l3-agent
systemctl mask neutron-l3-agent


ADMIN_TOKEN=$(crudini --get /etc/keystone/keystone.conf DEFAULT admin_token)
ADMIN_PASSWORD=$(grep "OS_PASSWORD" /root/keystonerc_admin | sed -e 's/"//g' | cut -f2 -d'=')
NEUTRON_DBPASS=$(crudini --get /root/packstack-answers-* general CONFIG_NEUTRON_DB_PW)

cp /root/keystonerc_admin /home/vagrant/
chown vagrant:vagrant /root/keystonerc_admin /home/vagrant/keystonerc_admin


#
# Zookeeper
#

yum install -y java-1.7.0-openjdk-headless zookeeper

# Zookeeper expects the JRE to be found in the /usr/java/default/bin/ directory
# so if it is in a different location, you must create a symbolic link pointing
# to that location. To do so run the 2 following commands:

mkdir -p /usr/java/default/bin/
ln -s /usr/lib/jvm/jre-1.7.0-openjdk/bin/java /usr/java/default/bin/java

# Next we need to create the zookeeper data directory and assign permissions:

mkdir /var/lib/zookeeper/data
chmod 777 /var/lib/zookeeper/data

# Now we can edit the Zookeeper configuration file. We need to add the servers
# (in a prod installation you would have more than one zookeeper server in a
# cluster. For this example we are only using one. ). Edit the Zookeeper config
# file at /etc/zookeeper/zoo.cfg and add the following to the bottom of the
# file:

echo "server.1=$IP:2888:3888" >> /etc/zookeeper/zoo.cfg

# We need to set the Zookeeper ID on this server:

echo 1 > /var/lib/zookeeper/data/myid

systemctl enable zookeeper.service
systemctl start zookeeper.service

#
# Cassandra
#

yum install -y dsc20

sed -i -e "s/cluster_name:.*/cluster_name: 'midonet'/" /etc/cassandra/conf/cassandra.yaml

sed -i -e "s/listen_address:.*/listen_address: $IP/" /etc/cassandra/conf/cassandra.yaml

sed -i -e "s/seeds:.*/seeds: \"$IP\"/" /etc/cassandra/conf/cassandra.yaml
sed -i -e "s/rpc_address:.*/rpc_address: $IP/" /etc/cassandra/conf/cassandra.yaml

rm -rf /var/lib/cassandra/data/system

systemctl enable cassandra.service
systemctl start cassandra.service


#
# Midolman
#

yum install -y midolman
systemctl enable midolman.service
systemctl start midolman.service


#
# Midonet-api
#

yum install -y midonet-api

# Small inline python program that sets the proper xml values to midonet-api's
# web.xml
cat << MIDO_EOF | python -
from xml.dom import minidom

DOCUMENT_PATH = '/usr/share/midonet-api/WEB-INF/web.xml'


def set_value(param_name, value):
    value_node = param_node.parentNode.getElementsByTagName('param-value')[0]
    value_node.childNodes[0].data = value


doc = minidom.parse(DOCUMENT_PATH)
params = doc.getElementsByTagName('param-name')

for param_node in params:
    if param_node.childNodes[0].data == 'rest_api-base_uri':
        set_value(param_node, 'http://$IP:8081/midonet-api')
    elif param_node.childNodes[0].data == 'keystone-service_host':
        set_value(param_node, '$IP')
    elif param_node.childNodes[0].data == 'keystone-admin_token':
        set_value(param_node, '$ADMIN_TOKEN')
    elif param_node.childNodes[0].data == 'zookeeper-zookeeper_hosts':
        set_value(param_node, '$IP:2181')

with open(DOCUMENT_PATH, 'w') as f:
    f.write(doc.toprettyxml())
MIDO_EOF

yum install -y tomcat

cat << MIDO_EOF | augtool -L
set /augeas/load/Shellvars/incl[last()+1] /etc/tomcat/tomcat.conf
load
set /files/etc/tomcat/tomcat.conf/CONNECTOR_PORT 8081
save
MIDO_EOF

cat << MIDO_EOF | sudo augtool -L
set /augeas/load/Xml/incl[last()+1] /etc/tomcat/server.xml
load
set /files/etc/tomcat/server.xml/Server/Service/Connector[1]/#attribute/port 8081
save
MIDO_EOF

cat << MIDO_EOF > /etc/tomcat/Catalina/localhost/midonet-api.xml
<Context
    path="/midonet-api"
    docBase="/usr/share/midonet-api"
    antiResourceLocking="false"
    privileged="true"
/>
MIDO_EOF

systemctl enable tomcat.service
systemctl start tomcat.service


#
# Midonet-cli
#
#

yum install -y python-midonetclient
cat > ~/.midonetrc << MIDO_EOF
[cli]
api_url = http://$IP:8081/midonet-api
username = admin
password = $ADMIN_PASSWORD
project_id = admin
MIDO_EOF

cp /root/.midonetrc /home/vagrant/

# Create a tunnel zone
TUNNEL_ZONE=$(midonet-cli -e tunnel-zone create name gre type gre)
HOST_UUID=$(midonet-cli -e list host | awk '{print $2;}')
midonet-cli -e tunnel-zone $TUNNEL_ZONE add member host $HOST_UUID address $IP


#
# Keystone Integration
#

source ~/keystonerc_admin
keystone service-create --name midonet --type midonet --description "Midonet API Service"

keystone user-create --name midonet --pass midonet --tenant admin
keystone user-role-add --user midonet --role admin --tenant admin


#
# Neutron integration
#
yum install -y openstack-neutron python-neutron-plugin-midonet

crudini --set /etc/neutron/neutron.conf DEFAULT core_plugin midonet.neutron.plugin.MidonetPluginV2

mkdir /etc/neutron/plugins/midonet
cat > /etc/neutron/plugins/midonet/midonet.ini << MIDO_EOF
[DATABASE]
sql_connection = mysql://neutron:$NEUTRON_DBPASS@$IP/neutron
sql_max_retries = 100
[MIDONET]
# MidoNet API URL
midonet_uri = http://$IP:8081/midonet-api
# MidoNet administrative user in Keystone
username = midonet
password = midonet
# MidoNet administrative user's tenant
project_id = admin
auth_url = http://$IP:5000/v2.0
MIDO_EOF

rm -f /etc/neutron/plugin.ini
ln -s /etc/neutron/plugins/midonet/midonet.ini /etc/neutron/plugin.ini

# Comment out the service_plugins definitions
crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins neutron.services.firewall.fwaas_plugin.FirewallPlugin
sed -i -e 's/^router_scheduler_driver/#router_scheduler_driver/' /etc/neutron/neutron.conf

# dhcp agent config
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.MidonetInterfaceDriver
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver midonet.neutron.agent.midonet_driver.DhcpNoOpDriver
crudini --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata True
crudini --set /etc/neutron/dhcp_agent.ini MIDONET midonet_uri http://$IP:8081/midonet-api
crudini --set /etc/neutron/dhcp_agent.ini MIDONET username midonet
crudini --set /etc/neutron/dhcp_agent.ini MIDONET password midonet


systemctl restart neutron-server
systemctl restart neutron-dhcp-agent
systemctl restart neutron-metadata-agent

#
# Create internal network for usage on the instances
#
neutron net-create foo
neutron subnet-create foo 172.16.1.0/24 --name foo


function install_nova_docker_with_midonet() {
    yum install -y git python-pip
    git clone https://review.openstack.org/stackforge/nova-docker
    pushd nova-docker
    git checkout origin/stable/juno
    git cherry-pick 06dabc0aecf95003e2558da3899ceed43367c237
    pip install pbr
    python setup.py install --record /root/nova_docker_installed_files.txt
    mkdir -p /etc/nova/rootwrap.d
    cp etc/nova/rootwrap.d/docker.filters /etc/nova/rootwrap.d/
    popd
}

function configure_glance_for_docker() {
    source keystonerc_admin
    local glance_formats=$(crudini --get /etc/glance/glance-api.conf DEFAULT container_formats)
    local glance_with_docker=$(case "$glance_formats" in *docker* ) echo "$glance_formats";; * ) echo "$glance_formats,docker";; esac)
    crudini --set /etc/glance/glance-api.conf DEFAULT container_formats "$glance_with_docker"
    systemctl restart openstack-glance-api
}

function configure_nova_for_docker() {
    crudini --set /etc/nova/nova.conf DEFAULT compute_driver novadocker.virt.docker.DockerDriver
    systemctl restart openstack-nova-compute
}

#
#Setting docker up
#
yum install -y docker
# add nova compute to the docker group so it can use its socket
gpasswd -a nova docker
systemctl enable docker
systemctl start docker

configure_glance_for_docker

# Add docker's cirros to glance
docker pull cirros
docker save cirros | glance image-create --is-public=True --container-format=docker --disk-format=raw --name cirros

install_nova_docker_with_midonet
configure_nova_for_docker

# Define more appropriately sized instance for cirros containers
nova flavor-create "m1.nano" auto 64 0 1

EOF
  config.vm.box = "centos7"

  config.vm.define :nova_docker do |nova_docker|
      nova_docker.vm.hostname = "nova-docker.local"
      nova_docker.vm.network :private_network, ip: "192.168.124.185"
      nova_docker.vm.network :private_network, ip: "192.168.124.186"
      nova_docker.vm.provision "shell",
    inline: $provisioning_nova_docker
      nova_docker.vm.provider :virtualbox do |vb|
          vb.memory = 4096
          vb.cpus = 2
      end
  end
end
