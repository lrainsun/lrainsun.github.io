---
layout: post
title:  "openstack keystone unified limit"
date:   2020-07-14 23:01:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack keystone unified limit
mathjax: true
typora-root-url: ../
---

# openstack keystone unified limit

As of the Queens release, keystone has the ability to store and relay information known as a limit. Limits can be used by services to enforce quota on resources across OpenStack. This section describes the basic concepts of limits, how the information can be consumed by services, and how operators can manage resource quota across OpenStack using limits.

## What is a limit?[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#what-is-a-limit)

A limit is a threshold for resource management and helps control resource utilization. A process for managing limits allows for reallocation of resources to different users or projects as needs change. Some information needed to establish a limit may include:

- project_id
- domain_id
- API service type (e.g. compute, network, object-storage)
- a resource type (e.g. ram_mb, vcpus, security-groups)
- a default limit
- a project specific limit i.e resource limit
- user_id (optional)
- a region (optional depending on the service)

Since keystone is the source of truth for nearly everything in the above list, limits are a natural fit as a keystone resource. Two different limit resources exist in this design.

- The first is a registered limit
- The second is a project limit.

In OpenStack, a quota system mainly contains two parts: `limit` and `usage`. The Unified limits in Keystone is a replacement of the `limit` part. It contains two kinds of resouces: `Registered Limit` and `Limit`. A `registered limit` is a default limit. It is usually created by the services which are registered in Keystone. A `limit` is the limit that override the registered limit for each project.

### Registered limits[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#registered-limits)

A registered limit accomplishes two important things in order to enforce quota across multi-tenant, distributed systems. First, it establishes resource types and associates them to services. Second, it sets a default resource limit for all projects. The first part maps specific resource types to the services that provide them. For example, a registered limit can map vcpus, to the compute service. The second part sets a default of 20 vcpus per project. This provides all the information needed for basic quota enforcement for any resource provided by a service.

### Domain limits[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#domain-limits)

A domain limit is a limit associated to a specific domain and it acts as an override for a registered limit. Similar to registered limits, domain limits require a resource type and a service. Additionally, a registered limit must exist before you can create a domain-specific override. For example, let’s assume a registered limit exists for vcpus provided by the compute service. It wouldn’t be possible to create a domain limit for cores on the compute service. Domain limits can only override limits that have already been registered. In a general sense, registered limits are likely established when a new service or cloud is deployed. Domain limits are used continuously to manage the flow of resource allocation.

Domain limits may affect the limits of projects within the domain. This is particularly important to keep in mind when choosing an enforcement model, documented below.

### Project limits[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#project-limits)

Project limits have the same properties as domain limits, but are specific to projects instead of domains. You must register a limit before creating a project-specific override. Just like with domain limits, the flow of resources between related projects may vary depending on the configured enforcement model. The support enforcement models below describe how limit validation and enforcement behave between related projects and domains.

Together, registered limits, domain limits, and project limits give deployments the ability to restrict resources across the deployment by default, while being flexible enough to freely marshal resources across projects.

## Limits and usage[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#limits-and-usage)

When we talk about a quota system, we’re really talking about two systems. A system for setting and maintaining limits, the theoretical maximum usage, and a system for enforcing that usage does not exceed limits. While they are coupled, they are distinct.

Up to this point, we’ve established that keystone is the system for maintaining limit information. Keystone’s responsibility is to ensure that any changes to limits are consistent with related limits currently stored in keystone.

Individual services maintain and enforce usage. Services check enforcement against the current limits at the time a user requests a resource. Usage reflects the actual resource allocation in units to a consumer.

## Enforcement models[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#enforcement-models)

### Flat[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#flat)

Flat enforcement ignores all aspects of a project hierarchy. Each project is considered a peer to all other projects. The limits associated to the parents, siblings, or children have no affect on a particular project. This model exercises the most isolation between projects because there are no assumptions between limits, regardless of the hierarchy. Validation of limits via the API will allow operations that might not be considered accepted in other models.

### Strict Two Level[¶](https://docs.openstack.org/keystone/latest/admin/unified-limits.html#strict-two-level)

The `strict_two_level` enforcement model assumes the project hierarchy does not exceed two levels. The top layer can consist of projects or domains. For example, project Alpha can have a sub-project called Beta within this model. Project Beta cannot have a sub-project. The hierarchy is restrained to two layers. Alpha can also be a domain that contains project Beta, but Beta cannot have a sub-project. Regardless of the top layer consisting of projects or domains, the hierarchical depth is limited to two layers.

Resource utilization is allowed to flow between projects in the hierarchy, depending on the limits. This property allows for more flexibility than the `flat` enforcement model. The model is strict in that operators can set limits on parent projects or domains and the limits of the children may never exceed the parent.

