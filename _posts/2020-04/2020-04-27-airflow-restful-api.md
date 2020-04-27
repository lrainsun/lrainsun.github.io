---
layout: post
title:  "Airflow Restful API"
date:   2020-04-27 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow Restful API
mathjax: true
typora-root-url: ../
---

# Airflow Restful API

## Trigger a dag run

```shell
curl -X POST \
  http://ocp-dev-002.qa.webex.com:8080/api/experimental/dags/ocp_server_build_dag/dag_runs \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -d '{"conf":"{\"ritm_num\":\"RITM0071298\", \"deployment\":\"ci22ln\", \"native_vlan_check\":\"True\"}"}'
```

## Get a dag run

```shell
curl -i http://ocp-dev-002.qa.webex.com:8080/api/experimental/dags/ocp_server_build_dag/dag_runs -X GET
```

## Get one dag run by execution data

```shell
curl -i http://ocp-dev-002.qa.webex.com:8080/api/experimental/dags/ocp_server_build_dag/dag_runs/2020-04-26T03:17:59+00:00 -X GET
```

## Get task

```shell
curl -i http://ocp-dev-002.qa.webex.com:8080/api/experimental/dags/ocp_server_build_dag/tasks/fetch_servers_from_snow -X GET
```

## Get task instances's public instance variables

```shell
curl -i http://ocp-dev-002.qa.webex.com:8080/api/experimental/dags/ocp_server_build_dag/dag_runs/2020-04-26T09:40:32+00:00/tasks/fetch_servers_from_snow -X GET
```

