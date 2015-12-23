==============
VMware vSphere
==============

Introduction
~~~~~~~~~~~~

OpenStack Compute supports the VMware vSphere product family and enables
access to advanced features such as vMotion, High Availability, and
Dynamic Resource Scheduling (DRS).

This section describes how to configure VMware-based virtual machine
images for launch. vSphere versions 4.1 and later are supported.

The VMware vCenter driver enables the ``nova-compute`` service to communicate
with a VMware vCenter server that manages one or more ESX host clusters.
The driver aggregates the ESX hosts in each cluster to present one
large hypervisor entity for each cluster to the Compute scheduler.
Because individual ESX hosts are not exposed to the scheduler, Compute
schedules to the granularity of clusters and vCenter uses DRS to select
the actual ESX host within the cluster. When a virtual machine makes
its way into a vCenter cluster, it can use all vSphere features.

The following sections describe how to configure the VMware vCenter driver.

High-level architecture
~~~~~~~~~~~~~~~~~~~~~~~

The following diagram shows a high-level view of the VMware driver
architecture:

**VMware driver architecture**

.. figure:: ../../../common/figures/vmware-nova-driver-architecture.jpg
   :width: 100%

As the figure shows, the OpenStack Compute Scheduler sees
three hypervisors that each correspond to a cluster in vCenter.
``nova-compute`` contains the VMware driver. You can run with multiple
``nova-compute`` services. While Compute schedules at the granularity
of a cluster, the VMware driver inside ``nova-compute`` interacts with
the vCenter APIs to select an appropriate ESX host within the cluster.
Internally, vCenter uses DRS for placement.

The VMware vCenter driver also interacts with the Image service to copy
VMDK images from the Image service back-end store.
The dotted line in the figure represents VMDK images being copied from
the OpenStack Image service to the vSphere data store.
VMDK images are cached in the data store so the copy operation is only
required the first time that the VMDK image is used.

After OpenStack boots a VM into a vSphere cluster, the VM becomes visible
in vCenter and can access vSphere advanced features. At the same time,
the VM is visible in the OpenStack dashboard and you can manage it as you
would any other OpenStack VM. You can perform advanced vSphere operations
in vCenter while you configure OpenStack resources such as VMs through the
OpenStack dashboard.

The figure does not show how networking fits into the architecture.
Both ``nova-network`` and the OpenStack Networking Service are supported.
For details, see :ref:`vmware-networking`.

Configuration overview
~~~~~~~~~~~~~~~~~~~~~~

To get started with the VMware vCenter driver, complete the following
high-level steps:

#. Configure vCenter. See :ref:`vmware-prereqs`.
#. Configure the VMware vCenter driver in the ``nova.conf`` file.
   See :ref:`vmware-vcdriver`.
#. Load desired VMDK images into the Image Service. See :ref:`vmware-images`.
#. Configure networking with either ``nova-network`` or
   the Networking service. See :ref:`vmware-networking`.

.. _vmware-prereqs:

Prerequisites and limitations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the following list to prepare a vSphere environment that runs with
the VMware vCenter driver:

Copying VMDK files (vSphere 5.1 only)
  In vSphere 5.1, copying large image files (for example, 12 GB and
  greater) from the Image service can take a long time.
  To improve performance, VMware recommends that you upgrade to VMware
  vCenter Server 5.1 Update 1 or later. For more information,
  see the `Release Notes <https://www.vmware.com/support/vsphere5/doc/
  vsphere-vcenter-server-51u1-release-notes.html#resolvedissuescimapi>`_.

DRS
  For any cluster that contains multiple ESX hosts, enable DRS and enable
  fully automated placement.

Shared storage
  Only shared storage is supported and data stores must be shared among
  all hosts in a cluster. It is recommended to remove data stores not
  intended for OpenStack from clusters being configured for OpenStack.

Clusters and data stores
  Do not use OpenStack clusters and data stores for other purposes.
  If you do, OpenStack displays incorrect usage information.

Networking
  The networking configuration depends on the desired networking model.
  See :ref:`vmware-networking`.

Security groups
  If you use the VMware driver with OpenStack Networking and the NSX
  plug-in, security groups are supported. If you use ``nova-network``,
  security groups are not supported.

  .. note::

     The NSX plug-in is the only plug-in that is validated for vSphere.

