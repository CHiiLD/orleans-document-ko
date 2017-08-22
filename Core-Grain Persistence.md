# Core
## Grain Persistence
### Grain Persistence의 목표
1. Storage provider 타입이 서로 다르거나(예: 한 놈은 Azura table, 다른 한 놈은 AOD.NET을 쓰는 경우) 또는 동일한 storage provider를 사용하나, 다른 설정(예: 한 놈은 계정A를 쓰고 다른 한 놈은 계정B를 쓰는 경우)을 사용하는 grain을 허용한다.
2. 설정 파일을 수정하여 storage provider의 설정을 변경할 수 있다.
3. 최소 수준의 프로덕션급 storage provider를 제공
4. Storage provider는 'persist backing store의 grain 상태를 어떻게 저장할지에 대해' 설정할 수 있다. Orleans는 ORM 저장 솔루션을 지원하지 않으나, custom storage provider를 통해 지원할 수 있다.

### Grain persistence API
`Grain` - persist state를 가지지 않거나 모든 persist state를 처리하지 않을 경우  
`Grain<T>` - persist state를 가지고 있고, 이것을 처리하고자 할 경우

### Grain State Stores
Grain<T>(T는 persisted state 타입)를 상속한 Grain 클래스는 특정 스토러지에서 state를 자동적으로 불러올 수 있다.  
`[StorageProvider]` 어트리뷰트는 grain의 state data를 read/write하는데 사용될 storage provider 인스턴스를의 이름을 명시한다.
```
[StorageProvider(ProviderName="store1")]
public class MyGrain<MyGrainState> ...
{
  ...
}
```
Orleans Provider Manager 프레임워크는 Silo 설정파일에 storage provider 및 storage option을 등록하는 매커니즘을 제공한다.
```
<OrleansConfiguration xmlns="urn:orleans">
    <Globals>
    <StorageProviders>
        <Provider Type="Orleans.Storage.MemoryStorage" Name="DevStore" />
        <Provider Type="Orleans.Storage.AzureTableStorage" Name="store1"
            DataConnectionString="DefaultEndpointsProtocol=https;AccountName=data1;AccountKey=SOMETHING1" />
        <Provider Type="Orleans.Storage.AzureBlobStorage" Name="store2"
            DataConnectionString="DefaultEndpointsProtocol=https;AccountName=data2;AccountKey=SOMETHING2"  />
    </StorageProviders>
```

### Storage provider 설정하기
#### AzureTableStorage
```
<Provider Type="Orleans.Storage.AzureTableStorage" Name="TableStore"
    DataConnectionString="UseDevelopmentStorage=true" />
```
`<Provider />`의 요소
+ `DataConnectionString="..."`(필수) - Azure storage 연결을 위한 문자열
+ `TableName="OrleansGrainState"`(선택) - table stage에 사용될 table name
+ `DeleteStateOnClear="false"`(선택)
    + if true - grain의 state가 지워질 때 레코드를 삭제
    + if false - null record로 쓰인다.
+ `UseJsonFormat="false"`(선택)
    + if true - json serializer 사용
    + if false - orleans bianry serializer 사용
+ `UseFullAssemblyNames="false"`(선택, if UseJsonFormat="true")
    + if true - 풀 어셈블리 이름으로 저장
    + if false - 간단한 어셈블리 이름으로 저장
+ `IndentJSON="false"`(선택, if UseJsonFormat="true") - json 들여쓰기 여부

