# Core
## Dependency Injection
### 의존성 주입이란?
DI(Dpendency Injection)은 종속성을 해결하기 위한 소프트웨어 디자인 패턴이다.

### Orleans의 DI
현재 서버 사이드에서만 지원한다. 일반적으로 grain, slio를 생성하는데 사용된다. 또한 silo가 시작할 때 `IServiceCollection`로 사전 등록한 클래스를 DI할 수 있다.

### Configuring DI
우리는 `ConfigureService` 메서드를 포함하는 애플리케이션과 `Startup` 클래스를 가지고 있어야 한다. 이 메서드는 `IServiceProvider`를 리턴한다.

### Configuring form Code
코드 기반의 설정으로 Orleans를 설정할 수 있다. 이를 가능케할 `ClusterConfiguration` 클래스의 `UseStartup***` 메서드가 있다.
```
var configuration = new ClusterConfiguration();
configuration.UseStartupType<MyApplication.Configuration.MyStartup>();
```

### Configurating via XML
XML로 Startup 클래스를 등록할 수 있다.
```
<?xml version="1.0" encoding="utf-8" ?>
<tns:OrleansConfiguration xmlns:tns="urn:orleans">
  <tns:Defaults>
    <tns:Startup Type="MyApplication.Configuration.Startup,MyApplication" />
  </tns:Defaults>
</tns:OrleansConfiguration>
```

### 예제
Startup 클래스 예제
```
namespace MyApplication.Configuration
{
    public class MyStartup
    {
        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<IInjectedService, InjectedService>();

            return services.BuildServiceProvider();
        }
    }
}
```

다음 예제는 `IInjectedService`로 생성자 주입과 서비스를 구현하여 Grain을 활용하는 방법을 보여준다.
```
public interface ISimpleDIGrain : IGrainWithIntegerKey
{
    Task<long> GetTicksFromService();
}

public class SimpleDIGrain : Grain, ISimpleDIGrain
{
    private readonly IInjectedService injectedService;

    public SimpleDIGrain(IInjectedService injectedService)
    {
        this.injectedService = injectedService;
    }

    public Task<long> GetTicksFromService()
    {
        return injectedService.GetTicks();
    }
}

public interface IInjectedService
{
    Task<long> GetTicks();
}

public class InjectedService : IInjectedService
{
    public Task<long> GetTicks()
    {
        return Task.FromResult(DateTime.UtcNow.Ticks);
    }
}
```

### TDD로 DI 테스트
DI의 정확성을 검증한다. Moq를 사용한 예제
```
public class MockServices
{
    public IServiceProvider ConfigureServices(IServiceCollection services)
    {
        var mockInjectedService = new Mock<IInjectedService>();

        mockInjectedService.Setup(t => t.GetTicks()).Returns(knownDateTime);
        services.AddSingleton<IInjectedService>(mockInjectedService.Object);
        return services.BuildServiceProvider();
    }
}
```
테스트 코드
```
[TestClass]
public class IInjectedServiceTests: TestingSiloHost
{
    private static TestingSiloHost host;

    [TestInitialize]
    public void Setup()
    {
        if (host == null)
        {
            host = new TestingSiloHost(
                new TestingSiloOptions
                {
                    StartSecondary = false,
                    AdjustConfig = clusterConfig =>
                    {
                        clusterConfig.UseStartupType<MockServices>();
                    }
                });
        }
    }
}
```
