# DefaultBuildHandlerChain

```go
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
	handler := filterlatency.TrackCompleted(apiHandler)
	handler = genericapifilters.WithAuthorization(handler, c.Authorization.Authorizer, c.Serializer)
	handler = filterlatency.TrackStarted(handler, "authorization")

	if c.FlowControl != nil {
		requestWorkEstimator := flowcontrolrequest.NewWorkEstimator(c.StorageObjectCountTracker.Get, c.FlowControl.GetInterestedWatchCount)
		handler = filterlatency.TrackCompleted(handler)
		handler = genericfilters.WithPriorityAndFairness(handler, c.LongRunningFunc, c.FlowControl, requestWorkEstimator)
		handler = filterlatency.TrackStarted(handler, "priorityandfairness")
	} else {
		handler = genericfilters.WithMaxInFlightLimit(handler, c.MaxRequestsInFlight, c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
	}

	handler = filterlatency.TrackCompleted(handler)
	handler = genericapifilters.WithImpersonation(handler, c.Authorization.Authorizer, c.Serializer)
	handler = filterlatency.TrackStarted(handler, "impersonation")

	handler = filterlatency.TrackCompleted(handler)
	handler = genericapifilters.WithAudit(handler, c.AuditBackend, c.AuditPolicyRuleEvaluator, c.LongRunningFunc)
	handler = filterlatency.TrackStarted(handler, "audit")

	failedHandler := genericapifilters.Unauthorized(c.Serializer)
	failedHandler = genericapifilters.WithFailedAuthenticationAudit(failedHandler, c.AuditBackend, c.AuditPolicyRuleEvaluator)

	failedHandler = filterlatency.TrackCompleted(failedHandler)
	handler = filterlatency.TrackCompleted(handler)
	handler = genericapifilters.WithAuthentication(handler, c.Authentication.Authenticator, failedHandler, c.Authentication.APIAudiences)
	handler = filterlatency.TrackStarted(handler, "authentication")

	handler = genericfilters.WithCORS(handler, c.CorsAllowedOriginList, nil, nil, nil, "true")

	// WithTimeoutForNonLongRunningRequests will call the rest of the request handling in a go-routine with the
	// context with deadline. The go-routine can keep running, while the timeout logic will return a timeout to the client.
	handler = genericfilters.WithTimeoutForNonLongRunningRequests(handler, c.LongRunningFunc)

	handler = genericapifilters.WithRequestDeadline(handler, c.AuditBackend, c.AuditPolicyRuleEvaluator,
		c.LongRunningFunc, c.Serializer, c.RequestTimeout)
	handler = genericfilters.WithWaitGroup(handler, c.LongRunningFunc, c.HandlerChainWaitGroup)
	if c.SecureServing != nil && !c.SecureServing.DisableHTTP2 && c.GoawayChance > 0 {
		handler = genericfilters.WithProbabilisticGoaway(handler, c.GoawayChance)
	}
	handler = genericapifilters.WithAuditAnnotations(handler, c.AuditBackend, c.AuditPolicyRuleEvaluator)
	handler = genericapifilters.WithWarningRecorder(handler)
	handler = genericapifilters.WithCacheControl(handler)
	handler = genericfilters.WithHSTS(handler, c.HSTSDirectives)
	if c.ShutdownSendRetryAfter {
		handler = genericfilters.WithRetryAfter(handler, c.lifecycleSignals.AfterShutdownDelayDuration.Signaled())
	}
	handler = genericfilters.WithHTTPLogging(handler)
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerTracing) {
		handler = genericapifilters.WithTracing(handler, c.TracerProvider)
	}
	handler = genericapifilters.WithLatencyTrackers(handler)
	handler = genericapifilters.WithRequestInfo(handler, c.RequestInfoResolver)
	handler = genericapifilters.WithRequestReceivedTimestamp(handler)
	handler = genericapifilters.WithMuxAndDiscoveryComplete(handler, c.lifecycleSignals.MuxAndDiscoveryComplete.Signaled())
	handler = genericfilters.WithPanicRecovery(handler, c.RequestInfoResolver)
	handler = genericapifilters.WithAuditID(handler)
	return handler
}
```

## filterlatency

### TrackCompleted

