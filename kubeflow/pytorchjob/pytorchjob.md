# kubeflow pytorchjob

## CRD结构

首先需要看下pytorchjob的crd结构

```
type PyTorchJobSpec struct {
	// RunPolicy encapsulates various runtime policies of the distributed training
	// job, for example how to clean up resources and how long the job can stay
	// active.
	//+kubebuilder:validation:Optional
	RunPolicy common.RunPolicy `json:"runPolicy"`

	// A map of PyTorchReplicaType (type) to ReplicaSpec (value). Specifies the PyTorch cluster configuration.
	// For example,
	//   {
	//     "Master": PyTorchReplicaSpec,
	//     "Worker": PyTorchReplicaSpec,
	//   }
	PyTorchReplicaSpecs map[common.ReplicaType]*common.ReplicaSpec `json:"pytorchReplicaSpecs"`
}
type ReplicaType string

type RunPolicy struct {
	// CleanPodPolicy defines the policy to kill pods after the job completes.
	// Default to Running.
	CleanPodPolicy *CleanPodPolicy `json:"cleanPodPolicy,omitempty"`

	// TTLSecondsAfterFinished is the TTL to clean up jobs.
	// It may take extra ReconcilePeriod seconds for the cleanup, since
	// reconcile gets called periodically.
	// Default to infinite.
	TTLSecondsAfterFinished *int32 `json:"ttlSecondsAfterFinished,omitempty"`

	// Specifies the duration in seconds relative to the startTime that the job may be active
	// before the system tries to terminate it; value must be positive integer.
	// +optional
	ActiveDeadlineSeconds *int64 `json:"activeDeadlineSeconds,omitempty"`

	// Optional number of retries before marking this job failed.
	// +optional
	BackoffLimit *int32 `json:"backoffLimit,omitempty"`

	// SchedulingPolicy defines the policy related to scheduling, e.g. gang-scheduling
	// +optional
	SchedulingPolicy *SchedulingPolicy `json:"schedulingPolicy,omitempty"`
}
type ReplicaSpec struct {
	// Replicas is the desired number of replicas of the given template.
	// If unspecified, defaults to 1.
	Replicas *int32 `json:"replicas,omitempty"`

	// Template is the object that describes the pod that
	// will be created for this replica. RestartPolicy in PodTemplateSpec
	// will be overide by RestartPolicy in ReplicaSpec
	Template v1.PodTemplateSpec `json:"template,omitempty"`

	// Restart policy for all replicas within the job.
	// One of Always, OnFailure, Never and ExitCode.
	// Default to Never.
	RestartPolicy RestartPolicy `json:"restartPolicy,omitempty"`
}
```

   从结构上可以大致看到“RunPolicy”用来声明对应任务的运行策略，比如超时时间、调度策略，pod清理策略等等

   PyTorchReplicaSpecs则对应分布式集群的pod信息，在pytorch训练中，主要是用的master——worker模式。

## Reconcile

