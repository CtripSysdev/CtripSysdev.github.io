---
layout: post
title:  "elasticsearch: transport client bulk的时候如何选择目标node"
date:   2016-03-26 14:27:35 +0800
modifydate:   2016-03-26 14:27:35 +0800
abstract: "<div>我们之前用Losgtash做indexer把数据从kafka消费插入ES, 所有的数据都是先经过Logstash里面配置的四个client节点, 然后经他们再分配到数据节点.</div>
<div>后来因为logstash效率太低, 改成我们自己用java开发的的hangout做同样的事情, 发现数据不再走client, 而是直接到数据节点. 原因是构造transport client的时候设置成sniff: true.</div>
<div>但还是有一个困惑, bulk的一批数据, 可能最终会到多个节点上面索引, 那么是client在发送数据的时候就已经计算好应该把哪些数据发往哪个节点, 还是说随便发到nodeX, 然后nodeX再二次分发.</div>
<br> <br> "
categories: elasticsearch
category : elasticsearch
tags : [elasticsearch]

---

# 背景
我们之前用Losgtash做indexer把数据从kafka消费插入ES, 所有的数据都是先经过Logstash里面配置的四个client节点, 然后经他们再分配到数据节点.

后来因为logstash效率太低, 改成我们自己用java开发的的hangout做同样的事情, 发现数据不再走client, 而是直接到数据节点. 原因是构造transport client的时候设置成sniff: true.

但还是有一个困惑, bulk的一批数据, 可能最终会到多个节点上面索引, 那么是client在发送数据的时候就已经计算好应该把哪些数据发往哪个节点, 还是说随便发到nodeX, 然后nodeX再二次分发.

碰到这个问题的时候, 我想当然的以为是前者, 因为transport client可以拿到所有的metadata,应该可以算出来怎么分发. 如果是后者的话, 流量要复制一份, 过于浪费了.

但验证之后, 发现并非如此.

# 测试
1. 建一个有四个节点的集群, 并新建一个索引, 四个shards, **全部分布在一个节点上**

    ```sh
    GET hangouttest-2016.03.21/_settings
    {
       "hangouttest-2016.03.21": {
          "settings": {
             "index": {
                "routing": {
                   "allocation": {
                      "require": {
                         "_ip": "10.2.7.159"
                      }
                   }
                },
                "creation_date": "1458570866963",
                "number_of_shards": "4",
                "number_of_replicas": "0",
                "uuid": "FkWPR_WaQpG5LdIABCEzVw",
                "version": {
                   "created": "2010199"
                }
             }
          }
       }
    }

    ```
    GET _cat/shards/hangouttest-2016.03.21?v
    index                  shard prirep state   docs  store ip         node       
    hangouttest-2016.03.21 2     p      STARTED   28 28.5kb 10.2.7.159 10.2.7.159 
    hangouttest-2016.03.21 3     p      STARTED   22 27.8kb 10.2.7.159 10.2.7.159 
    hangouttest-2016.03.21 1     p      STARTED   30 28.8kb 10.2.7.159 10.2.7.159 
    hangouttest-2016.03.21 0     p      STARTED   20 27.5kb 10.2.7.159 10.2.7.159 
    ```

2. 写代码, 先生成一个transport client, 配置成20条数据bulk一次. 参考[https://www.elastic.co/guide/en/elasticsearch/client/java-api/2.2/java-docs-bulk-processor.html](https://www.elastic.co/guide/en/elasticsearch/client/java-api/2.2/java-docs-bulk-processor.html)

    ```java
    import org.elasticsearch.action.bulk.BackoffPolicy;
    import org.elasticsearch.action.bulk.BulkProcessor;
    import org.elasticsearch.common.unit.ByteSizeUnit;
    import org.elasticsearch.common.unit.ByteSizeValue;
    import org.elasticsearch.common.unit.TimeValue;

    BulkProcessor bulkProcessor = BulkProcessor.builder(
            client,  
            new BulkProcessor.Listener() {
                @Override
                public void beforeBulk(long executionId,
                                       BulkRequest request) { 
                    System.out.println("beforeBulk");
                    } 

                @Override
                public void afterBulk(long executionId,
                                      BulkRequest request,
                                      BulkResponse response) {
                    System.out.println("afterBulk");
                }

                @Override
                public void afterBulk(long executionId,
                                      BulkRequest request,
                                      Throwable failure) { ... } 
            })
            .setBulkActions(10000) 
            .setBulkSize(new ByteSizeValue(1, ByteSizeUnit.GB)) 
            .setFlushInterval(TimeValue.timeValueSeconds(5)) 
            .setConcurrentRequests(1) 
            .setBackoffPolicy(
                BackoffPolicy.exponentialBackoff(TimeValue.timeValueMillis(100), 3)) 
            .build();
    ```

3. tcpdump开起来, 抓包分析流量

    ```sh
    % sudo tcpdump -nn 'ip[2:2]>200'
    ```

4. 发送四次数据, 每次20条, 每条100字节左右.

5. 抓包结果, 可以看到四次bulk请求分别发往了四个节点

    ```sh
    % sudo tcpdump -nn 'ip[2:2]>200'
    22:39:16.222034 IP 10.170.30.45.63034 > 10.2.7.165.9300: Flags [P.], seq 1053720918:1053721314, ack 4146795087, win 4128, options [nop,nop,TS val 698503243 ecr 3217616506], length 396
    22:39:19.951115 IP 10.170.30.45.63047 > 10.2.7.159.9300: Flags [P.], seq 2684147573:2684147960, ack 2244520841, win 4128, options [nop,nop,TS val 698506819 ecr 3217621511], length 387
    22:39:23.385240 IP 10.170.30.45.63060 > 10.2.7.168.9300: Flags [P.], seq 318079750:318080148, ack 3392556208, win 4128, options [nop,nop,TS val 698510112 ecr 3217626519], length 398
    22:39:26.688067 IP 10.170.30.45.63021 > 10.2.7.161.9300: Flags [P.], seq 84160388:84160780, ack 4144775292, win 4128, options [nop,nop,TS val 698513218 ecr 3217626516], length 392
    ```

6. 源码分析

    选择node的代码在 org.elasticsearch.client.transport.TransportClientNodesService, getNodeNumber就是简单的+1

    ```java
    public <Response> void execute(NodeListenerCallback<Response> callback, ActionListener<Response> listener) {
        List<DiscoveryNode> nodes = this.nodes;
        ensureNodesAreAvailable(nodes);
        int index = getNodeNumber();
        RetryListener<Response> retryListener = new RetryListener<>(callback, listener, nodes, index);
        DiscoveryNode node = nodes.get((index) % nodes.size());
        try {
            callback.doWithNode(node, retryListener);
        } catch (Throwable t) {
            //this exception can't come from the TransportService as it doesn't throw exception at all
            listener.onFailure(t);
        }
    }
    ```

    然后回调到org.elasticsearch.client.transport.support.TransportProxyClient:

    ```java
    public void doWithNode(DiscoveryNode node, ActionListener<Response> listener) {
        proxy.execute(node, request, listener);
    }
    ```

    后面就是往这个node发送数据了.

# Over.