```go
// TrackCompleted measures the timestamp the given handler has completed execution and then
// it updates the corresponding metric with the filter latency duration.
// TrackCompleted测量给定的handler完成实现时候的时间戳，并用filter latency（？）时间更新相关metric（矩阵，向量？）
func TrackCompleted(handler http.Handler) http.Handler {
	return trackCompleted(handler, clock.RealClock{}, func(ctx context.Context, fr *requestFilterRecord, completedAt time.Time) {
		latency := completedAt.Sub(fr.startedTimestamp)
    // 感觉就是在metric中记录一下完成时间？
		metrics.RecordFilterLatency(ctx, fr.name, latency)
		if klog.V(3).Enabled() && latency > minFilterLatencyToLog {
			httplog.AddKeyValue(ctx, fmt.Sprintf("fl_%s", fr.name), latency.String())
		}
	})
}
```

```go
func trackCompleted(handler http.Handler, clock clock.PassiveClock, action func(context.Context, *requestFilterRecord, time.Time)) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// The previous filter has just completed.
		completedAt := clock.Now()

		defer handler.ServeHTTP(w, r)

		ctx := r.Context()
		if fr := requestFilterRecordFrom(ctx); fr != nil {
			action(ctx, fr, completedAt)
		}
	})
}
```

### TrackStarted

```go
// TrackStarted measures the timestamp the given handler has started execution
// by attaching a handler to the chain.
// TrackStarted测量给定的handler开始运行的时候的时间（通过将一个handler放入chain中）。
func TrackStarted(handler http.Handler, name string) http.Handler {
	return trackStarted(handler, name, clock.RealClock{})
}
```

```go
func trackStarted(handler http.Handler, name string, clock clock.PassiveClock) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
		if fr := requestFilterRecordFrom(ctx); fr != nil {
			fr.name = name
			fr.startedTimestamp = clock.Now()

			handler.ServeHTTP(w, r)
			return
		}

		fr := &requestFilterRecord{
			name:             name,
			startedTimestamp: clock.Now(),
		}
		r = r.WithContext(withRequestFilterRecord(ctx, fr))
		handler.ServeHTTP(w, r)
	})
}
```

## genericapifilters

### WithAuthorization

```go
// WithAuthorizationCheck passes all authorized requests on to handler, and returns a forbidden error otherwise.
// WithAuthorizationCheck会通过所有认证过的请求，否则就会返回禁止访问的错误
func WithAuthorization(handler http.Handler, a authorizer.Authorizer, s runtime.NegotiatedSerializer) http.Handler {
  // 如果没有认证器，就直接返回handler，（相当于直接通过？）但是会给出警告信息
	if a == nil {
		klog.Warning("Authorization is disabled")
		return handler
	}
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		ctx := req.Context()

		attributes, err := GetAuthorizerAttributes(ctx)
    // 从上下文中获取认证的属性
		if err != nil {
			responsewriters.InternalError(w, req, err)
			return
		}
		authorized, reason, err := a.Authorize(ctx, attributes)
		// an authorizer like RBAC could encounter evaluation errors and still allow the request, so authorizer decision is checked before error here.
    // 像RBAC这样的认证器可能会遇到（评估错误）然后仍然允许这个请求，所以认证器的决定要在这儿的错误发生前就被检查到 ？？ 这里没有明白诶🫡
    // 根据属性使用认证器的Authorize进行属性的认证
		if authorized == authorizer.DecisionAllow {
      // 如果说，认证通过了
			audit.AddAuditAnnotations(ctx,
				decisionAnnotationKey, decisionAllow,
				reasonAnnotationKey, reason)
      // 添加审计信息后，http直接访问
			handler.ServeHTTP(w, req)
			return
		}
		if err != nil {
      // 如果有错误，也要添加审计信息，并给出服务器错误的回应
			audit.AddAuditAnnotation(ctx, reasonAnnotationKey, reasonError)
			responsewriters.InternalError(w, req, err)
			return
		}
		// 如果认证没有通过，就添加禁止访问的审计，并且返回禁止访问的回应
		klog.V(4).InfoS("Forbidden", "URI", req.RequestURI, "Reason", reason)
		audit.AddAuditAnnotations(ctx,
			decisionAnnotationKey, decisionForbid,
			reasonAnnotationKey, reason)
		responsewriters.Forbidden(ctx, attributes, w, req, reason, s)
	})
}
```

#### GetAuthorizerAttributes

