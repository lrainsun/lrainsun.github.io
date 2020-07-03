---
layout: post
title:  "openstack keystone ocata版本变更"
date:   2020-07-03 23:00:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack keystone ocata版本变更
mathjax: true
typora-root-url: ../
---

# openstack keystone ocata版本变更

## token

关于token ocata有几个改进

### allow expired

token有个超期时间，这个我们都知道，o版本多加了一个配置

```shell
[token]
expiration = 86400
allow_expired_window = 172800
```

会有这样的情况，一些service的比较长的operations，可能在token超期后还没有完成，就会造成失败，改进是允许通过allow_expired flag，在规定的时间内可以仍然可以获取超期token，allow_expired_window的默认值是2天

### token provider

默认的token provider改成fernet了，而且，pki跟pkiz被remove掉了

fernet对比uuid的好处在于，uuid是需要持久化在数据库的，一旦规模上去了，每次validate token都需要去检索数据，这个容易造成性能问题 —— 所以需要定期的清理数据库中的token。

而fernet是一种对称加密的算法，它采用 [cryptography](http://cryptography.readthedocs.org/en/latest/fernet/) 对称加密库(symmetric cryptography，加密密钥和解密密钥相同) 加密 token，具体由 AES-CBC 加密和散列函数 SHA256 签名，它携带了少量的用户信息（user_id，project_id，domain_id，methods，expires_at 等信息），大小大概在255 byte左右，并且不需要存储在数据库中。而为了提高安全性，需要采用key rotation来更换密钥，这个密钥是放在disk上的一个文件。

而在将token provider升级到fernet token的时候，首先需要有key repository & key distribution mechanism，否则token validation会不工作

当以keystone-manage fernet-setup生成密钥时，会看到0、1两个索引表征

 ![img](/../assets/images/b7840a60-4a43-4ea3-9d78-29e90459a561.png)

三个概念：

* primary key（主密钥）有且只有一个，名为为x，当前用于加密解密token
* secondary key（次次密钥）有x-1个，从Primary退役下来的，用于解密当初它加密过的token
* staged key（次密钥）有且只有一个，命名为0，准备下一个rotation时变为Primary key，可以解密token

上述0 表示的是staged key，1 表示的是primary key，primary key相比较另外两种key，它的索引最高，并且可以加密、也可以解密；staged key 相较于secondary key，它更有机会变为primary key。

```shell
(keystone)[root@ci99qacmp001 /]$ ls -al /etc/keystone/fernet-keys/
total 8
drwxrwx---. 2 keystone keystone 24 Jul  1 05:07 .
drwxr-xr-x. 1 keystone keystone 72 Jul  3 06:54 ..
-rw-------. 1 keystone keystone 44 Jul  1 05:07 0
-rw-------. 1 keystone keystone 44 Jul  1 05:07 1
```

### cache_on_issue

tokne下的cache_on_issue配置默认改成enabled

当global的caching enable的时候，应该也要enable token的cache on issue，而global caching disable的时候，这个配置enable也不会起到作用，所以默认是enabled最好

使用cache可以提高性能，即使像fernet token这样不需要持久化的，也还是可以通过cache来优化token validation的性能

### revocation events

在token validation的时候，减少了revocation events的返回数，只返回一部分跟这个token相关的events，这样可以提高token validation的性能 

## password

### password expires at

token reponse会在user对象返回password_expires_at字段：

```json
{"token": {"user": {"password_expires_at": null}}}
```

如果pci（Payment Card Industry）enabled，根据[security_compliance]的配置项，这里会返回一个timestamp，否则会是null（说明密码不会过期）

### PCI-DSS events

CADF通知会扩展成PCI-DSS（Payment Card Industry - Data Security Standard）events，通知里会加一个reason对象（包括reason type —— 原因的简短介绍, reason code —— http返回code）

下面这些events会受到影响

> - If a user does not change their passwords at least once every X days. See `[security_compliance] password_expires_days`.
> - If a user is locked out after many failed authentication attempts. See `[security_compliance] lockout_failure_attempts`.
> - If a user submits a new password that was recently used. See `[security_compliance] unique_last_password_count`.
> - If a password does not meet the specified criteria. See `[security_compliance] password_regex`.
> - If a user attempts to change their password too often. See `[security_compliance] minimum_password_age`.

### change password upon first use

新增了一个pci-dss feature：当密码被重置的时候，强制要求用户立即修改他们的密码

### per-user multi-factor-auth rules

是一个针对特定用户可以自定义认证方法

### update password without auth

通过/v3/user/$user_id/password可以不需要认证（X-Auth-Token）修改自己的密码，也就是说，一个用户的密码如果过期了，可以自己更新密码而不需要管理员的帮助

## federation

### shadow mapping

可以自动地为一个federated authentication成功后，可以自动为其user创建一个project或者assign一个role

### domain

identity provider会被关联到一个domain（domain_id），新创建的identity provider会有一个uuid：domain id，一个name（匹配identity provider id）

## health check

oslo.middleware的healthcheck中间件会被默认加到keystone的application pipeline

## notification

notification_format default值从默认的basic改成了cadf，cadf notification包含了更多的用户信息

`notification_opt_out` 默认配置：

`identity.authenticate.success`, `identity.authenticate.pending` and `identity.authenticate.failed`. 

## policy

list_revoke_events本来只要是authenticated user就可以，现在改成需要admin或者service user

