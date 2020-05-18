---
layout: post
title:  "alert manager email模板配置"
date:   2020-05-18 23:00:00 +0800
categories: Prometheus
tags: Prometheus-AlertManager
excerpt: alert manager email模板配置
mathjax: true
typora-root-url: ../
---

# alert manager email模板配置

今天跟alert manager的email模板奋斗了一天

alertmanager本身有一个默认的模板，发alert邮件的时候，如果告警比较多的时候，那看起来就很不方便了。

![image-20200518203019890](/../assets/images/image-20200518203019890.png)

比如一条告警就占地面的那么大，如果告警比较多的话，就要不停的往下拖

所以想要改造成表格的形式

# external url

首先邮件这边可以有个link，连接到alertmanager

![image-20200518203119627](/../assets/images/image-20200518203119627.png)

`{{ define "__alertmanagerURL" }}{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver | urlquery }}{{ end }}`

那么这个externalurl是要在启动alertmanager的时候填进去的，像这样：

```shell
    docker run -itd \
        -p 9093:9093 \
        -v /root/ocp-monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
        -v /root/ocp-monitoring/email.ocpalert.tmpl:/etc/alertmanager/templates/email.ocpalert.tmpl \
        --name $NAME \
        prom/alertmanager:v0.20.0 --config.file=/etc/alertmanager/alertmanager.yml --web.external-url=http://10.225.28.176:9093
```

# go template中的multi condition

然后因为表格太长了，为了看起来方便，我把所有的label都打印出来放在表格里

![image-20200518203735780](/../assets/images/image-20200518203735780.png)

而其实，alertname, job, severity等等内容，都是一样的，完全没必要每行都打印一下，所以想把这三个内容提到上面标题去，在表格里去掉。就这么个问题，纠结了好久。

## GroupLabels

我们在配置alertmanger配置文件的时候，可以把alert group起来，而把alert group起来的定义，就是groupLabels

```yaml
route:
  group_by: ['alertname', 'job', 'severity']
```

这样定义了之后，在template里就可以这样

```go
          {{ .Alerts | len }} alert{{ if gt (len .Alerts) 1 }}s{{ end }} for {{ range .GroupLabels.SortedPairs }}
            {{ .Name }}={{ .Value }}
          {{ end }}
```
效果是这样的

![image-20200518204122926](/../assets/images/image-20200518204122926.png)

## multi condition

下一步就是怎么把这些字段从下面表格去掉，纠结到最后终于搞定，类似这样

```go
{{ range $i, $alert := .Alerts }}
{{ if lt ($i) 1 }}
{{ range .Labels.SortedPairs }}
{{ if and (and (ne (.Name) "alertname") (and (ne (.Name) "ocp_deployment_type") (ne (.Name) "ocp_release"))) (and (and (ne (.Name) "job") (ne (.Name) "datacenter")) (ne (.Name) "severity")) }}
<td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 12px; font-weight: 500; vertical-align: top; text-align: center; color: #999; margin: 0; padding: 0 0 20px; color:#000000; border: 1px solid #e9e9e9;" align="center" valign="top">{{ .Name }}</td>
{{ end }}
{{ end }}
{{ end }}
{{ end }}
```

