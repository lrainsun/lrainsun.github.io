---
layout: post
title:  "Airflow EmailOperator"
date:   2020-04-28 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow EmailOperator
mathjax: true
typora-root-url: ../
---

# 失败发送邮件

首先在DAG默认配置里可以配置失败后发送邮件

![image-20200428205548743](/../assets/images/image-20200428205548743.png)

在~/airflow/airflow.cfg中配置邮件：

```shell
[smtp]

# If you want airflow to send emails on retries, failure, and you want to use
# the airflow.utils.email.send_email_smtp function, you have to configure an
# smtp server here
smtp_host = outbound.cisco.com
smtp_starttls = False
smtp_ssl = False
# Example: smtp_user = airflow
# smtp_user =
# Example: smtp_password = airflow
# smtp_password =
smtp_port = 25
smtp_mail_from = ocp-airflow@cisco.com
```

然后如果失败了，会有这样的邮件

![image-20200428205747026](/../assets/images/image-20200428205747026.png)

link点进去，发现会到localhost:8080

这里还要配一下：

```shell
[webserver]
# The base url of your website as airflow cannot guess what domain or
# cname you are using. This is used in automated emails that
# airflow sends to point links to the right web server
#base_url = http://localhost:8080
base_url = http://ocp-dev-002.qa.webex.com:8080
```

# EmailOperator

在我们DAG代码里，可以用EmailOperator来主动发送邮件

比如这样

```python
send_result = EmailOperator(
    task_id='send_result',
    to=['minsu@cisco.com'],
    subject="OCP Server Build Finish - {{ end_date }} - {{ dag_run.conf['deployment'] }}",
    html_content="<h1>OCP Server Build Just Finished for {{ dag_run.conf['deployment'] }} at {{ end_date }}</h1> \
                    <h2>Run Id: {{ run_id }}</h2> \
                    <h2>Input Parameters: </h2> \
                    <ul> \
                    <li>req number: {{ dag_run.conf['req_num'] }}</li> \
                    <li>ritm number: {{ dag_run.conf['ritm_num'] }}</li> \
                    <li>file name: {{ dag_run.conf['filename'] }}</li> \
                    </ul> \
                    <h2>Running Result</h2> \
                    {{ task_instance.xcom_pull(task_ids='end') }}",
    dag=dag,
)
```

![image-20200428210701063](/../assets/images/image-20200428210701063.png)

# Customize Operator

我们还想把通知消息发送到teams room，所以可以基于baseoperator写一个

```python
class TeamsOperator(BaseOperator):
    """
    Sends an message to Webex teams.

    :param access_token: access token for Webex teams
    :type access_token: str
    :param spaces: list of spaces to send the message to.
    :type spaces: list
    :param markdown_content: content of the message, markdown is allowed.
    :type markdown_content: str
    :param files: file names to attach in message
    :type files: list
    """
    template_fields = ['markdown_content']
    template_ext = ('.md',)

    @apply_defaults
    def __init__(
            self,
            access_token,
            spaces,
            markdown_content,
            files=None,
            *args, **kwargs):
        super(TeamsOperator, self).__init__(*args, **kwargs)
        self.access_token = access_token
        self.spaces = spaces
        self.markdown_content = markdown_content
        self.files = files or []

    def execute(self, context):
        print(context)
        for space in self.spaces:
            send_teams_markdown_messages(self.access_token, space, self.markdown_content)
```

send_teams_markdown_messages是真正去发送消息

而`template_fields`指的是`markdown_content`这个字段将会用template来渲染，`template_ext`指的是格式是markdown的

然后这样调用

```python
send_teams_result = TeamsOperator(
    task_id='send_teams_result',
    access_token=access_token,
    spaces=spaces,
    markdown_content="""## OCP Server Build Finished \n
--- \n
- **Deployment**: {{ dag_run.conf['deployment'] }} \n
- **Execution Date**: {{ execution_date }} \n
- **Run Id**: [{{ run_id }}]("""+base_url+"""/admin/airflow/graph?dag_id=ocp_server_build_dag&execution_date={{ execution_date }}) \n
--- \n
### Input Parameters \n
- **req number**: {{ dag_run.conf['req_num'] }} \n
- **ritm number**: {{ dag_run.conf['ritm_num'] }} \n
- **file name**: {{ dag_run.conf['filename'] }} \n
--- \n
### Running Result \n
{{ task_instance.xcom_pull(key='teams_out_put') }}""",
    dag=dag,
)
```

![image-20200428210743226](/../assets/images/image-20200428210743226.png)

