#argo wait 机制浅析


## 现象
我们在试运行一个简单的pod，只有一个容器pod，作用是 sleep 100000
```
    image: python:3.7
    command:
    - sh
    - -c
    - sleep 1000000
```
然后可以发现生成的pod中 会有一个wait container，同时main的container中的启动命令变为
```
 - /var/run/argo/argoexec
    - emissary
    - --
    - sh
    - -c
    - sleep 1000000
```
对应wait container中启动命令为
```
 - argoexec
    - wait
    - --loglevel
    - info
```
## wait container作用
首先需要提前了解得失，argo在执行完对应脚本内容之后，会将对应的输出文件写入到minio中，以供下一个pod使用。
这个写入的动作即时有这个wait container完成的，他会等待main container运行完成之后，把对应的输出内容，写到对应的minio和pod的annotations中。
## wait代码
### 入口
```
func NewWaitCommand() *cobra.Command {
	command := cobra.Command{
		Use:   "wait",
		Short: "wait for main container to finish and save artifacts",
		Run: func(cmd *cobra.Command, args []string) {
			ctx := context.Background()
			err := waitContainer(ctx)
			if err != nil {
				log.Fatalf("%+v", err)
			}
		},
	}
	return &command
}

func waitContainer(ctx context.Context) error {
    // 初始化 executor
    // argo中提供多种exector方式，在1.7+之后，支持Emissary，默认为Emissary
    // https://www.kubeflow.org/docs/components/pipelines/installation/choose-executor/
	wfExecutor := initExecutor()
	defer wfExecutor.HandleError(ctx) // Must be placed at the bottom of defers stack.
	defer stats.LogStats()
	stats.StartStatsTicker(5 * time.Minute)

	defer func() {
		// Killing sidecar containers
		err := wfExecutor.KillSidecars(ctx)
		if err != nil {
			wfExecutor.AddError(err)
		}
	}()

	// Wait for main container to complete
    // 等待main container 运行完成
	err := wfExecutor.Wait(ctx)
	if err != nil {
		wfExecutor.AddError(err)
	}
	// Capture output script result
	err = wfExecutor.CaptureScriptResult(ctx)
	if err != nil {
		wfExecutor.AddError(err)
	}
	// Saving logs
	logArt, err := wfExecutor.SaveLogs(ctx)
	if err != nil {
		wfExecutor.AddError(err)
	}
	// Saving output parameters
	err = wfExecutor.SaveParameters(ctx)
	if err != nil {
		wfExecutor.AddError(err)
	}
	// Saving output artifacts
    // 把输出写入到minio中
	err = wfExecutor.SaveArtifacts(ctx)
	if err != nil {
		wfExecutor.AddError(err)
	}
	// Annotating pod with output
    // 把输出写到pod annotaions字段中
	err = wfExecutor.AnnotateOutputs(ctx, logArt)
	if err != nil {
		wfExecutor.AddError(err)
	}

	return wfExecutor.HasError()
}

```
### init executor
```
func initExecutor() *executor.WorkflowExecutor {
	// .......
    // 主要是根据环境变量选择对应exectuor
    // kubeflow pipeline默认使用了emissary
	switch executorType {
	case common.ContainerRuntimeExecutorK8sAPI:
		cre = k8sapi.NewK8sAPIExecutor(clientset, config, podName, namespace)
	case common.ContainerRuntimeExecutorKubelet:
		cre, err = kubelet.NewKubeletExecutor(namespace, podName)
	case common.ContainerRuntimeExecutorPNS:
		cre, err = pns.NewPNSExecutor(clientset, podName, namespace)
	case common.ContainerRuntimeExecutorEmissary:
		cre, err = emissary.New()
	default:
		cre, err = docker.NewDockerExecutor(namespace, podName)
	}
	checkErr(err)

	wfExecutor := executor.NewExecutor(clientset, restClient, podName, namespace, cre, *tmpl, includeScriptOutput, deadline)

	log.
		WithField("version", version.String()).
		WithField("namespace", namespace).
		WithField("podName", podName).
		WithField("template", wfv1.MustMarshallJSON(&wfExecutor.Template)).
		WithField("includeScriptOutput", includeScriptOutput).
		WithField("deadline", deadline).
		Info("Executor initialized")
	return &wfExecutor
}

```
### wait
```
func (we *WorkflowExecutor) Wait(ctx context.Context) error {
	containerNames := we.Template.GetMainContainerNames()
    // 另起goroutine container是否超时
	go we.monitorDeadline(ctx, containerNames)
	err := waitutil.Backoff(ExecutorRetry, func() (bool, error) {
        // 等main container跑完
		err := we.RuntimeExecutor.Wait(ctx, containerNames)
		return err == nil, err
	})
	if err != nil {
		return fmt.Errorf("failed to wait for main container to complete: %w", err)
	}
	log.Infof("Main container completed")
	return nil
}
```
#### monitorDeadline
```
func (we *WorkflowExecutor) monitorDeadline(ctx context.Context, containerNames []string) {
	terminate := make(chan os.Signal, 1)
	signal.Notify(terminate, syscall.SIGTERM)

	deadlineExceeded := make(chan bool, 1)
	if !we.Deadline.IsZero() {
		t := time.AfterFunc(time.Until(we.Deadline), func() {
            // deadline之后 给chanel发送"true"
			deadlineExceeded <- true
		})
		defer t.Stop()
	}

	var message string
	log.Infof("Starting deadline monitor")
	select {
	case <-ctx.Done():
		log.Info("Deadline monitor stopped")
		return
	case <-deadlineExceeded:
		message = "Step exceeded its deadline"
	case <-terminate:
		message = "Step terminated"
	}
	log.Info(message)
	util.WriteTeriminateMessage(message)
    // 干掉main container
	we.killContainers(ctx, containerNames)
}
```
#### wait
```
func (e emissary) Wait(ctx context.Context, containerNames []string) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
			if e.isComplete(containerNames) {
				return nil
			}
			time.Sleep(time.Second)
		}
	}
}

func (e emissary) isComplete(containerNames []string) bool {
	for _, containerName := range containerNames {
		_, err := os.Stat("/var/run/argo/ctr/" + containerName + "/exitcode")
		if os.IsNotExist(err) {
			return false
		}
	}
	return true
}
```
读取这个文件的退出码，emissary机制后面篇幅介绍
#### kill container
```
func (we *WorkflowExecutor) killContainers(ctx context.Context, containerNames []string) {
	log.Infof("Killing containers")
	terminationGracePeriodDuration, _ := we.GetTerminationGracePeriodDuration(ctx)
	if err := we.RuntimeExecutor.Kill(ctx, containerNames, terminationGracePeriodDuration); err != nil {
		log.Warnf("Failed to kill %q: %v", containerNames, err)
	}
}
// 和pod退出流程类似，先发送SIGTERM信号，确保一些后处理操作，等待一段时间后 发送SIGKILL
func (e emissary) Kill(ctx context.Context, containerNames []string, terminationGracePeriodDuration time.Duration) error {
	for _, containerName := range containerNames {
		// allow write-access by other users, because other containers
		// should delete the signal after receiving it
		if err := ioutil.WriteFile("/var/run/argo/ctr/"+containerName+"/signal", []byte(strconv.Itoa(int(syscall.SIGTERM))), 0o666); err != nil { //nolint:gosec
			return err
		}
	}
	ctx, cancel := context.WithTimeout(ctx, terminationGracePeriodDuration)
	defer cancel()
	err := e.Wait(ctx, containerNames)
	if err != context.Canceled {
		return err
	}
	for _, containerName := range containerNames {
		// allow write-access by other users, because other containers
		// should delete the signal after receiving it
		if err := ioutil.WriteFile("/var/run/argo/ctr/"+containerName+"/signal", []byte(strconv.Itoa(int(syscall.SIGKILL))), 0o666); err != nil { //nolint:gosec
			return err
		}
	}
	return e.Wait(ctx, containerNames)
}

func (e emissary) Wait(ctx context.Context, containerNames []string) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
			if e.isComplete(containerNames) {
				return nil
			}
			time.Sleep(time.Second)
		}
	}
}

func (e emissary) isComplete(containerNames []string) bool {
	for _, containerName := range containerNames {
		_, err := os.Stat("/var/run/argo/ctr/" + containerName + "/exitcode")
		if os.IsNotExist(err) {
			return false
		}
	}
	return true
}
```
判断main container是否completed，kill container等操作都是读取某个文件内容，这个是emissary executor特性
#### SaveParams和SaveArtifacts
```
// SaveParameters will save the content in the specified file path as output parameter value
func (we *WorkflowExecutor) SaveParameters(ctx context.Context) error {
	if len(we.Template.Outputs.Parameters) == 0 {
		log.Infof("No output parameters")
		return nil
	}
	log.Infof("Saving output parameters")
	for i, param := range we.Template.Outputs.Parameters {
		log.Infof("Saving path output parameter: %s", param.Name)
		// Determine the file path of where to find the parameter
		if param.ValueFrom == nil || param.ValueFrom.Path == "" {
			continue
		}

		var output *wfv1.AnyString
		if we.isBaseImagePath(param.ValueFrom.Path) {
			// 从main的文件中读取对应数据
			fileContents, err := we.RuntimeExecutor.GetFileContents(common.MainContainerName, param.ValueFrom.Path)
			if err != nil {
				// We have a default value to use instead of returning an error
				if param.ValueFrom.Default != nil {
					output = param.ValueFrom.Default
				} else {
					return err
				}
			} else {
				output = wfv1.AnyStringPtr(fileContents)
			}
		} else {
			log.Infof("Copying %s from volume mount", param.ValueFrom.Path)
			mountedPath := filepath.Join(common.ExecutorMainFilesystemDir, param.ValueFrom.Path)
			data, err := ioutil.ReadFile(filepath.Clean(mountedPath))
			if err != nil {
				// We have a default value to use instead of returning an error
				if param.ValueFrom.Default != nil {
					output = param.ValueFrom.Default
				} else {
					return err
				}
			} else {
				output = wfv1.AnyStringPtr(string(data))
			}
		}

		// Trims off a single newline for user convenience
        // 直接赋值  感觉无实际用处
		output = wfv1.AnyStringPtr(strings.TrimSuffix(output.String(), "\n"))
		we.Template.Outputs.Parameters[i].Value = output
		log.Infof("Successfully saved output parameter: %s", param.Name)
	}
	return nil
}

// SaveArtifacts uploads artifacts to the archive location
func (we *WorkflowExecutor) SaveArtifacts(ctx context.Context) error {
	if len(we.Template.Outputs.Artifacts) == 0 {
		log.Infof("No output artifacts")
		return nil
	}
	log.Infof("Saving output artifacts")
	err := os.MkdirAll(tempOutArtDir, os.ModePerm)
	if err != nil {
		return errors.InternalWrapError(err)
	}

	for i, art := range we.Template.Outputs.Artifacts {
        // 将文件内容压缩之后 存储到minio中
		err := we.saveArtifact(ctx, common.MainContainerName, &art)
		if err != nil {
			return err
		}
		we.Template.Outputs.Artifacts[i] = art
	}
	return nil
}
```
## emissary
emissary 是main container的启动命令 runtimeexectuor设置为emissary，后面跟实际pod的启动命令，如
```
 - /var/run/argo/argoexec
    - emissary
    - --
    - sh
    - -c
    - sleep 1000000
```
没错，他并不会直接执行你的启动命令，而是在代码中，启动一个子进程来执行你的启动命令
```
func NewEmissaryCommand() *cobra.Command {
	return &cobra.Command{
		Use:          "emissary",
		SilenceUsage: true, // this prevents confusing usage message being printed when we SIGTERM
		RunE: func(cmd *cobra.Command, args []string) error {
			exitCode := 64

			defer func() {
				err := ioutil.WriteFile(varRunArgo+"/ctr/"+containerName+"/exitcode", []byte(strconv.Itoa(exitCode)), 0o644)
				if err != nil {
					logger.Error(fmt.Errorf("failed to write exit code: %w", err))
				}
			}()

			......

			name, err = exec.LookPath(name)
			if err != nil {
				return fmt.Errorf("failed to find name in PATH: %w", err)
			}
                        // 执行你的启动命令
			command := exec.Command(name, args...)
			command.Env = os.Environ()
			command.SysProcAttr = &syscall.SysProcAttr{}
			osspecific.Setpgid(command.SysProcAttr)
			command.Stdout = os.Stdout
			command.Stderr = os.Stderr
			.......
	}
}
```
选择这种方式主要是为了通信，在wait容器中，是不断的读取某一个文件，获取他的状态码以确保main容器执行完成，main container则通过emissary的方式，确保可以感知子进程的运行状态，从而保证wait和main之间的通信
