---
layout: post
title:  "Krakend endpoint"
date:   2021-05-30 23:01:00 +0800
categories: Gateway
tags: API-Gateway
excerpt: Krakend endpoint
mathjax: true
typora-root-url: ../
---

# Krakend service

## service

可以enable TLS, CORS以及一些安全保护

## endpoint

定义endpoint很简单，只要在endpoints下面增加一个endpoint的定义就可以了，如果没有定义method，默认是read-only(GET)

```json
"endpoints": [
{
  "endpoint": "/v1/foo",
  "method": "GET",
  "backend": [
    {
      "url_pattern": "/bar",
      "method": "GET",
      "host": [
        "https://api.mybackend.com"
      ]
    }
  ]
}
]
```

属性

- `endpoint`: The resource URL you want to expose
- `method`: Must be **written in uppercase** `GET`, `POST`, `PUT`, `PATCH`, `DELETE`.
- `backend`: List of all the **backend objects** queried for this endpoint.
- `extra_config`: Configuration of components and middlewares that are executed with this endpoint.
- `querystring_params`: Recognized GET parameters. See [parameter forwarding](https://www.krakend.io/docs/endpoints/parameter-forwarding/).
- `headers_to_pass`: Forwarded headers. See [headers forwarding](https://www.krakend.io/docs/endpoints/parameter-forwarding/#headers-forwarding).
- `concurrent_calls`: A technique to improve response times. See [concurrent requests](https://www.krakend.io/docs/endpoints/concurrent-requests/)

当对同一个resource有多种method的时候，需要为每一个method定义一个endpoint对象

endpoint变量，"endpoint": "/user/{id}"只能够匹配/user/123` or `/user/A-B-C，不能匹配/user/1/2

routes是不能冲突的，如果有冲突在启动的时候会报错

## rate limiting

在endpoint里定义rate limit是限制了router rate，也就是定义**maximum requests per second**一个krakend endpoint可以接收的最大数，默认是没有limit的，可以定义如下的limit

1. **Endpoint rate limit** (`maxRate`): Maximum number of requests an endpoint accepts in a second, no matter where the traffic comes from.
2. **Client/User rate limit** (`clientMaxRate`): Maximum number of requests an endpoint accepts **per client**

```json
{
    "endpoint": "/limited-endpoint",
    "extra_config": {
      "github.com/devopsfaith/krakend-ratelimit/juju/router": {
          "maxRate": 50,
          "clientMaxRate": 5,
          "strategy": "ip"
        }
    }
}
```

- `maxRate` (*integer*): Sets the number of **maximum requests the endpoint can handle per second**. The absence of `maxRate` in the configuration or `0` is the equivalent to no limitation.
- `clientMaxRate` (*integer*): Number of requests per second this endpoint will accept for each user (*user quota*). The client is defined by `strategy`. Instead of counting all the connections to the endpoint as the option above, the `clientMaxRate` keeps a counter for every client and endpoint. Keep in mind that every KrakenD instance keeps its counters in memory for **every single client**.
- `strategy` (*string*): The strategy you will use to set client counters. One of `ip` or `header`. Only to be used in combination with `clientMaxRate`. 区分不同的client，可以用ip来区分，或者header来区分

## Response manipulation

### merging

默认的manipulation是merging，比如一个endpoint从2-3个backend获取响应，会自动被Merge成一个response返回给client。

merging是有timeout的，krakend不会卡在那边等所有的backend都返回，对于一个gateway来说，failing fast is better than succeeding slowly。可以有gloabl的timeout配置，或者针对每个endpoint的配置

如果response是不完整的，header中x-krakend-completed是false

### filtering

有时候可能你只需要backend回复的一部分fields回复给client，这样可以节省用户的bandwidth

* deny list
* allow list

### grouping

可以把backend的response分组到不同的group，分组后，krakend会创建一个新的key把response放在里面

这样就避免了不同的backend同样的key name冲突

### mapping

这个很好理解，把backend的字段mapping到对应的字段，这样就可以保证对client的一致性，不管backend怎么变

### Target (or capturing)

经常会有响应数据放在类似data, response, content里面的，用户还需要解析一下，target就可以把这个下面的字段直接解析出来，放在root level

### Collection -or Array- manipulation

针对返回对象是一个array的处理