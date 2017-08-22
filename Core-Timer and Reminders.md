# Core
## Timer and Reminders
Orleans 런타임은 타이머와 미리 알림이라는 2가지 매커니즘으로 개발자가 Grain에 대한 주기적 동작을 수행할 수 있도록 도와준다.
### 타이머
#### 설명
Grain에 대한 주기적인 동작을 만들기 위해 사용되며, 싱글 스레드 실행(grain 내부에 컨디션 레이스 없음)을 보장한다. 각각의 grain에는 0개 이상의 타이머가 루틴이 실행될 수 있다.
#### 사용법
Grain.RegisterTimer 메서드를 사용한다.
```
protected IDisposable RegisterTimer(Func<object, Task> asyncCallback, object state, TimeSpan dueTime, TimeSpan period)
```
+ asyncCallback - 콜백 메서드
+ state - asyncCallback에 전달될 매개변수
+ dueTime - 첫 번쨰 타이머틱을 실행할 대기시간
+ period - 타이머의 기한을 정한다. 기한으로 타이머를 취소시켜라. 타이머는 deactivation 상태나 silo에서 오류가 발생할 경우 중지됨을 고려해라.
+ activation의 idle state는 타이머에 의해 asyncCallback 메서드가 호출되어도 영향을 받지 않는다. 즉, deactivation을 연기하는데 사용할 수 없다.

### 미리알림
#### 설명
미리알림은 타이머와 비슷하나 차이점이 있다.
+ 명시적인 캔슬이 없는한 영구적이고 주기적으로 실행된다.
+ 미리알림은 (activation 같은) 특정한 상태에 영향을 받지 않는다.
+ Grain이 no activation 상태일 경우, 새로운 grain을 생성한다. 예를 들어 grian이 deactivation 상태에 있는 경우 다음 틱에서 해당 grain을 reactivate한다.
+ 미리알림은 메시지로 전달된다.
+ 미리알림을 high-frequency timer로 사용하면 안된다.

#### 설정
미리알림의 동작은 스토러지에 의존한다. 미리알림의 하위 시스템이 동작되기 위해선 백업 스토러지를 반드시 지정해야한다. 그리고 미리알림은 서버 사이드 환경설정인  SystemStore 요소에 의해 제어된다. Azure table or SQL Server와 함께 작동된다.
```
<SystemStore SystemStoreType="AzureTable" /> OR
<SystemStore SystemStoreType="SqlServer" />
```
또는 개발단계에서 테스트로 사용하기 위해 Azure나 SQL database없이 configuration(Globals 태그 안에)을 설정할 수 있다.
```
<ReminderService ReminderServiceType="ReminderTableGrain"/>
```
#### 사용법
Grain에 `IRemindable.ReceiveReminder` 메서드를 구현해야한다.
```
Task IRemindable.ReceiveReminder(string reminderName, TickStatus status)
{
    Console.WriteLine("Thanks for reminding me-- I almost forgot!");
    return TaskDone.Done;
}
```
미리알림을 시작할려면 `Grain.RegisterOrUndateReminder`를 호출한다.
```
protected Task<IOrleansReminder> RegisterOrUpdateReminder(string reminderName, TimeSpan dueTime, TimeSpan period)
```
+ reminderName - 고유 아이디
+ dueTime - 처음 타이머 틱이 실행될 대기시간
+ period - 타이머의 기한을 지정

캔슬할 경우 `Grain.UnregisterReminder`를 호출한다.
```
protected Task UnregisterReminder(IOrleansReminder reminder)
```
IOrelansReminder인스턴스가 필요한 경우 `Grain.GetReminder` 메서드를 호출한다.
```
protected Task<IOrleansReminder> GetReminder(string reminderName)
```

### 어디에 사용하는가?
어디에 타이머를 사용하는 것을 고려해보아야 하는가?
+ Grain의 deactivation 또는 에러로 동작이 중지되어도 괜찮은 로직
+ 작은 작업일 때
+ Grain의 메서드가 호출되거나 `Grain.OnActivateAsync` 이후에 타이머 콜백함수를 시작할 수 있는 곳

어디에 미리알림을 사용하는 것을 고려해보아야하는가?
+ 일정한 주기의 동작이 에러나 deactivation 상태에서도 동작되어야 할 때
+ 호출 간격이 적을 때

### 타이머와 미리알림의 조합
어떠한 이유로 중지된 Grain의 로컬 타이머를 깨우기 위해 미리알림과 조합하여 사용할 수 있다.
