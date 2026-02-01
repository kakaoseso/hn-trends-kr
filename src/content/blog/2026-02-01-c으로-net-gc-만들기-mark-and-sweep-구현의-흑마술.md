---
title: "C#으로 .NET GC 만들기: Mark and Sweep 구현의 흑마술"
description: "C#으로 .NET GC 만들기: Mark and Sweep 구현의 흑마술"
pubDate: "2026-02-01T01:18:31Z"
---

솔직히 말해서, 대부분의 애플리케이션 엔지니어들에게 Garbage Collector(GC)는 '마법의 블랙박스'입니다. 우리는 그저 객체를 할당할 뿐이고, 뒷정리는 누군가(GC)가 알아서 해주길 기대하죠. 하지만 시니어 레벨로 올라가거나 고성능 시스템을 다루다 보면 이 블랙박스를 열어보고 싶은 욕망, 아니 '필요'가 생깁니다.

Kevin Gosse가 연재 중인 **Writing a .NET Garbage Collector in C#** 시리즈는 바로 그 욕망을 정면으로 충족시켜 주는 프로젝트입니다. C#으로 작성된 런타임 위에서 돌아가는 GC를 다시 C#으로 작성한다니, 다소 재귀적이고 위험해 보이지만 엔지니어링 관점에서는 이보다 흥미진진한 주제도 드뭅니다. 이번 Part 6에서는 GC의 핵심인 **Mark and Sweep** 알고리즘을 구현하는 과정을 다룹니다.

