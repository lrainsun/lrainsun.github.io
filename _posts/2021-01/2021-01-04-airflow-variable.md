---
layout: post
title:  "airflow variable存json格式数据"
date:   2021-01-04 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: airflow variable存json格式数据
mathjax: true
typora-root-url: ../
---

# airflow variable存json格式数据

airflow variable如果想要存字典的话，需要转成json格式，否则会默认是string。。今天被坑了。。

```python
    SYNC_UP_REPO = {'kolla-ansible': 'ansible',
                'nova': 'nova',
                'neutron': 'neutron',
                'keystone': 'keystone',
                'horizon': ['horizon', 'openstack_auth', 'openstack_dashboard'],
                'ironic': 'ironic',
                'cinder': 'cinder',
                'placement': 'placement',
                'glance': 'glance'}
  
  	for repo_name in SYNC_UP_REPO.keys():
        if repo_name not in community_sync:
            community_sync[repo_name] = {'latest_commit': None,
                                         'commit_date': datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')}
    Variable.set('community_sync', community_sync, serialize_json=True)


    community_sync = Variable.get('community_sync', {}, deserialize_json=True)
    for repo_name in SYNC_UP_REPO.keys():
        print('***** repo %s: %s *****' % (repo_name, community_sync[repo_name]))
        commit_date = datetime.strptime(community_sync[repo_name]['commit_date'], '%Y-%m-%d %H:%M:%S')
```

