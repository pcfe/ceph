=================================
 Using libvirt with Ceph RBD
=================================

.. index:: Ceph Block Device; libvirt

The ``libvirt`` library creates a virtual machine abstraction layer between 
hypervisor interfaces and the software applications that use them. With 
``libvirt``, developers and system administrators can focus on a common 
management framework, common API, and common shell interface (i.e., ``virsh``)
to many different hypervisors, including: 

- QEMU/KVM
- XEN
- LXC
- VirtualBox
- etc.

Ceph block devices support QEMU/KVM. You can use Ceph block devices with
software that interfaces with ``libvirt``. The following stack diagram
illustrates how ``libvirt`` and QEMU use Ceph block devices via ``librbd``. 


.. ditaa::  +---------------------------------------------------+
            |                     libvirt                       |
            +------------------------+--------------------------+
                                     |
                                     | configures
                                     v
            +---------------------------------------------------+
            |                       QEMU                        |
            +---------------------------------------------------+
            |                      librbd                       |
            +------------------------+-+------------------------+
            |          OSDs          | |        Monitors        |
            +------------------------+ +------------------------+


The most common ``libvirt`` use case involves providing Ceph block devices to
cloud solutions like OpenStack or CloudStack. The cloud solution uses
``libvirt`` to  interact with QEMU/KVM, and QEMU/KVM interacts with Ceph block
devices via  ``librbd``. See `Block Devices and OpenStack`_ and `Block Devices
and CloudStack`_ for details. See `Installation`_ for installation details.

You can also use Ceph block devices with ``libvirt``, ``virsh`` and the
``libvirt`` API. See `libvirt Virtualization API`_ for details.


To create VMs that use Ceph block devices, use the procedures in the following
sections. In the exemplary embodiment, we have used ``libvirt-pool`` for the pool
name, ``client.libvirt`` for the user name, and ``new-libvirt-image`` for  the
image name. You may use any value you like, but ensure you replace those values
when executing commands in the subsequent procedures.


Configuring Ceph
================

To configure Ceph for use with ``libvirt``, perform the following steps:

#. `Create a pool`_. The following example uses the 
   pool name ``libvirt-pool`` with 16 placement groups. ::

	ceph osd pool create libvirt-pool 16 16

   Verify the pool exists. :: 

	ceph osd pool ls detail

#. Use the ``rbd`` tool to initialize the pool for use by RBD::

   rbd pool init <pool-name>

#. `Create a Ceph User`_ (or use ``client.admin`` for version 0.9.7 and
   earlier). The following example uses the Ceph user name ``client.libvirt``
   and references ``libvirt-pool``. ::

	ceph auth get-or-create client.libvirt mon 'profile rbd' osd 'profile rbd pool=libvirt-pool'
	
   Verify the name exists. :: 
   
	ceph auth ls

   **NOTE**: ``libvirt`` will access Ceph using the ID ``libvirt``, 
   not the Ceph name ``client.libvirt``. See `User Management - User`_ and 
   `User Management - CLI`_ for a detailed explanation of the difference 
   between ID and name.	


Configuring libvirt storage pool
================================

Please read https://libvirt.org/storage.html
to avoid confusing Ceph pools and libvirt storage pools

Then read https://libvirt.org/storage.html#StorageBackendRBD
To learn about configuring a libvirt storage pool of type Ceph RBD pool

#. If your Ceph Storage Cluster has `Ceph Authentication`_ enabled (it does by 
   default), you must generate a secret. On the libvirt machine :: 

	cat > secret.xml <<EOF
	<secret ephemeral='no' private='no'>
		<usage type='ceph'>
			<name>client.libvirt secret</name>
		</usage>
	</secret>
	EOF

#. Define the secret. On the libvirt machine ::

	sudo virsh secret-define --file secret.xml
	<uuid of secret is output here>

#. Get the ``client.libvirt`` key and save the key string to a file. On a Ceph machine ::

	ceph auth get-key client.libvirt | sudo tee client.libvirt.key

#. Transfer ``client.libvirt.key`` securely to you libvirt machine.

