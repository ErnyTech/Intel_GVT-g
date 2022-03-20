# How to setup Intel GVT-g with libvirt

Intel GVT-g is a mediated pass-through virtualization technology supported by Intel integrated GPUs starting from the Broadwell generation, thanks to virtualization it will be possible to obtain performances close to a native Intel GPU.

The following guide explains how to configure Intel GVT-g to use it with libvirt, the guide has been designed for GNU/Linux distributions based on ArchLinux.

<p align="center">
  <img src="https://i.imgur.com/scmiKWR.png" />
</p>

### Intel GVT-g setup
- Enable Intel GVT-g

  To enable Intel GVT-g you need to add the following parameters to the kernel cmdline:
  ```
  intel_iommu=on i915.enable_gvt=1 i915.enable_guc=0 kvm.ignore_msrs=1
  ```
  
  ```intel_iommu i915.enable_gvt``` enable Intel GVT-g while ```i915.enable_guc kvm.ignore_msrs``` are absolutely necessary to avoid issues when using vGPUs.
  
  In addition to the cmdline parameters it is also necessary to enable the loading of some modules from systemd according to the following:
  ```
  # File: /etc/modules-load.d/gvt.conf
 
  kvmgt
  vfio-iommu-type1
  mdev
  vfio_pci
  ```
- Check if Intel GVT-g is working

  After rebooting you should find a PCI device in ```/sys/bus/pci/drivers/i915```:
  ```
  $ ls /sys/bus/pci/drivers/i915
  0000:00:02.0  bind  module  new_id  remove_id  uevent  unbind
  ```
  ```
  $ ls /sys/bus/pci/drivers/i915/0000:00:02.0
  ari_enabled                current_link_speed  drm            i2c-2        link                  msi_bus      reset         revision
  boot_vga                   current_link_width  enable         i2c-3        local_cpulist         msi_irqs     reset_method  rom
  broken_parity_status       d3cold_allowed      firmware_node  index        local_cpus            numa_node    resource      subsystem
  class                      device              graphics       iommu        max_link_speed        power        resource0     subsystem_device
  config                     dma_mask_bits       gvt_firmware   iommu_group  max_link_width        power_state  resource2     subsystem_vendor
  consistent_dma_mask_bits   driver              i2c-0          irq          mdev_supported_types  remove       resource2_wc  uevent
  consumer:pci:0000:00:1f.3  driver_override     i2c-1          label        modalias              rescan       resource4     vendor
  ```
  In most devices the PCI device address should be ```0000:00:02.0``` but it may change so you need to check it.
  
  Inside the PCI device you should find (If Intel GVT-g is working) ```mdev_supported_types``` which will contain all types of vGPUs your hardware can create:
  ```
  $ ls /sys/bus/pci/drivers/i915/0000:00:02.0/mdev_supported_types
  i915-GVTg_V5_4  i915-GVTg_V5_8
  ```
- Create the Qemu hook to setup the vGPU when the VM starts.
  ```
  # File: /etc/libvirt/hooks/qemu
  
  #!/bin/sh
  GVT_DRIVER=i915
  GVT_PCI=0000:00:02.0

  function create_gpu { 
      local GVT_GUID="$1"
      local MDEV_TYPE="$2"

      pkexec sh -c "echo $GVT_GUID > /sys/bus/pci/drivers/$GVT_DRIVER/$GVT_PCI/mdev_supported_types/$MDEV_TYPE/create"
  }
 
  function remove_gpu { 
      local GVT_GUID="$1"

      pkexec sh -c "echo 1 > /sys/bus/pci/drivers/$GVT_DRIVER/$GVT_PCI/$GVT_GUID/remove"
  }

  function setup_gpu {
      local VM_DOMAIN="$1"
      local GVT_GUID="$2"
      local MDEV_TYPE="$3"

      if [ $# -ge 6 ]; then
          if [ "$4" = "$VM_DOMAIN" ] && [ "$5" = "prepare" ] && [ "$6" = "begin" ]; then
              create_gpu "$GVT_GUID" "$MDEV_TYPE"
          elif [ "$4" = "$VM_DOMAIN" ] && [ "$5" = "release" ] && [ "$6" = "end" ]; then
              remove_gpu "$GVT_GUID"
          fi
      fi
  }
  
  # Example
  # setup_gpu win10 ab54baba-ea9a-4ce5-b7b4-51b376b65849 i915-GVTg_V5_4 "$@"
  # This will create a vGPU for the <win10> VM which will have uuid <ab54baba-ea9a-4ce5-b7b4-51b376b65849> and vGPU type <i915-GVTg_V5_4>
  ```
  Check if ```GVT_PCI``` is equal to the PCI address obtained in the previous step, otherwise you have to copy the address in the hook and replace it with the one present by default.
  
