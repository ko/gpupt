
# gpupt

Gpupt  Roll
==================

.. contents::

Introduction
---------------
This roll enables GPU pass-through to a guest VM. 
The roll installs rocks commands for GPU management a frontend and vm-container nodes. 

This roll assumes that GPU cards are NVIDIA cards that have Video/Audio function for each card
which are designated by a function 0/1 respectively on the CPI bus.  For example, :: 

    0000:02:00.0 3D controller: NVIDIA Corporation GF100GL [Tesla T20 Processor] (rev a3)
    0000:02:00.1 Audio device: NVIDIA Corporation GF100 High Definition Audio Controller (rev a1)

Assumptions
-------------
#. This roll assumes that GPU cards for each physical host are given logical names with the prefix ``gpupci``
   The names are unique on a single host and the sequence start with ``gpupci1``. The name is not 
   unique across different hosts. 
#. A single GPU card will be assigne to each VM host. The logical name for a card on a VM is always ``gpupci``
   without indexes.  See example belwo for usage



Building
-------------

To build the roll, execute : ::

    # make roll

A successful build will create  ``gpupt-*.x86_64*.iso`` file.

Installing
------------

To add this roll to existing cluster, execute these instructions on a Rocks frontend: ::

    # rocks add roll gpupt-*.x86_64.disk1.iso
    # rocks enable roll gpupt
    # (cd /export/rocks/install; rocks create distro)
    # rocks run roll gpupt > add-roll.sh
    # bash add-roll.sh

On the vm-container nodes (only GPU-enabled): :: 

    # yum clean all 
    # yum install rocks-command-gpupt

What is installed:
-------------------

#. The following commands are enabled with the gpupt roll: :: 

     rocks add host gpu ...
     rocks dump host gpu ...
     rocks list host gpu ...
     rocks remove host gpu ...
     rocks report host gpu ...
     rocks set host gpu ...

#. A plugin ``plugin_device.py`` to manage guest VM GPU pass-through PCI addressing.
   Used in rocks command: ::
   
     rocks report host vm config

#. A command ``gpupci`` to maange GPU cards PCI addressing (list, detach, attach).
   This command is executed on the GPU-enabled hosts to get information (list) or to make GPU card
   available/non-available on the physical host CPI bus.  For more info use ``gpupci -h``


Using:
---------

The commands to detach G{PU card form ta physical host are run once for each GPU card. 

#. Run ``gpupci -l`` command on all GPU-enabled hosts to get ingormation about the GPU cards. 
   For example,  on vm-container-0-15  :: 

     # gpupci -l
     gpupci1 pci_0000_02_00_0
     gpupci2 pci_0000_03_00_0
   
   The output means there are 2 GPU cards and for each there is 
   a logincal GPU name and its PCI bus info.

#. Run commands to add this information in the rocks database: ::

    # rocks add host gpu vm-container-0-15 gpupci1 pci_0000_02_00_0
    # rocks add host gpu vm-container-0-15 gpupci2 pci_0000_03_00_0

#. Verify that  GPU info now is in the database: :: 

    # rocks list host gpu
    HOST               GPU     PCI_BUS         
    vm-container-0-15: gpupci1 pci_0000_02_00_0
    vm-container-0-15: gpupci2 pci_0000_03_00_0

#. Detach the GPU cards from the physical host. This needs to be done once  for each GPU card 
   before any VM can use the GPU PCI in pass-through mode. This can be done as a single command
   for all cards :: 

    # rocks run host vm-container-0-15 "gpupci --detach all"

   or using a specific logical name for a single GPU card  on a given host :: 

    # rocks run host vm-container-0-2 "gpupci --detach gpupci1"

Once the GPU card is detached from a physical host it is ready for use by the guest VM. 
We assume taht a singl GPU card is assigned to a VM and that a VM is run on a GPU-enabled vm-container. 
For example, if there is a VM rocks-33 that is created and running on a vm-container-0-15 and we want  
to assign a GPU  to it: ::  

   rocks stop host VM rocks-33
   rocks add host gpu rocks-33 gpupci pci_0000_02_00_0
   rocks report host vm config rocks-33

The first command stops VM, the ``add`` command adds a GPU attribute to the VM in the rocks database.   
The ``report`` command verifies that the xml file that describes the VM configuration has device information
for the GPU card. For this example, the output would contain  at the end of devices seciton: :: 

    ...
      <hostdev mode='subsystem' type='pci' managed='yes'>
        <source>
          <address domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
        </source>
      </hostdev>
    </devices>

At the next start of the VM  the  GPU card  will be available to the VM. 
On the VM the PCI bus address will be different form that of the physical host. 