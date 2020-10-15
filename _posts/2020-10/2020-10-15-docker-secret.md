---
layout: post
title:  "docker /run/secrets mount错误"
date:   2020-10-15 23:00:00 +0800
categories: Docker
tags: Docker-Service
excerpt: docker /run/secrets mount错误
mathjax: true
---

# docker /run/secrets mount错误

这两天在跑kolla ansible的时候持续碰到一个问题

```shell
[2020-10-13 08:49:11,439] {logging_mixin.py:112} WARNING - fatal: [ocp-dev-sjc02-a-edge-1]: FAILED! => {"changed": true, "details": "{\"changed\": true, \"msg\": \"'Traceback (most recent call last):\\\\n  File \\\"/tmp/ansible_kolla_docker_payload_QavY4R/ansible_kolla_docker_payload.zip/ansible/modules/kolla_docker.py\\\", line 1024, in main\\\\n  File \\\"/tmp/ansible_kolla_docker_payload_QavY4R/ansible_kolla_docker_payload.zip/ansible/modules/kolla_docker.py\\\", line 759, in recreate_or_restart_container\\\\n  File \\\"/tmp/ansible_kolla_docker_payload_QavY4R/ansible_kolla_docker_payload.zip/ansible/modules/kolla_docker.py\\\", line 779, in start_container\\\\n  File \\\"/usr/lib/python2.7/site-packages/docker/utils/decorators.py\\\", line 19, in wrapped\\\\n    return f(self, resource_id, *args, **kwargs)\\\\n  File \\\"/usr/lib/python2.7/site-packages/docker/api/container.py\\\", line 1108, in start\\\\n    self._raise_for_status(res)\\\\n  File \\\"/usr/lib/python2.7/site-packages/docker/api/client.py\\\", line 261, in _raise_for_status\\\\n    raise create_api_error_from_http_exception(e)\\\\n  File \\\"/usr/lib/python2.7/site-packages/docker/errors.py\\\", line 31, in create_api_error_from_http_exception\\\\n    raise cls(e, response=response, explanation=explanation)\\\\nNotFound: 404 Client Error: Not Found (\\\"oci runtime error: container_linux.go:247: starting container process caused \\\"process_linux.go:364: container init caused \\\\\\\\\\\"rootfs_linux.go:54: mounting \\\\\\\\\\\\\\\\\\\\\\\\\\\"/var/lib/docker/containers/ee75368f7b0290313a3cc69831a0a346adb860135e88ee3c0736d063422b8720/secrets\\\\\\\\\\\\\\\\\\\\\\\\\\\" to rootfs \\\\\\\\\\\\\\\\\\\\\\\\\\\"/var/lib/docker/overlay2/8773db707d40e4034259ab64714375464c0383a98a1286f025cf02878e1e64a0/merged\\\\\\\\\\\\\\\\\\\\\\\\\\\" at \\\\\\\\\\\\\\\\\\\\\\\\\\\"/var/lib/docker/overlay2/8773db707d40e4034259ab64714375464c0383a98a1286f025cf02878e1e64a0/merged/run/secrets\\\\\\\\\\\\\\\\\\\\\\\\\\\" caused \\\\\\\\\\\\\\\\\\\\\\\\\\\"no such file or directory\\\\\\\\\\\\\\\\\\\\\\\\\\\"\\\\\\\\\\\"\\\"\\\\n\\\")\\\\n'\"}", "ignore_errors": null, "msg": "'Traceback (most recent call last):\\n  File \"/tmp/ansible_kolla_docker_payload_QavY4R/ansible_kolla_docker_payload.zip/ansible/modules/kolla_docker.py\", line 1024, in main\\n  File \"/tmp/ansible_kolla_docker_payload_QavY4R/ansible_kolla_docker_payload.zip/ansible/modules/kolla_docker.py\", line 759, in recreate_or_restart_container\\n  File \"/tmp/ansible_kolla_docker_payload_QavY4R/ansible_kolla_docker_payload.zip/ansible/modules/kolla_docker.py\", line 779, in start_container\\n  File \"/usr/lib/python2.7/site-packages/docker/utils/decorators.py\", line 19, in wrapped\\n    return f(self, resource_id, *args, **kwargs)\\n  File \"/usr/lib/python2.7/site-packages/docker/api/container.py\", line 1108, in start\\n    self._raise_for_status(res)\\n  File \"/usr/lib/python2.7/site-packages/docker/api/client.py\", line 261, in _raise_for_status\\n    raise create_api_error_from_http_exception(e)\\n  File \"/usr/lib/python2.7/site-packages/docker/errors.py\", line 31, in create_api_error_from_http_exception\\n    raise cls(e, response=response, explanation=explanation)\\nNotFound: 404 Client Error: Not Found (\"oci runtime error: container_linux.go:247: starting container process caused \"process_linux.go:364: container init caused \\\\\"rootfs_linux.go:54: mounting \\\\\\\\\\\\\"/var/lib/docker/containers/ee75368f7b0290313a3cc69831a0a346adb860135e88ee3c0736d063422b8720/secrets\\\\\\\\\\\\\" to rootfs \\\\\\\\\\\\\"/var/lib/docker/overlay2/8773db707d40e4034259ab64714375464c0383a98a1286f025cf02878e1e64a0/merged\\\\\\\\\\\\\" at \\\\\\\\\\\\\"/var/lib/docker/overlay2/8773db707d40e4034259ab64714375464c0383a98a1286f025cf02878e1e64a0/merged/run/secrets\\\\\\\\\\\\\" caused \\\\\\\\\\\\\"no such file or directory\\\\\\\\\\\\\"\\\\\"\"\\n\")\\n'"}
```

每次跑到recreate kolla-toolbox就会报错，我的解决办法是`umount /run/secretes`再重新跑，但是下一次再跑还是会出现同样的问题

装的版本是`docker.x86_64                    2:1.13.1-53.git774336d.el7.centos`

后来换了`docker-ce.x86_64                 3:19.03.13-3.el7        @docker                `才解决了这个问题，查到社区也有一些讨论

https://bugs.launchpad.net/kolla/+bug/1668059

https://bugzilla.redhat.com/show_bug.cgi?id=1410118

https://bugs.launchpad.net/kolla/+bug/1650778

```
Try to run docker daemon with flan --enable-secrets=false
```

```
From docker devs:
it's a bug in the Red Hat fork of docker, most likely due to their "super secret patch". This works fine on the official packages.
```

```
but after I fix the issue #6, origin issue is existed,Finally I find that the issue is disappeared after I use docker-engine-1.12.6-1.el7.centos.x86_64 this docker community version instead of redhat docker version docker-1.12.5-14.el7.centos.
```

