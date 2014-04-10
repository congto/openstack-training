Title: Getting started with Red Hat Enterprise Linux OpenStack Platform<br>
Author: Rhys Oxenham <roxenham@redhat.com><br>
Date: April 2014

#**Lab Contents**#

1. **Getting Started**
2. **Creating Users, Roles and Tenants via Keystone (Identity Service)**
3. **Creating/Adding Images into Glance (Image Service)**
4. **Creation of Tenant Networks in Neutron (Networking Service)**
5. **Creation of Instances in Nova (Compute Service)**
6. **Creation and Attachment of Block Devices with Cinder (Volume Service)**
7. **Deployment of Application Stacks using Heat (Orchestration)**


<!--BREAK-->

#**Lab Overview**

First of all, it's my pleasure to welcome you to the Red Hat Summit 2014. The past twelve months have been an exciting time for both Red Hat and the OpenStack community; we've seen unprecedented interest in this new revolutionary technology and we're proud to be at the heart of it all.

This lab aims to get you, the attendees, a bit closer to OpenStack technology. It's comprised of a number of individual tasks that will run you through some of the common activities, such as virtual machine creation, network management, and image deployment; giving you a basic hands-on overview of OpenStack and how the components fit together. It will use a combination of command-line tools and the OpenStack Dashboard (Horizon).

You won't need to install or configure anything within this lab, you'll be using a pre-configured OpenStack environment in an "all-in-one" configuration; i.e. all OpenStack components configured inside of one machine. This machine will be a virtual machine itself, and is running on the workstation you're sat at. Please start with the first lab, this will get you used to the environment infront of you.

If you have any problems at all, please put your hand-up and an attendee will be with you shortly to assist.

<!--BREAK-->

#**Lab 1: Getting Started**

The OpenStack environment has been preinstalled and preconfigured for the purposes of this lab. We need to ensure that the machine is running and that we're able to log in to it. On the desktop of your workstation you'll find a number of shortcuts. You'll find an electronic copy of this guide (should you need to copy and paste any commands) as well as a link to a terminal emulator, and the OpenStack Dashboard that we'll use later.

Firstly, launch the 'Terminal' via the shortcut on the desktop. You won't have root access to your workstation, but you will be able to ensure the lab virtual machine is started:

	$ sudo virsh start lab2-openstack
	Domain lab2-openstack started
	
We'll need to wait a few minutes for the machine to become available. After a minute of two, let's connect into the machine to make sure it's ready for use:

	$ ssh root@openstack-demo
	[Enter the password: "redhat"]

If this is unsuccessful, attempt the following-

	$ ssh root@192.168.122.100
	[Enter the password: "redhat"]
	
You will have full root access and control over this virtual machine, and we'll run our tasks within this virtual machine. If you're still unable to connect into your virtual machine after a few minutes, please ask for assistance.	

#**Lab 2: Creating Users, Roles and Tenants via Keystone (Identity Service)**

##**Introduction**

Keystone is the identity management component of OpenStack, a common authentication and authorisation store. It's primarily responsible for managing users, their roles and the projects/tenants that they belong to, in other words - who's who, what can they do, and to which groups they belong to; essentially a centralised directory of users mapped to the services they are granted to use. In addition to providing a centralised repository of users, Keystone provides a catalogue of services deployed in the environment, allowing service discovery with their endpoints (or API entry-points) for access. Keystone is responsible for governance in an OpenStack cloud, it provides a policy framework for allowing fine grained access control over various components and responsibilities.

As a crucial and critical part of OpenStack infrastructure, Keystone is used to verify *all* requests from end-users to ensure that what clients are trying to do is both authenticated and authorised.


##**Creating Users**

As part of the lab we're going to allow you to create your own user and tenant (project) using Keystone.  

Once you've logged onto your OpenStack lab machine, you'll notice that a Keystone 'source' file has been placed into your home directory. It will allow you to have admin access to the environment so that you can create your own user and tenant - just like Linux, the root user is not used for day-to-day tasks. Note that this is only to be used temporarily for the purpose of this lab.

	# source keystonerc_admin
	
By running the above command, you will have configured environment variables that the OpenStack command line tools will use, e.g. the location of Keystone, the admin username and password. The next few steps create a user account, a 'tenant' (a group of users, or a project) and a 'role' which is used to determine permissions across the stack. You can choose your own username, password, and tenant name here, just remember what they are as we'll use them later:

	# keystone user-create --name <your username> --pass <your password>
	+----------+-----------------------------------+
	| Property |              Value                |
	+----------+-----------------------------------+
	| email    |                                   |
	| enabled  |              True                 |
	| id       | 679cae35033f4bbc9a18aff0c15b7a99  |
	| name     |              rdo                  |
	| tenantId |                                   |
	+----------+-----------------------------------+
	

Next, create a tenant (project) for your user to reside in:


	# keystone tenant-create --name <your tenant name>
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |                                  |
	|   enabled   |               True               |
	|      id     | 4ab1c31fcd2741afa551b5f76146abf6 |
	|     name    |              rdo                 |
	+-------------+----------------------------------+

Finally we can give the user a role and assign that user to the tenant. Note that we're using usernames, roles and tenants by their name here, it's possible to use their id's instead. It's best to give yourself the role "Member" to start off with - i.e. a basic, unpriveliged user.

	# keystone user-role-add --user <your user> --role Member --tenant <your tenant>

To save time and to not have to worry about specifying usernames/passwords or tokens on the command-line, it's prudent to create an rc file which will load in environment variables; saving a lot of time. This can be copied & pasted, although make sure you provide the correct username, tenant, and password used previously. Place the following into a file called "keystonerc_user".

	export OS_USERNAME=<your username>
	export OS_TENANT_NAME=<your tenant>
	export OS_PASSWORD=<your password>
	export OS_AUTH_URL=http://localhost:5000/v2.0/
	export PS1='[\u@\h \W(keystone_<your username>)]\$ '

You can test the file by logging out of your ssh session to the OpenStack virtual machine, logging back in and trying the following-

	# logout
	$ ssh root@openstack-demo

	# keystone token-get
	Expecting an auth URL via either --os-auth-url or env[OS_AUTH_URL]

	# source ~/keystonerc_user
	# keystone token-get
	....

Note that the first attempt at running the "keystone token-get" command failed; as it was either expecting you to specify the authorisation credentials as parameters or via environment variables. The second time it should have succeeded, if not, please check the contents of your keystonerc_user file, and ensure they match the user you created earlier in the lab.

#**Lab 3: Adding Images into Glance (Image Service)**

##**Introduction**

