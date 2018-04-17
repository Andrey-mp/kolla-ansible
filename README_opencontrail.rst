========
Overview
========

This is a fork from OpenStack project kolla-ansible.

It has additional role 'opencontrail' and ability to deploy OpenStack Cloud with Contrail.
This changes should be merged back to OpenStack repo.

Getting Started
===============

This readme shows how to deploy OpenStack with kolla-ansible with all-in-one inventory. So you need a machine with 32Gb memory at least.
In order to deploy with kolla-ansible you have to prepare registry with Contrails' images.

Prepare host
------------

If the registry is your own then you may need to add it to insecure registries list:
File /etc/docker/daemon.json should have addition:

::

  {
    "insecure-registries": ["$registry_ip:5000"]
  }

Or URL of registry instead of $registry_ip:5000

Then install/update required tools (with sudo):

::

  yum install -y epel-release
  yum install -y python-pip ntp
  systemctl enable ntpd.service && systemctl start ntpd.service
  pip install -U pip
  yum install -y python-devel libffi-devel gcc openssl-devel libselinux-python
  yum install -y ansible

Prepare kolla-ansible
---------------------

Download it to local directory, Install it to the system:

::

  git clone https://github.com/cloudscaling/kolla-ansible
  cd kolla-ansible
  pip install -r requirements.txt
  python setup.py install
  cd ..

Copy configuration files to /etc/kolla, Copy inventory files to local directory:

::

  cp -r /usr/share/kolla-ansible/etc_examples/kolla /etc/kolla/
  cp /usr/share/kolla-ansible/ansible/inventory/* .

Then you need to create /etc/kolla/globals.yml with next tempate. Please change openstack_release, network_interface, neutron_external_interface to values that you need.
Please use two different interfaces for all-in-one inventory - in other case after vrouter start all IP-s are moved to vhost0 and ansible playbooks willn't find anything on base interface and fail.
Please change {{contrail_version}} to your version of contrails' images like "5.0.0-126".
Please change {{contrail_docker_registry}} to URL of your docker registry.

::

  ---
  kolla_base_distro: "centos"
  kolla_install_type: "binary"
  openstack_release: "ocata"
  # This should be a VIP, an unused IP on your network that will float between
  # the hosts running keepalived for high-availability. If you want to run an
  # All-In-One without haproxy and keepalived, you can set enable_haproxy to no
  # in "OpenStack options" section, and set this value to the first IP on your
  # 'network_interface' as set in the Networking section below.
  kolla_internal_vip_address: "192.168.130.254"
  network_interface: "eth0"
  api_interface: "{{ network_interface }}"
  neutron_external_interface: "eth1"
  # Valid options are [ openvswitch, linuxbridge, vmware_nsxv, vmware_dvs, opendaylight, opencontrail ]
  neutron_plugin_agent: "opencontrail"
  ###################
  # OpenStack options
  ###################
  # Use these options to set the various log levels across all OpenStack projects
  # Valid options are [ True, False ]
  openstack_logging_debug: "True"
  ###################################
  # OpenContrail support
  ###################################
  enable_opencontrail: "yes"
  opencontrail_release: "{{contrail_version}}"
  opencontrail_docker_registry: "{{contrail_docker_registry}}"
  # while opencontrail is not a role in ansible and we didn't patch group_vars/all.yml we set fake IP here
  enable_openvswitch: "no"
  enable_neutron_bgp_dragent: "no"
  enable_neutron_dvr: "no"
  enable_neutron_lbaas: "no"
  enable_neutron_fwaas: "no"
  enable_neutron_qos: "no"
  enable_neutron_agent_ha: "no"
  enable_neutron_vpnaas: "no"
  enable_neutron_sfc: "no"
  # here additional env can be passed as a dict like this:
  opencontrail_env:
    JVM_EXTRA_OPTS: "-Xms1g -Xmx2g"

Run kolla-ansible
----------------

Run next commands:

::

  kolla-genpwd
  kolla-ansible -i all-in-one bootstrap-servers
  kolla-ansible pull -i all-in-one

Check images:

::

  docker images

Prepare libvirt:

::

  mkdir -p /etc/kolla/config/nova
  cat <<EOF > /etc/kolla/config/nova/nova-compute.conf
  [libvirt]
  virt_type = qemu
  cpu_mode = none
  EOF

Deploy all:

::

  kolla-ansible prechecks -i all-in-one
  kolla-ansible deploy -i all-in-one

Check running container:

::

  docker ps -a

Prepare adminrc for openstack:

::

  kolla-ansible post-deploy

And then you can run test steps:

::

  # test it
  pip install python-openstackclient
  source /etc/kolla/admin-openrc.sh
  /usr/share/kolla-ansible/init-runonce
