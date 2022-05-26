# k8s deployment缩容时各pod的优先级探究



## 引言

Deployment是k8s中最常用的一种工作负载，用于管理无状态的服务pod，对于无状态服务来说，每个pod自然是平等的，手动或通过HPA自动触发deployment的缩容逻辑时，一般不会关心deployment管理的各pod缩容时的优先级。

但笔者近期遇到一个实际的问题，简言之则是集群中的节点有一些是包年包月的节点，有一些是按量付费的节点，按量付费的节点在节点空闲的时候会触发回收逻辑，因此就希望deployment在缩容时能够优先删除运行在按量付费的节点上的pod，从而起到节约成本的效果。基于该背景，笔者决定深入k8s的调度器的源码中，对缩容时选择pod的机制一探究竟，并研究是否能够通过某种方式介入该过程。

## 

## 分析过程



首先我们在 pkg/controller/deployment/deployment_controller.go中查看deployment的控制器逻辑，因为控制器是通过周期性的同步来保证其管理的资源不断同步到用户提交的期望状态，因此我们看到这里的 syncDeployment 方法







未分配节点的<pending< unKnown < running < notReady



1. 判断pod是否被调度到节点上，优先删除未调度的节点，
2. 已调度的pod中，优先删除的顺序为 Pending, Unknown, Running
3. Running的pod中，优先删除未Ready的
4. 判断pod的 pod-deletion-cost,这是k8s v0.22的新特性，用于手动置顶pod的删除优先级
5. Ready且pod-deletion-cost相同的pod，则优先删除pod所在Node中同一个RS控制器控制的pod数量较多的pod
6. 优先删除Ready时间更晚的pod
7. Ready时间相同时，优先删除Container的重启次数较少的
8. 上述条件相同时，优先删除创建时间较新的pod

```go
func (s ActivePodsWithRanks) Less(i, j int) bool {
	// 1. Unassigned < assigned
	// If only one of the pods is unassigned, the unassigned one is smaller
	if s.Pods[i].Spec.NodeName != s.Pods[j].Spec.NodeName && (len(s.Pods[i].Spec.NodeName) == 0 || len(s.Pods[j].Spec.NodeName) == 0) {
		return len(s.Pods[i].Spec.NodeName) == 0
	}
	// 2. PodPending < PodUnknown < PodRunning
	if podPhaseToOrdinal[s.Pods[i].Status.Phase] != podPhaseToOrdinal[s.Pods[j].Status.Phase] {
		return podPhaseToOrdinal[s.Pods[i].Status.Phase] < podPhaseToOrdinal[s.Pods[j].Status.Phase]
	}
	// 3. Not ready < ready
	// If only one of the pods is not ready, the not ready one is smaller
	if podutil.IsPodReady(s.Pods[i]) != podutil.IsPodReady(s.Pods[j]) {
		return !podutil.IsPodReady(s.Pods[i])
	}

	// 4. lower pod-deletion-cost < higher pod-deletion cost
	if utilfeature.DefaultFeatureGate.Enabled(features.PodDeletionCost) {
		pi, _ := helper.GetDeletionCostFromPodAnnotations(s.Pods[i].Annotations)
		pj, _ := helper.GetDeletionCostFromPodAnnotations(s.Pods[j].Annotations)
		if pi != pj {
			return pi < pj
		}
	}

	// 5. Doubled up < not doubled up
	// If one of the two pods is on the same node as one or more additional
	// ready pods that belong to the same replicaset, whichever pod has more
	// colocated ready pods is less
	if s.Rank[i] != s.Rank[j] {
		return s.Rank[i] > s.Rank[j]
	}
	// TODO: take availability into account when we push minReadySeconds information from deployment into pods,
	//       see https://github.com/kubernetes/kubernetes/issues/22065
	// 6. Been ready for empty time < less time < more time
	// If both pods are ready, the latest ready one is smaller
	if podutil.IsPodReady(s.Pods[i]) && podutil.IsPodReady(s.Pods[j]) {
		readyTime1 := podReadyTime(s.Pods[i])
		readyTime2 := podReadyTime(s.Pods[j])
		if !readyTime1.Equal(readyTime2) {
			if !utilfeature.DefaultFeatureGate.Enabled(features.LogarithmicScaleDown) {
				return afterOrZero(readyTime1, readyTime2)
			} else {
				if s.Now.IsZero() || readyTime1.IsZero() || readyTime2.IsZero() {
					return afterOrZero(readyTime1, readyTime2)
				}
				rankDiff := logarithmicRankDiff(*readyTime1, *readyTime2, s.Now)
				if rankDiff == 0 {
					return s.Pods[i].UID < s.Pods[j].UID
				}
				return rankDiff < 0
			}
		}
	}
	// 7. Pods with containers with higher restart counts < lower restart counts
	if maxContainerRestarts(s.Pods[i]) != maxContainerRestarts(s.Pods[j]) {
		return maxContainerRestarts(s.Pods[i]) > maxContainerRestarts(s.Pods[j])
	}
	// 8. Empty creation time pods < newer pods < older pods
	if !s.Pods[i].CreationTimestamp.Equal(&s.Pods[j].CreationTimestamp) {
		if !utilfeature.DefaultFeatureGate.Enabled(features.LogarithmicScaleDown) {
			return afterOrZero(&s.Pods[i].CreationTimestamp, &s.Pods[j].CreationTimestamp)
		} else {
			if s.Now.IsZero() || s.Pods[i].CreationTimestamp.IsZero() || s.Pods[j].CreationTimestamp.IsZero() {
				return afterOrZero(&s.Pods[i].CreationTimestamp, &s.Pods[j].CreationTimestamp)
			}
			rankDiff := logarithmicRankDiff(s.Pods[i].CreationTimestamp, s.Pods[j].CreationTimestamp, s.Now)
			if rankDiff == 0 {
				return s.Pods[i].UID < s.Pods[j].UID
			}
			return rankDiff < 0
		}
	}
	return false
}
```






$$
targetPods = \frac  
$$