Glance is OpenStack's image service, it provides a mechanism for discovering, registering and retrieving virtual machine images. These images are typically standardised and generalised so that they will require post-boot configuration applied. Glance supports a wide variety of disk image formats, including raw, qcow2, vmdk, ami and iso, all of which can be stored in multiple types of back-ends, including OpenStack Swift (OpenStack's Object Storage Service, which we'll cover in a later lab) although by default it will use the local filesystem. When a user requests an instance within the cloud, it's Glance's responsibility to provide that image and allow it to be retrieved prior to instantiation.

Glance stores metadata alongside each image which helps identify it and describe the image, it can accomodate for multiple container types, e.g. an image could be completely self contained such as a qcow2 image however an image could also be just a kernel or an initrd file which need to be tied together to successfully boot an instance of that machine. Glance is made up of two components, the glance-registry which is the actual image registry service and glance-api which provides the end-point to the rest of the OpenStack services.

##**Adding an image to Glance**

As previously mentioned, images uploaded to Glance should be 're-initialised' or 'syspreped' so that any system-specific configuration is wiped away, this ensures that there are no conflicts between instances that are started. It's common practice to find pre-defined virtual machine images online that contain a base operating system and perhaps a set of packages for a particular purpose. The next few steps will allow you to take such an image and upload it into the Glance registry. 

As part of the lab, we'll first download an image already residing within Glance, and upload it as our own. So, let's get a list of the available images and download the file:

	# source ~/keystonerc_user
	# glance image-list
	+--------------------------------------+----------------+-------------+------------------+-----------+--------+
	| ID                                   | Name           | Disk Format | Container Format | Size      | Status |
	+--------------------------------------+----------------+-------------+------------------+-----------+--------+
	| 6b6e8ef2-23a5-4c79-9f0a-cf3e2b9f6119 | Cirros 0.3.1   | qcow2       | bare             | 13147648  | active |
	| df65596e-74ee-4a3f-83fa-538141d86ac0 | rhel6-cfntools | qcow2       | bare             | 988741632 | active |
	+--------------------------------------+----------------+-------------+------------------+-----------+--------+

Here you can see that there's already two images pre-installed into Glance. One is Cirros, a lightweight testing image, and the other is Red Hat Enterprise Linux 6.x. Let's download the Cirros image, as it's only 13MB:

	# glance image-download --file cirros-0.3.1-x86_64-disk.img --progress 6b6e8ef2-23a5-4c79-9f0a-cf3e2b9f6119
	[=============================>] 100%
	
This command specified a few options- it specified the filename to download the file as, to request an output of a progress bar, and to specify the image we wanted to download by its id (as shown in the image-list output).

Let's verify that the disk image is as expected and has the correct properties:

	# qemu-img info cirros-0.3.1-x86_64-disk.img
	image: cirros-0.3.1-x86_64-disk.img
	file format: qcow2
	virtual size: 39M (41126400 bytes)
	disk size: 13M
	cluster_size: 65536

Next we can create a new image within Glance and import its contents, it may take a few minutes to copy the data...

	# source ~/keystonerc_user

	# glance image-create --name "My Cirros Image" --is-public false \
		--disk-format qcow2 --container-format bare \
		--file ~/cirros-0.3.1-x86_64-disk.img
	+------------------+--------------------------------------+
	| Property         | Value                                |
	+------------------+--------------------------------------+
	| checksum         | d972013792949d0d3ba628fbe8685bce     |
	| container_format | bare                                 |
	| created_at       | 2014-03-01T18:56:09                  |
	| deleted          | False                                |
	| deleted_at       | None                                 |
	| disk_format      | qcow2                                |
	| id               | 2a95e20e-1cf8-47d0-9b64-6f7867b3103a |
	| is_public        | False                                |
	| min_disk         | 0                                    |
	| min_ram          | 0                                    |
	| name             | My Cirros Image                      |
	| owner            | 3883b9dbf0054d119becd5298c1fbe54     |
	| protected        | False                                |
	| size             | 13147648                             |
	| status           | active                               |
	| updated_at       | 2014-03-01T18:56:09                  |
	+------------------+--------------------------------------+
	

The container format is 'bare' because it doesn't require any additional images such as a kernel or initrd, it's completely self-contained. The 'is-public' option allows any user within the tenant (project) to use the image rather than locking it down for the specific user uploading the image - we'll set this to false to ensure that this image is only for you.

Finally, vertify that the image is available in the repository:

	# glance image-list
	+--------------------------------------+-----------------+-------------+------------------+-----------+--------+
	| ID                                   | Name            | Disk Format | Container Format | Size      | Status |
	+--------------------------------------+-----------------+-------------+------------------+-----------+--------+
	| 6b6e8ef2-23a5-4c79-9f0a-cf3e2b9f6119 | Cirros 0.3.1    | qcow2       | bare             | 13147648  | active |
	| 1b31eb27-5d0f-4cab-b870-cee0fadd818a | My Cirros Image | qcow2       | bare             | 13147648  | active |
	| df65596e-74ee-4a3f-83fa-538141d86ac0 | rhel6-cfntools  | qcow2       | bare             | 988741632 | active 	|
	+--------------------------------------+-----------------+-------------+------------------+-----------+--------+


#**Lab 4: Creation of Tenant Networks in Neutron (Networking Service)**

##**Introduction**

Neutron is OpenStack's Networking service and provides an abstract virtual networking service, enabling administrators and end-users to manage both virtual and physical networks for their instances on-top of OpenStack. Neutron simply provides an API for self-service and management but relies on underlying technologies for the actual implementation via Neutron-plugins. The class environment makes use of Open vSwitch, but there are many other plugins available upstream such as VMware NSX (Nicira NVP), Cisco UCS, Brocade etc. Neutron allows users to create rich networking topologies within their own tenants - having full control over their networks.

We're going to be starting our first instances in the next lab, but for an instance to start, it must be assigned a network to attach to. These are typically private networks, i.e. have no public connectivity and is primarily used for virtual machine interconnects and private networking - called a "tenant network". Within OpenStack we bridge the private network out to the real world via a public (or 'external') network, it is simply the network interface in which public traffic will connect into, and is typically where you'd assign "floating IP's", i.e. IP addresses that are dynamically assigned to instances so that external traffic can be routed through correctly. Instances don't actually have direct access to the public network interface, they only see the private network and the user is responsible for optionally connecting a virtual router to interlink the two networks for both external access and inbound access from outside the private network.