VNC
  The port range 5900 - 6105 (inclusive) is automatically enabled for VNC
  connections on every ESX host in all clusters under OpenStack control.
  For more information about using a VNC client to connect to virtual machine,
  see http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&amp;cmd=displayKC&amp;externalId=1246.

  .. note::

     In addition to the default VNC port numbers (5900 to 6000) specified
     in the above document, the following ports are also used:
     6101, 6102, and 6105.

  You must modify the ESXi firewall configuration to allow the VNC ports.
  Additionally, for the firewall modifications to persist after a reboot,
  you must create a custom vSphere Installation Bundle (VIB) which is then
  installed onto the running ESXi host or added to a custom image profile
  used to install ESXi hosts. For details about how to create a VIB
  for persisting the firewall configuration modifications, see
  http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&amp;cmd=displayKC&amp;externalId=2007381.

  .. note::

     The VIB can be downloaded from
     https://github.com/openstack-vmwareapi-team/Tools.

To use multiple vCenter installations with OpenStack, each vCenter
must be assigned to a separate availability zone. This is required
as the OpenStack Block Storage VMDK driver does not currently work
across multiple vCenter installations.

VMware vCenter service account
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenStack integration requires a vCenter service account with the
following minimum permissions. Apply the permissions to the ``Datacenter``
root object, and select the :guilabel:`Propagate to Child Objects` option.

.. list-table:: vCenter permissions tree
   :header-rows: 1
   :widths: 12, 12, 40, 36

   * - All Privileges
     -
     -
     -
   * -
     - Datastore
     -
     -
   * -
     -
     - Allocate space
     -
   * -
     -
     - Browse datastore
     -
   * -
     -
     - Low level file operation
     -
   * -
     -
     - Remove file
     -
   * -
     - Extension
     -
     -
   * -
     -
     - Register extension
     -
   * -
     - Folder
     -
     -
   * -
     -
     - Create folder
     -
   * -
     - Host
     -
     -
   * -
     -
     - Configuration
     -
   * -
     -
     -
     - Maintenance
   * -
     -
     -
     - Network configuration
   * -
     -
     -
     - Storage partition configuration
   * -
     - Network
     -
     -
   * -
     -
     - Assign network
     -
   * -
     - Resource
     -
     -
   * -
     -
     - Assign virtual machine to resource pool
     -
   * -
     -
     - Migrate powered off virtual machine
     -
   * -
     -
     - Migrate powered on virtual machine
     -
   * -
     - Virtual Machine
     -
     -
   * -
     -
     - Configuration
     -
   * -
     -
     -
     - Add existing disk
   * -
     -
     -
     - Add new disk
   * -
     -
     -
     - Add or remove device
   * -
     -
     -
     - Advanced
   * -
     -
     -
     - CPU count
   * -
     -
     -
     - Disk change tracking
   * -
     -
     -
     - Host USB device
   * -
     -
     -
     - Memory
   * -
     -
     -
     - Raw device
   * -
     -
     -
     - Remove disk
   * -
     -
     -
     - Rename
   * -
     -
     -
     - Swapfile placement
   * -
     -
     - Interaction
     -
   * -
     -
     -
     - Configure CD media
   * -
     -
     -
     - Power Off
   * -
     -
     -
     - Power On
   * -
     -
     -
     - Reset
   * -
     -
     -
     - Suspend
   * -
     -
     - Inventory
     -
   * -
     -
     -
     - Create from existing
   * -
     -
     -
     - Create new
   * -
     -
     -
     - Move
   * -
     -
     -
     - Remove
   * -
     -
     -
     - Unregister
   * -
     -
     - Provisioning
     -
   * -
     -
     -
     - Clone virtual machine
   * -
     -
     -
     - Customize
   * -
     -
     - Sessions
     -
   * -
     -
     -
     - Validate session
   * -
     -
     -
     - View and stop sessions
   * -
     -
     - Snapshot management
     -
   * -
     -
     -
     - Create snapshot
   * -
     -
     -
     - Remove snapshot
   * -
     - vApp
     -
     -
   * -
     -
     - Export
     -
   * -
     -
     - Import
     -

.. _vmware-vcdriver:

VMware vCenter driver
~~~~~~~~~~~~~~~~~~~~~

Use the VMware vCenter driver (VMwareVCDriver) to connect
OpenStack Compute with vCenter. This recommended configuration
enables access through vCenter to advanced vSphere features like
vMotion, High Availability, and Dynamic Resource Scheduling (DRS).

VMwareVCDriver configuration options
------------------------------------