```
func (r *PyTorchJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)
	logger := r.Log.WithValues(pytorchv1.Singular, req.NamespacedName)

	pytorchjob := &pytorchv1.PyTorchJob{}
	err := r.Get(ctx, req.NamespacedName, pytorchjob)
	if err != nil {
		logger.Info(err.Error(), "unable to fetch PyTorchJob", req.NamespacedName.String())
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}
	//校验pytorchjob
	if err = validation.ValidateV1PyTorchJobSpec(&pytorchjob.Spec); err != nil {
		logger.Info(err.Error(), "PyTorchJob failed validation", req.NamespacedName.String())
	}

	// Check if reconciliation is needed
	// get crd's name and its namespace
	jobKey, err := common.KeyFunc(pytorchjob)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get jobKey for job object %#v: %v", pytorchjob, err))
	}

	replicaTypes := util.GetReplicaTypes(pytorchjob.Spec.PyTorchReplicaSpecs)
	needReconcile := util.SatisfiedExpectations(r.Expectations, jobKey, replicaTypes)
	if !needReconcile || pytorchjob.GetDeletionTimestamp() != nil {
		logger.Info("reconcile cancelled, job does not need to do reconcile or has been deleted",
			"sync", needReconcile, "deleted", pytorchjob.GetDeletionTimestamp() != nil)
		return ctrl.Result{}, nil
	}

	// Set default priorities to pytorch job
	r.Scheme.Default(pytorchjob)

	// Use common to reconcile the job related pod and service
	// use common jobcontroller ReconcileJobs**
	err = r.ReconcileJobs(pytorchjob, pytorchjob.Spec.PyTorchReplicaSpecs, pytorchjob.Status, &pytorchjob.Spec.RunPolicy)
	if err != nil {
		logrus.Warnf("Reconcile PyTorch Job error %v", err)
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}
```
   他会先去check这个pytorch格式是否规范，比如是否是master——worker格式，然后调用common包的ReconcileJobs函数去创建对应的分布式集群
   
   可以看下对应的Reconcile代码。