#. Feed the secret to libvirt. On the libvirt machine ::

	sudo virsh secret-set-value --secret {uuid of secret} --base64 $(cat client.libvirt.key) && rm client.libvirt.key secret.xml

#. Use the UUID of the secret when defining your libvirt storage pool. On the libvirt machine
   You reference the secret by it's UUID in the ``<auth>`` section of your storage pool definition.
   See https://libvirt.org/storage.html#StorageBackendRBD for details.

   Write an xml file with pool type rbd, a name of your choice, Ceph relevant details
   in ``<source>`` (the Ceph pool name, your mons and the uuid of the secret you defined
   for libvirt).

   Sample file ``/root/tmp/Ceph-HouseNet-libvirt-pool.xml`` ::

      <!--
         https://libvirt.org/storage.html#StorageBackendRBD
      -->
      <pool type="rbd">
      <name>Ceph-HouseNet-libvirt-pool</name>
      <source>
         <name>libvirt-pool</name>
         <host name='mon-1'/>
         <host name='mon-2'/>
         <host name='mon-3'/>
         <auth username='libvirt' type='ceph'>
            <secret uuid='{uuid of secret}'/>
         </auth>
      </source>
      </pool>

   Define the storage pool. On the libvirt machine ::

      sudo virsh pool-define /root/tmp/Ceph-HouseNet-libvirt-pool.xml
      sudo virsh pool-start Ceph-HouseNet-libvirt-pool

   **NOTE:** The exemplary ID is ``libvirt``, not the Ceph name 
   ``client.libvirt`` as generated at step 2 of `Configuring Ceph`_. Ensure 
   you use the ID component of the Ceph name you generated. If for some reason 
   you need to regenerate the secret, you will have to execute 
   ``sudo virsh secret-undefine {uuid}`` before executing 
   ``sudo virsh secret-set-value`` again.


Test if virsh can create an image
=================================

#. Use virsh to `create an image`_ in your RBD pool. (
   The following example uses the image name ``new-libvirt-image``
   and references ``Ceph-HouseNet-libvirt-pool``. ::

	virsh vol-create-as Ceph-HouseNet-libvirt-pool new-libvirt-image 10g

   Verify on the Ceph side that the image exists. On a Ceph host :: 

	rbd -p libvirt-pool ls

   **NOTE:** You can also use `rbd create`_ to create an image, but we
   recommend ensuring that QEMU is working properly.

.. tip:: Optionally, if you wish to enable debug logs and the admin socket for
   this client, you can add the following section to ``/etc/ceph/ceph.conf``::

	[client.libvirt]
	log file = /var/log/ceph/qemu-guest-$pid.log
	admin socket = /var/run/ceph/$cluster-$type.$id.$pid.$cctid.asok

   The ``client.libvirt`` section name should match the cephx user you created
   above. If SELinux or AppArmor is enabled, note that this could prevent the
   client process (qemu via libvirt) from writing the logs or admin socket to
   the destination locations (``/var/log/ceph`` or ``/var/run/ceph``).

Test if virsh can create and delete an image
============================================

#. List your pool content ::

   root@hypervisor ~ # virsh vol-list --pool Ceph-HouseNet-libvirt-pool 
   Name                 Path                                    
   ------------------------------------------------------------------------------
   20200309-fio-testing-1TiB libvirt-pool/20200309-fio-testing-1TiB  
   F31_Ceph_RBD_test    libvirt-pool/F31_Ceph_RBD_test          

#. Create a new image ::

   root@hypervisor ~ # virsh vol-create-as Ceph-HouseNet-libvirt-pool new-libvirt-image 10g
   Vol new-libvirt-image created

#. Verify the new image shows up in the livirt storage pool ::

   root@hypervisor ~ # virsh vol-list --pool Ceph-HouseNet-libvirt-pool 
   Name                 Path                                    
   ------------------------------------------------------------------------------
   20200309-fio-testing-1TiB libvirt-pool/20200309-fio-testing-1TiB  
   F31_Ceph_RBD_test    libvirt-pool/F31_Ceph_RBD_test          
   new-libvirt-image    libvirt-pool/new-libvirt-image          