When you use the VMwareVCDriver (vCenter versions 5.1 and later) with
OpenStack Compute, add the following VMware-specific configuration
options to the ``nova.conf`` file:

.. code-block:: ini

   [DEFAULT]
   compute_driver=vmwareapi.VMwareVCDriver

   [vmware]
   host_ip=<vCenter host IP>
   host_username=<vCenter username>
   host_password=<vCenter password>
   cluster_name=<vCenter cluster name>
   datastore_regex=<optional datastore regex>

.. note::

   * vSphere vCenter versions 5.0 and earlier: You must specify the
     location of the WSDL files by adding the
     ``wsdl_location=http://127.0.0.1:8080/vmware/SDK/wsdl/vim25/vimService.wsdl``
     setting to the above configuration. For more information, see
     :ref:`vSphere 5.0 and earlier additional set up <vmware-additional>`.

   * Clusters: The vCenter driver can support multiple clusters.
     To use more than one cluster, simply add multiple ``cluster_name`` lines
     in ``nova.conf`` with the appropriate cluster name.
     Clusters and data stores used by the vCenter driver should not contain
     any VMs other than those created by the driver.

   * Data stores: The ``datastore_regex`` setting specifies the data stores
     to use with Compute.  For example, ``datastore_regex="nas.*"``
     selects all the data stores that have a name starting with "nas".
     If this line is omitted, Compute uses the first data store returned by
     the vSphere API. It is recommended not to use this field and instead
     remove data stores that are not intended for OpenStack.

   * Reserved host memory: The ``reserved_host_memory_mb`` option value is
     512 MB by default. However, VMware recommends that you set this option
     to 0 MB because the vCenter driver reports the effective memory
     available to the virtual machines.

   * The vCenter driver generates instance name by instance ID.
     Instance name template is ignored.

   * The minimum supported vCenter version is 5.1.0.
     In OpenStack Liberty release this will be logged as a warning.
     In OpenStack Mitaka release this will be enforced.

A ``nova-compute`` service can control one or more clusters containing
multiple ESXi hosts, making ``nova-compute`` a critical service from a
high availability perspective. Because the host that runs ``nova-compute``
can fail while the vCenter and ESX still run, you must protect the
``nova-compute`` service against host failures.

.. note::

   Many ``nova.conf`` options are relevant to libvirt but do not apply
   to this driver.

You must complete additional configuration for environments that use
vSphere 5.0 and earlier. See :ref:`vmware-additional`.

.. _vmware-images:

Images with VMware vSphere
~~~~~~~~~~~~~~~~~~~~~~~~~~

The vCenter driver supports images in the VMDK format. Disks in this
format can be obtained from VMware Fusion or from an ESX environment.
It is also possible to convert other formats, such as qcow2, to the VMDK
format using the ``qemu-img`` utility. After a VMDK disk is available,
load it into the Image service. Then, you can use it with the VMware
vCenter driver. The following sections provide additional details on the
supported disks and the commands used for conversion and upload.

Supported image types
---------------------

Upload images to the OpenStack Image service in VMDK format.
The following VMDK disk types are supported:

* ``VMFS Flat Disks`` (includes thin, thick, zeroedthick, and
  eagerzeroedthick). Note that once a VMFS thin disk is exported from VMFS
  to a non-VMFS location, like the OpenStack Image service, it becomes a
  preallocated flat disk. This impacts the transfer time from the Image
  service to the data store when the full preallocated flat disk,
  rather than the thin disk, must be transferred.

* ``Monolithic Sparse disks``. Sparse disks get imported from the Image
  service into ESXi as thin provisioned disks. Monolithic Sparse disks
  can be obtained from VMware Fusion or can be created by converting from
  other virtual disk formats using the ``qemu-img`` utility.

The following table shows the ``vmware_disktype`` property that applies
to each of the supported VMDK disk types:

.. list-table:: OpenStack Image service disk type settings
   :header-rows: 1

   * - vmware_disktype property
     - VMDK disk type
   * - sparse
     - Monolithic Sparse
   * - thin
     - VMFS flat, thin provisioned
   * - preallocated (default)
     - VMFS flat, thick/zeroedthick/eagerzeroedthick

The ``vmware_disktype`` property is set when an image is loaded into the
Image service. For example, the following command creates a Monolithic
Sparse image by setting ``vmware_disktype`` to ``sparse``:

