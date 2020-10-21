---
layout: post
title:  "openstack使用application credential做认证"
date:   2020-10-21 23:00:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack使用application credential做认证
mathjax: true
typora-root-url: ../
---

# openstack使用application credential做认证

在使用application credential做认证的时候，需要提供username, application credential id和application credential secret（其中secret只会在创建的时候在界面上显示一遍，需要记下，否则后面是查询不到的）

有几个问题需要注意，auth url里面需要带上v3，比如`http://10.121.246.195:5000/v3`，这里没有v3是不行的，会报错

```shell
Making authentication request to http://10.121.246.195:5000/auth/tokens
Starting new HTTP connection (1): 10.121.246.195:5000
http://10.121.246.195:5000 "POST /auth/tokens HTTP/1.1" 404 232
Request returned failure status: 404
```

通过application credential获取到token

```shell
curl --location --request POST 'http://10.121.246.195:5000/v3/auth/tokens' \
--header 'Content-Type: application/json' \
--header 'X-Auth-Token: gAAAAABfj8-DJXHpmSwmOBTs8MTTjmnD8GO6bfsCoCP3mVVcP9KLqaz4dmnsq5htTCnmCoUH3VrTLan2YMKKPOtFceTjm3XZ6F8nNIVtTLRp4qjN4YJUIn2XHGLqo0D4xBaT5L01PmAXt2qcvNs0796X0hG8OEvA7P_jpbaCw0v1bQszRj-CwrKXYbHIUdUq9IiTWOVov50q' \
--data-raw '{
    "auth": {
        "identity": {
            "methods": [
                "application_credential"
            ],
            "application_credential": {
                "id": "d28f32170d09455ea16a1451c3974d9f",
                "secret": "password"
            }
        }
    }
}'
```

拿着获取到的token去nova list的时候，会得到如下错误：

```shell
2020-10-21 06:16:31.202 24 WARNING keystonemiddleware.auth_token [-] Cannot validate request with restricted access rules. Set service_type in [keystone_authtoken] to allow access rule validation.
```

需要在nova.conf里增加如下配置：

```
[keystone_authtoken]
service_type = compute
```

注意在使用application credential的时候，只需要指定用户名，也就是说application credential是跟用户绑定的。不需要指定scope

否则会有如下错误：

```shell
ERROR (Unauthorized): Error authenticating with application credential: Application credentials cannot request a scope. (HTTP 401) (Request-ID: req-fad71866-476d-43a2-bf3e-444ecc4d8910)
```

那怎么确定在哪个project下呢？会通过获取application credential里带的Project信息

```shell
    def _set_scope_from_app_cred(self, app_cred_info):
        app_cred_ref = self._lookup_app_cred(app_cred_info)
        LOG.info("rain app_cred_ref %s" % app_cred_ref)
        self._scope_data = (None, app_cred_ref['project_id'], None, None, None)
        return
        
2020-10-21 03:55:30.257 24 INFO keystone.auth.core [req-7a481631-7646-4365-a0e4-bcab76cc98ad - - - - -] rain app_cred_ref {'id': '90abf3e9f560499bb501e736e66eea9a', 'name': 'rain-test1', 'description': None, 'user_id': 'e0ba31dce4c84fbbbdd349ea13f9afb2', 'project_id': '8f40dc8fecf44b28911db64643a3d0ba', 'system': None, 'expires_at': None, 'unrestricted': False, 'roles': [{'id': '0072591c8736405ba78bc8f17531da70', 'name': 'member', 'domain_id': None, 'description': None, 'options': {'immutable': True}}, {'id': '3f1dc59a32fb4a2683c3a862e0272c7e', 'name': 'admin', 'domain_id': None, 'description': None, 'options': {'immutable': True}}, {'id': 'd1c1f0498398489d996ca6a8bd96eefa', 'name': 'heat_stack_owner', 'domain_id': None, 'description': None, 'options': {}}, {'id': 'de4b2da6561f4486ac170cd1c8d92ef0', 'name': 'reader', 'domain_id': None, 'description': None, 'options': {'immutable': True}}]}
```

在创建application credentials的时候有一个access rules

![img](/../assets/images/52871945.png)

如果不填写，就是allow all

如果填写，就只允许填写的那些access rule，比如

```
[
  {"service": "compute",
  "method": "POST",
  "path": "/v2.1/servers"},
  {"service": "compute",
  "method": "GET",
  "path": "/v2.1/servers"}  
] 
```

<img src="/../assets/images/image-20201021155816939.png" alt="image-20201021155816939" style="zoom:50%;" />

还有一个unrestricted

**Unrestricted**: By default, for security reasons, application  credentials are forbidden from being used for creating additional  application credentials or keystone trusts. If your application  credential needs to be able to perform these actions, check  "unrestricted".  

出于安全考虑，默认application  credentials是不允许创建application  credentials的，如果勾选了unrestricted，就没有这个限制了

如何通过application credential去访问service呢？这里有个示例

```python
from keystoneclient.auth.identity import v3
from keystoneclient import session
from novaclient import client as nova_client
from keystoneclient import session
from novaclient import client as nova_client
from keystoneauth1.identity.v3.application_credential import ApplicationCredential

ApplicationCredential(auth_url='http://10.121.246.195:5000/v3', application_credential_id="a832e4bc73b8475db399d84ebfe66040", application_credential_secret='password', username='admin') 
auth = ApplicationCredential(auth_url='http://10.121.246.195:5000/v3', application_credential_id="a832e4bc73b8475db399d84ebfe66040", application_credential_secret='password', username='aådmin')
sess = session.Session(auth=auth)
_nova_cli = nova_client.Client('2', session=sess)
_nova_cli.services.list()
[<Service: 3>, <Service: 12>, <Service: 24>, <Service: 39>, <Service: 54>, <Service: 69>, <Service: 3>, <Service: 15>, <Service: 27>, <Service: 42>, <Service: 45>, <Service: 48>, <Service: 51>, <Service: 54>]
```