#. Delete an image ::

   root@hypervisor ~ # virsh vol-delete --pool Ceph-HouseNet-libvirt-pool --vol new-libvirt-image
   Vol new-libvirt-image deleted

#. Verify the image disappears ::

   root@hypervisor ~ # virsh vol-list --pool Ceph-HouseNet-libvirt-pool 
   Name                 Path                                    
   ------------------------------------------------------------------------------
   20200309-fio-testing-1TiB libvirt-pool/20200309-fio-testing-1TiB  
   F31_Ceph_RBD_test    libvirt-pool/F31_Ceph_RBD_test  

Installing the VM Manager
=========================

You may use ``libvirt`` without a VM manager, but you may find it simpler to
create your first domain with ``virt-manager``. 

#. Install a virtual machine manager. See `VirtManager`_ and `KVM/VirtManager`_ for details. ::

	sudo dnf install virt-manager     # on Fedora and RHEL8
	sudo yum install virt-manager     # on RHEL7
	sudo apt-get install virt-manager # on Debian

#. Download an OS image (if necessary).

#. Launch the virtual machine manager. :: 

	virt-manager

#. Connect to a hypervisor (expect ``qemu:///system`` to be defined, but you may want to connect
   to a remote host using ``qemu+ssh://root@hypervisor.example.com/system`` or similar).


Verifying your pool is functional from within virt-manager
==========================================================

To ensure you configured both Ceph and libvirt correctly,
perform the following steps

#. Edit / Connection details
   of the libvirt host

#. Navigate to the Storage tab

#. Verify that you see the Ceph pool
   and the test volume you created earlier.
   If you do not see your pool, ensure ``libvirtd.service`` is aware
   of your changes to the storage pool definition.

#. Verify that you can create a volume from within ``virt-manager``
   by using the + icon.

#. Ensure you see the volume on the Ceph side ::

	[root@odroid-hc2-00 ~]# rbd -p libvirt-pool ls
	test1
	test2

#. Close the Connection Details window

Using your RBD pool
===================

Use it like just any other libvirt storage pool

This means;

#) in ``virt-manager`` using Select or create custom storage and then choosing your Ceph pool
   unless you made that pool the default

#) with ``virt-install`` you just reference it by using ``--disk pool=`` ::

	virt-install \
		--name rhel7.5-testmachine \
		--os-variant rhel7 \
		--disk pool=Ceph-HouseNet-libvirt-pool,boot_order=1,format=raw,bus=virtio,sparse=yes,size=10 \
	[...]

FIXME:: most of what follows binds one vol from a Ceph pool, the above tries to
get the use to use a pool on the libvirt side too, reword below as necessary

Creating a VM in virt-manager
=============================

To create a VM with ``virt-manager``, perform the following steps:

#. Press the **Create New Virtual Machine** button. 

#. Name the new virtual machine domain. In the following exemple, we
   use the name ``libvirt-virtual-machine``. You may use any name you wish,
   but ensure you replace ``libvirt-virtual-machine`` with the name you 
   choose in subsequent commandline and configuration examples. :: 

	libvirt-virtual-machine

#. Import the image. If you choose to create your VM from an image that is  ::

	/path/to/image/recent-linux.img

   **NOTE:** Import a recent image. Some older images may not rescan for 
   virtual devices properly.

   **NOTE:** You can also install your VM by attaching an ISO or network booting.
   
#. Configure and start the VM.
   Be sure to choose your Ceph pool for storage, unless you made that pool the default.

#. You may use ``virsh list`` on the command line to verify the VM domain exists. ::

	sudo virsh list

#. Login to the VM (root/root on some recent-linux.img)

#. Use ``fio`` to test the RBD backed disk your VM is running on if you want to get
   an indication of what performance your Ceph cluster can give the VM.

Creating a VM from the commandline
==================================

You may prefer to create your VM from CLI instead of GUI.