.. code-block:: console

   $ glance image-create --name "ubuntu-sparse" --disk-format vmdk \
     --container-format bare \
     --property vmware_disktype="sparse" \
     --property vmware_ostype="ubuntu64Guest" < ubuntuLTS-sparse.vmdk

.. note::

   Specifying ``thin`` does not provide any advantage over ``preallocated``
   with the current version of the driver. Future versions might restore
   the thin properties of the disk after it is downloaded to a vSphere
   data store.

Convert and load images
-----------------------

Using the ``qemu-img`` utility, disk images in several formats (such as,
qcow2) can be converted to the VMDK format.

For example, the following command can be used to convert a
`qcow2 Ubuntu Trusty cloud image <http://cloud-images.ubuntu.com/trusty/
current/trusty-server-cloudimg-amd64-disk1.img>`_:

.. code-block:: console

   $ qemu-img convert -f qcow2 ~/Downloads/trusty-server-cloudimg-amd64-disk1.img \
     -O vmdk trusty-server-cloudimg-amd64-disk1.vmdk

VMDK disks converted through ``qemu-img`` are ``always`` monolithic sparse
VMDK disks with an IDE adapter type. Using the previous example of the
Ubuntu Trusty image after the ``qemu-img`` conversion, the command to
upload the VMDK disk should be something like:

.. code-block:: console

   $ glance image-create --name trusty-cloud \
     --container-format bare --disk-format vmdk \
     --property vmware_disktype="sparse" \
     --property vmware_adaptertype="ide" < \
     trusty-server-cloudimg-amd64-disk1.vmdk

Note that the ``vmware_disktype`` is set to ``sparse`` and the
``vmware_adaptertype`` is set to ``ide`` in the previous command.

If the image did not come from the ``qemu-img`` utility, the
``vmware_disktype`` and ``vmware_adaptertype`` might be different.
To determine the image adapter type from an image file, use the
following command and look for the ``ddb.adapterType=`` line:

.. code-block:: console

   $ head -20 <vmdk file name>

Assuming a preallocated disk type and an iSCSI lsiLogic adapter type,
the following command uploads the VMDK disk:

.. code-block:: console

   $ glance image-create --name "ubuntu-thick-scsi" --disk-format vmdk \
     --container-format bare \
     --property vmware_adaptertype="lsiLogic" \
     --property vmware_disktype="preallocated" \
     --property vmware_ostype="ubuntu64Guest" < ubuntuLTS-flat.vmdk

Currently, OS boot VMDK disks with an IDE adapter type cannot be attached
to a virtual SCSI controller and likewise disks with one of the SCSI
adapter types (such as, busLogic, lsiLogic, lsiLogicsas, paraVirtual)
cannot be attached to the IDE controller. Therefore, as the previous
examples show, it is important to set the ``vmware_adaptertype`` property
correctly. The default adapter type is lsiLogic, which is SCSI, so you can
omit the ``vmware_adaptertype`` property if you are certain that the image
adapter type is lsiLogic.

Tag VMware images
-----------------

In a mixed hypervisor environment, OpenStack Compute uses the
``hypervisor_type`` tag to match images to the correct hypervisor type.
For VMware images, set the hypervisor type to ``vmware``.
Other valid hypervisor types include:
``hyperv``, ``ironic``, ``lxc``, ``qemu``, ``uml``, and ``xen``.
Note that ``qemu`` is used for both QEMU and KVM hypervisor types.

.. code-block:: console

   $ glance image-create --name "ubuntu-thick-scsi" --disk-format vmdk \
     --container-format bare \
     --property vmware_adaptertype="lsiLogic" \
     --property vmware_disktype="preallocated" \
     --property hypervisor_type="vmware" \
     --property vmware_ostype="ubuntu64Guest" < ubuntuLTS-flat.vmdk

Optimize images
---------------

Monolithic Sparse disks are considerably faster to download but have the
overhead of an additional conversion step. When imported into ESX, sparse
disks get converted to VMFS flat thin provisioned disks. The download and
conversion steps only affect the first launched instance that uses the
sparse disk image. The converted disk image is cached, so subsequent
instances that use this disk image can simply use the cached version.

To avoid the conversion step (at the cost of longer download times)
consider converting sparse disks to thin provisioned or preallocated disks
before loading them into the Image service.

Use one of the following tools to pre-convert sparse disks.

