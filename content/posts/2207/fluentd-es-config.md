---
title: "Fluentd配置文件最佳实践"
description: ""
summary: "Fluentd负责Kubernetes中容器日志的收集工作, 以Daemonset形式运行在每一个节点上. 下面这个配置是在多个生产集群使用的配置, 经过多次调优的. 有一些关键的配置增加了配置解释说明. 目前使用问题不大. 持续更新配置中..."
date: "2022-07-31"
menu: "main"
bookToC: false
draft: false
tags:
- kubernetes
- fluentd
categories:
- "technology"
---

Fluentd负责Kubernetes中容器日志的收集工作, 以Daemonset形式运行在每一个节点上. 下面这个配置是在多个生产集群使用的配置, 经过多次调优的. 有一些关键的配置增加了配置解释说明. 目前使用问题不大. 持续更新配置中...

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-es-config
  namespace: logging
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>

  containers.input.conf: |-
    # Json Log Example:
    # {"log":"[info:2016-02-16T16:04:05.930-08:00] Some log text here\n","stream":"stdout","time":"2016-02-17T00:04:05.931087621Z"}
    # CRI Log Example:
    # 2016-02-17T00:04:05.931087621Z stdout F [info:2016-02-16T16:04:05.930-08:00] Some log text here
    <source>
      @id fluentd-containers.log
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/es-containers.log.pos
      tag raw.kubernetes.*
      read_from_head true
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key time
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>

    # Detect exceptions in the log output and forward them as one log entry.
    <match raw.kubernetes.**>
      @id raw.kubernetes
      @type detect_exceptions
      remove_tag_prefix raw
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>

    # Concatenate multi-line logs
    # <filter kubernetes.**>
    #   @id filter_concat
    #   @type concat
    #   key log
    #   multiline_end_regexp /\n$/
    #   separator ""
    # </filter>

    # Enriches records with Kubernetes metadata
    <filter kubernetes.**>
      @id filter_kubernetes_metadata
      @type kubernetes_metadata
    </filter>

    # 防止ES中出现重复数据
    <filter collect-logs.**>
      @type elasticsearch_genid
      hash_id_key _hash # storing generated hash id key (default is _hash)
    </filter>

  output.conf: |-
    # 根据pod.metadata.labels来判断是否收集日志
    # collect-logs: true
    # 添加标识集群ID的tag: collect-logs.<clustername> 
    <match kubernetes.**>
      @type rewrite_tag_filter
      <rule>
        key $.kubernetes.labels.collect-logs
        pattern /^true$/
        tag collect-logs.<clustername>
      </rule>
    </match>

    <match collect-logs.**>
      @type elasticsearch
      @log_level info
      hosts 10.138.1.51:9200,10.138.1.52:9200,10.138.1.53:9200
      user <user>
      password <password>
      reload_connections false
      reconnect_on_error true
      reload_on_failure true
      request_timeout 30s
      suppress_type_name true
      id_key _hash # specify same key name which is specified in hash_id_key
      remove_keys _hash # Elasticsearch doesn't like keys that start with _
      include_tag_key true
      logstash_format true
      <buffer>
        @type file
        path /var/log/fluentd-buffers/elasticsearch01.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        # total_limit_size 500M # 默认64G, 太小会因为buffer灌满卡住并丢失数据
        overflow_action block
      </buffer>
    </match>

  monitoring.conf: |-
    # Prometheus Exporter Plugin
    # input plugin that exports metrics
    <source>
      @id prometheus
      @type prometheus
    </source>
    <source>
      @id monitor_agent
      @type monitor_agent
    </source>
    # input plugin that collects metrics from MonitorAgent
    <source>
      @id prometheus_monitor
      @type prometheus_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
    # input plugin that collects metrics for output plugin
    <source>
      @id prometheus_output_monitor
      @type prometheus_output_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
    # input plugin that collects metrics for in_tail plugin
    <source>
      @id prometheus_tail_monitor
      @type prometheus_tail_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
```