```go
func GetAuthorizerAttributes(ctx context.Context) (authorizer.Attributes, error) {
	// 拿到认证器需要认证的结构体
  attribs := authorizer.AttributesRecord{}
	
  // 从http上下文中拿到需要的信息，这里填充了user信息
	user, ok := request.UserFrom(ctx)
	if ok {
		attribs.User = user
	}
	
  // 拿到响应信息，如果没有响应信息，就直接报错。
	requestInfo, found := request.RequestInfoFrom(ctx)
	if !found {
		return nil, errors.New("no RequestInfo found in the context")
	}

	// Start with common attributes that apply to resource and non-resource requests
  // 填充从响应信息中拿到的内容
	attribs.ResourceRequest = requestInfo.IsResourceRequest
	attribs.Path = requestInfo.Path
	attribs.Verb = requestInfo.Verb

	attribs.APIGroup = requestInfo.APIGroup
	attribs.APIVersion = requestInfo.APIVersion
	attribs.Resource = requestInfo.Resource
	attribs.Subresource = requestInfo.Subresource
	attribs.Namespace = requestInfo.Namespace
	attribs.Name = requestInfo.Name

	return &attribs, nil
}
```

### WithPriorityAndFairness

> 如果设置了FlowControl，那么就要使用这个filter

```go
// WithPriorityAndFairness limits the number of in-flight
// requests in a fine-grained way.
// WithPriorityAndFairness更细粒度得限制同时运行请求的数量
func WithPriorityAndFairness(
	handler http.Handler,
	longRunningRequestCheck apirequest.LongRunningRequestCheck,
	fcIfc utilflowcontrol.Interface,
	workEstimator flowcontrolrequest.WorkEstimatorFunc,
) http.Handler {
    // fcIfc 其实就是FlowControl这个属性，如果为空应该不会进来，但这里是为了保险一点。
	if fcIfc == nil {
		klog.Warningf("priority and fairness support not found, skipping")
		return handler
	}
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()
        // RequestInfoFrom 返回 ctx 上的 RequestInfo 键的值
		requestInfo, ok := apirequest.RequestInfoFrom(ctx)
		if !ok {
			handleError(w, r, fmt.Errorf("no RequestInfo found in context"))
			return
		}
        // UserFrom 返回 ctx 上的 user 键的值
		user, ok := apirequest.UserFrom(ctx)
		if !ok {
			handleError(w, r, fmt.Errorf("no User found in context"))
			return
		}
		// 看看在requestInfo.Verb中有没有watch，watchVerbs其实就是watch
		isWatchRequest := watchVerbs.Has(requestInfo.Verb)

		// Skip tracking long running non-watch requests.
        // 跳过追踪长运行且没有watch操作的请求
		if longRunningRequestCheck != nil && longRunningRequestCheck(r, requestInfo) && !isWatchRequest {
			klog.V(6).Infof("Serving RequestInfo=%#+v, user.Info=%#+v as longrunning\n", requestInfo, user)
			handler.ServeHTTP(w, r)
			return
		}
		// PriorityAndFairnessClassification 标识API优先级和公平的分类的结果
		var classification *PriorityAndFairnessClassification
        // noteFn是一个函数，classification会在这里面被设置，而且会写httplog
		noteFn := func(fs *flowcontrol.FlowSchema, pl *flowcontrol.PriorityLevelConfiguration, flowDistinguisher string) {
			classification = &PriorityAndFairnessClassification{
				FlowSchemaName:    fs.Name,
				FlowSchemaUID:     fs.UID,
				PriorityLevelName: pl.Name,
				PriorityLevelUID:  pl.UID}

			httplog.AddKeyValue(ctx, "apf_pl", truncateLogField(pl.Name))
			httplog.AddKeyValue(ctx, "apf_fs", truncateLogField(fs.Name))
		}
		// estimateWork is called, if at all, after noteFn
        // estimateWork 在 noteFn 之后调用，如果有的话
		estimateWork := func() flowcontrolrequest.WorkEstimate {
			if classification == nil {
				// workEstimator is being invoked before classification of
				// the request has completed, we should never be here though.
                // workEstimator 在请求 classification 完成之前被调用，我们不应该在这里。
				klog.ErrorS(fmt.Errorf("workEstimator is being invoked before classification of the request has completed"),
					"Using empty FlowSchema and PriorityLevelConfiguration name", "verb", r.Method, "URI", r.RequestURI)

				return workEstimator(r, "", "")
			}
			// 调用workEstimator来提供评估时间
			workEstimate := workEstimator(r, classification.FlowSchemaName, classification.PriorityLevelName)
			// ObserveWorkEstimatedSeats 记录了与请求相关的估计席位的抽样
            // 这里没有细看，后面就太复杂了，应该就是做了记录
			fcmetrics.ObserveWorkEstimatedSeats(classification.PriorityLevelName, classification.FlowSchemaName, workEstimate.MaxSeats())
			// nolint:logcheck // Not using the result of klog.V
			// inside the if branch is okay, we just use it to
			// determine whether the additional information should
			// be added.
			if klog.V(4).Enabled() {
				httplog.AddKeyValue(ctx, "apf_iseats", workEstimate.InitialSeats)
				httplog.AddKeyValue(ctx, "apf_fseats", workEstimate.FinalSeats)
			}
			return workEstimate
		}

		var served bool
        // nonMutatingRequestVerbs = sets.NewString("get", "list", "watch")
        // nonMutatingRequestVerbs中包含了get、list、watch
        // 如果requestInfo的verb不在这三个里面，那么就是MutatingRequest
		isMutatingRequest := !nonMutatingRequestVerbs.Has(requestInfo.Verb)
        // noteExecutingDelta是一个函数，如果是MutatingRequest，那么就在watermark中记录在recordMutating中，否则就记录在ReadOnly中 这个针对的是atomicMutatingExecuting
		noteExecutingDelta := func(delta int32) {
			if isMutatingRequest {
				watermark.recordMutating(int(atomic.AddInt32(&atomicMutatingExecuting, delta)))
			} else {
				watermark.recordReadOnly(int(atomic.AddInt32(&atomicReadOnlyExecuting, delta)))
			}
		}
        // noteWaitingDelta是一个函数，如果是MutatingRequest，那么就在watermark中记录在recordMutating中，否则就记录在ReadOnly中 这个针对的是atomicReadOnlyWaiting
		noteWaitingDelta := func(delta int32) {
			if isMutatingRequest {
				waitingMark.recordMutating(int(atomic.AddInt32(&atomicMutatingWaiting, delta)))
			} else {
				waitingMark.recordReadOnly(int(atomic.AddInt32(&atomicReadOnlyWaiting, delta)))
			}
		}
        // queueNote是一个函数，根据传入的值，如果是真，就调用noteWaitingDelta(1),否则调用noteWaitingDelta(-1)
		queueNote := func(inQueue bool) {
			if inQueue {
				noteWaitingDelta(1)
			} else {
				noteWaitingDelta(-1)
			}
		}
		// 保存requestInfo和user的信息
		digest := utilflowcontrol.RequestDigest{
			RequestInfo: requestInfo,
			User:        user,
		}

		if isWatchRequest {
            // 如果是watch请求，就执行下面内容
			// This channel blocks calling handler.ServeHTTP() until closed, and is closed inside execute().
			// If APF rejects the request, it is never closed.
            // 此通道阻止调用处理程序。ServeHTTP（） 直到关闭，并在 execute（） 中关闭。如果 APF 拒绝请求，则永远不会关闭该请求。
			shouldStartWatchCh := make(chan struct{})

			watchInitializationSignal := newInitializationSignal()
			// This wraps the request passed to handler.ServeHTTP(),
			// setting a context that plumbs watchInitializationSignal to storage
            // 这会包装传递给处理程序的请求ServeHTTP（），设置一个上下文，
            // 将 watchInitializationSignal 探测到存储
			var watchReq *http.Request
			// This is set inside execute(), prior to closing shouldStartWatchCh.
			// If the request is rejected by APF it is left nil.
            // 这是在 execute（） 中设置的，在关闭 shouldStartWatchCh 之前。
            // 如果请求被 APF 拒绝，则保留为零。
			var forgetWatch utilflowcontrol.ForgetWatchFunc

			defer func() {
				// Protect from the situation when request will not reach storage layer
				// and the initialization signal will not be send.
                // 防止请求无法到达存储层且初始化信号无法发送的情况。
				if watchInitializationSignal != nil {
					watchInitializationSignal.Signal()
				}
				// Forget the watcher if it was registered.
                // 如果观察程序已注册，请忘记它。
				//
				// // This is race-free because by this point, one of the following occurred:
				// case <-shouldStartWatchCh: execute() completed the assignment to forgetWatch
				// case <-resultCh: Handle() completed, and Handle() does not return
				//   while execute() is running
                // 这是race-free的，因为此时发生了以下情况之一：
				// case <-shouldStartWatchCh： execute（） 完成了忘记Watch的作业
				// case <-resultCh： Handle（） 已完成，并且 Handle（） 不返回当执行（）正在运行时
				if forgetWatch != nil {
					forgetWatch()
				}
			}()

			execute := func() {
				startedAt := time.Now()
				defer func() {
					httplog.AddKeyValue(ctx, "apf_init_latency", time.Since(startedAt))
				}()
				noteExecutingDelta(1)
				defer noteExecutingDelta(-1)
				served = true
				setResponseHeaders(classification, w)

				forgetWatch = fcIfc.RegisterWatch(r)

				// Notify the main thread that we're ready to start the watch.
                // 通知主线程我们已准备好启动监视。
				close(shouldStartWatchCh)

				// Wait until the request is finished from the APF point of view
				// (which is when its initialization is done).
                // 从 APF 的角度来看，等到请求完成（这是完成初始化时）。
				watchInitializationSignal.Wait()
			}

			// Ensure that an item can be put to resultCh asynchronously.
            // 确保可以将项目异步放入 resultCh。
			resultCh := make(chan interface{}, 1)

			// Call Handle in a separate goroutine.
			// The reason for it is that from APF point of view, the request processing
			// finishes as soon as watch is initialized (which is generally orders of
			// magnitude faster then the watch request itself). This means that Handle()
			// call finishes much faster and for performance reasons we want to reduce
			// the number of running goroutines - so we run the shorter thing in a
			// dedicated goroutine and the actual watch handler in the main one.
            // 在单独的 go例程中调用句柄。原因是从APF的角度来看，请求处理在手表初始化后立即完成
            //（这通常是比watch请求本身快）。这意味着 Handle（）呼叫完成速度更快，
            // 出于性能原因，我们希望减少
			// 运行 goroutines 的数量 - 所以我们在专用的goroutine和主程序中的实际手表处理程序。
			go func() {
				defer func() {
					err := recover()
					// do not wrap the sentinel ErrAbortHandler panic value
					if err != nil && err != http.ErrAbortHandler {
						// Same as stdlib http server code. Manually allocate stack
						// trace buffer size to prevent excessively large logs
						const size = 64 << 10
						buf := make([]byte, size)
						buf = buf[:runtime.Stack(buf, false)]
						err = fmt.Sprintf("%v\n%s", err, buf)
					}

					// Ensure that the result is put into resultCh independently of the panic.
					resultCh <- err
				}()

				// We create handleCtx with explicit cancelation function.
				// The reason for it is that Handle() underneath may start additional goroutine
				// that is blocked on context cancellation. However, from APF point of view,
				// we don't want to wait until the whole watch request is processed (which is
				// when it context is actually cancelled) - we want to unblock the goroutine as
				// soon as the request is processed from the APF point of view.
				//
				// Note that we explicitly do NOT call the actuall handler using that context
				// to avoid cancelling request too early.
				handleCtx, handleCtxCancel := context.WithCancel(ctx)
				defer handleCtxCancel()

				// Note that Handle will return irrespective of whether the request
				// executes or is rejected. In the latter case, the function will return
				// without calling the passed `execute` function.
				fcIfc.Handle(handleCtx, digest, noteFn, estimateWork, queueNote, execute)
			}()

			select {
			case <-shouldStartWatchCh:
				watchCtx := utilflowcontrol.WithInitializationSignal(ctx, watchInitializationSignal)
				watchReq = r.WithContext(watchCtx)
				handler.ServeHTTP(w, watchReq)
				// Protect from the situation when request will not reach storage layer
				// and the initialization signal will not be send.
				// It has to happen before waiting on the resultCh below.
				watchInitializationSignal.Signal()
				// TODO: Consider finishing the request as soon as Handle call panics.
				if err := <-resultCh; err != nil {
					panic(err)
				}
			case err := <-resultCh:
				if err != nil {
					panic(err)
				}
			}
		} else {
			execute := func() {
				noteExecutingDelta(1)
				defer noteExecutingDelta(-1)
				served = true
				setResponseHeaders(classification, w)

				handler.ServeHTTP(w, r)
			}

			fcIfc.Handle(ctx, digest, noteFn, estimateWork, queueNote, execute)
		}

		if !served {
			setResponseHeaders(classification, w)

			epmetrics.RecordDroppedRequest(r, requestInfo, epmetrics.APIServerComponent, isMutatingRequest)
			epmetrics.RecordRequestTermination(r, requestInfo, epmetrics.APIServerComponent, http.StatusTooManyRequests)
			tooManyRequests(r, w)
		}
	})
}
```

