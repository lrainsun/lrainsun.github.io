---
layout: post
title:  "Ansible ssh timeout"
date:   2021-03-21 23:00:00 +0800
categories: Ansible
tags: Ansible
excerpt: Ansible ssh timeout
mathjax: true
typora-root-url: ../
---

# Ansible ssh timeout

在跑ansible的过程中，总是收到如下提示`Timeout (2s) waiting for privilege escalation prompt`

但是通过ansible-playbook跑就没问题，通过我们python代码去跑就会失败

网上提示跟ansible.cfg里面的timeout配置有关

```yaml
[defaults]
# SSH timeout
timeout = 30
```

如果把这个值改小，就会出现如上的问题，但我们明明配置了默认值30呀

后来发现，通过Python去调用的时候，有一个Play的context，里面会带一个timeout的参数，如果这里面不传的话，默认就会把这个timeout设置成0

`/Users/minsu/.pyenv/versions/ansible/lib/python3.7/site-packages/ansible/plugins/connection/ssh.py`

```python
        # select timeout should be longer than the connect timeout, otherwise
        # they will race each other when we can't connect, and the connect
        # timeout usually fails
        timeout = 2 + self._play_context.timeout
        for fd in (p.stdout, p.stderr):
            fcntl.fcntl(fd, fcntl.F_SETFL, fcntl.fcntl(fd, fcntl.F_GETFL) | os.O_NONBLOCK)

        # TODO: bcoca would like to use SelectSelector() when open
        # filehandles is low, then switch to more efficient ones when higher.
        # select is faster when filehandles is low.
        selector = selectors.DefaultSelector()
        selector.register(p.stdout, selectors.EVENT_READ)
        selector.register(p.stderr, selectors.EVENT_READ)

        # If we can send initial data without waiting for anything, we do so
        # before we start polling
        if states[state] == 'ready_to_send' and in_data:
            self._send_initial_data(stdin, in_data, p)
            state += 1

        try:
            while True:
                poll = p.poll()
                events = selector.select(timeout)

                # We pay attention to timeouts only while negotiating a prompt.

                if not events:
                    # We timed out
                    if state <= states.index('awaiting_escalation'):
                        # If the process has already exited, then it's not really a
                        # timeout; we'll let the normal error handling deal with it.
                        if poll is not None:
                            break
                        self._terminate_process(p)
                        raise AnsibleError('Timeout (%ds) waiting for privilege escalation prompt: %s' % (timeout, to_native(b_stdout)))
```

在`timeout = 2 + self._play_context.timeout`时间内，如果没有获取到target host的response，就会raise timeout的提示，当目标环境网络比较慢的时候，就可能会出现超时。

context的初始化

```python
context.CLIARGS = ImmutableDict(timeout=60, ......)
```

d