Confirm that your tenant is able to see the administratively created external network:

	# source keystonerc_user

	# neutron net-show external
	+-----------------+--------------------------------------+
	| Field           | Value                                |
	+-----------------+--------------------------------------+
	| admin_state_up  | True                                 |
	| id              | 9e3cc60a-3354-4528-9e4b-3f3a0825156f |
	| name            | external                             |
	| router:external | True                                 |
	| shared          | False                                |
	| status          | ACTIVE                               |
	| subnets         | 618937af-9858-468d-a3f3-b3c7fc81b51f |
	| tenant_id       | be369cab7e2845ea9905c0461cd6f8e3     |
	+-----------------+--------------------------------------+	

The key parameter is 'router:external=True'. It roughly translates to "this network is a real datacenter network and it can be used to route private networks to the outside".

This is great, but instances won't have direct access to this network, we need to create private networks for our tenants. This is the responsibility of a user within a tenant, the external network is just exposed to all of the tenants that are created; whilst they cannot modify it, they can attach a virtual router to it for connectivity.

##**Creating tenant networks**

Let's create a tenant network for our instances to use...

	# source keystonerc_user
	# neutron net-create int
	Created a new network:
	+-----------------+--------------------------------------+
	| Field           | Value                                |
	+-----------------+--------------------------------------+
	| admin_state_up  | True                                 |
	| id              | 0191c293-365d-4798-aa2d-f5afe47100c2 |
	| name            | int                                  |
	| router:external | False                                |
	| shared          | False                                |
	| status          | ACTIVE                               |
	| subnets         |                                      |
	| tenant_id       | 97b43bd18e7c4f7ebc45b39b090e9265     |
	+-----------------+--------------------------------------+
	
	# neutron subnet-create int 30.0.0.0/24 --name my_subnet
	Created a new subnet:
	+------------------+--------------------------------------------+
	| Field            | Value                                      |
	+------------------+--------------------------------------------+
	| allocation_pools | {"start": "30.0.0.2", "end": "30.0.0.254"} |
	| cidr             | 30.0.0.0/24                                |
	| dns_nameservers  |                                            |
	| enable_dhcp      | True                                       |
	| gateway_ip       | 30.0.0.1                                   |
	| host_routes      |                                            |
	| id               | df839eb2-8efc-413d-a19a-3e008da4858f       |
	| ip_version       | 4                                          |
	| name             |  my_subnet                                 |
	| network_id       | 0191c293-365d-4798-aa2d-f5afe47100c2       |
	| tenant_id        | 97b43bd18e7c4f7ebc45b39b090e9265           |
	+------------------+--------------------------------------------+
	
This means that any instances that we start on this network will receive an IP address (via DHCP) on the 30.0.0.0/24 network. We don't yet have external connectivity for our instances - at the moment it's purely a private overlay network. Let's change that...

	# neutron router-create my_router
	Created a new router:
	+-----------------------+--------------------------------------+
	| Field                 | Value                                |
	+-----------------------+--------------------------------------+
	| admin_state_up        | True                                 |
	| external_gateway_info |                                      |
	| id                    | c9f04fa3-e1cc-4357-b427-970917789c70 |
	| name                  | my_router                            |
	| status                | ACTIVE                               |
	| tenant_id             | c6d79dbd835842f5b36d09a737c8e530     |
	+-----------------------+--------------------------------------+	
Now let's connect the two networks together, firstly we need to set the gateway, i.e. the external network.

	# neutron router-gateway-set my_router external
	Set gateway for router my_router
	
Then add an interface which is the subnet we're linking to (our internal network 30.0.0.0/24). For this we need the subnet ID for 30.0.0.0/24 in our tenant:

	# neutron subnet-list
	
Grab the ID for the subnet and use it in the following command:
	
	# neutron router-interface-add my_router df839eb2-8efc-413d-a19a-3e008da4858f
	Added interface to router my_router

In conclusion, we've created a private 'tenant network' for our instances to use and have created a virtual router for our private network to route traffic to the outside; onto the admin-provided external network ("external").

#**Lab 5: Creation of Instances in Nova (Compute Service)**

##**Introduction**

Nova is OpenStack's compute service, it's responsible for providing and scheduling compute resource for OpenStack instances. It's undoubtably one of the most important parts of the OpenStack project. It was designed to massively scale horizontally to cater for the largest clouds. Nova supports a wide variety of hardware and software platforms, including the most popular hypervisors such as KVM, Xen, Hyper-V and VMware vCenter, turning traditional virtualisation environments into cloud resource pools. Nova has many different individual components, each of which are responsible for a specific task, examples include a compute driver which is responsible for providing resource to the cloud, a scheduler to respond to and allocate requests from cloud consumers and a network layer which provides inward and outward network traffic to compute instances.

This lab enables us to start instances based on the previous two labs; each instance requires an image to boot from and a network to attach to.

##**Starting instances via the console**

Let's launch our first instance in OpenStack using the command line. The first task that we'll do is create our own custom flavor; flavors define the specification of the instance in terms of vCPUs, memory, disk space, etc. For this we'll need to switch back to our user with administrative access:

	# source keystonerc_admin
	
Then, create a new flavor for us to use:

	# nova flavor-create my_new_flavor 6 1024 20 2
	+----+---------------+-----------+------+-----------+------+-------+-------------+-----------+
	| ID | Name          | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
	+----+---------------+-----------+------+-----------+------+-------+-------------+-----------+
	| 6  | my_new_flavor | 1024      | 20   | 0         |      | 2     | 1.0         | True      |
	+----+---------------+-----------+------+-----------+------+-------+-------------+-----------+

What we've done there is to create a new flavor, with the name "my_new_flavor" with 1024MB RAM, 2 vCPU's, and 20GB of disk space. Therefore, any instance that we create with this flavor profile will be assigned this specification.

__Note:__ Ensure you switch back to your normal user after the flavor creation.

Now we've created a flavor, switch back to the normal unprivileged user and find the image id that we want to boot:

	# source keystonerc_user

	# nova flavor-list
	+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
	| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public | extra_specs |
	+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
	| 1  | m1.tiny   | 512       | 0    | 0         |      | 1     | 1.0         | True      | {}          |
	| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      | {}          |
	| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      | {}          |
	| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      | {}          |
	| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      | {}          |
	| 6  | my_new_flavor | 1024      | 20   | 0         |      | 2     | 1.0         | True      |
	+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+

	# nova image-list
	+--------------------------------------+-----------------+--------+--------+
	| ID                                   | Name            | Status | Server |
	+--------------------------------------+-----------------+--------+--------+
	| 6b6e8ef2-23a5-4c79-9f0a-cf3e2b9f6119 | Cirros 0.3.1    | ACTIVE |        |
	| 1b31eb27-5d0f-4cab-b870-cee0fadd818a | My Cirros Image | ACTIVE |        |
	| df65596e-74ee-4a3f-83fa-538141d86ac0 | rhel6-cfntools  | ACTIVE |        |
	+--------------------------------------+-----------------+--------+--------+

