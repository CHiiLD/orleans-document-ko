# Getting Started
## Grains
### Grains
Grains는 Orleans 프로그래밍의 핵심 요소이다. Akka의 Actor 또는 OOP의 객체와 비슷한 성격을 가진다.   
+ 공유 메모리 사용 안함
+ 1 Grain 1 Thread

### Grain Identity
Grain은 ID(고유식별자)를 가진다.

### Grain 접근하기
Grain의 인터페이스와 ID를 사용하여 호출한다.
```
var user = GrainFactory.GetGrain<IUserProfile>(userEmail);
await user.UpdateAddress(newAddress);
```
### Grain Lifecycle
#### Grain 수명주기
1. 클러스터의 silo(Grain의 집합)중에 grain 인스턴스가 있는지 확인 -> 없으면 런타임 생성(Activation)
2. Grain이 Grain Persistence를 사용할 경우 생성된 Grain이 backing store에서 상태를 읽음
3. Silo에서 활성화되면 grain은 작업 처리
4. Grain이 일정한 시간동안 idle 상태일 때, grain은 비활성화(Deactivation)상태가 된다.
5. grain은 persisted 상태가 되어 메모리에서 제거된다.

![Grain Lifecycle](images/grain-lifecycle.png)

#### Grain 핵심 이벤트 순서
1. 타 grain이나 클라이언트로부터 grain인스턴스로 grain의 메서드 호출
2. grain activation
  + [Dependency Injection](http://dotnet.github.io/orleans/Documentation/Core-Features/Dependency-Injection.html)를 사용시, grain의 생성자가 레버러지됨(?)
  + [Declarative Persistence](http://dotnet.github.io/orleans/Documentation/Core-Features/Grain-Persistence.html)를 사용시, grain은 storage에서 상태를 읽어들인다.
  + `OnActivateAsync` 호출
3. grain 작업 처리
4. grain이 일정한 시간동안 idle상태
5. Silo는 grain을 Deactivation 상태로 전환
6. Silo는 `OnDeactivateAsync` 호출
7. Silo는 메모리에서 grain제거

#### Silo 종료와 복구
Slio를 종료하면 종속된 모든 grain은 Deactivation된다. grain큐에 저장된 모든 요청은 다른 slio에 전달된다.  
Slio가 비정상적으로 종료되면 클러스터의 타 slio가 오류를 감지하고, 요청이 들어오면, 종료된 slio에서 grain을 다시 activation한다.

### Chunk
grain은 chunk로 작업을 수행하고 끝난 다음, 다음 chunk로 이동한다. 이러한 chunk의 작업 단위를 turn이라 한다. 이들 각 turn들은 순차적으로 실행된다.

***
## Grain으로 개발해보기
### 설치
.NET 4.6.1이상의 프로젝트
```
PM> Install-Package Microsoft.Orleans.OrleansCodeGenerator.Build
```

### Interface And Class
grain은 인터페이스 타입으로 메서드를 외부에 노출한다. 모든 메서드는 Task를 반환형이다.
```
//an example of a Grain Interface
public interface IPlayerGrain : IGrainWithGuidKey
{
    Task<IGameGrain> GetCurrentGame();
    Task JoinGame(IGameGrain game);
    Task LeaveGame(IGameGrain game);
}

//an example of a Grain class implementing a Grain Interface
public class PlayerGrain : Grain, IPlayerGrain
{
    ...
}
```

### Task Return
__void가 아닌 경우__
```
public Task<SomeType> GrainMethod1()
{
    ...
    return Task.FromResult(<variable or constant with result>);
}
```
__void 인 경우__
```
public Task GrainMethod2()
{
    ...
    return Task.CompletedTask;
}
```
__메서드에 async가 붙으면__
```
public async Task<SomeType> GrainMethod3()
{
    ...
    return <variable or constant with result>;
}

public async Task GrainMethod4()
{
    ...
    return;
}
```

### Grain 참조하기
Grain Reference는 grain 클래스와 동일한 인터페이스를 구현하는 프록시(대리) 인스턴스, `GrainFactory.GetGrain<T>(key)` 메서드로 참조 가능하다.

__Grain 클래스 내부에서 참조할 경우__
```
//construct the grain reference of a specific player
IPlayerGrain player = GrainFactory.GetGrain<IPlayerGrain>(playerId);
```
__Orleans Client를 사용할 경우__
```
IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
```

### Task 메서드 비동기 호출
생략

### Virtual 메서드
+ `OnActivateAsync` - grain 초기화할 때 사용한다.
+ `OnDeactivateAsync` - __모든 상황에서 호출되는 것을 보장하지 않는다. 따라서 사용에 주의해야함__

***
## Client 개발하기
### What Is Grain Client?
Grain 클라이언트는 클러스터와 프로그램의 모든 grain과 연결한다.
![](images/Frontend-Cluster.png)
일반적으로 클라이언트는 웹 프론트엔드 서버에서 클러스터와 연결되는데 사용된다. 일반적은 프론트엔드 서버는 아래와 같은 일을 한다.
+ 웹 요청 수신
+ 인증과 권한 부여
+ 클러스터의 grain의 메서드 호출
+ grain의 리턴값을 처리
+ 요청에 대한 응답 보내기

### Grain Client 초기화하기
클러스터의 Grain을 사용하기위해, Client 설정 및 초기화를 통해 클러스터와 연결한다.
```
// ClientConfiguration으로 Client 설정
ClientConfiguration clientConfig = ClientConfiguration.LocalhostSilo();
// Client 생성
IClusterClient client = new ClientBuilder().UseConfiguration(clientConfig).Build();
// 연결 시도
await client.Connect();
```
[ClientConfiguration 가이드](http://dotnet.github.io/orleans/Documentation/Deployment-and-Operations/Configuration-Guide/Client-Configuration.html)

### Grain 메서드 호출하기
```
IPlayerGrain player = client.GetGrain<IPlayerGrain>(playerId);
Task t = player.JoinGame(game)
await t;
```

### Notification 수신하기
클라이언트가 클러스트의 알림을 받기 위해 Orleans는 2가지 매커니즘을 제공한다.  
[Observer](http://dotnet.github.io/orleans/Documentation/Core-Features/Observers.html)  
[Stream](http://dotnet.github.io/orleans/Documentation/Orleans-Streams/index.html)

***
## 응용프로그램 만들기
### Orleans Application
Application = 클러스터 프로세스 + 클라이언트 프로세스  
클러스터가 작동중일 때, 하나 이상의 클라이언트 프로세스는 클러스터에 접속하여 grain에게 요청을 보낼 수 있다. 모든 silo는 클라이언트 게이트웨이를 가지고 있다. __성능향상을 위해 클라이언트들은 병렬로 모든 slio에 연결될 수 있다.__

### Silo의 설정과 시작
`ClusterConfiguration`을 사용하여 Silo를 설정한다. 로컬 테스트인 경우 `ClusterConfiguration.LocalPrimarySilo()`를 헬퍼 클래스로 사용할 수 있다.

메타 패키지를 프로젝트에 추가
```
PM> Install-Package Microsoft.Orleans.Server
```
로컬 silo를 시작하는 예시
```
var siloConfig = ClusterConfiguration.LocalhostPrimarySilo();
var silo = new SiloHost("Test Silo", siloConfig);
silo.InitializeOrleansSilo();
silo.StartOrleansSilo();

Console.WriteLine("Press Enter to close.");
// wait here
Console.ReadLine();

// shut the silo down after we are done.
silo.ShutdownOrleansSilo();
```

### 클라이언트의 설정과 연결
`ClientConfiguration`으로 설정, 로컬 테스트를 위해 `ClientConfiguration.LocalhostSilo()`를 사용한다.

매타 패키지 추가
```
PM> Install-Package Microsoft.Orleans.Client
```
클라이언트가 로컬 silo에 연결하는 예시
```
var config = ClientConfiguration.LocalhostSilo();
var builder = new ClientBuilder().UseConfiguration(config).
var client = builder.Build();
await client.Connect();
```

***
## 디버깅
생략