#### AzureBlobStorage
[참조](http://dotnet.github.io/orleans/Documentation/Core-Features/Grain-Persistence.html)
#### DynamoDBStorageProvider
[참조](http://dotnet.github.io/orleans/Documentation/Core-Features/Grain-Persistence.html)
#### ADO.NET Storage Provider (SQL Storage Provider)
[참조](http://dotnet.github.io/orleans/Documentation/Core-Features/Grain-Persistence.html)
#### MemoryStorage
Silo 종료시 persist state는 사라짐, 테스트에서만 사용하라.  
XML configuration 설정
```
<?xml version="1.0" encoding="utf-8"?>
<OrleansConfiguration xmlns="urn:orleans">
  <Globals>
    <StorageProviders>
      <Provider Type="Orleans.Storage.MemoryStorage"
                Name="OrleansStorage"
                NumStorageGrains="10" />
    </StorageProviders>
  </Globals>
</OrleansConfiguration>
```
and 코드에서 설정
```
siloHost.Config.Globals.RegisterStorageProvider<MemoryStorage>("OrleansStorage");
```
|Name|Type|Description|
|----|----|-----------|
|Name|String|Storage provider를 흉내내는데 사용할 임시 이름|
|Type|String|`Orleans.Storage.MemoryStorage`로 설정|
|NumStorageGrains|Integer|State를 저장하기 위해 사용되는 grain의 개수, default 10|

#### ShardedStorageProvider
[참조](http://dotnet.github.io/orleans/Documentation/Core-Features/Grain-Persistence.html)

### Storage Provider의 참고사항
`Grain<T>` 클래스에 `[StorageProvider]` 어트리뷰트가 선언되지 않으면, grains에 설정된 storage provider 중에 `Default`를 찾아서 StorageProvider로 설정된다. 아무것도 찾을 수 없으면 missing  storage provider로 처리된다.  
Silo 설정파일에 storage provider가 하나만 있다면, 그것이 `Default`로 취급된다.  
Storage Provider의 모든 설정요소는 slio가 시작될 때 configuration에 의해 정적으로 정의된다. 이 시점에서 slio에 사용하는 storage provider list를 동적으로 업데이트/변경할 수 없다.

### State Storage APIs
Grain state/persistence API는 2개로 구분된다. Grain-to-Runtime, Runtime-to-Storage-Provider

### Grain State Read/Write
`GrainState`는 (activation이면 `OnActivateAsync()`가 호출되기 전에) `base.ReadStateAsync()`를 호출하여 storage provider를 통해 state data를 읽는다.  
`base.WriteStateAsync()`를 사용하여 명시적으로 grain state data를 storage provider를 통해 저장할 수 있다. 런타임에는 grain state를 자동으로 저장하지 아니한다. 따라서 `base.WriteStateAsync`를 실행할 메서드를 외부로 노출시켜야 한다.  
State를 읽을 때, grain state metadata로써 `Etag`(`string`)를 사용할 수 있다.  
런타임에서 write operation을 수행할 때는 grain state  data를 깊은 복사하여 복제본을 만든다.

State persistence 매커니즘을 구현하기 위해 Grain<T> 클래스를 만든다.
```
public class MyGrainState
{
    public int Field1 { get; set; }
    public string Field2 { get; set; }
}

[StorageProvider(ProviderName="store1")]
public class MyPersistenceGrain : Grain<MyGrainState>, IMyPersistenceGrain
{
    ...
    public Task DoWrite(int val)
    {
        State.Field1 = val;
        return base.WriteStateAsync();
    }

    public async Task<int> DoRead()
    {
        await base.ReadStateAsync();
        return State.Field1;
    }
}
```

### Grain State Persistence Operations의 실패 모델
Grain이 activation할 때 `ReadStateAsync()`메서드에서 storage provider에 의해 읽기가 실패할 경우, activation 작업이 취소된다. grain은 이러한 실패를 핸들링하거나 무시할 수 있다. silo 시작시 잘못된 설정으로 로드에 실패한 경우, `Orleans.BadproviderCOnfigException`이 던져진다.  
마찬가지로 `WriteStateAsync()` 작업도 실패될수 있으며, try-catch로 실패를 핸들링할 수 있다.

### IStorageProvider
Grain state data를 read/write할 Persistence Provider API 서비스를 제공한다.
```
public interface IStorageProvider
{
    Logger Log { get; }
    Task Init();
    Task Close();

    Task ReadStateAsync(string grainType, GrainReference grainReference, IGrainState grainState);
    Task WriteStateAsync(string grainType, GrainReference grainReference, IGrainState grainState);
}
```

### ADO.NET Persistence Rationale
[참조](http://dotnet.github.io/orleans/Documentation/Core-Features/Grain-Persistence.html)