Next, boot a new instance on OpenStack using our new flavor and the image we uploaded earlier ("My Cirros Image"), ensure you specify a name for the instance, below we use "MyNewVM":

	# nova boot --flavor 6 --image 1b31eb27-5d0f-4cab-b870-cee0fadd818a MyNewVM
	+--------------------------------------+--------------------------------------+
	| Property                             | Value                                |
	+--------------------------------------+--------------------------------------+
	| status                               | BUILD                                |
	| updated                              | 2014-04-08T20:41:58Z                 |
	| OS-EXT-STS:task_state                | scheduling                           |
	| key_name                             | None                                 |
	| image                                | My Cirros Image                      |
	| hostId                               |                                      |
	| OS-EXT-STS:vm_state                  | building                             |
	| OS-SRV-USG:launched_at               | None                                 |
	| flavor                               | my_new_flavor                        |
	| id                                   | 36c09f37-a23b-44ce-b180-bdd113fcec38 |
	| security_groups                      | [{u'name': u'default'}]              |
	| OS-SRV-USG:terminated_at             | None                                 |
	| user_id                              | 86c656ef36c94d299471395146bfc5d0     |
	| name                                 | MyNewVM                              |
	| adminPass                            | BktGwVX2djNj                         |
	| tenant_id                            | c6d79dbd835842f5b36d09a737c8e530     |
	| created                              | 2014-04-08T20:41:58Z                 |
	| OS-DCF:diskConfig                    | MANUAL                               |
	| metadata                             | {}                                   |
	| os-extended-volumes:volumes_attached | []                                   |
	| accessIPv4                           |                                      |
	| accessIPv6                           |                                      |
	| progress                             | 0                                    |
	| OS-EXT-STS:power_state               | 0                                    |
	| OS-EXT-AZ:availability_zone          | nova                                 |
	| config_drive                         |                                      |
	+--------------------------------------+--------------------------------------+
	
Note that because we only have one private network, it assumes we want to join this one. Otherwise we would have had to specify a network to use.

	# nova list
	+--------------------------------------+-----------+--------+------------------+
	| ID                                   | Name      | Status | Networks         |
	+--------------------------------------+-----------+--------+------------------+
	| 76209a2f-a9df-4100-9cd8-4b7f875d1c3a | MyNewVM   | ACTIVE | int=30.0.0.4     |
	+--------------------------------------+-----------+--------+------------------+

As you can see, our machine has been given a network address of 30.0.0.4 and has been started. Finally, lets remove this instance and repeat the process via the dashboard:

	# nova delete MyNewVM
	
Note that the above command doesn't provide any output (unless there's an error), but can be verified using the following..

	# nova list
	+----+------+--------+------------+-------------+----------+
	| ID | Name | Status | Task State | Power State | Networks |
	+----+------+--------+------------+-------------+----------+
	+----+------+--------+------------+-------------+----------+
	

##**Starting instances via the Dashboard**

1. Login to the dashboard (with your user account) by clicking on the shortcut on the desktop of your workstation
2. Select 'Instances' on the left-hand side
3. Select 'Launch Instance' in the top-right
4. Select 'Boot from Image' from 'Instance Boot Source' at the bottom of the page
5. Choose 'My Cirros Image' from the Image drop-down box
6. Give the instance a name (Use "MyVM")
7. Ensure that 'my_new_flavor' is selected in the Flavour drop-down box
8. Select the 'Networking' tab at the top and drag the 'int' network from the 'available networks' area
9. Select 'Launch' in the bottom right-hand corner of the pop-up window

Leave this machine running before we proceed with the rest of the lab...

##**Viewing Console Output (VNC)**

Via the OpenStack Dashboard (Horizon) as well as via the command-line tools we can access both the console log and the VNC console. OpenStack provides a component called novncproxy, this brokers connections from clients to the compute nodes running the instances themselves. Connections come into the novncproxy service only, it then creates a tunnel through to the VNC server, meaning VNC servers need not be open to everyone, purely the proxy service. 
	
There's two ways of accessing the VNC console using novncproxy; either via the dashboard, simply select the instance and select the 'VNC' tab, or use the command-line utility and navigate to the URL specified:

	# nova list
	+--------------------------------------+------+--------+------------------+
	| ID                                   | Name | Status | Networks         |
	+--------------------------------------+------+--------+------------------+
	| c38fb239-370a-4d6f-87e2-5adf34aaa936 | MyVM | ACTIVE | int=30.0.0.4     |
	+--------------------------------------+------+--------+------------------+

	# nova get-vnc-console MyVM novnc
	+-------+--------------------------------------------------------------------------------------+
	| Type  | Url                                                                                  |
	+-------+--------------------------------------------------------------------------------------+
	| novnc | http://openstack-demo:6080/vnc_auto.html?token=164d6792-53b8-40d4-a2ba-6fee409bd514  |
	+-------+--------------------------------------------------------------------------------------+

__Warning:__ You may find that if you run the above command before the machine has booted, you'll receive the following error. Simply wait a minute or two for the instance to boot.

	# nova get-vnc-console MyVM novnc
	ERROR: Instance not yet ready (HTTP 409) (Request-ID: req-da372916-10ba-4158-91a2-0cc10f4d083e)

##**Attaching Floating IP's to Instances**

So far we've started instances, these instances have received private internal IP addresses (not typically routable outside of the OpenStack environment) and we can view the console via a VNC proxy in a web-browser. The next step is to configure access to these instances from outside of the private network.

OpenStack allows us to assign 'floating IPs' to instances to allow network traffic from any external interface to be routed to a specific instance, running on a private "tenant network". The IP's assigned come directly from one or more external networks, thankfully we already have one ("external"). Behind the scenes the node running the L3-agent listens on an additional IP address and uses NAT to tunnel the traffic to the correct instance on the private network. 

##**Creating Floating Addresses**

For this task you'll need an instance running first, if you don't have one running revisit the previous lab and start an instance. We'll request floating IP's from a pool that we've already assigned, this is known as the allocation pool on the external network.

OpenStack makes you 'claim' an IP from the available list of IP addresses for the tenant (project) you're currently running in before you can assign it to an instance, we specify the 'ext' network to claim from:

	# neutron floatingip-create external
	Created a new floatingip:
	+---------------------+--------------------------------------+
	| Field               | Value                                |
	+---------------------+--------------------------------------+
	| fixed_ip_address    |                                      |
	| floating_ip_address | 192.168.122.202                      |
	| floating_network_id | 7382ead9-faba-405a-a78f-404c236c9334 |
	| id                  | 2f8a9079-55fa-44ab-b2c9-99685d7f3664 |
	| port_id             |                                      |
	| router_id           |                                      |
	| tenant_id           | 97b43bd18e7c4f7ebc45b39b090e9265     |
	+---------------------+--------------------------------------+

