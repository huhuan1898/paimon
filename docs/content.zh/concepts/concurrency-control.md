---
title: "Concurrency Control"
weight: 3
type: docs
aliases:
- /concepts/concurrency-control.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Concurrency Control

Paimon 支持对多个并发写作业进行乐观并发。

每个作业都以自己的速度写入数据，并在提交时刻通过应用增量文件(删除或添加文件)基于当前快照生成新的快照。

这里可能有两种类型的提交失败:
1. 快照冲突: 快照id被抢占，表从其他作业生成了新的快照。好吧，我们再来一次。
2. 文件冲突: 此作业要删除的文件已被其他作业删除。此时，作业只能失败。(对于流作业，它将失败

## Snapshot conflict

Paimon 的快照ID是唯一的，因此只要作业将其快照文件写入文件系统，就认为它成功了。

{{< img src="/img/snapshot-conflict.png">}}

Paimon 使用文件系统的重命名机制来提交快照，这对HDFS来说是安全的，因为它确保了事务性和原子性的重命名。

但是对于像OSS和S3这样的对象存储，它们的 `RENAME` 没有原子语义。我们需要配置 Hive 或 jdbc metastore 并启用 `lock.enable` 目录选项。
否则，可能会导致快照丢失。

## Files conflict

当 Paimon 提交文件删除(这只是一个逻辑删除)时，它会检查与最新快照的冲突。
如果存在冲突(这意味着文件在逻辑上已被删除)，它就不能再在此提交节点上继续，
因此它只能有意地触发故障转移以重新启动，作业将从文件系统检索最新状态，以期解决此冲突。

{{< img src="/img/files-conflict.png">}}

Paimon 将确保这里没有数据丢失或重复，但如果两个流作业同时写入并且存在冲突，您将看到它们不断重新启动，这不是一件好事。

冲突的本质在于删除文件(逻辑上)，而删除文件是从压缩中诞生的，所以只要我们关闭写作业的压缩(将'write-only'设置为true)，并启动一个单独的作业来做压缩工作，一切都很好。



See [dedicated compaction job]({{< ref "maintenance/dedicated-compaction#dedicated-compaction-job" >}}) for more info.
