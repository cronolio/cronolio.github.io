---
layout: post
sitemap:
  lastmod: 2023-09-01
title: мониторинг - храним только нужное
descr: мониторинг - храним только нужное
keywords: monitoring, prometheus, exporter, reduse data size
---
Если хранимые в prometheus метрики занимают много места, то можно хранить только нужное!!!

##### prometheus
Начнем с самого прометея. Ниже пример конфига:
```
global:
  evaluation_interval: 1m
  scrape_interval: 10s
  scrape_timeout: 10s
scrape_configs:
- job_name: 'prometheus'
  static_configs:
  - targets:
    - localhost:9090
  metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'go_.*|net_conntrack_.*|process_.*|prometheus_http_.*|prometheus_sd_(azure|consul|dns|kubernetes|kuma|nomad)_.*'
    action: drop
```

Удаляет добрую половину на мой взгляд ненужных метрик.
Если у вас есть что-то из `prometheus_sd_`, то поменяйте как необходимо.

##### node-exporter
Для достижения цели нужна версия выше v1.2.0 для `collector.disable-defaults`,
но речь пойдет о [node-exporter v1.5.0](https://github.com/prometheus/node_exporter/tree/v1.5.0).
В аргументах к node-exporter пишем такое:
```
--web.config.file=/etc/default/prometheus-web-config.yml
--web.disable-exporter-metrics
--collector.disable-defaults
```

Практически полностью убирает все метрики, кроме [бага](https://github.com/prometheus/node_exporter/issues/2265).
Чтож, поправим эту ситуацию.
```
- job_name: 'node'
  metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'promhttp_metric_handler_errors_total'
    action: drop 
```

Далее добавляем коллекторы и смотрим что нужно / не нужно.
Например в `collector.cpu` можно убрать `node_cpu_guest_seconds_total`.