You can see that it's reserved for our freshly created tenant, although it's not attached to an instance *yet*...

##**Assigning an address**

Next, we can assign our claimed IP address to an instance:

	# nova add-floating-ip MyVM 192.168.122.202
	
__Note:__ Your IP address may vary from the output shown above.

Verify using:

	# neutron floatingip-list
	+--------------------------------------+------------------+---------------------+--------------------------------------+
	| id                                   | fixed_ip_address | floating_ip_address | port_id                              |
	+--------------------------------------+------------------+---------------------+--------------------------------------+
	| 2f8a9079-55fa-44ab-b2c9-99685d7f3664 | 30.0.0.4         | 192.168.122.202     | d8233763-a214-47db-80bb-76885a06205b |
	+--------------------------------------+------------------+---------------------+--------------------------------------+
	

	# nova list
	+--------------------------------------+------+--------+----------------------------------+
	| ID                                   | Name | Status | Networks                         |
	+--------------------------------------+------+--------+----------------------------------+
	| 843441f6-b8ab-4ed9-9231-3326ee19e6ec | test | ACTIVE | int=30.0.0.4, 192.168.122.202    |
	+--------------------------------------+------+--------+----------------------------------+


NOTE: It may take a while for the Floating IP to show up in Nova (or via the Dashboard), so keep trying the above command. Also note that trying to ping your floating IP will currently fail, see the next section for details...

##**OpenStack Security Groups**

By default, OpenStack Security Groups prevent any access to instances via the public network, including ICMP/ping! Therefore, we have to manually edit the security policy to ensure that the firewall is opened up for us. Let's add two rules, firstly for all instances to have ICMP and SSH access. By default, Neutron ships with a 'default' security group, it's possible to create new groups and assign custom rules to these groups and then assign these groups to individual servers. For this lab, we'll just configure the default group.

First, enable ICMP for *every* node:

	# neutron security-group-rule-create --protocol icmp --remote-ip-prefix 0.0.0.0/0 default
	Created a new security_group_rule:
	+-------------------+--------------------------------------+
	| Field             | Value                                |
	+-------------------+--------------------------------------+
	| direction         | ingress                              |
	| ethertype         | IPv4                                 |
	| id                | dba33ff9-c616-4d8c-90ee-40512de5313d |
	| port_range_max    |                                      |
	| port_range_min    |                                      |
	| protocol          | icmp                                 |
	| remote_group_id   |                                      |
	| remote_ip_prefix  | 0.0.0.0/0                            |
	| security_group_id | b4f0829a-b4b6-45ed-972b-bc5d0e55bd58 |
	| tenant_id         | 97b43bd18e7c4f7ebc45b39b090e9265     |
	+-------------------+--------------------------------------+

Within a few seconds (for the hypervisor to pick up the changes) you should be able to ping your floating IP:

	# ping -c4 192.168.122.202
	PING 192.168.122.202 (192.168.122.202) 56(84) bytes of data.
	64 bytes from 192.168.122.202: icmp_seq=1 ttl=63 time=2.32 ms
	64 bytes from 192.168.122.202: icmp_seq=2 ttl=63 time=0.965 ms
	...

We can ping, but we can't SSH yet, as that's still not allowed:

	# ssh -v cirros@192.168.122.202
	OpenSSH_5.3p1, OpenSSL 1.0.0-fips 29 Mar 2010
	debug1: Reading configuration data /etc/ssh/ssh_config
	debug1: Applying options for *
	debug1: Connecting to 192.168.122.202 [192.168.122.202] port 22.
	debug1: connect to address 192.168.122.202 port 22: Connection timed out
	ssh: connect to host 192.168.122.202 port 22: Connection timed out

Next, let's try adding another rule, to allow SSH access for all instances in the 'default' group, but only allow ssh from the 192.168.122.0/24 network:

	# neutron security-group-rule-create --protocol tcp --port-range-min 22 --port-range-max 22 --remote-ip-prefix 192.168.122.0/24 default
	Created a new security_group_rule:
	+-------------------+--------------------------------------+
	| Field             | Value                                |
	+-------------------+--------------------------------------+
	| direction         | ingress                              |
	| ethertype         | IPv4                                 |
	| id                | b8ed2037-be88-4f26-bd4d-887558f94288 |
	| port_range_max    | 22                                   |
	| port_range_min    | 22                                   |
	| protocol          | tcp                                  |
	| remote_group_id   |                                      |
	| remote_ip_prefix  | 192.168.122.0/24                     |
	| security_group_id | b4f0829a-b4b6-45ed-972b-bc5d0e55bd58 |
	| tenant_id         | 97b43bd18e7c4f7ebc45b39b090e9265     |
	+-------------------+--------------------------------------+

And, let's retry the SSH connection.. Note: Cirros user password is "cubswin:)"

	# ssh cirros@192.168.122.202
	The authenticity of host '192.168.122.202 (192.168.122.202)' can't be established.
	RSA key fingerprint is 18:71:dd:3d:c5:cf:6b:5a:73:d4:e3:0b:11:af:7a:ec.
	Are you sure you want to continue connecting (yes/no)? yes
	Warning: Permanently added '192.168.122.202' (RSA) to the list of known hosts.
	cirros@192.168.122.202's password: 
	$

When you see the "$" prompt, you're connected into your instance successfully. Check the network configuration within the instance, note that it is *not* aware of the "192.168.122.202" address - this is being NAT'd by the L3 agent from the outside external network:

	$ ip a
	1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue 
	    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
	    inet 127.0.0.1/8 scope host lo
	    inet6 ::1/128 scope host 
	       valid_lft forever preferred_lft forever
	2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
	    link/ether fa:16:3e:ad:a3:9d brd ff:ff:ff:ff:ff:ff
	    inet 30.0.0.4/24 brd 30.0.0.255 scope global eth0
	    inet6 fe80::f816:3eff:fead:a39d/64 scope link 
	       valid_lft forever preferred_lft forever

Press "CTRL+D", or simply type "exit" to return to your OpenStack environment:

	$ exit
	Connection to 192.168.122.202 closed.
	#

Let's clean-up this instance before we proceed with the next section:

	# nova list
	+--------------------------------------+------+--------+----------------------------------+
	| ID                                   | Name | Status | Networks                         |
	+--------------------------------------+------+--------+----------------------------------+
	| 843441f6-b8ab-4ed9-9231-3326ee19e6ec | MyVM | ACTIVE | int=30.0.0.4, 192.168.122.202    |
	+--------------------------------------+------+--------+----------------------------------+
	# nova delete MyVM


