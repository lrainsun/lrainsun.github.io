---
layout: post
title:  "Ansible Python API role path"
date:   2021-02-26 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: Ansible Python API role path
mathjax: true
typora-root-url: ../
---

# Ansible Python API role path

昨天在调试ansible的时候，由于role并不都在base url下，所以需要单独配置role path，本来是直接在context.CLIARGS里传了roles_path，但是调试下来发现并不起作用

查看了ansible代码`ansible//playbook/role/definition.py`:

```python
        # We didn't find a collection role, look in defined role paths
        # FUTURE: refactor this to be callable from internal so we can properly order
        # ansible.legacy searches with the collections keyword

        # we always start the search for roles in the base directory of the playbook
        role_search_paths = [
            os.path.join(self._loader.get_basedir(), u'roles'),
        ]

        # also search in the configured roles path
        if C.DEFAULT_ROLES_PATH:
            role_search_paths.extend(C.DEFAULT_ROLES_PATH)

        # next, append the roles basedir, if it was set, so we can
        # search relative to that directory for dependent roles
        if self._role_basedir:
            role_search_paths.append(self._role_basedir)

        # finally as a last resort we look in the current basedir as set
        # in the loader (which should be the playbook dir itself) but without
        # the roles/ dir appended
        role_search_paths.append(self._loader.get_basedir())

        # now iterate through the possible paths and return the first one we find
        for path in role_search_paths:
            path = templar.template(path)
            role_path = unfrackpath(os.path.join(path, role_name))
            if self._loader.path_exists(role_path):
                return (role_name, role_path)
```

只会在下面这几个地方search role：

* C.DEFAULT_ROLES_PATH 
* _role_basedir
* basedir
* ansible.cfg里配置的roles_path

basedir就是run ansible的目录

C.DEFAULT_ROLES_PATH在`ansible/config/base.yml`这里定义：

```yaml
DEFAULT_ROLES_PATH:
  name: Roles path
  default: ~/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles
  description: Colon separated paths in which Ansible will search for Roles.
  env: [{name: ANSIBLE_ROLES_PATH}]
  expand_relative_paths: True
  ini:
  - {key: roles_path, section: defaults}
  type: pathspec
  yaml: {key: defaults.roles_path}
```

就是从ANSIBLE_ROLES_PATH环境变量取到的值

但是尝试了在程序运行期间去改配置的roles_path，配置是可以改掉，但是并没有生效。尝试修改环境变量也没有生效。

最后试了直接修改DEFAULT_ROLES_PATH，才起了作用

```python
constants.DEFAULT_ROLES_PATH = [self.roles_path]
```