### Setup VM to use Intel GVT-g
- Add VM to Qemu hook

  You need to generate a uuid that will represent the vGPU dedicated to your VM:
  ```
  $ uuidgen
  ab54baba-ea9a-4ce5-b7b4-51b376b65849
  ```
  
  Now it is necessary to configure the Qemu hook to create the vGPU when the VM starts, for example if my VM is called ```win10``` and I want to use profile ```i915-GVTg_V5_4``` using the uuid previously generated I will have to add the following line to the end of the hook file:
  ```
  # End of File: /etc/libvirt/hooks/qemu
  
  setup_gpu win10 ab54baba-ea9a-4ce5-b7b4-51b376b65849 i915-GVTg_V5_4 "$@"
  ```
- Remove the GPU previously used by libvirt

  From the libvirt VM xml file you need to remove the following tags:
  ```
  <graphics .. some value ..>
    ..
    Another tags
    ..
  </graphics>
  ```
  ```
  <video>
    ..
    Another tags
    ..
  </video>
  ```
- Add the Intel GVT-g vGPU to libvirt

  To add the vGPU to libvirt you need to add the device as mdev using the uuid generated before, just add the following code between the ```devices``` tag:
  ```
  <hostdev mode="subsystem" type="mdev" managed="no" model="vfio-pci" display="off">
    <source>
      <address uuid="your-gpu-uuid-here"/>
    </source>
  </hostdev>
  ```
- Configure the vGPU output

  To get maximum vGPU performance you need to use Qemu's default display and not Spice as is typically used with libvirt, to do this you need to do the following:
  - Replace the line:
    ```
    <domain type='kvm'>
    ```
    with:
    ```
    <domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
  - Add the following lines in the ```domain``` tag
    ```
    <qemu:commandline>
      <qemu:arg value="-set"/>
      <qemu:arg value="device.hostdev0.x-igd-opregion=on"/>
      <qemu:arg value="-set"/>
      <qemu:arg value="device.hostdev0.ramfb=on"/>
      <qemu:arg value="-set"/>
      <qemu:arg value="device.hostdev0.driver=vfio-pci-nohotplug"/>
      <qemu:arg value="-set"/>
      <qemu:arg value="device.hostdev0.display=on"/>
      <qemu:arg value="-display"/>
      <qemu:arg value="gtk,gl=on"/>
      <qemu:env name="DISPLAY" value=":0"/>
      <qemu:env name="GDK_SCALE" value="1.0"/>
    </qemu:commandline>
    ```
    Pay attention to ```<qemu:env name="DISPLAY" value=":0"/>```, the value must contain the value of the environment variable ```$DISPLAY```:
    ```
    $ echo $DISPLAY
    :0
    ```
- Start the VM
    
  The VM should now work using the vGPU via Intel GVT-g!
    
  Be aware that some operating systems may cause problems if you have already booted the operating system for example using QXL or Virtio, in this case a complete reinstallation of the guest operating system can help.
    
  If you are using the Qemu system session you will probably need to execute the following command before the VM starts to allow Qemu to create the video output window:
  ```
  $ xhost +
  ```
  Without this command the VM startup will fail due to the lack of permissions.
  
### Appendix: vGPU Types
- i915-GVTg_V5_1:

  ```
  low_gm_size: 512MB
  high_gm_size: 2048MB
  fence: 4
  resolution: 1920x1200
  weight: 1
  ```
- i915-GVTg_V5_2:

  ```
  low_gm_size: 256MB
  high_gm_size: 1024MB
  fence: 4
  resolution: 1920x1200
  weight: 2
  ```
- i915-GVTg_V5_4:

  ```
  low_gm_size: 128MB
  high_gm_size: 512MB
  fence: 4
  resolution: 1920x1200
  weight: 4
  ```
- i915-GVTg_V5_8:

  ```
  low_gm_size: 64MB
  high_gm_size: 384MB
  fence: 4
  resolution: 1024x768
  weight: 8
  ```
  
  Note: Not all GPUs supported all types of vGPU described, generally ```i915-GVTg_V5_4``` and ```i915-GVTg_V5_8``` are supported by most GPUs for others it depends.
