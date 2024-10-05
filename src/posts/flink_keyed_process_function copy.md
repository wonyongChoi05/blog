---
title: Flink Task LifeCycle
description: 이 글에서는 JobMaster가 ExecutionGraph를 생성하여 TaskExecutor에 제출하는 전체 프로세스의 순서와, Task와 StreamTask의 라이프사이클에 대해 설명한다.
permalink: posts/{{ title | slug }}/index.html
date: "2024-10-05"
updated: "2024-10-05"
tags: [Flink, Flink Task, Flink LifeCycle]
---

## 들어가며
이 글에서는 JobMaster가 ExecutionGraph를 생성하여 TaskExecutor에 제출하는 전체 프로세스의 순서와, Task와 StreamTask의 라이프사이클에 대해 소개한다.

# 객체와 개념

## Task
- Flink의 분산 실행을 위한 기본 단위.
- 독립적인 스레드로 실행되며 여러 Task가 하나의 Task slot에 포함될 수 있음.
- Operator 체인에서 Element(Input element, Watermark, Checkpoint barriers)를 전달하고 처리.

## Operator
- 데이터 처리의 논리를 구현한 작업 단위.
- 병렬성에 따라 여러 **ExecutionVertex**를 포함.
- 연속된 Operator는 조건에 따라 하나의 **JobVertex**로 체인(Chained)될 수 있음.

## JobVertex
- **JobGraph**의 구성 단위로, 여러 **Chained operator**를 하나로 묶은 것.
- 각 JobVertex는 병렬 버전의 여러 **ExecutionVertex**로 확장됨.

## ExecutionGraph
- 전체 작업의 병렬화 구조를 나타내는 그래프.
- JobVertex를 여러 ExecutionVertex로 확장한 구조로 작업의 병렬 처리를 수행.

## ExecutionJobVertex
- JobGraph의 한 정점을 나타내며, map 또는 join과 같은 연산을 수행
- 모든 병렬 하위 작업의 집계된 상태를 보유
- JobGraph의 해당 JobVertex에서 가져온 JobVertexID로 식별됨

## ExecutionVertex
- 하나의 병렬 하위 작업을 나타냄

![image.png](/images/flink-subtask.png)
            
- 여기서 각 하위작업은 단일 스레드로 처리되는데, 아래서 설명
- 각 ExecutionJobVertex와 병렬 하위 작업의 인덱스에 의해 식별됨

## Execution
- ExecutionVertex를 실행하려는 시도를 나타냄
- 실패할 경우 또는 후속 작업에서 요청할 때 데이터가 더 이상 사용 가능하지 않은 경우에 대해 여러 번의 Execution이 발생할 수 있음
- ExecutionAttempID로 식별하고, JM, TM 간의 모든 메시지는 이 ID를 이용하여 수신자를 지정함

## StreamTask
- Flink의 스트림 처리를 위한 모든 Task 하위 유형의 기본 클래스.
- 라이프사이클 동안 **OperatorChain**을 사용하여 여러 Operator를 조작하고 실행.

## OperatorChain
- 하나의 Task 내에서 여러 Operator를 연결하여 파이프라인 형태로 묶는 역할.
- 각 Element를 Operator 간에 전달하면서 처리하는 구조.

### 간단한 흐름
- Task 안에서 Operator들이 데이터를 처리
- 각 Operator는 병렬적으로 여러 ExecutionVertex로 실행
- JobVertex는 이러한 Operator를 묶어 하나의 단위로 처리하고, 전체 작업은 ExecutionGraph를 통해 병렬처리

## Execution Graph의 생성 및 배포