```go
// RequestInfoFrom returns the value of the RequestInfo key on the ctx
// RequestInfoFrom 返回 ctx 上的 RequestInfo 键的值
func RequestInfoFrom(ctx context.Context) (*RequestInfo, bool) {
	info, ok := ctx.Value(requestInfoKey).(*RequestInfo)
	return info, ok
}
```

```go
// UserFrom returns the value of the user key on the ctx
// UserFrom 返回 ctx 上的 user 键的值
func UserFrom(ctx context.Context) (user.Info, bool) {
	user, ok := ctx.Value(userKey).(user.Info)
	return user, ok
}
```

```go
// Has returns true if and only if item is contained in the set.
// 当这个item在set中时Has会返回真
func (s String) Has(item string) bool {
	_, contained := s[item]
	return contained
}
```

```go
// PriorityAndFairnessClassification identifies the results of
// classification for API Priority and Fairness
// PriorityAndFairnessClassification 标识API优先级和公平的分类的结果
type PriorityAndFairnessClassification struct {
	FlowSchemaName    string
	FlowSchemaUID     apitypes.UID
	PriorityLevelName string
	PriorityLevelUID  apitypes.UID
}
```

```go
// WorkEstimate carries three of the four parameters that determine the work in a request.
// The fourth parameter is the duration of the initial phase of execution.
// WorkEstimate包含确定请求中工作的四个参数中的三个。
// 第四个参数是执行初始阶段的持续时间。
type WorkEstimate struct {
	// InitialSeats is the number of seats occupied while the server is
	// executing this request.
    // InitialSeats是服务器执行此请求时占用的席位数。
	InitialSeats uint

	// FinalSeats is the number of seats occupied at the end,
	// during the AdditionalLatency.
    // FinalSeats是在额外延迟期间最后占用的席位数。
	FinalSeats uint

	// AdditionalLatency specifies the additional duration the seats allocated
	// to this request must be reserved after the given request had finished.
	// AdditionalLatency should not have any impact on the user experience, the
	// caller must not experience this additional latency.
    // 额外延迟指定在给定请求完成后必须保留分配给此请求的席位的额外持续时间。
    // 额外延迟不应对用户体验产生任何影响，调用方不得体验此额外延迟。
	AdditionalLatency time.Duration
}
```