원본 글: [Writing a .NET Garbage Collector in C# – Part 6: Mark and Sweep](https://minidump.net/writing-a-net-gc-in-c-part-6/)

---

### GC의 심장부로 진입하다

지난 Part 4에서 힙(Heap) 레이아웃을 잡았다면, 이번에는 실제로 살아있는 객체를 찾아내는(Mark) 과정과 죽은 객체를 정리하는(Sweep) 과정을 구현합니다. 이론적으로는 컴공과 2학년 때 배우는 내용이지만, 실제 .NET 런타임 레벨에서 이걸 구현하는 건 전혀 다른 차원의 이야기입니다.

![Pro .NET Memory Management](https://minidump.net/images/progc.png)

### 1. Marking: 비트 단위의 흑마술

Mark 단계의 목표는 간단합니다. Root에서 시작해서 도달 가능한 모든 객체에 "나 살아있음" 표시를 남기는 것입니다. 여기서 엔지니어로서 가장 흥미로웠던 부분은 **"어디에 마킹 정보를 저장할 것인가?"** 에 대한 결정이었습니다.

보통 별도의 비트맵을 쓰거나 객체 헤더의 남는 공간을 활용한다고 생각하기 쉽습니다. 하지만 Kevin은 실제 .NET GC가 사용하는 방식과 동일한, 아주 고전적이고 효율적인 트릭을 사용합니다.

바로 **MethodTable 포인터의 하위 비트를 훔쳐 쓰는 것** 입니다.

모든 객체는 `MethodTable` 포인터를 가지고 있습니다. 그리고 아키텍처 특성상 포인터 주소는 정렬(Alignment)되므로, 32비트에서는 하위 2비트, 64비트에서는 하위 3비트가 항상 `0`입니다. 이 점을 이용해 최하위 비트(LSB)를 1로 세팅하면 "Marked" 상태로 간주하는 것이죠.

```csharp
[StructLayout(LayoutKind.Sequential)]
public unsafe struct GCObject
{
    public MethodTable* RawMethodTable;
    
    // 하위 비트를 제외하고 실제 주소만 반환
    public readonly MethodTable* MethodTable => (MethodTable*)((nint)RawMethodTable & ~1);
    
    // 하위 비트가 1이면 마킹된 것으로 판단
    public bool IsMarked() => ((nint)RawMethodTable & 1) != 0;
    
    // 하위 비트를 1로 세팅 (Mark)
    public void Mark() => RawMethodTable = (MethodTable*)((nint)MethodTable | 1);
    
    // 하위 비트를 0으로 복구 (Unmark)
    public void Unmark() => RawMethodTable = (MethodTable*)((nint)MethodTable & ~1);
}
```

이 코드를 보면 "Dirty hack"이라고 느낄 수도 있지만, 시스템 프로그래밍에서는 메모리 오버헤드 없이 상태를 저장하는 매우 우아한 기법입니다. 수백만 개의 객체를 다룰 때 별도의 불리언 필드 하나가 얼마나 큰 낭비인지 아는 분들이라면 무릎을 탁 칠 겁니다.

### 2. Scanning Roots: 런타임과의 대화

GC가 혼자서 모든 걸 할 순 없습니다. 스택 변수나 레지스터 같은 RootSet은 런타임(CLR)이 알려줘야 합니다. `IGCToCLR.GcScanRoots` API를 통해 콜백을 등록하는데, 여기서 재미있는 점은 `ScanContext` 구조체를 해킹(?)해서 사용하는 방식입니다.

`ScanContext`는 원래 Server GC를 위해 설계된 복잡한 구조체지만, 여기서는 `_unused1` 필드에 우리 GC의 힙 인스턴스 포인터를 숨겨서 전달합니다. Native 코드와 Managed 코드 사이를 오갈 때 컨텍스트를 유지하기 위한 전형적인 패턴이죠.

```csharp
private void MarkPhase()
{
    ScanContext scanContext = new();
    scanContext.promotion = true;
    // 포인터 크기의 _unused1 필드에 GC 인스턴스 핸들을 숨김
    scanContext._unused1 = GCHandle.ToIntPtr(_handle);
    
    var scanRootsCallback = (delegate* unmanaged<GCObject**, ScanContext*, uint, void>)&ScanRootsCallback;
    _gcToClr.GcScanRoots((IntPtr)scanRootsCallback, 2, 2, &scanContext);
}
```

### 3. Traversal: DFS와 Cache Locality

객체 그래프를 순회할 때 BFS(너비 우선)와 DFS(깊이 우선) 중 무엇을 선택해야 할까요? 저자는 **DFS** 를 선택했습니다. 이유는 명확합니다.

1.  **Cache Locality:** 서로 참조하는 객체들은 메모리상에 인접해 있거나, 적어도 시간적으로 가깝게 접근될 확률이 높습니다. DFS는 이를 활용하기 좋습니다.
2.  **Stack Usage:** 재귀 호출 대신 명시적인 `Stack<IntPtr>`을 사용하여 `StackOverflowException`을 방지합니다. (GC가 스택 오버플로우로 죽으면 정말 꼴사나우니까요.)

### 4. Sweeping: 죽은 자들의 처리

Marking이 끝나면 힙을 쭉 훑으면서(Sweep) 마킹되지 않은 객체를 정리합니다. 여기서 중요한 디테일은 **"빈 공간을 어떻게 처리하느냐"** 입니다.

단순히 메모리를 0으로 미는 것으로 끝나지 않습니다. .NET 힙은 **Walkable** 해야 합니다. 즉, 힙의 시작부터 끝까지 객체 크기를 더해가며 이동할 수 있어야 합니다. 중간에 구멍(void)이 생기면 순회가 끊깁니다.

그래서 해제된 메모리 영역에는 `Free Object`(더미 객체)를 생성해서 채워 넣습니다. 이렇게 하면 다음 GC 사이클이나 힙 검사 도구가 이 영역을 "빈 객체"로 인식하고 건너뛸 수 있습니다.

### The Elephant in the Room: Interior Pointers

이 글을 읽으며 가장 우려되었던(그리고 저자도 인정한) 부분은 **Interior Pointer** 의 부재입니다. C#의 `ref` 키워드나 배열 내부 요소에 대한 포인터 같은 것들 말이죠.

```csharp
private static ref int GetInteriorPointer()
{
    var array = new int[10];
    return ref array[5]; // 배열의 시작이 아닌 중간을 가리킴
}
```

GC 입장에서 이건 악몽입니다. 포인터가 객체의 헤더를 가리키지 않고 뜬금없이 몸통 중간을 가리키고 있으니까요. 이를 통해 원본 객체(배열)의 시작 주소를 역추적해서 살려두는 로직이 필요한데, 이번 파트에서는 복잡도를 이유로 생략되었습니다. (실제로 이 코드를 돌리면 바로 크래시가 날 겁니다.)

### 마치며: 장난감이 아닌 진짜 학습 도구

이 시리즈는 단순히 "나만의 GC 만들기" 튜토리얼을 넘어서고 있습니다. .NET 런타임이 메모리를 어떻게 관리하는지, 객체 헤더가 어떻게 생겼는지, 그리고 Native와 Managed 세계가 어떻게 교신하는지를 가장 적나라하게 보여주는 자료입니다.

물론 지금 단계의 코드는 Interior Pointer 미지원, Finalization Queue 무시 등으로 인해 실사용은 불가능합니다. 하지만 **"왜 .NET GC가 이렇게 복잡하게 설계되었는가?"** 에 대한 질문에 코드로 답을 해주고 있습니다.

다음 파트에서 Interior Pointer와 GC Handle을 어떻게 처리할지 기대가 됩니다. 특히 Interior Pointer의 역추적 로직을 C#으로 어떻게 구현할지가 관전 포인트가 되겠네요.

**엔지니어로서의 한 줄 평:**
> "MethodTable의 하위 비트를 1로 바꾸는 순간, 여러분은 더 이상 C# 개발자가 아니라 시스템 아키텍트가 된 것입니다."
