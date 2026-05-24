---
title: "C# 15 Union Types 도입: 드디어 Rust와 F#을 따라잡는가? (솔직한 리뷰)"
description: "C# 15 Union Types 도입: 드디어 Rust와 F#을 따라잡는가? (솔직한 리뷰)"
pubDate: "2026-05-24T02:40:19Z"
---

10년 넘게 C#을 주력으로 사용해 온 엔지니어로서, 솔직히 말해 이번 소식은 꽤나 감격스럽다. 우리는 그동안 `Result<T>` 패턴을 흉내 내기 위해 수많은 래퍼(wrapper) 클래스를 만들거나, `OneOf` 같은 외부 라이브러리에 의존해 왔다. 예외(Exception)를 제어 흐름에 사용하는 안티 패턴과 싸우는 것도 지긋지긋했다.

마침내 .NET 11 (C# 15) 프리뷰에 Union Types(공용체 타입)가 도입되었다. Rust나 F# 같은 언어에서는 이미 숨 쉬듯 자연스러운 기능이지만, C# 생태계에서는 꽤 오랜 시간 기다려온 기능이다. Andrew Lock의 블로그 글과 최근 Hacker News의 뜨거운 반응을 바탕으로, 이 기능이 내부적으로 어떻게 동작하는지, 그리고 프로덕션에 도입할 만한 가치가 있는지 깊게 파헤쳐 보겠다.

## C# 15 Union Types: 어떻게 생겼나?

이전까지 C#에서 여러 타입 중 하나를 반환해야 할 때 우리가 선택할 수 있는 옵션은 처참했다. 공통 Base Class를 억지로 만들거나, 타입 안정성을 포기하고 `object`를 쓰거나, Enum으로 태그를 달아 관리하는 식이었다.

이제 C# 15에서는 `union` 키워드를 통해 이를 우아하게 해결할 수 있다.

```csharp
public record Windows(string Version);
public record Linux(string Distro, string Version);
public record MacOS(string Name, int Version);

// 👇 union 키워드 사용
public union SupportedOS(Windows, Linux, MacOS);
```

객체를 생성하는 것도 암시적 변환(Implicit conversion)을 지원하여 매우 직관적이다.

```csharp
SupportedOS os = new MacOS("Tahoe", 25);
```

하지만 내가 가장 열광하는 부분은 바로 **Pattern Matching** 과의 결합이다. Union 타입의 진가는 `switch` expression에서 발휘된다.

```csharp
string GetDescription(SupportedOS os) => os switch
{
    Windows windows => $"Windows {windows.Version}",
    Linux linux => $"{linux.Distro} {linux.Version}",
    MacOS macOS => $"MacOS {macOS.Name} ({macOS.Version})",
};
```

여기서 주목할 점은 `_ =>` (discard) 케이스가 없다는 것이다. 컴파일러가 모든 케이스가 처리되었는지 **Exhaustive checking** (철저한 검사)을 수행한다. 만약 하나라도 빼먹으면 `CS8509` 경고를 뱉어낸다. Rust에서 수없이 경험했던 그 견고한 타입 안정성을 드디어 C#에서도 맛볼 수 있게 된 것이다.

## 내부 구현과 뼈아픈 'Boxing' 문제

엔지니어라면 겉보기 좋은 문법(Syntactic sugar) 뒤에 숨겨진 내부 구현을 의심해 봐야 한다. 컴파일러가 도대체 이 `union`을 어떻게 C# 코드로 번역할까?

생성된 코드를 보면 `[Union]` 어트리뷰트가 붙은 `struct`로 구현되며, `IUnion` 인터페이스를 상속받는다.

```csharp
public interface IUnion
{
    object? Value { get; }
}
```

여기서 성능에 민감한 시니어 엔지니어라면 바로 눈치챘을 것이다. "잠깐, `object?`라고? 그럼 Value Type을 넣으면 무조건 Boxing이 발생한다는 거잖아?"

맞다. 이것이 현재 프리뷰 버전의 가장 큰 한계다. 예를 들어 `public union IntOrBool(int, bool);`를 선언하면, 내부적으로 `int`나 `bool`이 `object`로 박싱(Boxing)되어 힙(Heap) 메모리를 할당하게 된다. Hot path나 메모리 제약이 심한 시스템에서는 치명적일 수 있다.

다행히 우회 방법이 존재한다. `HasValue`와 `TryGetValue` 패턴을 직접 구현하여 Non-boxing Union을 만들 수 있다.

```csharp
[Union]
public struct IntOrBool : IUnion
{
    private readonly bool _isBool;
    private readonly int _value;

    public IntOrBool(int value) { _isBool = false; _value = value; }
    public IntOrBool(bool value) { _isBool = true; _value = value ? 1 : 0; }

    public bool HasValue => true;

    public bool TryGetValue(out int value)
    {
        value = _value;
        return !_isBool;
    }

    public bool TryGetValue(out bool value)
    {
        value = _isBool && _value is 1;
        return _isBool;
    }

    public object Value => _isBool ? _value is 1 : _value;
}
```

컴파일러는 `TryGetValue`가 존재하면 `switch` expression에서 `Value` 프로퍼티 대신 이를 우선적으로 호출하여 박싱을 방지한다. 

솔직히 말해, 이 보일러플레이트(Boilerplate) 코드를 매번 짜야 한다면 `union` 키워드의 의미가 퇴색된다. Hacker News의 한 유저는 "왜 처음부터 제대로 만들지 않았나? 지난 10년간 성능 개선에 목매던 팀 치고는 기이한 결정이다"라고 비판했다. 개인적으로도 동의하지만, Microsoft 특유의 'MVP(최소 기능 제품) 선출시 후 최적화' 전략을 고려하면 정식 릴리스나 다음 버전에서는 컴파일러 레벨의 최적화가 들어갈 것으로 기대한다.

## Hacker News의 반응: 생태계와 언어의 진화

이번 발표에 대한 Hacker News의 반응은 매우 흥미롭다. 언어 자체의 발전은 환영하지만, C#이 처한 현실적인 위치에 대한 냉정한 평가들이 오갔다.

- **Rust 경험자들의 환호:** Rust나 TypeScript를 쓰다가 C#으로 돌아오면 가장 역겨운(?) 것 중 하나가 바로 Enum과 Null 처리다. C#의 Enum은 근본적으로 그저 이름 붙은 정수일 뿐이라 범위 밖의 값을 넣어도 합법이다. Union 타입과 철저한 패턴 매칭은 이러한 방어적 프로그래밍(Defensive programming)의 피로도를 급격히 낮춰줄 것이다.
- **F#의 그림자:** "C#은 그저 C 문법을 가진 F#이 되어가고 있다"는 뼈있는 농담이 많은 공감을 얻었다. F#은 이 기능을 수십 년 전부터 가지고 있었다. 하지만 현실적으로 대다수의 팀은 언어를 바꾸지 않는다. 주류 언어인 C#이 이런 함수형 패러다임을 흡수하는 것은 실용적인 관점에서 엄청난 이득이다.
- **Java와의 생태계 비교:** 언어 자체는 C#이 Java를 압도한 지 오래되었다는 의견이 지배적이다. 하지만 "C# 언어는 사랑하지만, 오픈소스 생태계는 슬럼가(Ghetto) 같다"는 뼈아픈 지적도 있었다. Apache Spark, Kafka 클라이언트, AWS SDK 등 엔터프라이즈 인프라 생태계에서는 여전히 Java가 압도적이다. Microsoft가 제공하는 라이브러리 안에서는 천국이지만, 그 밖으로 나가는 순간 가시밭길이 펼쳐진다는 점은 여전히 C#의 아킬레스건이다.

## 결론: 프로덕션에 도입할 준비가 되었는가?

나는 C#이 성능과 표현력 사이에서 줄타기를 매우 잘해왔다고 생각한다. 이번 Union Types의 도입은 C# API 설계 방식을 근본적으로 바꿀 게임 체인저다. 더 이상 예외를 던지거나 `Tuple`과 `out` 파라미터를 섞어 쓰는 더러운 코드를 보지 않아도 된다.

다만, **Boxing 이슈** 는 명확한 걸림돌이다. 고성능이 요구되는 Core 로직에서는 당분간 기본 `union` 키워드 사용을 자제하고, 커스텀 Non-boxing 구현체를 사용하거나 기존 방식(예: Memory-aligned structs)을 유지하는 것이 좋아 보인다.

아직 .NET 11 프리뷰 단계이지만, 이 기능은 C#의 미래를 보여주는 강력한 지표다. 라이브러리 작성자라면 지금부터라도 공개 API에 Union Type을 어떻게 녹여낼지 고민을 시작해야 할 때다.

---

**References:**
- Original Article: [Exploring the .NET 11 preview 2: .NET gets union types!](https://andrewlock.net/exploring-the-dotnet-11-preview-2-dotnet-gets-union-types/)
- Hacker News Thread: [Discussion on C# Union Types](https://news.ycombinator.com/item?id=48234954)