For example, 2 vHD. One in you default pool (probably, but not necessarily, 
``/var/lib/libvirt/images/``) and one on the previously defined, Ceph RBD pool backed, storage pool.
you may also want to measure performance wirth ``bus=virtio`` instead of ``bus=scsi`` ::

   virt-install --name F31_Ceph_RBD_test \
      --os-variant fedora29 \
      --location http://download.fedoraproject.org/pub/fedora/linux/releases/31/Server/x86_64/os \
      --extra-args "console=ttyS0,115200" \
      --ram 4096 \
      --disk pool=default,boot_order=1,format=qcow2,bus=scsi,discard=unmap,sparse=yes,size=10 \
      --disk pool=Ceph-HouseNet-libvirt-pool,boot_order=2,bus=scsi,sparse=yes,size=50 \
      --controller scsi,model=virtio-scsi \
      --rng /dev/random \
      --boot useserial=on \
      --vcpus 1 \
      --cpu host \
      --nographics \
      --accelerate \
      --network network=default,model=virtio

Please note that when you create a VM with ``virt-install`` you may need to refresh your
storage pool in ``virt-manager`` under *your hypervisor* / *Details* / *Storage*
if you do not yet see the newly created disk(s)

Example xml of a VM using the above pool
========================================

You may want to check if your VM configuration is using RBD, to do this
use the ``virsh dumpxml`` command and look for the relevant disk definituion block.

``virsh dumpxml rhel7.5-testmachine`` ::

	[...]
	<disk type='network' device='disk'>
		<driver name='qemu' type='raw'/>
		<auth username='libvirt'>
			<secret uuid='{uuid of secret}'/>
		</auth>
		<source protocol='rbd' name='libvirt-pool/test1'>
			<host name='mon-1'/>
			<host name='mon-2'/>
			<host name='mon-3'/>
		</source>
		<target dev='vda' bus='virtio'/>
		<alias name='virtio-disk0'/>
		<address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
	</disk>
	[...]


Summary
=======

Once you have configured ``libvirt`` to use a Ceph pool, you can run any VM with
virtual disk that reside on Ceph.
To verify that the VM and Ceph are communicating, you may perform the
following procedures.


#. Check to see if Ceph is running:: 

	ceph health

#. Check to see if the VM is running. :: 

	sudo virsh list

#. Check to see if the VM is communicating with Ceph. Replace 
   ``{vm-domain-name}`` with the name of your VM domain:: 

	sudo virsh qemu-monitor-command --hmp {vm-domain-name} 'info block'

#. Check to see if the device from ``virsh dumpxml {vm-domain-name}``, look for ``<target dev='vda' bus='virtio'/>``, appears
   under ``/dev`` or under ``/proc/partitions`` in the VM. :: 
   
	lsblk
	ls dev
	cat proc/partitions

If everything looks okay, you are using the Ceph block device 
within your VM.

missing storage backend for network files using rbd protocol
============================================================

Please see;

- https://access.redhat.com/solutions/4252871
- https://bugzilla.redhat.com/show_bug.cgi?id=1724808

If you find your logs spammed with ::

	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol
	error : virStorageFileBackendForType:142 : internal error: missing storage backend for network files using rbd protocol

.. _Installation: ../../install
.. _libvirt Virtualization API: http://www.libvirt.org
.. _Block Devices and OpenStack: ../rbd-openstack
.. _Block Devices and CloudStack: ../rbd-cloudstack
.. _Create a pool: ../../rados/operations/pools#create-a-pool
.. _Create a Ceph User: ../../rados/operations/user-management#add-a-user
.. _create an image: ../qemu-rbd#creating-images-with-qemu
.. _Virsh Command Reference: http://www.libvirt.org/virshcmdref.html
.. _VirtManager: https://virt-manager.org/
.. _KVM/VirtManager: https://help.ubuntu.com/community/KVM/VirtManager
.. _Ceph Authentication: ../../rados/configuration/auth-config-ref
.. _Disks: http://www.libvirt.org/formatdomain.html#elementsDisks
.. _rbd create: ../rados-rbd-cmds#creating-a-block-device-image
.. _User Management - User: ../../rados/operations/user-management#user
.. _User Management - CLI: ../../rados/operations/user-management#command-line-usage
.. _Virtio: http://www.linux-kvm.org/page/Virtio
