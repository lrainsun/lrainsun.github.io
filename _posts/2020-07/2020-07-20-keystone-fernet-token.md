---
layout: post
title:  "openstack keystone fernet token"
date:   2020-07-20 23:00:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack keystone application credential
mathjax: true
typora-root-url: ../
---

# openstack keystone fernet token

## What is a fernet token?[¶](https://docs.openstack.org/keystone/pike/admin/identity-fernet-token-faq.html#what-is-a-fernet-token)

A fernet token is a bearer token that represents user authentication. Fernet tokens contain a limited amount of identity and authorization data in a [MessagePacked](http://msgpack.org/) payload. The payload is then wrapped as a [Fernet](https://github.com/fernet/spec) message for transport, where Fernet provides the required web safe characteristics for use in URLs and headers. The data inside a fernet token is protected using symmetric encryption keys, or fernet keys.

## What is a fernet key?[¶](https://docs.openstack.org/keystone/pike/admin/identity-fernet-token-faq.html#what-is-a-fernet-key)

A fernet key is used to encrypt and decrypt fernet tokens. Each key is actually composed of two smaller keys: a 128-bit AES encryption key and a 128-bit SHA256 HMAC signing key. The keys are held in a key repository that keystone passes to a library that handles the encryption and decryption of tokens.

## What are the different types of keys?[¶](https://docs.openstack.org/keystone/pike/admin/identity-fernet-token-faq.html#what-are-the-different-types-of-keys)

A key repository is required by keystone in order to create fernet tokens. These keys are used to encrypt and decrypt the information that makes up the payload of the token. Each key in the repository can have one of three states. The state of the key determines how keystone uses a key with fernet tokens. The different types are as follows:

**Primary key**:There is only ever one primary key in a key repository. The primary key is allowed to encrypt and decrypt tokens. This key is always named as the highest index in the repository.

**Secondary key**:A secondary key was at one point a primary key, but has been demoted in place of another primary key. It is only allowed to decrypt tokens. Since it was the primary at some point in time, its existence in the key repository is justified. Keystone needs to be able to decrypt tokens that were created with old primary keys.

**Staged key**:The staged key is a special key that shares some similarities with secondary keys. There can only ever be one staged key in a repository and it must exist. Just like secondary keys, staged keys have the ability to decrypt tokens. Unlike secondary keys, staged keys have never been a primary key. In fact, they are opposites since the staged key will always be the next primary key. This helps clarify the name because they are the next key staged to be the primary key. This key is always named as 0 in the key repository.

```shell
(keystone)[root@ci99qacmp001 /]# ls -al /etc/keystone/fernet-keys/
total 12
drwxrwx---. 2 keystone keystone 33 Jul 19 00:00 .
drwxr-xr-x. 1 keystone keystone 72 Jul 16 13:46 ..
-rw-------. 1 keystone keystone 44 Jul 19 00:00 0
-rw-------. 1 keystone keystone 44 Jul  8 00:00 4
-rw-------. 1 keystone keystone 44 Jul 12 00:00 5
```

## Where do I put my key repository?[¶](https://docs.openstack.org/keystone/pike/admin/identity-fernet-token-faq.html#where-do-i-put-my-key-repository)

The key repository is specified using the `key_repository` option in the keystone configuration file. The keystone process should be able to read and write to this location but it should be kept secret otherwise. Currently, keystone only supports file-backed key repositories.

```ini
[fernet_tokens]
key_repository = /etc/keystone/fernet-keys/
```

## What is the recommended way to rotate and distribute keys?[¶](https://docs.openstack.org/keystone/latest/admin/fernet-token-faq.html#what-is-the-recommended-way-to-rotate-and-distribute-keys)

The keystone-manage command line utility includes a key rotation mechanism. This mechanism will initialize and rotate keys but does not make an effort to distribute keys across keystone nodes. The distribution of keys across a keystone deployment is best handled through configuration management tooling, however ensure that the new primary key is distributed first. Use keystone-manage fernet_rotate to rotate the key repository.

```shell
(keystone-fernet)[root@ci99qacmp001 /]# cat /usr/bin/fernet-rotate.sh
#!/bin/bash
 
set -o errexit
set -o pipefail
 
keystone-manage --config-file /etc/keystone/keystone.conf fernet_rotate --keystone-user keystone --keystone-group keystone
 
/usr/bin/fernet-push.sh

(keystone-fernet)[root@ci99qacmp001 /]# ls -al /etc/keystone/fernet-keys/
total 12
drwxrwx---. 2 keystone keystone 33 Jul 19 00:00 .
drwxr-xr-x. 1 keystone keystone 46 Jul 13 04:00 ..
-rw-------. 1 keystone keystone 44 Jul 19 00:00 0
-rw-------. 1 keystone keystone 44 Jul  8 00:00 4
-rw-------. 1 keystone keystone 44 Jul 12 00:00 5
(keystone-fernet)[root@ci99qacmp001 /]# set -o errexit
(keystone-fernet)[root@ci99qacmp001 /]# set -o pipefail
(keystone-fernet)[root@ci99qacmp001 /]#
(keystone-fernet)[root@ci99qacmp001 /]# keystone-manage --config-file /etc/keystone/keystone.conf fernet_rotate --keystone-user keystone --keystone-group keystone
2020-07-20 07:50:43.010 2023 INFO keystone.common.fernet_utils [-] Starting key rotation with 3 key files: ['/etc/keystone/fernet-keys/4', '/etc/keystone/fernet-keys/5', '/etc/keystone/fernet-keys/0']
2020-07-20 07:50:43.010 2023 INFO keystone.common.fernet_utils [-] Created a new temporary key: /etc/keystone/fernet-keys/0.tmp
2020-07-20 07:50:43.011 2023 INFO keystone.common.fernet_utils [-] Current primary key is: 5
2020-07-20 07:50:43.012 2023 INFO keystone.common.fernet_utils [-] Next primary key will be: 6
2020-07-20 07:50:43.012 2023 INFO keystone.common.fernet_utils [-] Promoted key 0 to be the primary: 6
2020-07-20 07:50:43.012 2023 INFO keystone.common.fernet_utils [-] Become a valid new key: /etc/keystone/fernet-keys/0
2020-07-20 07:50:43.013 2023 INFO keystone.common.fernet_utils [-] Excess key to purge: /etc/keystone/fernet-keys/4
(keystone-fernet)[root@ci99qacmp001 /]# ls -al /etc/keystone/fernet-keys/
total 12
drwxrwx---. 2 keystone keystone 33 Jul 20 07:50 .
drwxr-xr-x. 1 keystone keystone 46 Jul 13 04:00 ..
-rw-------. 1 keystone keystone 44 Jul 20 07:50 0   (staged key)
-rw-------. 1 keystone keystone 44 Jul 12 00:00 5   (secondary key)
-rw-------. 1 keystone keystone 44 Jul 19 00:00 6   (primary key)
```

## Generate & Validate Token

```python
def generate_id_and_issued_at(self, token):
    token_payload_class = self._determine_payload_class_from_token(token)
    token_id = self.token_formatter.create_token(
        token.user_id,
        token.expires_at,
        token.audit_ids,
        token_payload_class,
        methods=token.methods,
        system=token.system,
        domain_id=token.domain_id,
        project_id=token.project_id,
        trust_id=token.trust_id,
        federated_group_ids=token.federated_groups,
        identity_provider_id=token.identity_provider_id,
        protocol_id=token.protocol_id,
        access_token_id=token.access_token_id,
        app_cred_id=token.application_credential_id
    )
    creation_datetime_obj = self.token_formatter.creation_time(token_id)
    issued_at = ks_utils.isotime(
        at=creation_datetime_obj, subsecond=True
    )
    return token_id, issued_at
 
 
def _determine_payload_class_from_token(self, token):
    if token.oauth_scoped:
        return tf.OauthScopedPayload
    elif token.trust_scoped:
        return tf.TrustScopedPayload
    elif token.is_federated:
        if token.project_scoped:
            return tf.FederatedProjectScopedPayload
        elif token.domain_scoped:
            return tf.FederatedDomainScopedPayload
        elif token.unscoped:
            return tf.FederatedUnscopedPayload
    elif token.application_credential_id:
        return tf.ApplicationCredentialScopedPayload
    elif token.project_scoped:
        return tf.ProjectScopedPayload
    elif token.domain_scoped:
        return tf.DomainScopedPayload
    elif token.system_scoped:
        return tf.SystemScopedPayload
    else:
        return tf.UnscopedPayload
 
 
def create_token(self, user_id, expires_at, audit_ids, payload_class,
                 methods=None, system=None, domain_id=None,
                 project_id=None, trust_id=None, federated_group_ids=None,
                 identity_provider_id=None, protocol_id=None,
                 access_token_id=None, app_cred_id=None):
    """Given a set of payload attributes, generate a Fernet token."""
    version = payload_class.version
    payload = payload_class.assemble(
        user_id, methods, system, project_id, domain_id, expires_at,
        audit_ids, trust_id, federated_group_ids, identity_provider_id,
        protocol_id, access_token_id, app_cred_id
    )
 
    versioned_payload = (version,) + payload
    serialized_payload = msgpack.packb(versioned_payload)
    token = self.pack(serialized_payload)
 
    # NOTE(lbragstad): We should warn against Fernet tokens that are over
    # 255 characters in length. This is mostly due to persisting the tokens
    # in a backend store of some kind that might have a limit of 255
    # characters. Even though Keystone isn't storing a Fernet token
    # anywhere, we can't say it isn't being stored somewhere else with
    # those kind of backend constraints.
    if len(token) > 255:
        LOG.info('Fernet token created with length of %d '
                 'characters, which exceeds 255 characters',
                 len(token))
 
    return token
```

```python
def validate_token(self, token):
    """Validate a Fernet token and returns the payload attributes.
 
    :type token: str
 
    """
    serialized_payload = self.unpack(token)
    versioned_payload = msgpack.unpackb(serialized_payload)
    version, payload = versioned_payload[0], versioned_payload[1:]
 
    for payload_class in _PAYLOAD_CLASSES:
        if version == payload_class.version:
            (user_id, methods, system, project_id, domain_id,
             expires_at, audit_ids, trust_id, federated_group_ids,
             identity_provider_id, protocol_id, access_token_id,
             app_cred_id) = payload_class.disassemble(payload)
            break
    else:
        # If the token_format is not recognized, raise ValidationError.
        raise exception.ValidationError(_(
            'This is not a recognized Fernet payload version: %s') %
            version)
 
    # FIXME(lbragstad): Without this, certain token validation tests fail
    # when running with python 3. Once we get further along in this
    # refactor, we should be better about handling string encoding/types at
    # the edges of the application.
    if isinstance(system, bytes):
        system = system.decode('utf-8')
 
    # rather than appearing in the payload, the creation time is encoded
    # into the token format itself
    issued_at = TokenFormatter.creation_time(token)
    issued_at = ks_utils.isotime(at=issued_at, subsecond=True)
    expires_at = timeutils.parse_isotime(expires_at)
    expires_at = ks_utils.isotime(at=expires_at, subsecond=True)
 
    return (user_id, methods, audit_ids, system, domain_id, project_id,
            trust_id, federated_group_ids, identity_provider_id,
            protocol_id, access_token_id, app_cred_id, issued_at,
            expires_at)
```

