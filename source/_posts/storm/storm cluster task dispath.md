---
title: storm源码分析--任务分发
tags: 
    - storm
categories: 
    - storm
---



# Storm源码分析之任务分发

> version: storm-1.2.3

## 简介

当storm项目提交到`nimbus`的时候，storm需要对提交的topology任务进行分发，将topology中的各个components分发到集群中各个supervisor的worker进程执行，任务的分发过程由storm的nimbus节点完成，默认情况下，storm使用`DefaultResourceAwareStrategy`进行分发逻辑的计算

## 源码简析

storm任务分发调度的超类为`IStrategy`接口，它抽象了调度的两个核心动作: 

```java
/**
 * 任务调度的策略
 */
public interface IStrategy {

    /**
     * 初始化一些调度准备操作
     */
    void prepare(SchedulingState schedulingState);

    /**
     * 任务调度的核心逻辑，返回一个调度的结果对象
     */
    SchedulingResult schedule(TopologyDetails td);
}
```

在默认情况下，storm默认使用`DefaultResourceAwareStrategy`来作为调度策略类，这是一个资源感知策略，顾名思义，此类会根据系统的资源使用情况来择优选择合适的节点和slot来完成任务的调度，其中包括总内存、可用内存、总cpu、可用cpu、可用slot数、执行器数量等

通过接口的抽象，我们先看`DefaultResourceAwareStrategy`的`prepare`和`schedule`方法

```java
public class DefaultResourceAwareStrategy implements IStrategy {
    ...
    
    /**
     * 初始化基本信息，包括集群，topology，机器节点等
     */
    public void prepare (SchedulingState schedulingState) {
        _cluster = schedulingState.cluster;
        _topologies = schedulingState.topologies;
        _nodes = schedulingState.nodes;
        _clusterInfo = schedulingState.cluster.getNetworkTopography();
        LOG.debug(this.getClusterInfo());
    }

    /**
     * 核心调度逻辑
     * @param td 当前提交的topology
     */
    public SchedulingResult schedule(TopologyDetails td) {
        // 检查可用节点的个数，不满足则调度失败
        if (_nodes.getNodes().size() <= 0) {
            LOG.warn("No available nodes to schedule tasks on!");
            return SchedulingResult.failure(SchedulingStatus.FAIL_NOT_ENOUGH_RESOURCES, "No available nodes to schedule tasks on!");
        }
        
        // 从当前提交的topology中获取未分配执行器的集合
        Collection<ExecutorDetails> unassignedExecutors = new HashSet<>(_cluster.getUnassignedExecutors(td));
        
        // 初始化一个调度分配HashMap
        Map<WorkerSlot, Collection<ExecutorDetails>> schedulerAssignmentMap = new HashMap<>();
        // 初始化已调度的任务列表
        Collection<ExecutorDetails> scheduledTasks = new ArrayList<>();
      
        // spout可用性检查
        List<Component> spouts = this.getSpouts(td);
        if (spouts.size() == 0) {
            LOG.error("Cannot find a Spout!");
            return SchedulingResult.failure(SchedulingStatus.FAIL_INVALID_TOPOLOGY, "Cannot find a Spout!");
        }

        // 对topology中未分配的执行器进行排序
        // 根据toplology中components上下游的连接数(exec数量)进行降序排序(参考sortComponents()) 
        List<ExecutorDetails> orderedExecutors = orderExecutors(td, unassignedExecutors);

        Collection<ExecutorDetails> executorsNotScheduled = new HashSet<>(unassignedExecutors);
        
        // 遍历已经排序的执行器列表，按照系统资源信息，对任务进行调度，将已经调度的执行器，添加到scheduledTasks中
        for (ExecutorDetails exec : orderedExecutors) {
            scheduleExecutor(exec, td, schedulerAssignmentMap, scheduledTasks);
        }
        
        // 将已完成调度的执行器从未分配的执行器列表中移除
        executorsNotScheduled.removeAll(scheduledTasks);
        LOG.debug("/* Scheduling left over task (most likely sys tasks) */");
        // 处理剩下未调度的执行器
        for (ExecutorDetails exec : executorsNotScheduled) {
            scheduleExecutor(exec, td, schedulerAssignmentMap, scheduledTasks);
        }

        SchedulingResult result;
        // 将已完成调度的执行器从未分配的执行器列表中移除
        executorsNotScheduled.removeAll(scheduledTasks);

        // 如果还是存在未调度的执行器, 则返回失败，此次调度失败, 反之成功
        if (executorsNotScheduled.size() > 0) {
            LOG.error("Not all executors successfully scheduled: {}",
                    executorsNotScheduled);
            schedulerAssignmentMap = null;
            result = SchedulingResult.failure(SchedulingStatus.FAIL_NOT_ENOUGH_RESOURCES,
                    (td.getExecutors().size() - unassignedExecutors.size()) + "/" + td.getExecutors().size() + " executors scheduled");
        } else {
            LOG.debug("All resources successfully scheduled!");
            result = SchedulingResult.successWithMsg(schedulerAssignmentMap, "Fully Scheduled by DefaultResourceAwareStrategy");
        }
        if (schedulerAssignmentMap == null) {
            LOG.error("Topology {} not successfully scheduled!", td.getId());
        }
        return result;
    }
     
    ...
}
```

上述源码主体的调度逻辑非常清晰，就是一个对任务进行分发的过程，先按照components的连接数进行降序排序，再遍历components对任务进行调度，调度过程中会对资源的可用性进行选择，对于节点的优先级选择，主要在`scheduleExecutor()`中进行

```java
		/**
     * Schedule executor exec from topology td
     *
     * @param exec the executor to schedule
     * @param td the topology executor exec is a part of
     * @param schedulerAssignmentMap the assignments already calculated
     * @param scheduledTasks executors that have been scheduled
     */
    private void scheduleExecutor(ExecutorDetails exec, TopologyDetails td, Map<WorkerSlot,
            Collection<ExecutorDetails>> schedulerAssignmentMap, Collection<ExecutorDetails> scheduledTasks) {
        // 找到合适的worker slot用来执行当前ExecutorDetails
        WorkerSlot targetSlot = this.findWorkerForExec(exec, td, schedulerAssignmentMap);
        if (targetSlot != null) {
            RAS_Node targetNode = this.idToNode(targetSlot.getNodeId());
            if (!schedulerAssignmentMap.containsKey(targetSlot)) {
                schedulerAssignmentMap.put(targetSlot, new LinkedList<ExecutorDetails>());
            }

            schedulerAssignmentMap.get(targetSlot).add(exec);
            targetNode.consumeResourcesforTask(exec, td);
            scheduledTasks.add(exec);
            LOG.debug("TASK {} assigned to Node: {} avail [ mem: {} cpu: {} ] total [ mem: {} cpu: {} ] on slot: {} on Rack: {}", exec,
                    targetNode.getHostname(), targetNode.getAvailableMemoryResources(),
                    targetNode.getAvailableCpuResources(), targetNode.getTotalMemoryResources(),
                    targetNode.getTotalCpuResources(), targetSlot, nodeToRack(targetNode));
        } else {
            LOG.error("Not Enough Resources to schedule Task {}", exec);
        }
    }
```



## 总结