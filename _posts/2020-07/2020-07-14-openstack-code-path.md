---
layout: post
title:  "openstack Horizon Release Highlights"
date:   2020-07-15 23:00:00 +0800
categories: Openstack
tags: Openstack-Horizon
excerpt: openstack Horizon Release Highlights
mathjax: true
typora-root-url: ../
---

# openstack Horizon Release Highlights

## Ocata

- Added a new `create_volume` setting under the `LAUNCH_INSTANCE_DEFAULTS` dict. This allows you to set the default value of “Create Volume”, when Cinder is available.
- The Access & Security panel’s tabs have been moved to their own panels for clearer navigation and better performance. API Access and Key Pairs now reside in the Compute panel group. Floating IPs and Security Groups are now in the Network panel group.
- Download buttons for OpenStack RC files have been added to the user dropdown menu in the top right of Horizon.
- Configurable setting to allow operators to require admin users provide search criteria first, before loading any data
- ANGULAR_FEATURES now allows for a key ‘flavors_panel’ to be specified as True or False indicating whether the Angular version of the panel is enabled.

```
FILTER_DATA_FIRST={``  ``'admin.instances'``: True,``  ``'admin.volumes'``: True,``  ``'admin.networks'``: True,``  ``'admin.images'``: True``}
```

- The logos in Horizon (previously logo-splash.png and logo.png) now load SVG files instead of PNG. The default logos have been updated to reflect the new OpenStack branding.

![image-20200715173507859](/../assets/images/image-20200715173507859.png)

## Pike