1. `Dispatcher`는 Flink에서 Job을 수신하고 관리하는 주요 컴포넌트다.`Dispatcher`는 Job을 수신하면 `JobManagerRunner`를 생성하고, 그 안에서 `JobMaster`가 생성된다.
2.  이때 `JobMaster`는 `JobGraph`를 통해 `ExecutionGraph`를 생성하며, `ExecutionGraph`는 작업의 물리적인 병렬화 실행 계획을 나타낸다.
> JobMaster가 초기화될 때 `createScheduler(…)`를 호출하여 ExecutionGraph를 생성하며, 생성된 그래프는 `schedulerNG` 변수에 저장한다.
3. 변수에 저장했던 SchedulerNG의 `startScheduling()` 메서드를 통해 SchedulerNG를 구현하고 있는 SchedulerBase가 트리거되는데, 이 때 `createScheduler(...)` 로 초기화할 때 생성되었던 executionGraph가 사용된다.
> SchedulerBase#createAndRestoreExecutionGraph

## Task 제출 및 실행

1. Execution의 `deploy()` 메서드는 TaskManagerGateway의 `submitTask(…)` 를 사용하여 Task를 제출한다. 
2. `RpcTaskManagerGateway` 인스턴스를 통해 TMRunner 프로세스로 전달되고, Akka RPC를 통해 해당 Task가 TM으로 전달된다.
3. TMExecutor의 `submitTask()` 는 RPC로 전달된 task를 수신하여, 필요한 데이터를 준비한 후 Task 객체를 생성한다. 생성된 task는 `startTaskThread()` 메서드를 통해 스레드로 실행한다.
    
> TaskExecutor#submitTask
```java
    @Override
        public CompletableFuture<Acknowledge> submitTask(
                TaskDeploymentDescriptor tdd, JobMasterId jobMasterId, Time timeout) {
    				...
    				try {
    						...
    
                Task task =
                        new Task(
                                jobInformation,
                                taskInformation,
                                tdd.getExecutionAttemptId(),
                                tdd.getAllocationId(),
                                tdd.getProducedPartitions(),
                                tdd.getInputGates(),
                                memoryManager,
                                sharedResources,
                                taskExecutorServices.getIOManager(),
                                taskExecutorServices.getShuffleEnvironment(),
                                taskExecutorServices.getKvStateService(),
                                taskExecutorServices.getBroadcastVariableManager(),
                                taskExecutorServices.getTaskEventDispatcher(),
                                externalResourceInfoProvider,
                                taskStateManager,
                                taskManagerActions,
                                inputSplitProvider,
                                checkpointResponder,
                                taskOperatorEventGateway,
                                aggregateManager,
                                classLoaderHandle,
                                fileCache,
                                taskManagerConfiguration,
                                taskMetricGroup,
                                partitionStateChecker,
                                MdcUtils.scopeToJob(jobId, getRpcService().getScheduledExecutor()),
                                channelStateExecutorFactoryManager.getOrCreateExecutorFactory(jobId));
    						...
                boolean taskAdded;
    
                try {
                    taskAdded = taskSlotTable.addTask(task);
                } catch (SlotNotFoundException | SlotNotActiveException e) {
                    throw new TaskSubmissionException("Could not submit task.", e);
                }
    
                if (taskAdded) {
    		            // TaskThread 시작
                    task.startTaskThread();
    
                    setupResultPartitionBookkeeping(
                            tdd.getJobId(), tdd.getProducedPartitions(), task.getTerminationFuture());
                    return CompletableFuture.completedFuture(Acknowledge.get());
                } else {
                    ...
                }
            } catch (TaskSubmissionException e) {
                return FutureUtils.completedExceptionally(e);
            }
        }
```
    
- Task는 Thread를 상속하고, Runnable을 구현하므로, 각 Task 객체는 독립적인 스레드로 실행된다.
    
* Task 시그니처
    
```java
public class Task
        implements Runnable, TaskSlotPayload, TaskActions, PartitionProducerStateProvider { }
```
    