```
func (jc *JobController) ReconcileJobs(
	job interface{},
	replicas map[apiv1.ReplicaType]*apiv1.ReplicaSpec,
	jobStatus apiv1.JobStatus,
	runPolicy *apiv1.RunPolicy) error {

	metaObject, ok := job.(metav1.Object)
	jobName := metaObject.GetName()
	if !ok {
		return fmt.Errorf("job is not of type metav1.Object")
	}
	runtimeObject, ok := job.(runtime.Object)
	if !ok {
		return fmt.Errorf("job is not of type runtime.Object")
	}
	jobKey, err := KeyFunc(job)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for job object %#v: %v", job, err))
		return err
	}
	// Reset expectations
	// 1. Since `ReconcileJobs` is called, we expect that previous expectations are all satisfied,
	//    and it's safe to reset the expectations
	// 2. Reset expectations can avoid dirty data such as `expectedDeletion = -1`
	//    (pod or service was deleted unexpectedly)
	jc.ResetExpectations(jobKey, replicas)

	log.Infof("Reconciling for job %s", metaObject.GetName())
	pods, err := jc.Controller.GetPodsForJob(job)
	if err != nil {
		log.Warnf("GetPodsForJob error %v", err)
		return err
	}

	services, err := jc.Controller.GetServicesForJob(job)
	if err != nil {
		log.Warnf("GetServicesForJob error %v", err)
		return err
	}

	oldStatus := jobStatus.DeepCopy()
	if commonutil.IsSucceeded(jobStatus) || commonutil.IsFailed(jobStatus) {
		// If the Job is succeed or failed, delete all pods and services.
        // 运行成功或失败后当前任务都会结束，需要根据事先定义的清理策略清理对应的pod，svc等资源
		if err := jc.DeletePodsAndServices(runPolicy, job, pods); err != nil {
			return err
		}

		if err := jc.CleanupJob(runPolicy, jobStatus, job); err != nil {
			return err
		}

		// At this point the pods may have been deleted.
		// 1) If the job succeeded, we manually set the replica status.
		// 2) If any replicas are still active, set their status to succeeded.
        // 任务成功之后更新status状态
		if commonutil.IsSucceeded(jobStatus) {
			for rtype := range jobStatus.ReplicaStatuses {
				jobStatus.ReplicaStatuses[rtype].Succeeded += jobStatus.ReplicaStatuses[rtype].Active
				jobStatus.ReplicaStatuses[rtype].Active = 0
			}
		}

		// No need to update the job status if the status hasn't changed since last time.
		// 需要比较是否一致，如果oldStatus、jobStatus一致时候仍然去uodate时候，
        // status改动会生效，导致resourceversion+1
        // 从而导致再一次触发reconcile，下一次仍然执行到这一步，出现死循环
        if !reflect.DeepEqual(*oldStatus, jobStatus) {
			return jc.Controller.UpdateJobStatusInApiServer(job, &jobStatus)
		}

		return nil
	}

	// retrieve the previous number of retry
	previousRetry := jc.WorkQueue.NumRequeues(jobKey)

	activePods := k8sutil.FilterActivePods(pods)

	jc.recordAbnormalPods(activePods, runtimeObject)

	active := int32(len(activePods))
	failed := k8sutil.FilterPodCount(pods, v1.PodFailed)
	totalReplicas := k8sutil.GetTotalReplicas(replicas)
	prevReplicasFailedNum := k8sutil.GetTotalFailedReplicas(jobStatus.ReplicaStatuses)

	var failureMessage string
	jobExceedsLimit := false
	exceedsBackoffLimit := false
	pastBackoffLimit := false

	if runPolicy.BackoffLimit != nil {
		jobHasNewFailure := failed > prevReplicasFailedNum
		// new failures happen when status does not reflect the failures and active
		// is different than parallelism, otherwise the previous controller loop
		// failed updating status so even if we pick up failure it is not a new one
		exceedsBackoffLimit = jobHasNewFailure && (active != totalReplicas) &&
			(int32(previousRetry)+1 > *runPolicy.BackoffLimit)

		pastBackoffLimit, err = jc.PastBackoffLimit(jobName, runPolicy, replicas, pods)
		if err != nil {
			return err
		}
	}

	if exceedsBackoffLimit || pastBackoffLimit {
		// check if the number of pod restart exceeds backoff (for restart OnFailure only)
		// OR if the number of failed jobs increased since the last syncJob
		jobExceedsLimit = true
		failureMessage = fmt.Sprintf("Job %s has failed because it has reached the specified backoff limit", jobName)
	} else if jc.PastActiveDeadline(runPolicy, jobStatus) {
		failureMessage = fmt.Sprintf("Job %s has failed because it was active longer than specified deadline", jobName)
		jobExceedsLimit = true
	}
    // 出现超时或者超过重试次数
    // 需要清理对应资源和更新status状态
	if jobExceedsLimit {
		// Set job completion time before resource cleanup
		if jobStatus.CompletionTime == nil {
			now := metav1.Now()
			jobStatus.CompletionTime = &now
		}

		// If the Job exceeds backoff limit or is past active deadline
		// delete all pods and services, then set the status to failed
		if err := jc.DeletePodsAndServices(runPolicy, job, pods); err != nil {
			return err
		}

		if err := jc.CleanupJob(runPolicy, jobStatus, job); err != nil {
			return err
		}

		if err := commonutil.UpdateJobConditions(&jobStatus, apiv1.JobFailed, commonutil.JobFailedReason, failureMessage); err != nil {
			log.Infof("Append job condition error: %v", err)
			return err
		}

		return jc.Controller.UpdateJobStatusInApiServer(job, &jobStatus)
	} else {

	    // Diff current active pods/services with replicas.
            // 创建对应的pod和svc
           // replicas为{"master": masterRs.spec, "worker": workerRs.spec}
		for rtype, spec := range replicas {
			err := jc.Controller.ReconcilePods(metaObject, &jobStatus, pods, rtype, spec, replicas)
			if err != nil {
				log.Warnf("ReconcilePods error %v", err)
				return err
			}

			err = jc.Controller.ReconcileServices(metaObject, services, rtype, spec)

			if err != nil {
				log.Warnf("ReconcileServices error %v", err)
				return err
			}
		}
	}

	err = jc.Controller.UpdateJobStatus(job, replicas, &jobStatus)
	if err != nil {
		log.Warnf("UpdateJobStatus error %v", err)
		return err
	}
	// No need to update the job status if the status hasn't changed since last time.
	if !reflect.DeepEqual(*oldStatus, jobStatus) {
		return jc.Controller.UpdateJobStatusInApiServer(job, &jobStatus)
	}
	return nil
}
```
   在最后他会遍历所有的rs，按照一定规则生成svc name
   
   然后根据这些name作为环境变量放置pod中，并且生成对应的svc
