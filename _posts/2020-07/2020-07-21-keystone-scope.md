---
layout: post
title:  "openstack keystone Authorization Scopes & Roles"
date:   2020-07-21 23:00:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack keystone Authorization Scopes & Roles
mathjax: true
typora-root-url: ../
---

# openstack keystone Authorization Scopes & Roles

# Scope

End users use the Identity API as a way to express their authoritative power to other OpenStack services. This is done using tokens, which can be scoped to one of several targets depending on the users’ role assignments. This is typically referred to as a token’s *scope*. This happens when a user presents credentials, in some form or fashion, to keystone in addition to a desired scope. If keystone can prove the user is who they say they are (authN), it will then validate that the user has access to the scope they are requesting (authZ). If successful, the token response will contain a token ID and data about the transaction, such as the scope target and role assignments. Users can use this token ID in requests to other OpenStack services, which consume the authorization information associated to that token to make decisions about what that user can or cannot do within that service.

This section describes the various scopes available, and what they mean for services consuming tokens.

## System Scope[¶](https://docs.openstack.org/keystone/latest/contributor/services.html#system-scope)

A *system-scoped* token implies the user has authorization to act on the *deployment system*. These tokens are useful for interacting with resources that affect the deployment as a whole, or exposes resources that may otherwise violate project or domain isolation.

Good examples of system-scoped resources include:

- Services: Service entities within keystone that describe the services deployed in a cloud.
- Endpoints: Endpoints that tell users where to find services deployed in a cloud.
- Hypervisors: Physical compute infrastructure that hosts instances where the instances may, or may not, be owned by the same project.

## Domain Scope[¶](https://docs.openstack.org/keystone/latest/contributor/services.html#domain-scope)

A *domain-scoped* token carries a user’s authorization on a specific domain. Ideally, these tokens would be useful for listing resources aggregated across all projects with that domain. They can also be useful for creating entities that must belong to a domain. Users and groups are good examples of this. The following is an example of how a domain-scoped token could be used against a service.

## Project Scope[¶](https://docs.openstack.org/keystone/latest/contributor/services.html#project-scope)

A *project-scoped* token carries the role assignments a user has on a project. This type of scope is great for managing resources that fit nicely within project boundaries. Good examples of project-level resources that can be managed with project-scoped tokens are:

- Instances: Virtual compute servers that require a project association in order to be created.
- Volumes: Storage devices that can be attached to instances.

## Unscoped[¶](https://docs.openstack.org/keystone/latest/contributor/services.html#unscoped)

An *unscoped* token is a token that proves authentication, but doesn’t carry any authorization. Users can obtain unscoped tokens by simply proving their identity with credentials. Unscoped tokens can be exchanged for any of the various scoped tokens if a user has authorization on the requested scope.

# Roles

Like most OpenStack services, keystone protects its API using role-based access control (RBAC).

Users can access different APIs depending on the roles they have on a project, domain, or system.

As of the Rocky release, keystone provides three roles called `admin`, `member`, and `reader` by default. Operators can grant these roles to any actor (e.g., group or user) on any target (e.g., system, domain, or project).

## Roles Definitions[¶](https://docs.openstack.org/keystone/latest//admin/service-api-protection.html#roles-definitions)

The default roles imply one another. The `admin` role implies the `member` role, and the `member` role implies the `reader` role. This implication means users with the `admin` role automatically have the `member` and `reader` roles. Additionally, users with the `member` role automatically have the `reader` role. Implying roles reduces role assignments and forms a natural hierarchy between the default roles. It also reduces the complexity of default policies by making check strings short.

### Reader[¶](https://docs.openstack.org/keystone/latest//admin/service-api-protection.html#reader)

The `reader` role provides read-only access to resources within the system, a domain, or a project. Depending on the assignment scope, two users with the `reader` role can expect different API behaviors. For example, a user with the `reader` role on the system can list all projects within the deployment. A user with the `reader` role on a domain can only list projects within their domain.

By analyzing the scope of a role assignment, we increase the re-usability of the `reader` role and provide greater functionality without introducing more roles. For example, to accomplish this without analyzing assignment scope, you would need `system-reader`, `domain-reader`, and `project-reader` roles in addition to custom policies for each service.

### Member[¶](https://docs.openstack.org/keystone/latest//admin/service-api-protection.html#member)

Within keystone, there isn’t a distinct advantage to having the `member` role instead of the `reader` role. The `member` role is more applicable to other services. The `member` role works nicely for introducing granularity between `admin` and `reader` roles. Other services might write default policies that require the `member` role to create resources, but the `admin` role to delete them. For example, users with `reader` on a project could list instance, users with `member` on a project can list and create instances, and users with `admin` on a project can list, create, and delete instances. Service developers can use the `member` role to provide more flexibility between `admin` and `reader` on different scopes.

### Admin[¶](https://docs.openstack.org/keystone/latest//admin/service-api-protection.html#admin)

We reserve the `admin` role for the most privileged operations within a given scope. It is important to note that having `admin` on a project, domain, or the system carries separate authorization and are not transitive. For example, users with `admin` on the system should be able to manage every aspect of the deployment because they’re operators. Users with `admin` on a project shouldn’t be able to manage things outside the project because it would violate the tenancy of their role assignment (this doesn’t apply consistently since services are addressing this individually at their own pace).