vSphere CLI tools
  Sometimes called the remote CLI or rCLI.

  Assuming that the sparse disk is made available on a data store accessible
  by an ESX host, the following command converts it to preallocated format:

  .. code-block:: console

     vmkfstools --server=ip_of_some_ESX_host -i \
       /vmfs/volumes/datastore1/sparse.vmdk \
       /vmfs/volumes/datastore1/converted.vmdk

  Note that the vifs tool from the same CLI package can be used to upload
  the disk to be converted. The vifs tool can also be used to download
  the converted disk if necessary.

vmkfstools directly on the ESX host
  If the SSH service is enabled on an ESX host, the sparse disk can be
  uploaded to the ESX data store through scp and the vmkfstools local
  to the ESX host can use used to perform the conversion.
  After you log in to the host through ssh, run this command:

  .. code-block:: console

     vmkfstools -i /vmfs/volumes/datastore1/sparse.vmdk /vmfs/volumes/datastore1/converted.vmdk

vmware-vdiskmanager
  ``vmware-vdiskmanager`` is a utility that comes bundled with VMware
  Fusion and VMware Workstation. The following example converts a sparse
  disk to preallocated format:

  .. code-block:: console

     '/Applications/VMware Fusion.app/Contents/Library/vmware-vdiskmanager' -r sparse.vmdk -t 4 converted.vmdk

In the previous cases, the converted vmdk is actually a pair of files:

* The descriptor file ``converted.vmdk``.
* The actual virtual disk data file ``converted-flat.vmdk``.

The file to be uploaded to the Image Service is ``converted-flat.vmdk``.

Image handling
--------------

The ESX hypervisor requires a copy of the VMDK file in order to boot up a
virtual machine. As a result, the vCenter OpenStack Compute driver must
download the VMDK via HTTP from the Image service to a data store that is
visible to the hypervisor. To optimize this process, the first time a
VMDK file is used, it gets cached in the data store.
A cached image is stored in a folder named after the image ID.
Subsequent virtual machines that need the VMDK use the cached version and
don't have to copy the file again from the Image service.

Even with a cached VMDK, there is still a copy operation from the cache
location to the hypervisor file directory in the shared data store.
To avoid this copy, boot the image in linked_clone mode. To learn how to
enable this mode, see :ref:`vmware-config`.

.. note::

   You can also use the ``vmware_linked_clone`` property in the Image
   service to override the linked_clone mode on a per-image basis.

   If spawning a virtual machine image from ISO with a VMDK disk,
   the image is created and attached to the virtual machine as a blank disk.
   In that case ``vmware_linked_clone`` property for the image is just ignored.

If multiple compute nodes are running on the same host, or have a shared
file system, you can enable them to use the same cache folder on the back-end
data store. To configure this action, set the ``cache_prefix`` option in the
``nova.conf`` file. Its value stands for the name prefix of the folder where
cached images are stored.

.. note::

   This can take effect only if compute nodes are running on the same host,
   or have a shared file system.

You can automatically purge unused images after a specified period of time.
To configure this action, set these options in the ``DEFAULT`` section in
the ``nova.conf`` file:

remove_unused_base_images
  Set this option to ``True`` to specify that unused images should
  be removed after the duration specified in the
  ``remove_unused_original_minimum_age_seconds`` option.
  The default is ``True``.

remove_unused_original_minimum_age_seconds
  Specifies the duration in seconds after which an unused image is
  purged from the cache. The default is ``86400`` (24 hours).

.. _vmware-networking:

Networking with VMware vSphere
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The VMware driver supports networking with the ``nova-network`` service
or the Networking Service. Depending on your installation,
complete these configuration steps before you provision VMs:

#. **The nova-network service with the FlatManager or FlatDHCPManager**.
   Create a port group with the same name as the ``flat_network_bridge``
   value in the ``nova.conf`` file. The default value is ``br100``.
   If you specify another value, the new value must be a valid Linux bridge
   identifier that adheres to Linux bridge naming conventions.

   All VM NICs are attached to this port group.

   Ensure that the flat interface of the node that runs the ``nova-network``
   service has a path to this network.

   .. note::

      When configuring the port binding for this port group in vCenter,
      specify ``ephemeral`` for the port binding type. For more information,
      see `Choosing a port binding type in ESX/ESXi <http://kb.vmware.com/
      selfservice/microsites/search.do?language=en_US&amp;cmd=displayKC
      &amp;externalId=1022312>`_ in the VMware Knowledge Base.

#. **The nova-network service with the VlanManager**.
   Set the ``vlan_interface`` configuration option to match the ESX host
   interface that handles VLAN-tagged VM traffic.

   OpenStack Compute automatically creates the corresponding port groups.

