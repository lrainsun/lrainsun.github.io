---
layout: post
title:  "Bot发送消息失败"
date:   2021-05-28 23:00:00 +0800
categories: Teams
tags: Teams-Bot
excerpt: Bot发送消息失败
mathjax: true
typora-root-url: ../
---

# Bot发送消息失败

前几天在调试bot的时候，一直有一个card发送失败，但是看json内容都是没问题的，贴到测试网站上也可以显示，搞了半天不明所以

```python
Error handling request
Traceback (most recent call last):
  File "/Users/minsu/.pyenv/versions/bot/lib/python3.6/site-packages/aiohttp/web_protocol.py", line 418, in start
    resp = await task
  File "/Users/minsu/Documents/minsu/gitlab/ocp_onesearch/bot/bot_helper.py", line 467, in webhook_handler
    response = await msg_handler(msg_json)
  File "/Users/minsu/Documents/minsu/gitlab/ocp_onesearch/bot/bot_helper.py", line 433, in spark_msg_in
    await self.spark_msg_reply(msg_data)
  File "/Users/minsu/Documents/minsu/gitlab/ocp_onesearch/bot/bot_helper.py", line 395, in spark_msg_reply
    reply_rooms, mentioned_people, replies, is_code_block, file_type, cards, mentioned_msg = self.spark_msg_direct_reply(msg_data)
  File "/Users/minsu/Documents/minsu/gitlab/ocp_onesearch/bot/bot_helper.py", line 389, in spark_msg_direct_reply
    return cmd['func'](msg_data, params)
  File "onesearchbot.py", line 588, in add_flavor
    return self.add_flavor_in_ocp(msg_data, params, reply_rooms, mentioned_people, requester)
  File "onesearchbot.py", line 568, in add_flavor_in_ocp
    message_id = self.bothelper.send_spark_reply([room_id], [requester.id], 'Flavor Request', False, 'CARD', card if isinstance(card, list) else [card], '')
  File "/Users/minsu/Documents/minsu/gitlab/ocp_onesearch/bot/bot_helper.py", line 328, in send_spark_reply
    return self.send_spark_card_message(room, texts, cards, to_person=to_person, parentId=parentId)
  File "/Users/minsu/Documents/minsu/gitlab/ocp_onesearch/bot/bot_helper.py", line 252, in send_spark_card_message
    return self.send_spark_text_message(roomId, texts, attachments, to_person=to_person, parentId=parentId)
  File "/Users/minsu/Documents/minsu/gitlab/ocp_onesearch/bot/bot_helper.py", line 197, in send_spark_text_message
    response = _session.messages.create(roomId=roomId, markdown=message, attachments=attachments, parentId=parentId)
  File "/Users/minsu/.pyenv/versions/bot/lib/python3.6/site-packages/webexteamssdk/api/messages.py", line 217, in create
    json_data = self._session.post(API_ENDPOINT, json=post_data)
  File "/Users/minsu/.pyenv/versions/bot/lib/python3.6/site-packages/webexteamssdk/restsession.py", line 401, in post
    **kwargs)
  File "/Users/minsu/.pyenv/versions/bot/lib/python3.6/site-packages/webexteamssdk/restsession.py", line 258, in request
    check_response_code(response, erc)
  File "/Users/minsu/.pyenv/versions/bot/lib/python3.6/site-packages/webexteamssdk/utils.py", line 220, in check_response_code
    raise ApiError(response)
webexteamssdk.exceptions.ApiError: [400] Bad Request - Unable to retrieve content.
```

最后才发现，是其中引用的一个图片，估计是webex teams的服务器访问不到，就会有这样的错，后来换了一张图片就好了。。