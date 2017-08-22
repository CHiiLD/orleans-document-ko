# Core
## Observers
### 서문
클러스터가 클라이언트에게 알리는 비동기 단방향 수신 매커니즘이다. 이를 위해 클라이언트 클래스는 `IGrainObserver`를 상속하고 모든 메서드는 void 반환형이야 한다. 알림을 전파하는 grain은 Observer를 추가하거나 제거하고, 구독을 취소하는 API를 제공하여야 한다. 이는 `ObserverSubscriptionManager<T>` 제네릭클래스를 사용하여 단순화 할 수 있다.

클라이언트가 Observer의 Notification을 수신하기 위해서 observer interface를 상속한 인스턴스를 생성하여야 한다. 그리고 정적 observer factory 메서드 `CreateObjectReference()`를 호출하여 인스턴스를 grain reference로 전환한다. 이러면 grain으로부터 구독이 가능하다.

이러한 모델은 다른 grain이 비동기 Notification을 받는데도 사용할 수 있다.
### 예제
메시지를 받는 클라이언트의 인터페이스와 클래스를 정의한다. 주의할 점은 `IGrainObserver` 인터페이스를 상속한다는 점
```
public interface IChat : IGrainObserver
{
    void ReceiveMessage(string message);
}

public class Chat : IChat
{
    public void ReceiveMessage(string message)
    {
        Console.WriteLine(message);
    }
}
```

이제 서버에서 채팅 메시지를 클라이언트에 보내는 grain을 구현해야한다. 또한 알림을 수신하는 구독/탈퇴의 기능이 있어야 한다. 구독기능을 위해 유틸리티 클래스 `ObserverSubscriptionManager`를 사용한다.
```
class HelloGrain : Grain, IHello
{
    private ObserverSubscriptionManager<IChat> _subsManager;

    public override async Task OnActivateAsync()
    {
        // We created the utility at activation time.
        _subsManager = new ObserverSubscriptionManager<IChat>();
        await base.OnActivateAsync();
    }

    // Clients call this to subscribe.
    public Task Subscribe(IChat observer)
    {
        _subsManager.Subscribe(observer);
        return TaskDone.Done;
    }

    //Also clients use this to unsubscribe themselves to no longer receive the messages.
    public Task UnSubscribe(IChat observer)
    {
        _subsManager.Unsubscribe(observer);
        return TaskDone.Done;
    }
}
```

클라이언트에게 메시지를 보내고자 한다면 `ObserverSubscriptionManager`의 `Notify` 메서드를 사용할 수 있다.
```
public Task SendUpdateMessage(string message)
{
    _subsManager.Notify(s => s.ReceiveMessage(message));
    return TaskDone.Done;
}
```


```
//First create the grain reference
var friend = GrainClient.GrainFactory.GetGrain<IHello>(0);
Chat c = new Chat();

//Create a reference for chat usable for subscribing to the observable grain.
var obj = await GrainClient.GrainFactory.CreateObjectReference<IChat>(c);
//Subscribe the instance to receive messages.
await friend.Subscribe(obj);
```