```go
// 调用的时候为：
requestWorkEstimator := flowcontrolrequest.NewWorkEstimator(c.StorageObjectCountTracker.Get, c.FlowControl.GetInterestedWatchCount)
---
// NewWorkEstimator estimates the work that will be done by a given request,
// if no WorkEstimatorFunc matches the given request then the default
// work estimate of 1 seat is allocated to the request.
// NewWorkEstimator 估计给定请求将完成的工作，如果没有 WorkEstimatorFunc 与给定的请求匹配，则默认为请求分配 1 个席位的工作估计。
func NewWorkEstimator(objectCountFn objectCountGetterFunc, watchCountFn watchCountGetterFunc) WorkEstimatorFunc {
	estimator := &workEstimator{
		listWorkEstimator:     newListWorkEstimator(objectCountFn),
		mutatingWorkEstimator: newMutatingWorkEstimator(watchCountFn),
	}
	return estimator.estimate
}
```

```go
// ObserveWorkEstimatedSeats notes a sampling of estimated seats associated with a request
// ObserveWorkEstimatedSeats 记录了与请求相关的估计席位的抽样
func ObserveWorkEstimatedSeats(priorityLevel, flowSchema string, seats int) {
	apiserverWorkEstimatedSeats.WithLabelValues(priorityLevel, flowSchema).Observe(float64(seats))
}
```

```go
// watermark tracks requests being executed (not waiting in a queue)
// watermark追踪正在执行的请求（不在队列中等待）
var watermark = &requestWatermark{
	phase:            metrics.ExecutingPhase,
	readOnlyObserver: fcmetrics.ReadWriteConcurrencyObserverPairGenerator.Generate(1, 1, []string{metrics.ReadOnlyKind}).RequestsExecuting,
	mutatingObserver: fcmetrics.ReadWriteConcurrencyObserverPairGenerator.Generate(1, 1, []string{metrics.MutatingKind}).RequestsExecuting,
}
```

```go
// RequestDigest holds necessary info from request for flow-control
// RequestDigest保存来自流控制请求的必要信息
type RequestDigest struct {
	RequestInfo *request.RequestInfo
	User        user.Info
}
```

