# Core Streams
## Overview
### Orleans Streams
일련의 이벤트에서 작동하는 반응형 응용 프로그램을 개발할 수 있다.

### 왜 스트림을 알고 있어야 하는가?
[참고](http://dotnet.github.io/orleans/Documentation/Orleans-Streams/Streams-Why.html)

### 프로그래밍 모델
Orleans Streams Programming Model에는 아래와 같은 원칙이 있다.
1. Virtual actor model의 철학에 따라 스트림 또한 항상 존재한다. 명시적으로 생성되거나 파괴되지 않는다.
2. 스트림은 스트림 ID로 식별된다.
3. 시공간에서 처리되는 데이터의 생성을 분리할 수 있다. (?)
4. 가볍고 역동적이다.  
5. 생략

### 프로그래밍 API
생략

### 퀵 스타트 샘플
[참고](http://dotnet.github.io/orleans/Documentation/Orleans-Streams/Streams-Quick-Start.html)

### Stream Provider
스트림은 여러 형태의 물리적 채널을 통해 올 수 있으며, 서로 다른 의미를 가질 수 있다. 따라서 Stream Provider를 통해 다양성을 지원하도록 설계되었다. Orleans는 현재 TCP기반의 Simple Message Stream Provider와 Azure Queue 기반의 Azure Queue Stream Provider를 제공한다. 자세한 내용은 여기를 [참고](http://dotnet.github.io/orleans/Documentation/Orleans-Streams/Stream-Providers.html)

### 스트림의 의미
생략

### 스트림 구현
생략

### 스트림 확장성
생략

### 코드 샘플
[참고](https://github.com/dotnet/orleans/blob/master/test/TestGrains/SampleStreamingGrain.cs)

***
## 퀵 스타트
### 필수 설정
스트림을 생성하기 위해 configuration 설정한다. 테스트를 위해 InMemory 스토러지를 사용한다.
```
<Globals>
    <StorageProviders>
        <Provider Type="Orleans.Storage.MemoryStorage" Name="Default" />
        <Provider Type="Orleans.Storage.MemoryStorage" Name="PubSubStore" />
    </StorageProviders>
    <StreamProviders>
        <Provider Type="Orleans.Providers.Streams.SimpleMessageStream.SimpleMessageStreamProvider" Name="SMSProvider"/>
    </StreamProviders>
```

### 이벤트 제작
configuration에서 설정한 Strema Provider(`SMSProvider`)에 접근하여 스트림을 선택한다.
```
//Pick a guid for a chat room grain and chat room stream
var guid = some guid identifying the chat room
//Get one of the providers which we defined in config
var streamProvider = GetStreamProvider("SMSProvider");
//Get the reference to a stream
var stream = streamProvider.GetStream<int>(guid, "RANDOMDATA");
```

`OnNext` 메서드로 데이터를 스트림에 푸시할 수 있다.
```
RegisterTimer(s =>
    {
        return stream.OnNextAsync(new System.Random().Next());
    }, null, TimeSpan.FromMilliseconds(1000), TimeSpan.FromMilliseconds(1000));

```

### 스트리밍 데이터의 구독과 수신
데이터를 수신하기 위해서 암시/명시적으로 구독을 설정할 수 있다. 지금은 암시적 구독을 사용하기 위해 `ImplicitStreamSubscription(namespace)`를 사용한다.
```
[ImplicitStreamSubscription("RANDOMDATA")]
public class ReceiverGrain : Grain, IRandomReceiver
```
타이머가 데이터를 푸시할 때 마다 `ReceiverGrain`은 메시지를 계속 수신한다. grain이 deactivation 상태일 때는 새로운 grain을 생성하고 메시지를 보낸다.

다음은 `ReceiverGrain`에서 데이터 수신을 설정하는 방법이다.
```
//Create a GUID based on our GUID as a grain
var guid = this.GetPrimaryKey();
//Get one of the providers which we defined in config
var streamProvider = GetStreamProvider("SMSProvider");
//Get the reference to a stream
var stream = streamProvider.GetStream<int>(guid, "RANDOMDATA");
//Set our OnNext method to the lambda which simply prints the data, this doesn't make new subscriptions
await stream.SubscribeAsync<int>(async (data, token) => Console.WriteLine(data));
```

***
## Streams Programming API
### 비동기 스트림
Stream Provider로 스트림에 대한 핸들링을 가져온다. `Grain` 클래스 내부에서 `GetStreamProvider()` 메서드를 호출하거나 클라이언트에서 `GrainClient.GetStreamProvider()`로 Stream Provider에 대한 참조를 가져올 수 있다.
```
IStreamProvider streamProvider = base.GetStreamProvider("SimpleStreamProvider");
IAsyncStream<T> stream = streamProvider.GetStream<T>(Guid, "MyStreamNamespace");
```
`Orleans.Streams.IAsyncStream<T>`는 가상 스트림을 핸들링하기 위한 가장 적합한 인스턴스이다. 인자는 스트림 네임스페이스와 GUID로 구성한다.

### 이벤트 생성과 구독
`IAsyncStream<T>` 인터페이스는 `Orelans.Streams.IAsyncObserver<T>`(새로운 이벤트를 생성), `Orlenas.Streams.IAsyncObservable<T>`(구독, 이벤트 처리) 인터페이스를 모두 상속한다.
```
public interface IAsyncObserver<in T>
{
    Task OnNextAsync(T item, StreamSequenceToken token = null);
    Task OnCompletedAsync();
    Task OnErrorAsync(Exception ex);
}

public interface IAsyncObservable<T>
{
    Task<StreamSubscriptionHandle<T>> SubscribeAsync(IAsyncObserver<T> observer);
}
```

스트림에서 이벤트를 생성하고자한다면, 아래의 메서드를 호출
```
await stream.OnNextAsync<T>(event)
```

스트림을 구독하고자 한다면, 다음을 호출
```
StreamSubscriptionHandle<T> subscriptionHandle = await stream.SubscribeAsync(IAsyncObserver)
```

`IAsyncObserver`는 `AsyncObservableExtensions` 클래스로 확장할 수 있다.
리턴값 `StreamSubscriptionHandle<T>`는 스트림의 구독을 취소하는데 사용한다.
```
await subscriptionHandle.UnsubscribeAsync()
```

또한, 스트림의 __구독__ 은 __grain의 수명주기와 상관없이 동작__ 됨이 중요하다.

### 중복 구독
스트림은 다수의 프로듀서(이벤트를 생산하는 인스턴스)와 구독자를 가질 수 있다. 프로듀서로부터 발행(Publush)된 메시지는 모든 구독자에게 전파된다. 구독자는 동일한 스트림을 중복하여 구독할 수 있는데, 만약에 grain or client가 동일한 스트림을 n번 중복하여 구독신청했다면 매 시간마다 n번의 이벤트가 전달될 것이다. 구독자는 각각의 구독 또는 현재 수신중인 모든 구독을 취소할 수 있다.
```
IList<StreamSubscriptionHandle<T>> allMyHandles = await IAsyncStream<T>.GetAllSubscriptionHandles()
```

### 실패 복구
스트림 프로듀서가 죽어도(혹은 deactivation된 경우) 별 다른 처리를 하지 않아도 된다. grain이 이벤트를 생성하길 원할 때, 스트림 핸들링을 얻고 동일한 방식으로 이벤트를 생성할 수 있다.  
구독자가 죽고 (또는 deactivation된 경우) 새로운 이벤트가 스트림으로부터 생성되면, 구독자 grain은 자동적으로 re-activation된다. 구독자가 해야할 일은 데이터를 처리할 `IAsyncObserver<T>`를 제공하는 것이다. 구독자는 `OnActivateAsync` 메서드에서 다음과 같은 로직을 추가할 필요가 있다.

```
StreamSubscriptionHandle<int> newHandle = await subscriptionHandle.ResumeAsync(IAsyncObserver)
```
구독자는 이벤트 처리를 위해 처음에 구독한 이전 핸들을 사용한다. `ResumeAsync`는 단지 새로운 `IASyncObserver` 인스턴스와 함께 기존의 구독을 업데이트할 뿐이다. 구독자는 이미 스트림에 가입되어 있다는 사실은 변하지 않는다.  

구독자가 기존의 구독을 유지하는 방법은 `SubscribeAsync` 메서드로 리턴된 핸들을 유지하거나(위의 내용처럼), `IAsyncStream<T>`를 호출하여 구독중인 핸들을 요청할 수 있다.
```
IList<StreamSubscriptionHandle<T>> allMyHandles = await IAsyncStream<T>.GetAllSubscriptionHandles()
```

### 명시/암시적 구독
기본적으로 스트림의 구독자는 명시적으로 구독한다. 또한 스트림은 __암시적 구독__ 을 지원한다. grain은 자동적으로 구독되며, `ImplicitStreamSubscription`어트리뷰트를 사용한다.  

`MyGrainType` 클래스를 구현한 grain은 `ImplicitStreamSubscription("MyStreamNamespace")` 어트리뷰트를 선언할 수 있다.
이는 런타임에 `MyStreamNamespace`라는 네임스페이스와 XXX라는 ID를 가진 스트림에서 이벤트가 생성되었을 때, XXX의 ID를 가진 `MyGrainType` 인스턴스(구독자)에 전파된다. 스트림 `<XXX, MyStreamNamespace>`은 구독자 grain `<XXX, MyGrainType>`에 매핑된다.  
이제 `ImplicitStreamSubscription`에 따라 런타임은 자동으로 grain이 스트림을 구독하게 하고 이벤트를 스트림으로 전달한다. 그러나 grain은 이벤트 처리 코드를 작성해야 한다. 따라서 grain이 activation되면 `OnActivateAsync` 메서드 내에서 처리하자.
```
IStreamProvider streamProvider = base.GetStreamProvider("SimpleStreamProvider");
IAsyncStream<T> stream = streamProvider.GetStream<T>(this.GetPrimaryKey(), "MyStreamNamespace");
StreamSubscriptionHandle<T> subscription = await stream.SubscribeAsync(IAsyncObserver<T>);
```
### 구독 로직 작성하기
암시적 구독은 모든 스트림 네임 스페이스에 대해 암시적으로 하나의 구독만 가진다. 따라서 여러 구독(중복 구독도 포함)을 만들 수 있는 방법이 없다. 또한 구독을 취소할 수 없다.  
명시적 구독은 재구독(Resume) 기능이 필요하다. 그렇지 않으면 grain이 재시작할 때 마다, 중복으로 구독을 신청하게 된다.
#### 암시적 구독
암시적 구독은 이벤트 처리 로직만 코드로 짜면 된다. 간단하게 `OnActivateAsync` 메서드 내에서 `await stream.SubscribeAsync(OnNext...)`을 실행하면 된다. `OnNext`메서드는 이벤트를 처리하는 함수이다. 또한 to 인수로 `StreamSequenceToken` 인스턴스를 넣을 수도 있다. 또한 암시적 구독에서 `ResumeAsync`를 사용할 필요는 없다.
```
public async override Task OnActivateAsync()
{
    var streamProvider = GetStreamProvider(PROVIDER_NAME);
    var stream = streamProvider.GetStream<string>(this.GetPrimaryKey(), "MyStreamNamespace");
    await stream.SubscribeAsync(OnNextAsync)
}
```
#### 명시적 구독
명시적 구독을 위해 grian은 반드시 `SubscribeAsync`를 호출해야한다. 이렇게 하면 구독을 생성하고 핸들링 로직(콜백)을 추가할 수 있다. 명시적 구독은 grain이 구독 취소할 때까지 존재함으로, grain이 deactivation에서 activation되면 명시적으로 가입되지면 핸들링 로직은 추가되지 않는다. 이 경우 grain은 핸들링 로직을 다시 추가해 주어야 한다. 이를 위해 grain은 `OnActivateAsync` 메서드 내에서 자신이 어떤 것을 구독중인지를 알기 위해 `stream.GetAllSubscriptionHandles()`를 호출한다. 그리고 원하는 핸들링 로직을 다시 추가하기 위해 `ResumeAsync`를 반드시 실행한다.(혹은 구독 해지를 위해 `UnsubscribeAsync`를 사용할수도...) grain은 선택적으로 `StreamSequenceToken`을 `ResumeAsync` 메서드의 인자로 넣을 수 있다. 그러면 토큰 인스턴스에서 구독을 처리하게 한다.

```
public async override Task OnActivateAsync()
{
    var streamProvider = GetStreamProvider(PROVIDER_NAME);
    var stream = streamProvider.GetStream<string>(this.GetPrimaryKey(), "MyStreamNamespace");
    var subscriptionHandles = await stream.GetAllSubscriptionHandles();
    if (!subscriptionHandles.IsNullOrEmpty())
        subscriptionHandles.ForEach(async x => await x.ResumeAsync(OnNextAsync));
}
```

### 스트림 순서와 Sequence Tokens
이벤트의 전달(Delivery)순서 방식은 프로듀서와 구독자의 stream provider에 따라 다르다.  
SMS(Simpel Message Stream Provider)는 기본적으로 프로듀서가 `OnNextAsync`를 호출하면 FIFO순서로 이벤트가 전달된다. `OnNextAsync`가 리턴한 `Task`가 의미한 실패를 의미한다면, 이를 처리하는 것은 프로듀서가 결정한다.  
Azure Queue 스트림은 FIFO 순서를 보장하지 않는다. 이벤트를 생성하고 Enqueue 작업이 실패하면, 다른 대기열에 Enqueue 작업을 시도한다. 이러한 잠재적인 중복 메시지를 처리하는 것은 프로듀서의 책임이다. Orelans 스트리밍 런타임은 전달과 처리가 성공적으로 이루어졌을 때, Queue에서 이벤트를 삭제한다. 전달 또는 처리가 실패되면 삭제되지 않고 자동으로 Queue에 들어간다. 따라서 FIFO 순서를 보장하지 않는다.  

위의 순서 문제를 해결하기 위해 우리는 `StreamSequenceToken`을 사용하여, 선택적으로 순서를 지정할 수 있다.

### Rewinable Streams
현재 구현중   
Akka의 Persistence Actor의 Recover 매커니즘과 비슷하다.

### Stateless 자동 스케일 아웃 처리
현재 구현 중

### Grain and Orleans Client
스트림은 grain과 cleint에 동일하게 작동한다.

### Pub-Sub 스트리밍
스트림의 구독을 추적하기 위해 구독자와 프로듀서의 집결지 역할을 하는 Streaming Pub-Sub 런타임 설정 요소를 제공한다. Pub-Sub은 모든 구독을 추적하고 구독자와 프로듀서를 매칭시킨다.

이하 생략

### 구성
스트림을 사용할려면 stream provider를 활성화해야한다. stream provider에 대한 내용은 [여기](http://dotnet.github.io/orleans/Documentation/Orleans-Streams/Stream-Providers.html)를 참조
```
<OrleansConfiguration xmlns="urn:orleans">
  <Globals>
    <StreamProviders>
      <Provider Type="Orleans.Providers.Streams.SimpleMessageStream.SimpleMessageStreamProvider" Name="SMSProvider"/>
      <Provider Type="Orleans.Providers.Streams.AzureQueue.AzureQueueStreamProvider" Name="AzureQueueProvider"/>
    </StreamProviders>
  </Globals>
</OrleansConfiguration>
```
또는 코드로 등록할 수 있다.
```
public void RegisterStreamProvider(string providerTypeFullName, string providerName, IDictionary<string, string> properties = null)

public void RegisterStreamProvider<T>(string providerName, IDictionary<string, string> properties = null) where T : IStreamProvider
```

## Stream Providers
## Stream 구현하기
## Stream 확장하기
