---
layout: post
title:  "helm template里的list"
date:   2021-06-01 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Deploy
excerpt: helm template里的list
mathjax: true
typora-root-url: ../

---

# helm template里的list

我有一个配置文件，某个字段是个list

```yaml
ocpadmin:
- user1
- user2
- user3
```

然后通过helm的template里，生成configmap，最终mount到pod里去

本来在configmap的template里是这样定义的

```yaml
ocpadmin: {{ .Values.ocpadmin }}
```

最终到了pod里的配置文件变成这样了

```yaml
ocpadmin: [user1 user2 user3]
```

其实我是想展开成list的

所以应该这样

```yaml
ocpadmin:
  {{- range .Values.ocpadmin }}
  {{ print "- " ( . ) }}
  {{- end }}
```

