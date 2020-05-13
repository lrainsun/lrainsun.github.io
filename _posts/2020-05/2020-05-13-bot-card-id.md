---
layout: post
title:  "Webex Teams区分不同的Card"
date:   2020-05-13 23:00:00 +0800
categories: Teams
tags: Teams-Bot
excerpt: Webex Teams区分不同的Card
mathjax: true
typora-root-url: ../
---

# Webex Teams区分不同的Card

上次讲过card submit之后，获取到的message内容是类似下面这样

```yaml
GET /attachment/actions/Y2lzY29zcGFyazovL3VzL09SR0FOSVpBVElPTi85NmFiYzJhYS0zZGNjLTE
{
  "type": "submit",
  "messageId": "GFyazovL3VzL1BFT1BMRS80MDNlZmUwNy02Yzc3LTQyY2UtOWI4NC",
  "inputs": {
    "Name": "John Andersen",
    "Url": "https://example.com",
    "Email": "john.andersen@example.com",
    "Tel": "+1 408 526 7209"
  },
  "id": "Y2lzY29zcGFyazovL3VzL09SR0FOSVpBVElPTi85NmFiYzJhYS0zZGNjLTE",
  "personId": "Y2lzY29zcGFyazovL3VzL1BFT1BMRS83MTZlOWQxYy1jYTQ0LTRmZ",
  "roomId": "L3VzL1BFT1BMRS80MDNlZmUwNy02Yzc3LTQyY2UtOWI",
  "created": "2016-05-10T19:41:00.100Z"
}
```

可以看到，除了inputs之外，其他都是一些card内容无关的信息，只有inputs是可以带回来的某一个card特定的信息。那如果有两个card，都submit了，那我们要如何区分不同的card呢？

似乎没有专门的值来定义，所以想了个法子，在定义card的时候，就定义一个专门用来放card id（自定义）的inputs，比如我用了text类型，并且设置它的isVisible是false

```json
{'id': 'card_id', 'type': 'Input.Text', 'value': 'ServerBuild', 'isVisible': False}
```

最后我们收到的submit的结果里inputs是这样的

 ```json
"inputs": {"deployment": "ci22am", "card_id": "ServerBuild", "ritm_num": "RITM0071298", "native_vlan_check": "false"}
 ```

就可以通过'card_id'来区分不同的card了