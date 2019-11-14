---
layout: post
title:  "Ironic with Redfish Support"
date:   2019-11-14 17:30:00 +0800
categories: Openstack
tags: Openstack-Ironic
excerpt: Research on redfish support for ironic
mathjax: true
---

# Introduction

The Redfish standard is a suite of specifications that deliver an industry standard protocol providing a RESTful interface for the management of servers, storage, networking, and converged infrastructure. It based on the Open Data Protocol (OData) which utilizes HTTPS and JSON to transfer data. It helps customers integrate solutions within their existing tool chains. This standard was developed by the [Distributed Management Task Force (DMTF)](https://www.dmtf.org/about) with the participation of many server vendors, including Cisco. It utilizes a range of scalable IT technologies that are widely used, and by using these accepted technologies, it makes the use of Redfish™ easier. 

It is ideal for cloud and web-based infrastructures, which typically have large quantities of servers in heterogeneous environments. Redfish also provides developers a familiar and comfortable suite of protocols.

## Management Standard

IT solution models have evolved over the years and given way to several Out-of-Band (OOB) systems management standards, or lights-out management (LOM) systems that work within emerging programming standards and can be implemented in the embedded systems. While this has worked fairly well, there was still a need for a single management standard that could handle the various demands of IT solutions robustly. Expanded scale, higher security, and multi-vendor openness call for equally diverse DevOps tools and processes.

Keeping these requirements in mind, the DMTF took on the responsibility of creating a new management interface standard, which resulted in Redfish™ version 1.0, which was formally launched in July, 2015.

Key features of the Redfish™ management standard include:

- Simple to use and highly secure
- Encrypted connections and generally heightened security
- Simple programmatic interface that can be easily managed using scripts
- Meets Open Compute Project’s Remote Machine Management requirements

Key Technologies:

* HTTPS Communications
* RESTful Application Programming Interface

Cisco has now built capabilities of using RESTful APIs to configure the UCS C-series servers using the Redfish™ technology.

# Refish Architecture

The Redfish™ API comprises a folder structure that starts with the Redfish root at “/redfish/”. In case of a C-Series server, the root is accessed through the URI https://\<Cisco IMC IP\>/redfish/v1/ - the “v1” at the end of the URI denotes the version of the API.

The URI is the primary unique identifier of resources. Redfish™ URIs consist of three parts (https://mgmt.vendor.com/redfish/v1/Systems/SvrID):

1. the scheme and authority of the URI (https://mgmt.vendor.com)
2. the root service and version (/redfish/v1)
3. a unique resource identifier (/Systems/SvrID)

## Redfish Tree Structure

The Redfish tree structure comprises a top-level root from where the RESTful interface branches out to cover a number of “Collections” that subsequently include multiple levels within, creating a tree-like structure. 

## Redfish Operations

Redfish™ uses the HTTPS method to perform operations of a RESTful API. You can specify the type of request being made. It adheres to a standard CRUD (Create, Retrieve, Update, and Delete) format. Depending on the desired result, you can issue the following types of commands:

- **GET**: View data
- **POST**: Create resources or use actions
- **PATCH**: Change one or more properties on a resource
- **DELETE**: Remove a resource

> Currently, HEAD and PUT operations are not supported for Redfish™ URIs.

# Redfish REST API Examples

## Retrieving Service Root without Authentication

```shell
MINSU-M-M1RW:k8s_github minsu$ curl -Get https://10.225.11.137/redfish/v1 -k
{
  "Chassis":{
    "@odata.id":"/redfish/v1/Chassis"
  },
  "@odata.id":"/redfish/v1/",
  "JSONSchemas":{
    "@odata.id":"/redfish/v1/JSONSchemas"
  },
  "RedfishVersion":"1.0.0",
  "EventService":{
    "@odata.id":"/redfish/v1/EventService"
  },
  "Systems":{
    "@odata.id":"/redfish/v1/Systems"
  },
  "Description":"Root Service",
  "Name":"Cisco RESTful Root Service",
  "Links":{
    "Sessions":{
      "@odata.id":"/redfish/v1/SessionService/Sessions"
    }
  },
  "TaskService":{
    "@odata.id":"/redfish/v1/TaskService"
  },
  "Managers":{
    "@odata.id":"/redfish/v1/Managers"
  },
  "@odata.type":"#ServiceRoot.1.0.0.ServiceRoot",
  "SessionService":{
    "@odata.id":"/redfish/v1/SessionService"
  },
  "@odata.context":"/redfish/v1/$metadata#ServiceRoot",
  "Id":"RootService",
  "AccountService":{
    "@odata.id":"/redfish/v1/AccountService"
  },
  "MessageRegistry":{
    "@odata.id":"/redfish/v1/MessageRegistry"
  }
}
```

## Viewing Server ID

```shell
MINSU-M-M1RW:k8s_github minsu$ curl --insecure -u admin:pass https://10.225.11.137/redfish/v1/Systems
{
  "Members":[{
      "@odata.id":"/redfish/v1/Systems/FCH2131V13Z"
    }],
  "Description":"Collection of Computer Systems",
  "@odata.type":"#Cisco_ComputerSystemCollection",
  "@odata.id":"/redfish/v1/Systems",
  "Members@odata.count":1,
  "Name":"Computer System Collection",
  "@odata.context":"/redfish/v1/$metadata#Systems"
}
```

## Viewing Specific System information

```shell
MINSU-M-M1RW:k8s_github minsu$ curl --insecure -u admin:pass https://10.225.11.137/redfish/v1/Systems/FCH2131V13Z
{
  "SerialNumber":"FCH2131V13Z",
  "Boot":{
    "BootSourceOverrideTarget":"None",
    "BootSourceOverrideTarget@Redfish.AllowableValues":["None","Pxe","Floppy","Cd","Hdd","BiosSetup","Diags"],
    "BootSourceOverrideEnabled@Redfish.AllowableValues":["Once","Continuous","Disabled"],
    "BootSourceOverrideEnabled":"Disabled"
  },
  "Id":"FCH2131V13Z",
  "AssetTag":"Unknown",
  "PowerState":"On",
  "SystemType":"Physical",
  "ProcessorSummary":{
    "Model":"Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz",
    "Count":2
  },
  "HostName":"C220M4-10-225-11-137",
  "MemorySummary":{
    "TotalSystemMemoryGiB":256,
    "State":{
      "HealthRollup":"OK",
      "Health":"OK"
    }
  },
  "Processors":{
    "@odata.id":"/redfish/v1/Systems/FCH2131V13Z/Processors"
  },
  "Description":"",
  "Links":{
    "CooledBy":["/redfish/v1/Chassis/1/Thermal"],
    "Chassis":["/redfish/v1/Chassis/1"],
    "PoweredBy":["/redfish/v1/Chassis/1/Power"],
    "ManagedBy":["/redfish/v1/Managers/CIMC"]
  },
  "SimpleStorage":{
    "@odata.id":"/redfish/v1/Systems/FCH2131V13Z/SimpleStorage"
  },
  "UUID":"DC3C7EF8-39ED-451B-805A-496546AB9209",
  "Status":{
    "State":"Enabled",
    "Health":"Warning"
  },
  "BiosVersion":"C220M4.3.0.3c.0.0831170216",
  "Name":"UCS C220 M4S",
  "LogServices":{
    "@odata.id":"/redfish/v1/Systems/FCH2131V13Z/LogServices"
  },
  "Actions":{
    "#System.Reset":{
      "Target":"/redfish/v1/Systems/FCH2131V13Z/Actions/System.Reset",
      "ResetType@Redfish.AllowableValues":["On","ForceOff","GracefulShutdown","ForceRestart","Nmi"]
    }
  },
  "@odata.context":"/redfish/v1/$metadata#Systems/Members/$entity",
  "@odata.type":"#Cisco_ComputerSystem",
  "@odata.id":"/redfish/v1/Systems/FCH2131V13Z",
  "Manufacturer":"Cisco Systems",
  "IndicatorLED":"Off",
  "Model":"UCS C220 M4S",
  "EthernetInterfaces":{
    "@odata.id":"/redfish/v1/Systems/FCH2131V13Z/EthernetInterfaces"
  }
}
```

# Ironic with Redfish Support

Enable redfish driver for ironic, redfish driver add a new `redfish` hardware type, power and management interfaces. It was supported since OpenStack Pike release. But we're now on Newton, need to backport.

And for neutron part, we want to configurare uplink switch access vlan, need to check if redfish api can do this. 

For ironic part, we need: 

1. set boot device (BootSourceOverrideTarget)

2. power on 

   ```shell
   curl -vv https://10.10.10.10/redfish/v1/Systems
   /FCH2005V1EN/Actions/System.Reset -d
   '{"ResetType":"On"}' --insecure -u admin:Admin123
   ```

3. power off 

   ```shell
   curl -vv https://10.10.10.10/redfish/v1/Systems/FCH2005V1EN/Actions/System.Reset -d
   '{"ResetType":"ForceOff"}' --insecure -u admin:Admin123
   ```

4. reboot

For neutron part, we need:

1. bind port, configure uplink switch vlan

```shell
   curl -XPATCH -k -u admin https://10.10.10.10/redfish/v1/Managers/CIMC/EthernetInterfaces/NICs -d'
   {"VLAN": {"VLANId": 2,"VLANEnable": true}}
```

But it's a little bit complicated. Cause: 

* **redfish is not a driver**

The Bare Metal service delegates actual hardware management to **drivers**. Starting with the Ocata release, two types of drivers are supported: *classic drivers* (for example, `pxe_ipmitool`, `agent_ilo`, etc.) and the newer *hardware types* (for example, generic `redfish` and `ipmi` or vendor-specific `ilo` and `irmc`).

Drivers, in turn, consist of *hardware interfaces*: sets of functionality dealing with some aspect of bare metal provisioning in a vendor-specific way. *Classic drivers* have all *hardware interfaces* hardcoded, while *hardware types* only declare which *hardware interfaces* they are compatible with.

Please refer to the [driver composition reform specification](https://specs.openstack.org/openstack/ironic-specs/specs/approved/driver-composition-reform.html) for technical details behind *hardware types*.

From API user’s point of view, both *classic drivers* and *hardware types* can be assigned to the `driver` field of a node. However, they are configured differently.

But Newton only have one configuration "enabled_drivers", but redfish is not a driver, but a hardware_type, a boot_interface, a power_interface, a management_interface, an inspect_interface...

```shell
[DEFAULT]
...
enabled_hardware_types = ipmi,redfish
enabled_boot_interfaces = ipmitool,redfish-virtual-media
enabled_power_interfaces = ipmitool,redfish
enabled_management_interfaces = ipmitool,redfish
enabled_inspect_interfaces = inspector,redfish
```

It's hard to backport all these code from community, so I think what we can do is to refactor redfish code to let it be a "driver". 

* **The [Sushy](https://opendev.org/openstack/sushy) library is needed**

The sushy library should be installed on the ironic conductor node. Seems not a bit deal, right? But there are many package dependencies conflict issue. 😢

# References

[1] [https://youtu.be/uIgHMD3FQW0](https://youtu.be/uIgHMD3FQW0)

[2] [https://www.cisco.com/c/en/us/td/docs/unified_computing/ucs/c/sw/api/3_0/b_Cisco_IMC_REST_API_guide_301/b_Cisco_IMC_api_3x60_chapter_01.html](https://www.cisco.com/c/en/us/td/docs/unified_computing/ucs/c/sw/api/3_0/b_Cisco_IMC_REST_API_guide_301/b_Cisco_IMC_api_3x60_chapter_01.html)

[3] [https://blogs.cisco.com/datacenter/cisco-supports-redfish-standard-api-enhances-ucs-programmability](https://blogs.cisco.com/datacenter/cisco-supports-redfish-standard-api-enhances-ucs-programmability)

[4] [https://docs.openstack.org/ironic/pike/install/enabling-drivers.html](https://docs.openstack.org/ironic/pike/install/enabling-drivers.html)