```
func (jc *JobController) ReconcilePods(
	job interface{},
	jobStatus *apiv1.JobStatus,
	pods []*v1.Pod,
	rtype apiv1.ReplicaType,
	spec *apiv1.ReplicaSpec,
	replicas map[apiv1.ReplicaType]*apiv1.ReplicaSpec) error {

	numReplicas := int(*spec.Replicas)
	var masterRole bool

	initializeReplicaStatuses(jobStatus, rtype)

	// GetPodSlices will return enough information here to make decision to add/remove/update resources.
	//
	// For example, let's assume we have pods with replica-index 0, 1, 2
	// If replica is 4, return a slice with size 4. [[0],[1],[2],[]], a pod with replica-index 3 will be created.
	//
	// If replica is 1, return a slice with size 3. [[0],[1],[2]], pod with replica-index 1 and 2 are out of range and will be deleted.
	podSlices := jc.GetPodSlices(pods, numReplicas, logger)
    // podSlices是个二维数组，表示应该有多少个master或worker，每个master或worker找到了多少个pod
    // worker中不存在pod就创建"if len(podSlice) == 0"，index不规范就把对应pod删掉。
    // 应该是用于update pytorchjob的时候
	for index, podSlice := range podSlices {
		if len(podSlice) > 1 {
			logger.Warningf("We have too many pods for %s %d", rt, index)
		} else if len(podSlice) == 0 {
			logger.Infof("Need to create new pod: %s-%d", rt, index)

			// check if this replica is the master role
            // 创建 master还是worker
			masterRole = jc.Controller.IsMasterRole(replicas, rtype, index)
			err = jc.createNewPod(job, rt, strconv.Itoa(index), spec, masterRole, replicas)
			if err != nil {
				return err
			}
		} else {
			// Check the status of the current pod.
			pod := podSlice[0]

			// check if the index is in the valid range, if not, we should kill the pod
			if index < 0 || index >= numReplicas {
				err = jc.PodControl.DeletePod(pod.Namespace, pod.Name, runtimeObject)
				if err != nil {
					return err
				}
				// Deletion is expected
				jc.Expectations.RaiseExpectations(expectationPodsKey, 0, 1)
			}

			updateJobReplicaStatuses(jobStatus, rtype, pod)
		}
	}
	return nil
}
```

   podSlices是的大小是通过计算当前pod的数量，并与rs数量取其大值
