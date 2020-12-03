# 参考文献：

## 1、Automatically and Adaptively Identifying Severe Alerts for Online Service Systems

## 2、Diagnosing Root Causes of Intermittent Slow Queries in Cloud Databases

在云数据库中出现了数据库本身和计算机级别的性能问题，导致服务之间读取数据库过程出现了间歇性慢查询，提出了基于机器学习算法的间歇性异常诊断器。
其中，该框架有两个阶段：离线分析和在线诊断、更新，又包含4个模块：异常提取、依赖清理、面向类型聚类、贝叶斯模型。

* 异常提取

解决问题：kpi异常多样、异常表现复杂

方法：指定时间段，收集规定时间段内的KPI片段，从其中提取异常类型。其中在识别尖峰的时候，使用了一种鲁棒阈值。

* 依赖清理

解决问题：kpi异常相关

方法：计算关联规则置信度

* 面向类型聚类

解决问题：数据多、标注难、4个基本kpi异常表现类型、kpi类型相关而根因相关

方法：TOPIC算法

* 贝叶斯模型

解决问题：可解释化

## 3、Localizing Failure Root Causes in a Microservice

## 4、两篇硕士论文

# 思路：

多维度监测数据，发现问题，对LB问题起作用，找自己的规则，使用数理统计或深度学习的方法。

特征提取、融合（特征来自多维metric，同时感觉一定要考虑时间序列特性）  自动发现负载不均衡点（这里可以考虑将问题分类）    治理不均衡（不同类型如何治理熔断、分发）
