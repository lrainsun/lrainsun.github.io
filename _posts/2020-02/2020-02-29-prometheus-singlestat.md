---
layout: post
title:  "grafana singlestat时间的秘密"
date:   2020-02-29 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: grafana singlestat时间的秘密
mathjax: true
typora-root-url: ../
---

# SingleStat

![image-20200229231045173](/assets/images/image-20200229231045173.png)

如果我们在query设置的是instant，意味着取的是当前timestamp的值，所以无论右上角的时间range怎么变，这个值是不会改变的

反之如果不是instant，这里就是一个取值的range，随着右上角的变化，整个图会变

![image-20200229231207761](/../assets/images/image-20200229231207761.png)

那么中间这个值哪来的呢？

![image-20200229231320055](/../assets/images/image-20200229231320055.png)

设置中有个stat，这里设置的是average，也就是会把time range内的采样值求平均

在我们这个case中，用current更为合适

## Stats

The Stats field let you set the function  (min, max, average, current, total, first, delta, range) that your  entire query is reduced into a single value with. This reduces the  entire query into a single summary value that is displayed

- **min** - The smallest value in the series
- **max** - The largest value in the series
- **avg** - The average of all the non-null values in the series
- **current** - The last value in the series. If the series ends on null the previous value will be used.
- **total** - The sum of all the non-null values in the series
- **first** - The first value in the series
- **delta** - The total incremental increase (of a  counter) in the series. An attempt is made to account for counter  resets, but this will only be accurate for single instance metrics. Used to show total counter increase in time series.
- **diff** - The difference between ‘current’ (last value) and ‘first’.
- **range** - The difference between ‘min’ and ‘max’. Useful the show the range of change for a gauge.