1. 이 때 Task 내부에서 StreamTask가 생성되는데, StreamTask는 Flink의 주요 태스크로, 실제 스트림 데이터 처리를 담당한다.
2. Streamtask의 `invoke()` 메서드에서는 `OperatorChain` 이 생성되며, OperatorChain은 현재 Task에 속한 모든 Operator를 관리하고 연결한다.
3. OperatorChain의 생성자에서 StreamTask에 연결된 Operator들을 체인형태로 초기화한다.
- StreamTask#invoke        
```java
        @Override
            public final void invoke() throws Exception {
                // task가 실행중이 아니라면 restoreInternal()로 task 상태를 복구한다.
                // restoreInternal()은 실제로 체크포인트나 세이브포인트로부터의 상태 복원을 담당한다.
                // 만약 체크포인트나 세이브포인트가 없다면 단순히 초기 상태로 설정된다.
                if (!isRunning) {
                    LOG.debug("Restoring during invoke will be called.");
                    restoreInternal();
                }
        
                // task 취소 여부를 확인한다.
                // 이미 취소 상태라면 실행을 멈추고 종료한다.
                // task가 중간에 취소되거나 오류가 발생하면, gracefulshutdown을 위해 이 부분에서 확인한다.
                ensureNotCanceled();
        
        				// 버퍼 관리를 최적화하기 위한 스케줄링 작업
                scheduleBufferDebloater();
        
                // metricgroup을 사용하여 태스크의 시작을 마킹한다.
                // 모니터링 및 성능 분석등에 사용된다.
                getEnvironment().getMetricGroup().getIOMetricGroup().markTaskStart();
                // 이 부분이 stream task의 가장 중요한 부분인데,
                // 메일박스는 데이터 처리, 타이머 이벤트, 체크포인트 트리거 등의 메시지를 관리하고 있다.
                // 따라서 이 메서드에서는 메일박스를 계속 확인하여 메시지를 처리하고, 데이터의 흐름과 이벤트를 처리한다.
                runMailboxLoop();
                
                // runMailboxLoop가 정상적으로 종료되었는지 확인한다.
        	      ensureNotCanceled();
        				// 리소스를 정리한다.
                afterInvoke();
            }
        
```
        

### 하나의 Task안에 있는 여러 개의 Operator를 체이닝 하는 순서 분석

OperatorChain은 아래와 같은 플링크 코드의 연산마다 생성된다. ex)

```java
DataStream<String> sourceStream = env.fromSource(kafkaSource, ...);
DataStream<UserClickCount> aggregatedStream = sourceStream
    .map(value -> new ClickEvent(value)) // 내부적으로 MapOperator 생성
    .keyBy(clickEvent -> clickEvent.getUserId()) // Keyed Aggregation Operator 생성
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .sum("clickCount"); // 집계 연산을 수행하는 Operator 생성

aggregatedStream.addSink(new RedisSink<>(...)); // RedisSinkOperator 생성
```

