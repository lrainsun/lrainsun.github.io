---
layout: post
title:  "openstack keystone application credential"
date:   2020-07-16 23:00:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack keystone application credential
mathjax: true
typora-root-url: ../
---

# openstack keystone application credential

Users can create application credentials to allow their applications to authenticate to keystone. Users can delegate a subset of their role assignments on a project to an application credential, granting the application the same or restricted authorization to a project. With application credentials, applications authenticate with the application credential ID and a secret string which is not the user’s password. This way, the user’s password is not embedded in the application’s configuration, which is especially important for users whose identities are managed by an external system such as LDAP or a single-signon system.

See the [Identity API reference](https://docs.openstack.org/api-ref/identity/v3/index.html#application-credentials) for more information on authenticating with and managing application credentials.

# api

 POST
  /v3/auth/tokens

Authenticating with an Application Credential

 POST
/v3/users/{user_id}/application_credentials

Create application credential

 GET
/v3/users/{user_id}/application_credentials

List application credentials

 GET
/v3/users/{user_id}/application_credentials/{application_credential_id}

Show application credential details

 DELETE
/v3/users/{user_id}/application_credentials/{application_credential_id}

Delete application credential

 GET
/v3/users/{user_id}/access_rules

List access rules

 GET
/v3/users/{user_id}/access_rules/{access_rule_id}

Show access rule details

 DELETE
/v3/users/{user_id}/access_rules/{access_rule_id}

Delete access rule

# dashboard

![image-20200716221339806](/../assets/images/image-20200716221339806.png)

for example

```shell
curl --location --request POST 'http://10.100.32.50:5000/v3/auth/tokens' \
--header 'Content-Type: application/json' \
--data-raw '{
    "auth": {
        "identity": {
            "methods": [
                "application_credential"
            ],
            "application_credential": {
                "id": "**************",
                "secret": "*************************"
            }
        }
    }
}'
```