##**Uploading public keys**

Public keys are used in OpenStack (and other cloud platforms) to uniquely identify a user, avoiding any password requirements. It's also useful when you have passwords installed by users in their VM images but they aren't shared; with a key a user can log-in and change the password to something they're happy with. When an instance is created, one of the options is to select a public key to assign or to upload one into the OpenStack database.

Create a new ssh-key with the following instructions, note that you can choose to either create a passphrase of your own, or leave it blank (for ease I recommend you leave it blank):

	# ssh-keygen
	Generating public/private rsa key pair.
	Enter file in which to save the key (/root/.ssh/id_rsa):
	Created directory '/root/.ssh'.
	Enter passphrase (empty for no passphrase):
	Enter same passphrase again:
	Your identification has been saved in /root/.ssh/id_rsa.
	Your public key has been saved in /root/.ssh/id_rsa.pub.
	The key fingerprint is:
	f7:29:b0:5e:aa:1a:73:a0:73:f8:54:c3:c6:12:8b:73 root@openstack-demo
	The key's randomart image is:
	+--[ RSA 2048]----+
	|                 |
	|                 |
	|    .            |
	|   . =           |
	|  o E * S .      |
	|   = = . + . .   |
	|  + = . . o o    |
	|   = + . o .     |
	|    o...o        |
	+-----------------+

Either way, you can upload your key either via the dashboard (Project --> Access & Security --> Import Keypair --> Copy/paste code from your id_rsa.pub) or from the command-line:

	# nova keypair-add --pub-key /root/.ssh/id_rsa.pub mypublickey


##**Launching an Instance with a Keypair**

