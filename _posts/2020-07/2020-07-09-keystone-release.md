---
layout: post
title:  "openstack Keystone Release Highlights"
date:   2020-07-09 23:00:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack Keystone Release Highlights
mathjax: true
typora-root-url: ../
---

# openstack Keystone Release Highlights (N -> U)

# New Feature

【framework】Keystone has historically used a custom rolled WSGI framework based loosely on [[webob](https://github.com/Pylons/webob)] which was in turn loaded by the [[pythonpaste library](https://paste.readthedocs.io/en/latest/)]. The Keystone team has been planning to move away from the home-rolled solution and to a common framework for a number of release cycles. As of the Rocky release Keystone is moving to the `Flask` framework.

【api】v2 → v3

【python】python2.7 → python3.6

【token】Keystone now supports a JSON Web Signature (JWS) token provider in addition to fernet tokens. Fernet token remain the default token provider.

https://docs.openstack.org/keystone/ussuri/admin/token-provider.html

https://specs.openstack.org/openstack/keystone-specs/specs/keystone/stein/json-web-tokens.html

【MFA】[[blueprint per-user-auth-plugin-reqs](https://blueprints.launchpad.net/keystone/+spec/per-user-auth-plugin-reqs)] Per-user Multi-Factor-Auth rules (MFA Rules) have been implemented. These rules define which auth methods can be used (e.g. Password, TOTP) and provides the ability to require multiple auth forms to successfully get a token.

https://docs.openstack.org/keystone/latest/admin/multi-factor-authentication.html

https://docs.openstack.org/keystone/pike/advanced-topics/auth-totp.html

【healthcheck】Health check mechanism allows an operator to configure the endpoint URL that will provide information to a load balancer if the given API endpoint at the node should be available or not.

https://docs.openstack.org/keystone/latest/admin/health-check-middleware.html

【application credential】[[blueprint application-credentials](https://blueprints.launchpad.net/keystone/+spec/application-credentials)] Users can now create Application Credentials, a new keystone resource that can provide an application with the means to get a token from keystone with a preset scope and role assignments. To authenticate with an application credential, an application can use the normal token API with the ‘application_credential’ auth method.

https://docs.openstack.org/keystone/latest/user/application_credentials.html

【unified limit】[[blueprint unified-limit](https://blueprints.launchpad.net/keystone/+spec/unified-limit)] Keystone now supports unified limits. Two resouces called `registered limit` and `limit` are added and a batch of related APIs are supported as well. These APIs are experimental now. It means that they are not stable enough and may be changed without backward compatibility. Once unified limit feature are ready for consuming, the APIs will be marked as stable.

https://docs.openstack.org/keystone/latest/admin/unified-limits.html

【system scope】[[blueprint system-scope](https://blueprints.launchpad.net/keystone/+spec/system-scope)] Keystone now supports the ability to assign roles to users and groups on the system. As a result, users and groups with system role assignment will be able to request system-scoped tokens. Additional logic has been added to `keystone-manage bootstrap` to ensure the administrator has a role on the project and system.

【default role】[[blueprint basic-default-roles](https://blueprints.launchpad.net/keystone/+spec/basic-default-roles)] Support has been added for deploying two new roles during the bootstrap process, reader and member, in addition to the admin role.

# Enhancement

【token】[[blueprint allow-expired](https://blueprints.launchpad.net/keystone/+spec/allow-expired)] An allow_expired flag is added to the token validation call (`GET/HEAD /v3/auth/tokens`) that allows fetching a token that has expired. This allows for validating tokens in long running operations.

https://specs.openstack.org/openstack/keystone-specs/specs/keystone/ocata/allow-expired.html

【token】[[bug 1642457](https://bugs.launchpad.net/keystone/+bug/1642457)] Handle disk write and IO failures when rotating keys for Fernet tokens. Rather than creating empty keys, properly catch and log errors when unable to write to disk.

【token】The token provider API has removed the `needs_persistence` property from the abstract interface. Token providers are expected to handle persistence requirement if needed. This will require out-of-tree token providers to remove the unused property and handle token storage.

【token】The option `[token] infer_roles=False` is being deprecated in favor of always expanding role implications during token validation. [Default roles](https://specs.openstack.org/openstack/keystone-specs/specs/keystone/rocky/define-default-roles.html) depend on a chain of implied role assignments, ex: an admin user will also have the reader and member role. Therefore by ensuring that all these roles will always appear on the token validation response, we can improve the simplicity and readability of policy files.

【policy】[[`blueprint policy-in-code](https://blueprints.launchpad.net/keystone/+spec/policy-in-code)] Keystone now supports the ability to register default policies in code. This makes policy file maintenance easier by allowing duplicated default policies to be removed from the policy file. The only policies that should exist within a deployment’s policy file after Pike should be policy overrides. Note that there is no longer a default value for the default rule. That rule is only checked when the more specific rule cannot be found, and with policy in code all rules should be found in code even if they are not in the policy file. To generate sample policy files from default values, prune default policies from existing policy files, or familiarize yourself with general policy usage, please see the [usage documentation](https://docs.openstack.org/developer/oslo.policy/usage.html) provided in oslo.policy.

https://specs.openstack.org/openstack/keystone-specs/specs/keystone/pike/policy-in-code.html

【upgrade】Added an option `--check` to `keystone-manage db_sync`, the option will allow a user to check the status of rolling upgrades in the database.

【upgrade】[[Community Goal](https://governance.openstack.org/tc/goals/stein/upgrade-checkers.html)] Support has been added for developers to write pre-upgrade checks. Operators can run these checks using `keystone-status upgrade check`. This allows operators to be more confident when upgrading their deployments by having a tool that automates programmable checks against the deployment configuration or dataset.

【password】[[bug 1641645](https://bugs.launchpad.net/keystone/+bug/1641645)] RBAC protection was removed from the Self-service change user password API (`/v3/user/$user_id/password`), meaning, a user can now change their password without a token specified in the `X-Auth-Token` header. This change will allow a user, with an expired password, to update their password without the need of an administrator.

【federation】[[bug 1571878](https://bugs.launchpad.net/keystone/+bug/1571878)] A valid `mapping_id` is now required when creating or updating a federation protocol. If the `mapping_id` does not exist, a `400 - Bad Request` will be returned.

【federation】[[blueprint shadow-mapping](https://blueprints.launchpad.net/keystone/+spec/shadow-mapping)] The federated identity mapping engine now supports the ability to automatically provision `projects` for `federated users`. A role assignment will automatically be created for the user on the specified project. If the project specified within the mapping does not exist, it will be automatically created in the `domain` associated with the `identity provider`. This behavior can be triggered using a specific syntax within the `local` rules section of a mapping.

https://specs.openstack.org/openstack/keystone-specs/specs/keystone/ocata/shadow-mapping.html

【project】[[bug 1523369](https://bugs.launchpad.net/keystone/+bug/1523369)] Deleting a project will now cause it to be removed as a default project for users. If caching is enabled the changes may not be visible until the user’s cache entry expires.

【project】[[blueprint project-tags](https://blueprints.launchpad.net/keystone/+spec/project-tags)] Projects have a new property called tags. These tags are simple strings that can be used to allow projects to be filtered/searched.

【role】[[bug 1669080](https://bugs.launchpad.net/keystone/+bug/1669080)] Added support for a `description` attribute for V3 Identity Roles, see API docs for details.

【role】[[bug 1823258](https://bugs.launchpad.net/keystone/+bug/1823258)] Adds support for an “immutable” resource option for roles, which when enabled prevents accidental harmful modification or deletion of roles. Also adds a new flag `--immutable-roles` to the `keystone-manage bootstrap` command to make the default roles (admin, member, and reader) immutable by default, as well as a check in the `keystone-status upgrade check` command to check that these roles have been made immutable. In a future release, these three roles will be immutable by default.

【trust】The `enabled` config option of the `trust` feature is deprecated and will be removed in the next release. Trusts will then always be enabled.

【bootstrap】[[bug 1749268](https://bugs.launchpad.net/keystone/+bug/1749268)] The `keystone-manage bootstrap` command now ensures that an administrator has a system role assignment. This prevents the ability for operators to lock themselves out of system-level APIs.

【bootstrap】The `keystone-manage bootstrap` command can now be used to update existing endpoints idempotently, which is useful in conjunction with configuration management tools that use this command for both initialization and lifecycle management of keystone.

【policy】[[bug 1810983](https://bugs.launchpad.net/keystone/+bug/1810983)] With the removal of KeystoneToken from the token model, we longer have the ability to use the token data syntax in the policy rules. This change broke backward compatibility for anyone deploying customized Keystone policies. Unfortunately, we can’t go back to KeystoneToken model as the change was tightly coupled with the other refactored authorization functionalities.

Since the scope information is now available in the credential dictionary, we can just make use of it instead. Those who have custom policies must update their policy files accordingly.

![image-20200709155248307](/../assets/images/image-20200709155248307.png)

【unified limit】[[bug 1754185](https://bugs.launchpad.net/keystone/+bug/1754185)] Registered limits and project limits now support an optional, nullable property called description. Users can create/update a registered limit or project limit with description now.

【unified limit】[[blueprint strict-two-level-model](https://blueprints.launchpad.net/keystone/+spec/strict-two-level-model)] A new limit enforcement model called strict_two_level is added. Change the value of the option [unified_limit]/enforcement_model to strict_two_level to enable it. In this [[model](http://specs.openstack.org/openstack/keystone-specs/specs/keystone/rocky/strict-two-level-enforcement-model.html)]: 1. The project depth is force limited to 2 level. 2. Any child project’s limit can not exceed the parent’s. Please ensure that the previous project and limit structure deployment in your Keystone won’t break this model before starting to use it. If a newly created project results in a project tree depth greater than 2, a 403 Forbidden error will be raised. When try to use this model but the project depth exceed 2 already, Keystone process will fail to start. Operators should choose another available model to fix the issue first.

【unified limit】[[blueprint strict-two-level-model](https://blueprints.launchpad.net/keystone/+spec/strict-two-level-model)] The project_id filter is added for listing limits. This filter is used for system-scoped request only to fetch the specified project limits. Non system-scoped request will get empty response body instead.

【unified limit】[[blueprint strict-two-level-model](https://blueprints.launchpad.net/keystone/+spec/strict-two-level-model)] The include_limits filter is added to GET /v3/projects/{project_id} API. This filter should be used together with parents_as_list or subtree_as_list filter to add parent/sub project’s limit information the response body.

【unified limit】[[bug 1779903](https://bugs.launchpad.net/keystone/+bug/1779903)] When a project is deleted, the limits which belong to it will be deleted as well.

【unified limit】[[blueprint domain-level-limit](https://blueprints.launchpad.net/keystone/+spec/domain-level-limit)] Keystone now supports domain level unified limit. When creating a limit, users can specify a `domain_id` instead of `project_id`. For flat model, the domain limit is still non-hierarchical. For strict-two-level model, the domain limit is now considered as the first level, so that the project limit is the second level and the project can’t contain any child.

【MFA】[[blueprint mfa-auth-receipt](https://blueprints.launchpad.net/keystone/+spec/mfa-auth-receipt)] Added support for auth receipts. Allows multi-step authentication for users with configured MFA Rules. Partial authentication with successful auth methods will return an auth receipt that can be consumed in subsequent auth attempts along with the missing auth methods to complete auth and be provided with a valid token.

【credential】[[bug 1788415](https://bugs.launchpad.net/keystone/+bug/1788415)] [[bug 968696](https://bugs.launchpad.net/keystone/+bug/968696)] Policies protecting the `/v3/credentials` API have changed defaults in order to make the credentials API more accessible for all users and not just operators or system administrator. Please consider these updates when using this version of keystone since it could affect API behavior in your deployment, especially if you’re using a customized policy file.

【resource driver】Restores the configurability of the resource driver, so it is now possible to create a custom resource driver if the built-in sql driver does not meet business requirements.

# Deprecation Notes

【token】The UUID, PKI and PKIz token format has been removed

【api】[[blueprint removed-as-of-queens](https://blueprints.launchpad.net/keystone/+spec/removed-as-of-queens)] Support for all Identity V2 APIs, with the exception of the EC2 v2 API, has been removed from keystone.

【api】The `policies` API is deprecated. Keystone is not a policy management service.

【middleware】The keystone.middleware.core:TokenAuthMiddleware is deprecated for removal.

The functionality of the `ADMIN_TOKEN` remains, but has been incorporated into the main auth middleware (`keystone.middleware.auth.AuthContextMiddleware`).

The token_auth middleware functionality has been merged into the main auth middleware (keystone.middleware.auth.AuthContextMiddleware). admin_token_auth must be removed from the [pipeline:api_v3], [pipeline:admin_api], and [pipeline:public_api] sections of your paste ini file. The [filter:token_auth] block will also need to be removed from your paste ini file. Failure to remove these elements from your paste ini file will result in keystone to no longer start/run when the token_auth is removed in the Stein release.

【framework】Keystone no longer is loaded via `paste.deploy` and instead directly loads the `Flask` based application. If a deployment is relying on the entry-point generated wsgi files, it is important to get the newest ones. These new files have minor changes to support the new loading mechanisms. The files will be auto-generated via `PBR` and setup. The `paste.ini` file will now be ignored, but will remain on disk until the `Stein` release to ensure deployment tools are not inadvertently broken. The `paste.ini` file will have a comment added to indicate it is ignored.

【python】Dropping the Python2 support in OpenStack Ussuri according to [the TC deprecation timeline](https://governance.openstack.org/tc/resolutions/20180529-python2-deprecation-timeline.html)

【policy】[[bug 1806762](https://bugs.launchpad.net/keystone/+bug/1806762)] [[bug 1630434](https://bugs.launchpad.net/keystone/+bug/1630434)] The entire `policy.v3cloudsample.json` file has been removed. If you were using this policy file to supply overrides in your deployment, you should consider using the defaults in code and setting `keystone.conf [oslo_policy] enforce_scope=True`. The new policy defaults are more flexible, they’re tested extensively, and they solve all the problems the `policy.v3cloudsample.json` file was trying to solve.

# References

[1] [https://docs.openstack.org/keystone/latest/](https://docs.openstack.org/keystone/latest/)

[2] [https://www.openstack.org/software/Ussuri](https://www.openstack.org/software/Ussuri)

[3] [https://docs.openstack.org/releasenotes/keystone/ussuri.html](https://docs.openstack.org/releasenotes/keystone/ussuri.html)