1. Operator 이벤트 디스패처 초기화 (OperatorChain#OperatorChain)
```java
this.operatorEventDispatcher =
                new OperatorEventDispatcherImpl(
                        containingTask.getEnvironment()getUserCodeClassLoader().asClassLoader(),
                        containingTask.getEnvironment()getOperatorCoordinatorEventGateway());
```
    
- `OperatorEventDispatcherImpl` 는 Operator간의 이벤트 전송, 처리 등을 담당

2. StreamConfig와 chainedConfigs 로드 (OperatorChain#OperatorChain)
```java
final StreamConfig configuration = containingTask.getConfiguration();
Map<Integer, StreamConfig> chainedConfigs =
         configuration.getTransitiveChainedTaskConfigsWithSelf(userCodeClassloader);
```
    
- StreamConfig는 현재 stream task에 대한 정보를 제공한다.
- `chainedConfigs`는 체인으로 연결된 `Operator`들의 설정 정보를 포함하며, 이 설정을 통해 `OperatorChain`이 어떤 `Operator`로 구성되어야 하는지 결정된다.

3. Output 생성    
```java
List<NonChainedOutput> outputsInOrder =
                configuration.getVertexNonChainedOutputs(userCodeClassloader);
        Map<IntermediateDataSetID, RecordWriterOutput<?>> recordWriterOutputs =
                CollectionUtil.newHashMapWithExpectedSize(outputsInOrder.size());
        this.streamOutputs = new RecordWriterOutput<?[outputsInOrder.size()];
        this.finishedOnRestoreInput =
                this.isTaskDeployedAsFinished()
                        ? new FinishedOnRestoreInput(
                                streamOutputs, configuration.getInputs(userCodeClassloader).length)
                       : null;
```
    
- 현재 StreamTask의 Operator에서 생성된 데이터를 내보내기 위한 출력(output) 설정을 초기화한다.
- RecordWriteroutput은 각 operator가 데이터 처리 후 결과를 내보내는 역할이다.

4. 체인 출력 생성    
```java
createChainOutputs(
                    outputsInOrder,
                    recordWriterDelegate,
                    chainedConfigs,
                    containingTask,
                    recordWriterOutputs);
```
- `Operator` 간의 연결 및 데이터를 전달할 수 있는 출력(`Output`)을 생성한다. 각 `Operator`는 `OutputCollector`를 통해 다음 `Operator`로 데이터가 전달될 수 있도록 설정되며, 실제 `OperatorChain`에서 각 `Operator`가 서로 연결되어 파이프라인 형태로 데이터가 전달된다.

5. 메인 Operator 생성
```java
    if (operatorFactory != null) {
        Tuple2<OP, Optional<ProcessingTimeService>> mainOperatorAndTimeService =
                StreamOperatorFactoryUtil.createOperator(
                        operatorFactory,
                        containingTask,
                        configuration,
                        mainOperatorOutput,
                        operatorEventDispatcher);
    
        OP mainOperator = mainOperatorAndTimeService.f0;
        mainOperator
                .getMetricGroup()
                .gauge(
                        MetricNames.IO_CURRENT_OUTPUT_WATERMARK,
                        mainOperatorOutput.getWatermarkGauge());
        this.mainOperatorWrapper =
                createOperatorWrapper(
                        mainOperator,
                        containingTask,
                        configuration,
                        mainOperatorAndTimeService.f1,
                        true);
    
        // add main operator to end of chain
        allOpWrappers.add(mainOperatorWrapper);
    
        this.tailOperatorWrapper = allOpWrappers.get(0);
    }
```    
- `createOperatorWrapper()`는 메인 `Operator`를 래핑하고 메트릭 그룹을 통해 실행 중의 `Watermark` 등 메트릭을 모니터링할 수 있게 한다. 이 과정으로 인해 우리는 각 연산마다 메트릭을 자세히 모니터링 할 수 있게 되는 것..

6. 체인 연결하고 초기화
```java
    this.numOperators = allOpWrappers.size();
    firstOperatorWrapper = linkOperatorWrappers(allOpWrappers);
```
    
- `linkOperatorWrappers()`는 각 `Operator`의 순서에 맞게 **연결을 설정**하며, 첫 번째 `OperatorWrapper`가 `firstOperatorWrapper`로 설정한다.
- 이 부분에서 각 `Operator`가 순차적으로 실행될 수 있도록 모든 `OperatorWrapper`가 연결되며, `OperatorChain`이 완전히 구성된다.

## 요약
- `JobMaster`는 `ExecutionGraph`를 생성하고 스케줄링하여 `TaskManager`에 Task를 제출한다. (rpc를 통해)
- `Execution` 단위로 `deploy()`를 통해 `TaskManager`에 Task가 전달되고, `TaskExecutor`는 이 Task를 수신하여 스레드로 실행한다.
- 각 Task는 독립적인 스레드로 실행되며, 내부적으로 `StreamTask`를 통해 모든 Operator가 실행된다.