#. If you are using the OpenStack Networking Service:
   Before provisioning VMs, create a port group with the same name as the
   ``vmware.integration_bridge`` value in ``nova.conf`` (default is
   ``br-int``). All VM NICs are attached to this port group for management
   by the OpenStack Networking plug-in.

Volumes with VMware vSphere
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The VMware driver supports attaching volumes from the Block Storage service.
The VMware VMDK driver for OpenStack Block Storage is recommended and should be
used for managing volumes based on vSphere data stores. For more information
about the VMware VMDK driver, see :ref:`block_storage_vmdk_driver`.  Also an
iSCSI volume driver provides limited support and can be used only for
attachments.

.. _vmware-additional:

vSphere 5.0 and earlier additional set up
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Users of vSphere 5.0 or earlier must host their WSDL files locally.
These steps are applicable for vCenter 5.0 or ESXi 5.0 and you can either
mirror the WSDL from the vCenter or ESXi server that you intend to use or
you can download the SDK directly from VMware. These workaround steps fix
a `known issue <http://kb.vmware.com/selfservice/microsites/search.do?
cmd=displayKC&amp;externalId=2010507>`_ with the WSDL that was resolved
in later versions.

When setting the VMwareVCDriver configuration options, you must include the
``wsdl_location`` option. For more information, see :ref:`vmware-vcdriver`.

**To mirror WSDL from vCenter (or ESXi)**

#. Set the ``VMWAREAPI_IP`` shell variable to the IP address for your
   vCenter or ESXi host from where you plan to mirror files. For example:

   .. code-block:: console

      $ export VMWAREAPI_IP=<your_vsphere_host_ip>

#. Create a local file system directory to hold the WSDL files:

   .. code-block:: console

      $ mkdir -p /opt/stack/vmware/wsdl/5.0

#. Change into the new directory.

   .. code-block:: console

      $ cd /opt/stack/vmware/wsdl/5.0

#. Use your OS-specific tools to install a command-line tool that can
   download files like :command:`wget`.

#. Download the files to the local file cache:

   .. code-block:: console

      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/vimService.wsdl
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/vim.wsdl
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/core-types.xsd
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/query-messagetypes.xsd
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/query-types.xsd
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/vim-messagetypes.xsd
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/vim-types.xsd
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/reflect-messagetypes.xsd
      $ wget  --no-check-certificate https://$VMWAREAPI_IP/sdk/reflect-types.xsd

#. Because the ``reflect-types.xsd`` and ``reflect-messagetypes.xsd`` files
   do not fetch properly, you must stub out these files. Use the following
   XML listing to replace the missing file content. The XML parser underneath
   Python can be very particular and if you put a space in the wrong place, it
   can break the parser. Copy the following contents and formatting carefully.

   .. code-block:: xml

      <xml version="1.0" encoding="UTF-8"?>
        <schema
          targetNamespace="urn:reflect"
          xmlns="http://www.w3.org/2001/XMLSchema"
          xmlns:xsd="http://www.w3.org/2001/XMLSchema"
          elementFormDefault="qualified">
        </schema>

#. Now that the files are locally present, tell the driver to look for the
   SOAP service WSDLs in the local file system and not on the remote
   vSphere server. Add the following setting to the ``nova.conf`` file
   for your ``nova-compute`` node:

   .. code-block:: ini

      [vmware]
      wsdl_location=file:///opt/stack/vmware/wsdl/5.0/vimService.wsdl

Alternatively, download the version appropriate SDK from
http://www.vmware.com/support/developer/vc-sdk/ and copy it to the
``/opt/stack/vmware`` file. Make sure that the WSDL is available, in for
example ``/opt/stack/vmware/SDK/wsdl/vim25/vimService.wsdl``.
You must point ``nova.conf`` to fetch this WSDL file from the local file
system by using a URL.

When using the VMwareVCDriver (vCenter) with OpenStack Compute with
vSphere version 5.0 or earlier, ``nova.conf`` must include the following
extra config option:

.. code-block:: ini

   [vmware]
   wsdl_location=file:///opt/stack/vmware/SDK/wsdl/vim25/vimService.wsdl

.. _vmware-config:

Configuration reference
~~~~~~~~~~~~~~~~~~~~~~~

To customize the VMware driver, use the configuration option settings
documented in :ref:`nova-vmware`.