---
layout: post
title:  "Ephemeral disk format"
date:   2021-03-30 23:00:00 +0800
categories: Openstack
tags: Openstack-Nova
excerpt: Ephemeral disk format
mathjax: true
typora-root-url: ../
---

# Ephemeral disk format

我们发现当用户启动vm使用Ephemeral disk的时候，Ephemeral disk的format是vfat，导致chown等命令会报错

当image带上metadata os_type的时候

`openstack image set 3a8a28f1-d0ac-468a-8b6f-443b165c9c0c --property os_type=linux`

instance会从image继承这个属性

```python
def _inherit_properties_from_image(self, image, auto_disk_config):
    image_properties = image.get('properties', {})
    auto_disk_config_img = \
            utils.get_auto_disk_config_from_image_props(image_properties)
    self._ensure_auto_disk_config_is_valid(auto_disk_config_img,
                                           auto_disk_config,
                                           image.get("id"))
    if auto_disk_config is None:
        auto_disk_config = strutils.bool_from_string(auto_disk_config_img)

    return {
        'os_type': image_properties.get('os_type'),
        'architecture': image_properties.get('architecture'),
        'vm_mode': image_properties.get('vm_mode'),
        'auto_disk_config': auto_disk_config
    }
```

```python
    # Lookup the filesystem type if required
    os_type_with_default = nova.privsep.fs.get_fs_type_for_os_type(
        instance.os_type)
    # Generate a file extension based on the file system
    # type and the mkfs commands configured if any
    file_extension = nova.privsep.fs.get_file_extension_for_os_type(
        os_type_with_default, CONF.default_ephemeral_format)
    
def get_fs_type_for_os_type(os_type):
    global _MKFS_COMMAND

    return os_type if _MKFS_COMMAND.get(os_type) else 'default'
```

按理说传进get_fs_type_for_os_type的os_type是image里定义的linux，但是get_fs_type_for_os_type会去判断_MKFS_COMMAND

而

```python
_MKFS_COMMAND = {}
```

所以这里不管os_type传啥，都是会变成default的呀

```python
def get_file_extension_for_os_type(os_type, default_ephemeral_format,
                                   specified_fs=None):
    global _MKFS_COMMAND
    global _DEFAULT_MKFS_COMMAND

    mkfs_command = _MKFS_COMMAND.get(os_type, _DEFAULT_MKFS_COMMAND)
    if mkfs_command:
        extension = mkfs_command
    else:
        if not specified_fs:
            specified_fs = default_ephemeral_format
            if not specified_fs:
                specified_fs = _DEFAULT_FS_BY_OSTYPE.get(os_type,
                                                         _DEFAULT_FILE_SYSTEM)
        extension = specified_fs
    return _get_hash_str(extension)[:7]
```

workaround是定义一下CONF.default_ephemeral_format

```ini
default_ephemeral_format= ext4
```

