---
layout: post
title:  "Redfish Driver"
date:   2019-11-15 17:30:00 +0800
categories: Openstack
tags: Openstack-Ironic
excerpt: Redfish Driver
mathjax: true
---

# Redfish Driver

昨天对Redfish做了最基本的了解，然后大概看了一下，我们集成的问题，主要问题在于，版本的不一致性。我们在newton，而redfish driver是从p版本开始支持的。而且，从o版本之后，ironic driver进行了一个比较大的变更，使得backport这个工作变得有些困难。

> Starting with the Ocata release, two types of drivers are supported: *classic drivers* (for example, `pxe_ipmitool`, `agent_ilo`, etc.) and the newer *hardware types* (for example, generic `redfish` and `ipmi` or vendor-specific `ilo` and `irmc`).
>
> Drivers, in turn, consist of *hardware interfaces*: sets of functionality dealing with some aspect of bare metal provisioning in a vendor-specific way. *Classic drivers* have all *hardware interfaces* hardcoded, while *hardware types* only declare which *hardware interfaces* they are compatible with.
>
> Please refer to the [driver composition reform specification](https://specs.openstack.org/openstack/ironic-specs/specs/approved/driver-composition-reform.html) for technical details behind *hardware types*.
>
> From API user’s point of view, both *classic drivers* and *hardware types* can be assigned to the `driver` field of a node. However, they are configured differently.

之前的drivers对于所有的hardware interface都是hardcoded的，而新的方式更加灵活，使得可以更灵活的定义每一种hardware interface。把这一整套代码backport回来比较困难，所以初步决定把redfish改造成一个driver，已嵌入现在的逻辑。

# Hardware interfaces

有好多种hardware interfaces

## boot

管理ramdisk以及user instance的deploy

```python
class BootInterface(BaseInterface):
    """Interface for boot-related actions."""
    interface_type = 'boot'
    capabilities = []

    @abc.abstractmethod
    def prepare_ramdisk(self, task, ramdisk_params):
        """Prepares the boot of Ironic ramdisk.

        This method prepares the boot of the deploy or rescue ramdisk after
        reading relevant information from the node's database.

        :param task: A task from TaskManager.
        :param ramdisk_params: The options to be passed to the ironic ramdisk.
            Different implementations might want to boot the ramdisk in
            different ways by passing parameters to them.  For example,

            When Agent ramdisk is booted to deploy a node, it takes the
            parameters ipa-api-url, etc.

            Other implementations can make use of ramdisk_params to pass such
            information.  Different implementations of boot interface will
            have different ways of passing parameters to the ramdisk.
        :returns: None
        """

    @abc.abstractmethod
    def clean_up_ramdisk(self, task):
        """Cleans up the boot of ironic ramdisk.

        This method cleans up the environment that was setup for booting the
        deploy or rescue ramdisk.

        :param task: A task from TaskManager.
        :returns: None
        """

    @abc.abstractmethod
    def prepare_instance(self, task):
        """Prepares the boot of instance.

        This method prepares the boot of the instance after reading
        relevant information from the node's database.

        :param task: A task from TaskManager.
        :returns: None
        """

    @abc.abstractmethod
    def clean_up_instance(self, task):
        """Cleans up the boot of instance.

        This method cleans up the environment that was setup for booting
        the instance.

        :param task: A task from TaskManager.
        :returns: None
        """

    def validate_rescue(self, task):
        """Validate that the node has required properties for rescue.

        :param task: A TaskManager instance with the node being checked
        :raises: MissingParameterValue if node is missing one or more required
            parameters
        :raises: UnsupportedDriverExtension
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='validate_rescue')
```

## console

 管理baremetal node的serial console

```python
class ConsoleInterface(BaseInterface):
    """Interface for console-related actions."""
    interface_type = "console"

    @abc.abstractmethod
    def start_console(self, task):
        """Start a remote console for the task's node.

        This method should not raise an exception if console already started.

        :param task: A TaskManager instance containing the node to act on.
        """

    @abc.abstractmethod
    def stop_console(self, task):
        """Stop the remote console session for the task's node.

        :param task: A TaskManager instance containing the node to act on.
        """

    @abc.abstractmethod
    def get_console(self, task):
        """Get connection information about the console.

        This method should return the necessary information for the
        client to access the console.

        :param task: A TaskManager instance containing the node to act on.
        :returns: the console connection information.
        """
```

## deploy

把image传输到目标disk上，有两种方式:

* iscsi deploy method：
  * 将根硬盘作为iSCSI设备暴露给ironic conductor，由conductor将镜像复制到这里target disk
* direct deploy method:
  * 从http url（比如swift等对象存储，或者是用户提供的http url）获取deploy image，通常是conductor准备这样一个url，ipa负责下载镜像完成部署

```python
class DeployInterface(BaseInterface):
    """Interface for deploy-related actions."""
    interface_type = 'deploy'

    @abc.abstractmethod
    def deploy(self, task):
        """Perform a deployment to the task's node.

        Perform the necessary work to deploy an image onto the specified node.
        This method will be called after prepare(), which may have already
        performed any preparatory steps, such as pre-caching some data for the
        node.

        :param task: A TaskManager instance containing the node to act on.
        :returns: status of the deploy. One of ironic.common.states.
        """

    @abc.abstractmethod
    def tear_down(self, task):
        """Tear down a previous deployment on the task's node.

        Given a node that has been previously deployed to,
        do all cleanup and tear down necessary to "un-deploy" that node.

        :param task: A TaskManager instance containing the node to act on.
        :returns: status of the deploy. One of ironic.common.states.
        """

    @abc.abstractmethod
    def prepare(self, task):
        """Prepare the deployment environment for the task's node.

        If preparation of the deployment environment ahead of time is possible,
        this method should be implemented by the driver.

        If implemented, this method must be idempotent. It may be called
        multiple times for the same node on the same conductor.

        This method is called before `deploy`.

        :param task: A TaskManager instance containing the node to act on.
        """

    @abc.abstractmethod
    def clean_up(self, task):
        """Clean up the deployment environment for the task's node.

        If preparation of the deployment environment ahead of time is possible,
        this method should be implemented by the driver. It should erase
        anything cached by the `prepare` method.

        If implemented, this method must be idempotent. It may be called
        multiple times for the same node on the same conductor, and it may be
        called by multiple conductors in parallel. Therefore, it must not
        require an exclusive lock.

        This method is called before `tear_down`.

        :param task: A TaskManager instance containing the node to act on.
        """

    @abc.abstractmethod
    def take_over(self, task):
        """Take over management of this task's node from a dead conductor.

        If conductors' hosts maintain a static relationship to nodes, this
        method should be implemented by the driver to allow conductors to
        perform the necessary work during the remapping of nodes to conductors
        when a conductor joins or leaves the cluster.

        For example, the PXE driver has an external dependency:
            Neutron must forward DHCP BOOT requests to a conductor which has
            prepared the tftpboot environment for the given node. When a
            conductor goes offline, another conductor must change this setting
            in Neutron as part of remapping that node's control to itself.
            This is performed within the `takeover` method.

        :param task: A TaskManager instance containing the node to act on.
        """

    def prepare_cleaning(self, task):
        """Prepare the node for cleaning tasks.

        For example, nodes that use the Ironic Python Agent will need to
        boot the ramdisk in order to do in-band cleaning tasks.

        If the function is asynchronous, the driver will need to handle
        settings node.driver_internal_info['clean_steps'] and node.clean_step,
        as they would be set in ironic.conductor.manager._do_node_clean,
        but cannot be set when this is asynchronous. After, the interface
        should make an RPC call to continue_node_cleaning to start cleaning.

        NOTE(JoshNang) this should be moved to BootInterface when it gets
        implemented.

        :param task: A TaskManager instance containing the node to act on.
        :returns: If this function is going to be asynchronous, should return
            `states.CLEANWAIT`. Otherwise, should return `None`. The interface
            will need to call _get_cleaning_steps and then RPC to
            continue_node_cleaning
        """
        pass

    def tear_down_cleaning(self, task):
        """Tear down after cleaning is completed.

        Given that cleaning is complete, do all cleanup and tear
        down necessary to allow the node to be deployed to again.

        NOTE(JoshNang) this should be moved to BootInterface when it gets
        implemented.

        :param task: A TaskManager instance containing the node to act on.
        """
        pass

    def heartbeat(self, task, callback_url, agent_version):
        """Record a heartbeat for the node.

        :param task: A TaskManager instance containing the node to act on.
        :param callback_url: a URL to use to call to the ramdisk.
        :param agent_version: The version of the agent that is heartbeating
        :return: None
        """
        LOG.warning('Got heartbeat message from node %(node)s, but '
                    'the driver %(driver)s does not support heartbeating',
                    {'node': task.node.uuid, 'driver': task.node.driver})
```

## inspect

inspect是用来发现node的，可以out-of-band（连接node的BMC），或者in-band（通过在node上boot ramdisk）

```python
class InspectInterface(BaseInterface):
    """Interface for inspection-related actions."""
    interface_type = 'inspect'

    ESSENTIAL_PROPERTIES = {'memory_mb', 'local_gb', 'cpus', 'cpu_arch'}
    """The properties required by scheduler/deploy."""

    @abc.abstractmethod
    def inspect_hardware(self, task):
        """Inspect hardware.

        Inspect hardware to obtain the essential & additional hardware
        properties.

        :param task: A task from TaskManager.
        :raises: HardwareInspectionFailure, if unable to get essential
                 hardware properties.
        :returns: Resulting state of the inspection i.e. states.MANAGEABLE
                  or None.
        """

    def abort(self, task):
        """Abort asynchronized hardware inspection.

        Abort an ongoing hardware introspection, this is only used for
        asynchronize based inspect interface.

        NOTE: This interface is called with node exclusive lock held, the
        interface implementation is expected to be a quick processing.

        :param task: a task from TaskManager.
        :raises: UnsupportedDriverExtension, if the method is not implemented
                 by specific inspect interface.
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='abort')
```

## management

提供additional硬件管理，比如设置或者获取boot devices

```python
class ManagementInterface(BaseInterface):
    """Interface for management related actions."""
    interface_type = 'management'

    @abc.abstractmethod
    def get_supported_boot_devices(self, task):
        """Get a list of the supported boot devices.

        :param task: A task from TaskManager.
        :returns: A list with the supported boot devices defined
                  in :mod:`ironic.common.boot_devices`.
        """

    @abc.abstractmethod
    def set_boot_device(self, task, device, persistent=False):
        """Set the boot device for a node.

        Set the boot device to use on next reboot of the node.

        :param task: A task from TaskManager.
        :param device: The boot device, one of
                       :mod:`ironic.common.boot_devices`.
        :param persistent: Boolean value. True if the boot device will
                           persist to all future boots, False if not.
                           Default: False.
        :raises: InvalidParameterValue if an invalid boot device is
                 specified.
        :raises: MissingParameterValue if a required parameter is missing
        """

    @abc.abstractmethod
    def get_boot_device(self, task):
        """Get the current boot device for a node.

        Provides the current boot device of the node. Be aware that not
        all drivers support this.

        :param task: A task from TaskManager.
        :raises: MissingParameterValue if a required parameter is missing
        :returns: A dictionary containing:

            :boot_device:
                Ahe boot device, one of :mod:`ironic.common.boot_devices` or
                None if it is unknown.
            :persistent:
                Whether the boot device will persist to all future boots or
                not, None if it is unknown.

        """

    def get_supported_boot_modes(self, task):
        """Get a list of the supported boot modes.

        NOTE: Not all drivers support this method. Older hardware
              may not implement that.

        :param task: A task from TaskManager.
        :raises: UnsupportedDriverExtension if requested operation is
                 not supported by the driver
        :raises: DriverOperationError or its derivative in case
                 of driver runtime error.
        :raises: MissingParameterValue if a required parameter is missing
        :returns: A list with the supported boot modes defined
                  in :mod:`ironic.common.boot_modes`. If boot
                  mode support can't be determined, empty list
                  is returned.
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='get_supported_boot_modes')

    def set_boot_mode(self, task, mode):
        """Set the boot mode for a node.

        Set the boot mode to use on next reboot of the node.

        Drivers implementing this method are required to implement
        the `get_supported_boot_modes` method as well.

        NOTE: Not all drivers support this method. Hardware supporting only
            one boot mode may not implement that.

        :param task: A task from TaskManager.
        :param mode: The boot mode, one of
                     :mod:`ironic.common.boot_modes`.
        :raises: InvalidParameterValue if an invalid boot mode is
                 specified.
        :raises: MissingParameterValue if a required parameter is missing
        :raises: UnsupportedDriverExtension if requested operation is
                 not supported by the driver
        :raises: DriverOperationError or its derivative in case
                 of driver runtime error.
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='set_boot_mode')

    def get_boot_mode(self, task):
        """Get the current boot mode for a node.

        Provides the current boot mode of the node.

        NOTE: Not all drivers support this method. Older hardware
              may not implement that.

        :param task: A task from TaskManager.
        :raises: MissingParameterValue if a required parameter is missing
        :raises: DriverOperationError or its  derivative in case
                 of driver runtime error.
        :raises: UnsupportedDriverExtension if requested operation is
                 not supported by the driver
        :returns: The boot mode, one of :mod:`ironic.common.boot_mode` or
                  None if it is unknown.
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='get_boot_mode')

    @abc.abstractmethod
    def get_sensors_data(self, task):
        """Get sensors data method.

        :param task: A TaskManager instance.
        :raises: FailedToGetSensorData when getting the sensor data fails.
        :raises: FailedToParseSensorData when parsing sensor data fails.
        :returns: Returns a consistent format dict of sensor data grouped by
                  sensor type, which can be processed by Ceilometer.
                  eg,

                  ::

                      {
                        'Sensor Type 1': {
                          'Sensor ID 1': {
                            'Sensor Reading': 'current value',
                            'key1': 'value1',
                            'key2': 'value2'
                          },
                          'Sensor ID 2': {
                            'Sensor Reading': 'current value',
                            'key1': 'value1',
                            'key2': 'value2'
                          }
                        },
                        'Sensor Type 2': {
                          'Sensor ID 3': {
                            'Sensor Reading': 'current value',
                            'key1': 'value1',
                            'key2': 'value2'
                          },
                          'Sensor ID 4': {
                            'Sensor Reading': 'current value',
                            'key1': 'value1',
                            'key2': 'value2'
                          }
                        }
                      }
        """

    def inject_nmi(self, task):
        """Inject NMI, Non Maskable Interrupt.

        Inject NMI (Non Maskable Interrupt) for a node immediately.

        :param task: A TaskManager instance containing the node to act on.
        :raises: UnsupportedDriverExtension
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='inject_nmi')

    def get_supported_indicators(self, task, component=None):
        """Get a map of the supported indicators (e.g. LEDs).

        :param task: A task from TaskManager.
        :param component: If not `None`, return indicator information
            for just this component, otherwise return indicators for
            all existing components.
        :returns: A dictionary of hardware components
            (:mod:`ironic.common.components`) as keys with values
            being dictionaries having indicator IDs as keys and indicator
            properties as values.

            ::

                {
                    'chassis': {
                        'enclosure-0': {
                            "readonly": true,
                            "states": [
                                "off",
                                "on"
                            ]
                        }
                    },
                    'system':
                        'blade-A': {
                            "readonly": true,
                            "states": [
                                "pff",
                                "on"
                            ]
                        }
                    },
                    'drive':
                        'ssd0': {
                            "readonly": true,
                            "states": [
                                "off",
                                "on"
                            ]
                        }
                    }
                }

        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='get_supported_indicators')

    def set_indicator_state(self, task, component, indicator, state):
        """Set indicator on the hardware component to the desired state.

        :param task: A task from TaskManager.
        :param component: The hardware component, one of
            :mod:`ironic.common.components`.
        :param indicator: Indicator ID (as reported by
            `get_supported_indicators`).
        :state: Desired state of the indicator, one of
            :mod:`ironic.common.indicator_states`.
        :raises: InvalidParameterValue if an invalid component, indicator
            or state is specified.
        :raises: MissingParameterValue if a required parameter is missing
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='set_indicator_state')

    def get_indicator_state(self, task, component, indicator):
        """Get current state of the indicator of the hardware component.

        :param task: A task from TaskManager.
        :param component: The hardware component, one of
            :mod:`ironic.common.components`.
        :param indicator: Indicator ID (as reported by
            `get_supported_indicators`).
        :raises: InvalidParameterValue if an invalid component or indicator
            is specified.
        :raises: MissingParameterValue if a required parameter is missing
        :returns: Current state of the indicator, one of
            :mod:`ironic.common.indicator_states`.

        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='get_indicator_state')


class InspectInterface(BaseInterface):
    """Interface for inspection-related actions."""
    interface_type = 'inspect'

    ESSENTIAL_PROPERTIES = {'memory_mb', 'local_gb', 'cpus', 'cpu_arch'}
    """The properties required by scheduler/deploy."""

    @abc.abstractmethod
    def inspect_hardware(self, task):
        """Inspect hardware.

        Inspect hardware to obtain the essential & additional hardware
        properties.

        :param task: A task from TaskManager.
        :raises: HardwareInspectionFailure, if unable to get essential
                 hardware properties.
        :returns: Resulting state of the inspection i.e. states.MANAGEABLE
                  or None.
        """

    def abort(self, task):
        """Abort asynchronized hardware inspection.

        Abort an ongoing hardware introspection, this is only used for
        asynchronize based inspect interface.

        NOTE: This interface is called with node exclusive lock held, the
        interface implementation is expected to be a quick processing.

        :param task: a task from TaskManager.
        :raises: UnsupportedDriverExtension, if the method is not implemented
                 by specific inspect interface.
        """
        raise exception.UnsupportedDriverExtension(
            driver=task.node.driver, extension='abort')
```

## network

connect/disconnects baremetal node to/from virtual network

```python
class NetworkInterface(BaseInterface):
    """Base class for network interfaces."""

    interface_type = 'network'

    def get_properties(self):
        """Return the properties of the interface.

        :returns: dictionary of <property name>:<property description> entries.
        """
        return {}

    def validate(self, task):
        """Validates the network interface.

        :param task: A TaskManager instance.
        :raises: InvalidParameterValue, if the network interface configuration
            is invalid.
        :raises: MissingParameterValue, if some parameters are missing.
        """

    @abc.abstractmethod
    def port_changed(self, task, port_obj):
        """Handle any actions required when a port changes

        :param task: A TaskManager instance.
        :param port_obj: a changed Port object.
        :raises: Conflict, FailedToUpdateDHCPOptOnPort
        """

    @abc.abstractmethod
    def portgroup_changed(self, task, portgroup_obj):
        """Handle any actions required when a port changes

        :param task: A TaskManager instance.
        :param portgroup_obj: a changed Port object.
        :raises: Conflict, FailedToUpdateDHCPOptOnPort
        """

    @abc.abstractmethod
    def vif_attach(self, task, vif_info):
        """Attach a virtual network interface to a node

        :param task: A TaskManager instance.
        :param vif_info: a dictionary of information about a VIF.
            It must have an 'id' key, whose value is a unique identifier
            for that VIF.
        :raises: NetworkError, VifAlreadyAttached, NoFreePhysicalPorts
        """

    @abc.abstractmethod
    def vif_detach(self, task, vif_id):
        """Detach a virtual network interface from a node

        :param task: A TaskManager instance.
        :param vif_id: A VIF ID to detach
        :raises: NetworkError, VifNotAttached
        """

    @abc.abstractmethod
    def vif_list(self, task):
        """List attached VIF IDs for a node

        :param task: A TaskManager instance.
        :returns: List of VIF dictionaries, each dictionary will have an 'id'
            entry with the ID of the VIF.
        """

    @abc.abstractmethod
    def get_current_vif(self, task, p_obj):
        """Returns the currently used VIF associated with port or portgroup

        We are booting the node only in one network at a time, and presence of
        cleaning_vif_port_id means we're doing cleaning,
        of provisioning_vif_port_id - provisioning,
        of rescuing_vif_port_id - rescuing.
        Otherwise it's a tenant network.

        :param task: A TaskManager instance.
        :param p_obj: Ironic port or portgroup object.
        :returns: VIF ID associated with p_obj or None.
        """

    @abc.abstractmethod
    def add_provisioning_network(self, task):
        """Add the provisioning network to a node.

        :param task: A TaskManager instance.
        :raises: NetworkError
        """

    @abc.abstractmethod
    def remove_provisioning_network(self, task):
        """Remove the provisioning network from a node.

        :param task: A TaskManager instance.
        """

    @abc.abstractmethod
    def configure_tenant_networks(self, task):
        """Configure tenant networks for a node.

        :param task: A TaskManager instance.
        :raises: NetworkError
        """

    @abc.abstractmethod
    def unconfigure_tenant_networks(self, task):
        """Unconfigure tenant networks for a node.

        :param task: A TaskManager instance.
        """

    @abc.abstractmethod
    def add_cleaning_network(self, task):
        """Add the cleaning network to a node.

        :param task: A TaskManager instance.
        :returns: a dictionary in the form {port.uuid: neutron_port['id']}
        :raises: NetworkError
        """

    @abc.abstractmethod
    def remove_cleaning_network(self, task):
        """Remove the cleaning network from a node.

        :param task: A TaskManager instance.
        :raises: NetworkError
        """

    def validate_rescue(self, task):
        """Validates the network interface for rescue operation.

        :param task: A TaskManager instance.
        :raises: InvalidParameterValue, if the network interface configuration
            is invalid.
        :raises: MissingParameterValue, if some parameters are missing.
        """
        pass

    def add_rescuing_network(self, task):
        """Add the rescuing network to the node.

        :param task: A TaskManager instance.
        :returns: a dictionary in the form {port.uuid: neutron_port['id']}
        :raises: NetworkError
        :raises: InvalidParameterValue, if the network interface configuration
            is invalid.
        """
        return {}

    def remove_rescuing_network(self, task):
        """Removes the rescuing network from a node.

        :param task: A TaskManager instance.
        :raises: NetworkError
        :raises: InvalidParameterValue, if the network interface configuration
            is invalid.
        :raises: MissingParameterValue, if some parameters are missing.
        """
        pass

    def need_power_on(self, task):
        """Check if ironic node must be powered on before applying network changes

        :param task: A TaskManager instance.
        :returns: Boolean.
        """
        return False
```

## power

node的电源管理

```python
class PowerInterface(BaseInterface):
    """Interface for power-related actions."""
    interface_type = 'power'

    @abc.abstractmethod
    def get_power_state(self, task):
        """Return the power state of the task's node.

        :param task: A TaskManager instance containing the node to act on.
        :raises: MissingParameterValue if a required parameter is missing.
        :returns: A power state. One of :mod:`ironic.common.states`.
        """

    @abc.abstractmethod
    def set_power_state(self, task, power_state, timeout=None):
        """Set the power state of the task's node.

        :param task: A TaskManager instance containing the node to act on.
        :param power_state: Any power state from :mod:`ironic.common.states`.
        :param timeout: timeout (in seconds) positive integer (> 0) for any
          power state. ``None`` indicates to use default timeout.
        :raises: MissingParameterValue if a required parameter is missing.
        """

    @abc.abstractmethod
    def reboot(self, task, timeout=None):
        """Perform a hard reboot of the task's node.

        Drivers are expected to properly handle case when node is powered off
        by powering it on.

        :param task: A TaskManager instance containing the node to act on.
        :param timeout: timeout (in seconds) positive integer (> 0) for any
          power state. ``None`` indicates to use default timeout.
        :raises: MissingParameterValue if a required parameter is missing.
        """

    def get_supported_power_states(self, task):
        """Get a list of the supported power states.

        :param task: A TaskManager instance containing the node to act on.
        :returns: A list with the supported power states defined
                  in :mod:`ironic.common.states`.
        """
        return [states.POWER_ON, states.POWER_OFF, states.REBOOT]
```

## raid

设置or取消raid设置，也分out-of-band和in-band

```python
class RAIDInterface(BaseInterface):
    interface_type = 'raid'

    def __init__(self):
        """Constructor for RAIDInterface class."""
        with open(RAID_CONFIG_SCHEMA, 'r') as raid_schema_fobj:
            self.raid_schema = json.load(raid_schema_fobj)

    def get_properties(self):
        """Return the properties of the interface.

        :returns: dictionary of <property name>:<property description> entries.
        """
        return {}

    def validate(self, task):
        """Validates the RAID Interface.

        This method validates the properties defined by Ironic for RAID
        configuration. Driver implementations of this interface can override
        this method for doing more validations (such as BMC's credentials).

        :param task: A TaskManager instance.
        :raises: InvalidParameterValue, if the RAID configuration is invalid.
        :raises: MissingParameterValue, if some parameters are missing.
        """
        target_raid_config = task.node.target_raid_config
        if not target_raid_config:
            return
        self.validate_raid_config(task, target_raid_config)

    def validate_raid_config(self, task, raid_config):
        """Validates the given RAID configuration.

        This method validates the given RAID configuration.  Driver
        implementations of this interface can override this method to support
        custom parameters for RAID configuration.

        :param task: A TaskManager instance.
        :param raid_config: The RAID configuration to validate.
        :raises: InvalidParameterValue, if the RAID configuration is invalid.
        """
        raid.validate_configuration(raid_config, self.raid_schema)

    # NOTE(mgoddard): This is not marked as a deploy step, because it requires
    # the create_configuration method to support use during deployment, which
    # might not be true for all implementations. Subclasses wishing to expose
    # an apply_configuration deploy step should implement this method with a
    # deploy_step decorator. The RAID_APPLY_CONFIGURATION_ARGSINFO variable may
    # be used for the deploy_step argsinfo argument. The create_configuration
    # method must also accept a delete_existing argument.
    def apply_configuration(self, task, raid_config, create_root_volume=True,
                            create_nonroot_volumes=True,
                            delete_existing=True):
        """Applies RAID configuration on the given node.

        :param task: A TaskManager instance.
        :param raid_config: The RAID configuration to apply.
        :param create_root_volume: Setting this to False indicates
            not to create root volume that is specified in raid_config.
            Default value is True.
        :param create_nonroot_volumes: Setting this to False indicates
            not to create non-root volumes (all except the root volume) in
            raid_config.  Default value is True.
        :param delete_existing: Setting this to True indicates to delete RAID
            configuration prior to creating the new configuration.
        :raises: InvalidParameterValue, if the RAID configuration is invalid.
        :returns: states.DEPLOYWAIT if RAID configuration is in progress
            asynchronously or None if it is complete.
        """
        self.validate_raid_config(task, raid_config)
        node = task.node
        node.target_raid_config = raid_config
        node.save()
        return self.create_configuration(
            task,
            create_root_volume=create_root_volume,
            create_nonroot_volumes=create_nonroot_volumes,
            delete_existing=delete_existing)

    @abc.abstractmethod
    def create_configuration(self, task,
                             create_root_volume=True,
                             create_nonroot_volumes=True,
                             delete_existing=True):
        """Creates RAID configuration on the given node.

        This method creates a RAID configuration on the given node.
        It assumes that the target RAID configuration is already
        available in node.target_raid_config.
        Implementations of this interface are supposed to read the
        RAID configuration from node.target_raid_config. After the
        RAID configuration is done (either in this method OR in a call-back
        method), ironic.common.raid.update_raid_info()
        may be called to sync the node's RAID-related information with the
        RAID configuration applied on the node.

        :param task: A TaskManager instance.
        :param create_root_volume: Setting this to False indicates
            not to create root volume that is specified in the node's
            target_raid_config. Default value is True.
        :param create_nonroot_volumes: Setting this to False indicates
            not to create non-root volumes (all except the root volume) in the
            node's target_raid_config.  Default value is True.
        :param delete_existing: Setting this to True indicates to delete RAID
            configuration prior to creating the new configuration.
        :returns: states.CLEANWAIT (cleaning) or states.DEPLOYWAIT (deployment)
            if RAID configuration is in progress asynchronously, or None if it
            is complete.
        """

    @abc.abstractmethod
    def delete_configuration(self, task):
        """Deletes RAID configuration on the given node.

        This method deletes the RAID configuration on the give node.
        After RAID configuration is deleted, node.raid_config should be
        cleared by the implementation.

        :param task: A TaskManager instance.
        :returns: states.CLEANWAIT (cleaning) or states.DEPLOYWAIT (deployment)
            if deletion is in progress asynchronously, or None if it is
            complete.
        """

    def get_logical_disk_properties(self):
        """Get the properties that can be specified for logical disks.

        This method returns a dictionary containing the properties that can
        be specified for logical disks and a textual description for them.

        :returns: A dictionary containing properties that can be mentioned for
            logical disks and a textual description for them.
        """
        return raid.get_logical_disk_properties(self.raid_schema)
```

苦闷的是，最新redfish driver&sushy代码拿来，改造了一下，调了一天，认证还没过

redfish driver代码&sushy lost中。。。