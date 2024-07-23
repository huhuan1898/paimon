---
title: "Basic Concepts"
weight: 2
type: docs
aliases:
- /concepts/basic-concepts.html
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

# 基本概念

## 文件结构

表的所有文件都存储在一个基本目录下。Paimon 文件以分层的方式组织。下图说明了文件布局。从快照文件开始，Paimon 消费者可以递归地访问表中的所有记录。

{{< img src="/img/file-layout.png">}}

## 快照

所有快照文件都存储在 `snapshot` 目录下。

快照文件是一个 JSON 文件，其中包含有关该快照的信息，包括

* 正在使用的 Schema 文件
* 包含此快照中的所有数据变更的清单列表

快照捕获表在某个时间点的状态。
用户可以通过最近的快照访问表的最新数据。
通过时间旅行，用户还可以通过较早的快照访问表的先前状态。


## 清单文件

所有清单列表和清单文件都存储在 `manifest` 目录下。

清单列表是清单文件名的列表。

清单文件是包含有关LSM数据文件和更改日志文件的更改的文件。例如，在相应的快照中创建了哪个LSM数据文件，删除了哪个文件。

## 数据文件

数据文件按分区分组。目前 Paimon 支持使用 parquet、orc、avro 作为数据文件格式。默认格式为 parquet。

## 分区

Paimon 采用与 Apache Hive 相同的分区概念来分离数据。

分区是一种可选的方法，可以根据特定列（如日期、城市和部门）的值将表划分为相关部分。 每个表可以有一个或多个分区键来标识一个特定的分区。

通过分区，用户可以有效地操作表中的记录切片。

## 一致性保证

Paimon 生产者使用两阶段提交协议自动将一批记录提交到表中。

取决于增量写入策略和合并策略，每次提交过程在提交时最多产生两个[快照]({{< ref "concepts/basic-concepts#snapshot" >}})：

- 如果只执行增量写操作而不触发合并操作，则只创建增量快照。
- 如果触发了合并操作，将创建一个增量快照和一个全量快照。

对于任何两个同时修改表的生产者，只要他们不修改同一个分区，他们的提交可以并行进行。
如果修改的是同一个分区，则只保证快照隔离。
也就是说，最终的表状态可能是两次提交的混合，但不会丢失任何更改。
查看[专用压缩作业]({{< ref "maintenance/dedicated-compaction#dedicated-compact -job" >}})了解更多信息。