```
func (jc *JobController) GetPodSlices(pods []*v1.Pod, replicas int, logger *log.Entry) [][]*v1.Pod {
	podSlices := make([][]*v1.Pod, calculatePodSliceSize(pods, replicas))
	for _, pod := range pods {
		if _, ok := pod.Labels[apiv1.ReplicaIndexLabel]; !ok {
			logger.Warning("The pod do not have the index label.")
			continue
		}
		index, err := strconv.Atoi(pod.Labels[apiv1.ReplicaIndexLabel])
		if err != nil {
			logger.Warningf("Error when strconv.Atoi: %v", err)
			continue
		}
		if index < 0 || index >= replicas {
			logger.Warningf("The label index is not expected: %d, pod: %s/%s", index, pod.Namespace, pod.Name)
		}

		podSlices[index] = append(podSlices[index], pod)
	}
	return podSlices
}
func calculatePodSliceSize(pods []*v1.Pod, replicas int) int {
	size := 0
	for _, pod := range pods {
		if _, ok := pod.Labels[apiv1.ReplicaIndexLabel]; !ok {
			continue
		}
		index, err := strconv.Atoi(pod.Labels[apiv1.ReplicaIndexLabel])
		if err != nil {
			continue
		}
		size = MaxInt(size, index)
	}

	// size comes from index, need to +1 to indicate real size
	return MaxInt(size+1, replicas)
}
```
然后就是createPod了
```
// createNewPod creates a new pod for the given index and type.
func (jc *JobController) createNewPod(job interface{}, rt, index string, spec *apiv1.ReplicaSpec, masterRole bool,
	replicas map[apiv1.ReplicaType]*apiv1.ReplicaSpec) error {

	// Set type and index for the worker.
	labels := jc.GenLabels(metaObject.GetName())
	labels[apiv1.ReplicaTypeLabel] = rt
	labels[apiv1.ReplicaIndexLabel] = index

	if masterRole {
		labels[apiv1.JobRoleLabel] = "master"
	}

	podTemplate := spec.Template.DeepCopy()

	// Set name for the template.
	podTemplate.Name = GenGeneralName(metaObject.GetName(), rt, index)

	if podTemplate.Labels == nil {
		podTemplate.Labels = make(map[string]string)
	}

	for key, value := range labels {
		podTemplate.Labels[key] = value
	}
    // 设置环境变量
	if err := jc.Controller.SetClusterSpec(job, podTemplate, rt, index); err != nil {
		return err
	}
	setRestartPolicy(podTemplate, spec)
	err = jc.PodControl.CreatePodsWithControllerRef(metaObject.GetNamespace(), podTemplate, runtimeObject, controllerRef)
	return nil
}
```

    内容比较简单，其集群信息通过SetClusterSpec(job, podTemplate, rt, index)设置
```
// SetClusterSpec sets the cluster spec for the pod
func (r *PyTorchJobReconciler) SetClusterSpec(job interface{}, podTemplate *corev1.PodTemplateSpec, rtype, index string) error {
	return SetPodEnv(job, podTemplate, rtype, index)
}
func SetPodEnv(obj interface{}, podTemplateSpec *corev1.PodTemplateSpec, rtype, index string) error {
	// 生成master host（通过svc通信）和port
    masterAddr := genGeneralName(pytorchjob.Name, strings.ToLower(string(pytorchv1.PyTorchReplicaTypeMaster)), strconv.Itoa(0))
	if rtype == strings.ToLower(string(pytorchv1.PyTorchReplicaTypeMaster)) {
		if rank != 0 {
			return fmt.Errorf("invalid config: There should be only a single master with index=0")
		}
		masterAddr = "localhost"
	} else {
		rank = rank + 1
	}

	for i := range podTemplateSpec.Spec.Containers {
		if len(podTemplateSpec.Spec.Containers[i].Env) == 0 {
			podTemplateSpec.Spec.Containers[i].Env = make([]corev1.EnvVar, 0)
		}
		podTemplateSpec.Spec.Containers[i].Env = append(podTemplateSpec.Spec.Containers[i].Env, corev1.EnvVar{
			Name:  "MASTER_PORT",
			Value: strconv.Itoa(int(masterPort)),
		})
		podTemplateSpec.Spec.Containers[i].Env = append(podTemplateSpec.Spec.Containers[i].Env, corev1.EnvVar{
			Name:  "MASTER_ADDR",
			Value: masterAddr,
		})
		podTemplateSpec.Spec.Containers[i].Env = append(podTemplateSpec.Spec.Containers[i].Env, corev1.EnvVar{
			Name:  "WORLD_SIZE",
			Value: strconv.Itoa(int(totalReplicas)),
		})
		podTemplateSpec.Spec.Containers[i].Env = append(podTemplateSpec.Spec.Containers[i].Env, corev1.EnvVar{
			Name:  "RANK",
			Value: strconv.Itoa(rank),
		})
		podTemplateSpec.Spec.Containers[i].Env = append(podTemplateSpec.Spec.Containers[i].Env, corev1.EnvVar{
			Name:  "PYTHONUNBUFFERED",
			Value: "0",
		})
	}

	return nil
}

```
就这样把对应的pod全部生成出来