Next, launch an instance and ensure that you specify your keypair to use (note that if you do this via the dashboard it will automatically select it if it's the only one available) and check that you can connect in over SSH *without* it asking you for a password.

	# nova keypair-list
	+-------------+-------------------------------------------------+
	| Name        | Fingerprint                                     |
	+-------------+-------------------------------------------------+
	| mypublickey | bc:6e:27:db:47:71:b3:03:4c:37:b4:42:3d:b2:34:ee |
	+-------------+-------------------------------------------------+

Let's now boot up a RHEL 6 instance. First, get the image id for RHEL 6:

	# glance index | grep "rhel6-cfntools" | awk '{print $1};'
	
Now boot up an instance based on this image and the previously used flavor, but specify your keypair:


	# nova boot --flavor 6 --image 85201822-298c-4e6b-9f24-933f796973ae newinstance --key_name mypublickey
	[... output omitted...]

__Note:__ It will take substantially longer for this machine to become available; the image size for RHEL is a lot larger.

Next associate a floating IP address to your instance:

	# neutron floatingip-create external
	Created a new floatingip:
	+---------------------+--------------------------------------+
	| Field               | Value                                |
	+---------------------+--------------------------------------+
	| fixed_ip_address    |                                      |
	| floating_ip_address | 192.168.122.203                      |
	| floating_network_id | 9e3cc60a-3354-4528-9e4b-3f3a0825156f |
	| id                  | ab3e7d53-af3f-4f46-b68b-c2621894ccf9 |
	| port_id             |                                      |
	| router_id           |                                      |
	| tenant_id           | c6d79dbd835842f5b36d09a737c8e530     |
	+---------------------+--------------------------------------+

	# nova add-floating-ip newinstance 192.168.122.203
	
__Note:__ Your IP address may vary from the output shown above.

If the metadata server worked correctly you should be able to ssh into the machine without requiring the password. If you cannot login, ensure that you associated your instance with the keypair as above.

	# ssh root@192.168.122.203
	# uname -a
	Linux newinstance 2.6.32-431.3.1.el6.x86_64 #1 SMP Fri Dec 13 06:58:20 EST 2013 x86_64 x86_64 x86_64 GNU/Linux
	
Let's now delete this machine now we've confirmed the metadata service and keypair injection is working. You'll need to disconnect from this machine first:

	# exit
	logout
	Connection to 192.168.122.203 closed.
	
Then, delete the machine:

	# nova delete newinstance

##**Executing boot-time scripts**

In the OpenStack dashboard, we can insert script-code that will get executed at boot-time via cloud-init (or similar, depending on distribution), this can be carried out via the CLI also but using the Dashboard makes things easier:

1. Login to the dashboard by selecting the link on the desktop of your workstation 
2. Select 'Instances' on the left hand-side
3. Select 'Launch Instance' at the top right
4. Choose an appropriate name, e.g. "bootscript_test" and choose the "my_new_flavor" flavor
5. Select the 'rhel6-cfntools' image and 
6. Select the "int" network from the networking tab
7. Select the final tab at the top named 'Post-Creation'
8. Paste in the following (without the quotes):
	
	"#!/bin/sh
	
	uname -a > /tmp/uname
	
	date > /tmp/date"
9. Select 'Launch' at the bottom of the pop-up window (and wait a few minutes for it to become active)
10. Associate a floating-ip by choosing 'More' --> 'Associate Floating IP' --> Select the "+" --> 'Allocate IP' --> Associate
11. Refresh the page to display the floating IP that has been assigned (note this may take a few minutes to show)

Now, try and connect out to the instance by issuing a few commands via secure shell, testing to see whether the boot script worked. If successful, it should print out the output of uname and also the time and date of when the script was executed. Note that when you try to do this, the machine may still be booting, so be patient! Also, your IP address may vary from the output shown below.

	# nova list
	+--------------------------------------+--------+--------+----------------------------------+
	| ID                                   | Name   | Status | Networks                         |
	+--------------------------------------+--------+--------+----------------------------------+
	| c025278e-7067-42c0-977c-d428605daafc |  bootscript_test  | ACTIVE | int=30.0.0.4, 192.168.122.204    |
	+--------------------------------------+--------+--------+----------------------------------+
	
	# ssh root@192.168.122.204 cat /tmp/uname
	Linux bootscript-test 2.6.32-431.3.1.el6.x86_64 #1 SMP Fri Dec 13 06:58:20 EST 2013 x86_64 x86_64 x86_64 GNU/Linux
	
	# ssh root@192.168.122.204 cat /tmp/date
	Wed Apr  9 00:05:07 BST 2014

Leave this machine running so that we can carry on with the lab below.

#**Lab 6: Creation and Attachment of Block Devices with Cinder (Volume Service)**


##**Introduction**

Cinder is OpenStack's volume service, it's responsible for managing persistent block storage, i.e. the creation of a block device, it's lifecycle, connections to instances and it's eventual deletion. Block storage is an optional requirement in OpenStack but has many advantages, firstly persistence but also for performance scenarios, e.g. access to data backed by tiered storage. Cinder supports many different back-ends in which it can connect to and manage storage for OpenStack, including HP's LeftHand, EMC, IBM, Ceph, Red Hat Storage (Gluster) and NetApp, although it does support a basic Linux storage model, based on iSCSI. Cinder was once part of Nova and was known as nova-volume; it's since been divorced from Nova in order to allow each distinct component to evolve independently. 


##**Testing Cinder**

Let's test our Cinder configuration, making sure that it can create a volume, note that you'll need to be authenticated with Keystone for this to work:

	# cinder create --display-name Test 5
	+---------------------+--------------------------------------+
	|       Property      |                Value                 |
	+---------------------+--------------------------------------+
	|     attachments     |                  []                  |
	|  availability_zone  |                 nova                 |
	|      created_at     |      2013-03-30T18:34:15.049367      |
	| display_description |                 None                 |
	|     display_name    |                 Test                 |
	|          id         | d1907853-68e5-45e7-aa86-23b52a833258 |
	|       metadata      |                  {}                  |
	|         size        |                  5                   |
	|     snapshot_id     |                 None                 |
	|        status       |               creating               |
	|     volume_type     |                 None                 |
	+---------------------+--------------------------------------+

The above command will create a 5GB block device within the test environment although it won't be attached to any instances yet. Let's attach our volume to the running instance from the previous lab; if you don't have this - create a new instance as part of the previous lab.

Firstly, check that Nova can see the volume as expected:

	# nova volume-list
	+--------------------------------------+-----------+--------------+------+-------------+-------------+
	| ID                                   | Status    | Display Name | Size | Volume Type | Attached to |
	+--------------------------------------+-----------+--------------+------+-------------+-------------+
	| d1907853-68e5-45e7-aa86-23b52a833258 | available | Test         | 5    | None        |             |
	+--------------------------------------+-----------+--------------+------+-------------+-------------+

##**Attaching your volume**

Next, attach it using "nova volume-attach":

	# nova volume-attach bootscript_test ada30c3e-ebba-4d94-b9de-892ece32f28d auto
	+----------+--------------------------------------+
	| Property | Value                                |
	+----------+--------------------------------------+
	| device   | /dev/vdb                             |
	| serverId | a024b4de-3e41-4bee-8815-15d0b2928489 |
	| id       | ada30c3e-ebba-4d94-b9de-892ece32f28d |
	| volumeId | ada30c3e-ebba-4d94-b9de-892ece32f28d |
	+----------+--------------------------------------+

__Note:__ the 'auto' listed above means attach the device at the next available SCSI port, i.e. in our case, '/dev/vdb', otherwise you can override it to another. Also, if you chose a name for your machine other than "bootscript_test", replace the name as appropriate.


Let's reconnect back into our instance and verify that the volume is available, replace the IP address as necessary for your machine:

	# ssh root@192.168.122.204
	
Once connected, let's ensure that the block device is available for us to use:

	# fdisk -l /dev/vdb
	Disk /dev/vdb: 5368 MB, 5368709120 bytes
	16 heads, 63 sectors/track, 10402 cylinders
	Units = cylinders of 1008 * 512 = 516096 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disk identifier: 0x00000000
	
Next, we can create a filesystem and mount that up:

	# mkfs.ext4 /dev/vdb
	mke2fs 1.41.12 (17-May-2010)
	Filesystem label=
	OS type: Linux
	Block size=4096 (log=2)
	Fragment size=4096 (log=2)
	Stride=0 blocks, Stripe width=0 blocks
	327680 inodes, 1310720 blocks
	65536 blocks (5.00%) reserved for the super user
	First data block=0
	Maximum filesystem blocks=1342177280
	40 block groups
	32768 blocks per group, 32768 fragments per group
	8192 inodes per group
	Superblock backups stored on blocks: 
		32768, 98304, 163840, 229376, 294912, 819200, 884736

	Writing inode tables: done                            
	Creating journal (32768 blocks): done
	Writing superblocks and filesystem accounting information: done

	This filesystem will be automatically checked every 30 mounts or
	180 days, whichever comes first.  Use tune2fs -c or -i to override.
	
	# mount /dev/vdb /mnt
	# df -Th
	Filesystem     Type   Size  Used Avail Use% Mounted on
	/dev/vda3      ext4   5.8G  965M  4.6G  18% /
	tmpfs          tmpfs  1.9G     0  1.9G   0% /dev/shm
	/dev/vda1      ext4   485M   52M  409M  12% /boot
	/dev/vdb       ext4   5.0G  138M  4.6G   3% /mnt

Next unmount your volume:

	# umount /mnt
	# exit
	Connection to 192.168.122.204 closed.
	
Finally, detatch the volume from the instance and delete the instance and volume:

	# nova volume-list
	+--------------------------------------+--------+--------------+------+-------------+--------------------------------------+
	| ID                                   | Status | Display Name | Size | Volume Type | Attached to                          |
	+--------------------------------------+--------+--------------+------+-------------+--------------------------------------+
	| ada30c3e-ebba-4d94-b9de-892ece32f28d | in-use | Test         | 5    | None        | a024b4de-3e41-4bee-8815-15d0b2928489 |
	+--------------------------------------+--------+--------------+------+-------------+--------------------------------------+

	# nova volume-detach bootscript_test ada30c3e-ebba-4d94-b9de-892ece32f28d
	
	# cinder list
	+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
	|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
	+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
	| ada30c3e-ebba-4d94-b9de-892ece32f28d | available |     Test     |  5   |     None    |  false   |             |
	+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+

	# nova delete bootscript_test


#**Lab 7: Deployment of Application Stacks using OpenStack Orchestration (Heat)**

##**Introduction**



In this lab we'll deploy a simple resource-based Stack, comprising of a single server, with it's own private network, a floating IP address on the external network, and some small modifications to the instance upon boot.

##**Getting the Wordpress Stack Definition**

For this lab, the instructor has provided a pre-defined template. It can be deployed either using the dashboard or via the command line tools. This lab will guide you through deploying using the command line.

The following stack template can be copied into a file called heat-demo.yaml:

	heat_template_version: 2013-05-23

	description: >
	 'Red Hat Summit 2014 - Heat Example'

	parameters:
	  KeyName: 
	    description: The name of an existing SSH keypair to inject into the instance
	    type: string
	    default: "mypublickey"
	  FlavorSize: 
	    description: The flavor required for the instance
	    type: string
	    default: "my_new_flavor"
	    constraints:
	      - allowed_values: [m1.small, m1.medium, my_new_flavor]
	  TemplateName:
	    description: The name of a template to deploy
	    type: string
	    default: "rhel6-cfntools"
	  FloatingNetworkUUID:
	    description: UUID of the Public Network
	    type: string
	    default: ""

	resources:
	  SecurityGroup:
	    type: AWS::EC2::SecurityGroup
	    properties:
	      GroupDescription: Firewall Rules for Heat Demo
	      SecurityGroupIngress: [
	        { "CidrIp" : "0.0.0.0/0", "FromPort" : "22", "ToPort" : "22", "IpProtocol" : "TCP" },
	        { "CidrIp" : "0.0.0.0/0", "FromPort" : "-1", "ToPort" : "-1", "IpProtocol" : "ICMP" }
	        ]

	  NeutronNetwork:
	    type: OS::Neutron::Net
	    properties:
	     name: { "Ref" : "AWS::StackName" }
	     shared: false

	  NeutronSubnet:
	    type: OS::Neutron::Subnet
	    depends_on: NeutronNetwork
	    properties:
	      cidr: "60.0.0.0/24"
	      enable_dhcp: "true"
	      ip_version: 4
	      name: { "Ref" : "AWS::StackName" }
	      network_id: { get_resource: "NeutronNetwork" }

	  NeutronRouter:
	    type: OS::Neutron::Router
	    properties:
	      name: demo-router

	  RouterGateway:
	    type: OS::Neutron::RouterGateway
	    depends_on: NeutronRouter
	    properties:
	      network_id: { get_param: FloatingNetworkUUID }
	      router_id: { get_resource: NeutronRouter }

	  RouterInterface:
	    type: OS::Neutron::RouterInterface
	    depends_on: NeutronSubnet
	    properties:
	      router_id: { get_resource: "NeutronRouter" }
	      subnet_id: { get_resource: "NeutronSubnet" }

	  NeutronPort:
	    type: OS::Neutron::Port
	    properties:
	      admin_state_up: true
	      network_id: { "Ref" : "NeutronNetwork" }
	      name: { "Ref" : "AWS::StackName" }

	  DemoServer: 
	    type: OS::Nova::Server
	    depends_on: NeutronPort
	    properties: 
	      flavor: { get_param: FlavorSize }
	      image: { get_param: TemplateName }
	      key_name: { get_param: KeyName }
	      name: demo-server
	      networks: [{ "port" : { "Ref" : "NeutronPort" } }]
	      security_groups: [{ get_resource: "SecurityGroup" }]
	      user_data: 
	        str_replace:
	          template: |
	            #!/bin/bash
	            set -o verbose
            
	      	      # Setting up the motd and banners
	            sed -i '$ d' /etc/issue
	            echo -e "Red Hat Summit 2014 - San Francisco, California\n" >> /etc/issue
	            echo -e "\nWelcome to Red Hat Summit 2014\n" >> /etc/motd

	  FloatingIP:
	    type: OS::Neutron::FloatingIP
	    properties:
	      floating_network_id: { get_param: FloatingNetworkUUID }

	  FloatingIPAssociate:
	    type: OS::Neutron::FloatingIPAssociation
	    depends_on: DemoServer
	    properties:
	      floatingip_id: { get_resource: "FloatingIP" }
	      port_id: { get_resource: "NeutronPort" }

	outputs:
	  Server_IP:
	    description: Floating IP address of instance
	    value:
	      str_replace:
	        template: root@%host%
	        params:
	           "%host%": { get_attr: [ FloatingIP, floating_ip_address ] }


##**What the Stack Definition does**

For those not familiar with the template language used within OpenStack Heat, I suggest you take a look at the file, specifically where it lists the images used, the resources it requires, and any script data that the Heat (or CloudFormations) tools execute upon boot. You will notice that the Heat definition deploys the following elements-

* A new security group to allow ICMP and TCP on port 22 (SSH).
* A new private network for our application to deploy onto
* An associated subnet for the network previously created
* A router and a gateway to allow our private network to communicate with the outside
* An instance with a configuration script (that gets uploaded into the metadata)
* Finally, a floating IP address for external access

##**Deploying our Stack**

For us to successfully deploy this stack, the only parameter we need to pass over to Heat is the UUID of an external network for the floating IP to be pulled from, thankfully one is already available for us. In the template we define this parameter as "FloatingNetworkUUID". So let's get this first, using the administrator-provided external network 'external':

	# source keystonerc_<user>
	# FLOATING=`neutron net-show ext | grep -m1 id | awk '{print $4;}'`

Next, create the stack using the CLI tools, using the external network ID as the required paramater. Note that if you omit this parameter, the stack will fail:

	# heat stack-create Demo -f heat-demo.yaml -P FloatingNetworkUUID=$FLOATING
	+--------------------------------------+------------+--------------------+----------------------+
	| id                                   | stack_name | stack_status       | creation_time        |
	+--------------------------------------+------------+--------------------+----------------------+
	| 92473d0f-04e7-43bf-b643-1e7da105b454 | Demo       | CREATE_IN_PROGRESS | 2014-02-11T20:20:55Z |
	+--------------------------------------+------------+--------------------+----------------------+

After a few minutes, it should have completed, you can view the progress using the following-

	# heat stack-list
	+--------------------------------------+------------+-----------------+----------------------+
	| id                                   | stack_name | stack_status    | creation_time        |
	+--------------------------------------+------------+-----------------+----------------------+
	| 92473d0f-04e7-43bf-b643-1e7da105b454 | Demo       | CREATE_COMPLETE | 2014-02-11T20:20:55Z |
	+--------------------------------------+------------+-----------------+----------------------+

After a few minutes, your stack should have deployed successfully, simple use the following to find the output IP address for your instance:

	# heat stack-show Demo
	.... output omitted ....
		"output_value": "root@192.168.122.202",                                                                        
		"description": "Floating IP address of instance",                                                                                 
		"output_key": "Server_IP"                                                                                                                    

You should then be able to secure shell into the instance using the output as above. Note that the floating IP that OpenStack provides to you will likely be different from the one shown in the example above. If you cannot login, wait a few more minutes and try again. The root password for the machine is 'redhat' if your public key wasn't uploaded.

Finally, clean-up your deployment:

	# heat stack-delete Demo
	+--------------------------------------+------------+--------------------+----------------------+
	| id                                   | stack_name | stack_status       | creation_time        |
	+--------------------------------------+------------+--------------------+----------------------+
	| 92473d0f-04e7-43bf-b643-1e7da105b454 | Demo       | DELETE_IN_PROGRESS | 2014-02-11T20:20:55Z |
	+--------------------------------------+------------+--------------------+----------------------+