- Add support for horizon offering a clouds.yaml file for download along with the openrc files. For more information on clouds.yaml, see [os-client-config documentation](https://docs.openstack.org/os-client-config/latest/user/).
- Added settings OPENSTACK_KEYSTONE_DOMAIN_DROPDOWN (boolean) and OPENSTACK_KEYSTONE_DOMAIN_CHOICES (tuple of tuples) to support a dropdown list of keystone domains to choose from at login. This should NOT be enabled for public clouds, as advertising enabled domains to unauthenticated users irresponsibly exposes private information. This is useful for private clouds that sit behind a corprate firewall and that have a small number of domains mapped to known corporate structures, such as an LDAP directory, Active Directory domains, geopgraphical regions or business units.
- Added two charts to show the Number of Volumes and Total Volume Storage quotas on launch instance modal when cinder is enabled.
- Gives end-users the ability to create and delete ports in their networks. The functionality will be implemented into the project network details table. Following the discussions in the bug discussion. This functionality will be enabled/disabled via policy. Blueprint can be found at [[blueprint network-ports-tenant](https://blueprints.launchpad.net/horizon/+spec/network-ports-tenant)] Bug can be found at [[bug 1399252](https://bugs.launchpad.net/horizon/+bug/1399252)]
- Added a locked status column on admin/project instances table. It will show a locked or unlocked icon if nova API 2.9 or above is used. The locked status is also available on instance details panel.
- Added a new setting CREATE_IMAGE_DEFAULTS(dictionary) to configure the default options shown on create image modal. By default, the visibility option is public on create image modal. If `image_visibility` in this setting is set to `"private"`, the default visibility option is private.
- Now it is possible to enable/disable port security in Horizon, when the port-security extension is available. Note: Neutron allows disabling the port security on a port only when no security groups are associated to it
- Security group association per port is now shown in the port detail page. In Neutron different security groups can be associated on different ports of a same server instance, but previously it cannot be referred in Horizon.
- Securtiy group “Add rule” form now allows to specify ‘any’ IP protocol and ‘any’ port number (for TCP and UDP protocols). This feature is available when neutron is used as a networking back-end. You can specify ‘any’ IP protocol for ‘Other Protocol’ and `-1` means ‘any’ IP protocol. You can also see `All ports` choice in ‘Open Port’ field in case of TCP or UDP protocol is selected.
- Panel group is introduced in the Admin dashboard to organize admin panels better. Panels in “System” group of Admin dashboard are now categorized into four groups: “Compute”, “Volume”, “Network” and “System”.
- The ability to edit flavors is disabled by default. See `ENABLE_FLAVOR_EDIT` in the settings documentation for more information. (Deprecated config option ENABLE_FLAVOR_EDIT is removed. — Train)

## Queens

- [[blueprint ng-keypairs](https://blueprints.launchpad.net/horizon/+spec/ng-keypairs)] AngularJS-based Key Pairs panel is added. The features in the legacy panel are fully implemented. The Key Pairs panel now may be configured to use either the legacy or AngularJS-based codes. The ANGULAR_FEATURES setting now allows for a key_pairs_panel. If set to True, then the AngularJS-Based Key Pairs panel will be used, while the Django version will be used if set to False. Default value for key_pairs_panel is True.
- Added the way to specify an interface when attaching it to an instance. It can be specified by a network and a fixed IP address (optional) or a port.
- Cinder API v3 is used by default now. It was introduced in Mitaka release and has all features from API v2.
- The keystone v3 API now becomes the default keystone API version.
- Support security groups association per network port for operators and users. Note that the current implementation only supports to edit security groups of neutron port from the port tables in the network detail page (Further improvement is planned).

## Rocky

- Added server groups and server group members quota management. Users can specify their values when creating or modifying project information, and users can also change their quota default values on the Admin-> System-> Defaults page.
- [[blueprint application-credentials](https://blueprints.launchpad.net/horizon/+spec/application-credentials)] Adds a new panel for creating, viewing, and deleting keystone application credentials.
- Floating IP can be released when it is disassociated from a server. “Release Floating IP” checkbox is now available in “Disassociate Floating IP” form.
- Security groups now can be specified when creating a port. When the port security is enabled, the security groups tab will be displayed in create port workflow.
- “Interfaces” tab is added to the instance detail page. The new tab shows a list of ports attached to an instance. Users now have an easy way to access the list of ports of the instance and edit security groups per port. In addition, “Edit Port Security Groups” menu is added as an action of the instance table.
- Support has been added to set and display DNS attributes for Floating IPs (DNS Name and DNS Domain). These attributes are only available if Neutron has the dns-integration extension enabled.
- [[blueprint cinder-generic-volume-groups](https://blueprints.launchpad.net/horizon/+spec/cinder-generic-volume-groups)] Cinder generic groups is now supported. Consistency groups views will be disabled if the generic group support is available. User is able to create generic groups and snapshots now. Note that operators need to create at least one group type so that users can use the generic group feature. Otherwise, it might be better to disable the group and group snapshot panels by the horizon plugin `enabled` files.
- [[bug 1690433](https://bugs.launchpad.net/bugs/1690433)] “Get me a network” feature provided by nova and neutron is now exposed in the launch server form. This feature will sets up a neutron network topology for a project if there is no network in the project. It simplifies the workflow when launching a server. In the horizon support, when there is no network which can be used for a server, a dummy network named ‘auto_allocated_network’ is shown in the network choices. The feature is disabled by default because it requires preparations in your neutron deployment. To enable it, set `enable_auto_allocated_network` in `OPENSTACK_NEUTRON_NETWORK` to `True`.
- Quota information panel and forms are now tabbified per back-end service.
  - Admin -> Defaults -> Default Quotas table
  - Admin -> Defaults -> Update Defaults form
  - Identity -> Projects -> Modify Quotas form
- [[bug 1742332](https://bugs.launchpad.net/bugs/1742332)] Description for security group rule is supported.
- The “Quotas” tab in the “Create Project” form was split out into a new separate form “Modify Quotas”. Quotas for a new project need to be configured from “Modify Quotas” action after creating a new project.
- Remove deprecated Cinder API V1 support. Cinder V1 API was deprecated for a while and removed in Queens release. If you need to enable Cinder support you should update the OPENSTACK_API_VERSIONS configuration option to use Cinder V2 or V3 API.

## Stein

- Add “Create Router” button to Admin/Network/Routers panel.

- [[blueprint neutron-rbac-policies](https://blueprints.launchpad.net/horizon/+spec/rbac-policies)] This blueprint adds RBAC policies panel to the Admin Network group. This panel will be enabled by default when the RBAC extension is enabled. Remove this panel by setting “‘enable_rbac_policy’: False” in ‘local_settings.py’. RBAC policy supports the control of two resources: networks and qos policies, because qos policies is an extension function of neutron, need to enable this extension if wants to use it.

- [[blueprint instance-rescue-horizon-support](https://blueprints.launchpad.net/horizon/+spec/instance-rescue-horizon-support)] Support instance rescue feature

- [[bug 1785263](https://bugs.launchpad.net/bugs/1785263)] Modify the project detail view in a multi tabbed view, composed of:

  - `Overview` tab displaying general information about the project.
  - `Users` tab displaying all users which have roles on the project (and their roles on it), including users which have roles on the project throw their membership to a group.
  - `Group` tab displaying all groups which have roles on the project (and their roles on it).

- [[bug 1792524](https://bugs.launchpad.net/bugs/1792524)] Modify the user detail view in a multi tabbed view, composed of:

  - `Overview` tab displaying general information about the user.
  - `Roles assignments` tab displaying all the roles that the users have on project or domain, directly or throw their membership to a group. When the role comes from a membership to a group this will be indicated into the role column.
  - `Groups` tab displaying all groups where the user is a membership to.

- [[blueprint cinder-generic-volume-groups](https://blueprints.launchpad.net/horizon/+spec/cinder-generic-volume-groups)] Cinder generic groups is now supported for admin panel. Admin is now able to view all groups and group snapshots for differenet users. Also group-type and group-type-spec support added to admin panel. Admin is able to create group-type and group-type-spec now.

- New setting `SESSION_REFRESH` (defaults to `True`) that allows the user session expiry to be refreshed for every request until the token itself expires. `SESSION_TIMEOUT` acts as an idle timeout value now.

- Added a new `hide_create_volume` setting under the `LAUNCH_INSTANCE_DEFAULTS` dict. This allows you to hide the “Create New Volume” option in the “Launch Instance” form and instead rely on the default value you select with `create_volume` is the best suitable option for your users.

- [[bug 1795851](https://bugs.launchpad.net/bugs/1795851)] Operators now can control whether the links of “Download OpenRC” and “Download clouds.yaml” are displayed or not via new settings `SHOW_OPENRC_FILE` and `SHOW_OPENSTACK_CLOUDS_YAML`. `openrc` and `clouds.yaml` files provided by horizon now assume the basic simple deployment and do not cover keystone authentication like saml2, openid and so on. The default `openrc` and `clouds.yaml` from horizon do not make sense for such environments.

  Custom templates for `clouds.yaml` and `openrc` files can be configured now via `OPENSTACK_CLOUDS_YAML_CUSTOM_TEMPLATE` and `OPENRC_CUSTOM_TEMPLATE` settings. For more detail, see the [Settings Reference](https://docs.openstack.org/horizon/latest/configuration/settings.html).

  `ADD_TEMPLATE_DIRS` setting is also added so that operators can place custom templates for `clouds.yaml` at deployment-specific paths.

  

- Added an upgrade_check management command, that checks the configuration files for any settings that may potentially be problematic in the next version. The command is available as `./manage.py upgrade_check`.

- Adds the possibility to redirect the login to an identity provider by default. For that purpose the following variables have been added, `WEBSSO_DEFAULT_REDIRECT`, `WEBSSO_DEFAULT_REDIRECT_PROTOCOL`, `WEBSSO_DEFAULT_REDIRECT_REGION` and `WEBSSO_DEFAULT_REDIRECT_LOGOUT`.

## Train

- Cinder consistency group support in horizon has been dropped in Train release. It was deprecated in Pike release in Cinder and deprecated in Stein release in Horizon. The feature is superseded by the generic group feature and horizon provides full support of the generic group.

- Keystone API V2 support has been dropped in Train release. Keystone V2 API support was deprecated in Stein release. If you use Keystone V2 before, you should update the OPENSTACK_API_VERSIONS configuration option to use Keystone V3 API.

- Deprecated `SHOW_KEYSTONE_V2_RC` since Stein release is removed.

- The default values of the settings listed in `local_settings.py.example` in past releases have been moved to `openstack_dashboard/defaults.py`. By doing this, horizon can now provide the definitions of the default settings more explicitly. For the available settings, see `openstack_dashboard/defaults.py` and the horizon setting reference found at https://docs.openstack.org/horizon/latest/configuration/settings.html.

  Note that Django related settings and HORIZON_CONFIG still exist in `local_settings.py.example` in this release and they will be revisited in upcoming releases.

## Ussuri

- Added support for Keystone locking user option. Locked user can’t change own password using the self-service password change API. By default, users are unlocked and can change their own passwords. In case of older Keystone not supporting this feature, all users treated as unlocked.
- Introduced a new `DEFAULT_BOOT_SOURCE` config option to allow operators to configure a default instance boot source.
- Added support to retrieve supported disk formats from glance, so you can adjust disk_formats only inside glance-api.conf. You still can use IMAGE_BACKEND_SETTINGS to adjust format naming.
- Django 1.11 support was dropped. Django 1.11 ends its extended support in April 2020 which is before Ussuri release. Considering this, horizon dropped Django 1.11 support and use Django 2.2 as default.
- Python 2.7 support has been dropped. Last release of horizon to support python 2.7 is OpenStack Train. The minimum version of Python now supported by horizon is Python 3.6.
- `OPENSTACK_NOVA_EXTENSIONS_BLACKLIST` option is deprecated. All of the nova API extensions have been mainlined several releases ago and there is no potential performance issue in the nova API. This option is used only to toggle features in horizon and there seems no performance issues controlled by the option in horizon. Considering this situation, this option is deprecated now.
- Adds support for access rules for application credentials. Fine-grained restrictions can now be applied to application credentials by supplying a list of access rules upon creation. See the [keystone documentation](https://docs.openstack.org/api-ref/identity/v3/#application-credentials) for more information.
- The default OPENSTACK_KEYSTONE_URL value has been changed to `"http://%s/identity/v3" % OPENSTACK_HOST` from `"http://%s:5000/v3" % OPENSTACK_HOST`.
- Glance API V1 support has been dropped in Ussuri release. Glance V1 API support was deprecated in Stein release.