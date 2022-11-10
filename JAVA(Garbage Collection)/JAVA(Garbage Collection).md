# CS11/10(Garbage Collection)

# GC란?

---

가비지 컬렉션은 영어로 Garbage Collection 줄임말이다. GC는 자바의 메모리 관리 방법 중의 하나로 JVM의 Heap 영역에서 동적으로 할당했던 메모리 영역 중 필요 없게 된 메모리 영역을 주기적으로 삭제하는 프로세스를 말합니다. C나 C++과는 다르게 수동으로 메모리 할당과 해제를 일일이 해줄 필요가 없어서 편하다.

> System.gc()
>
> Runs the garbage collector.
> Calling the gc method suggests that the Java Virtual Machine expend effort toward recycling unused objects in order to make the memory they currently occupy available for quick reuse. When control returns from the method call, the Java Virtual Machine has made a best effort to reclaim space from all discarded objects.
>
> The call System.gc() is effectively equivalent to the call:
>
> Runtime.getRuntime().gc()

단점

- 개발자가 메모리가 언제 해제되는지 정확하게 알 수 없다.
- GC가 동작하는 동안에 다른 동작을 멈추기 때문에 오버헤드 발생

## **JNI( Java Native Interface)**

---

JNI는 C나 C++에서 작성된 모듈을 자바에서 호출할 수 있게 해주는 기능이다. Runtime 클래스에 있는 gc()메소드를 조회하면

![Untitled](<JAVA(Garbage%20Collection)/Untitled.png>)

사진 처럼 native가 붙어있다. 더 찾아보기 위해서 비교적 최근에 MS에서 자바 GC툴킷을 오픈소스로 공개 했다. https://github.com/microsoft/gctoolkit 쉽지 않다..

## **Unreachable & reachable**

---

![빨간색 객체가 가비지 컬렉션의 대상이 되는 객체](<JAVA(Garbage%20Collection)/Untitled%201.png>)

빨간색 객체가 가비지 컬렉션의 대상이 되는 객체

객체들은 실질적으로 Heap영역에서 생성되고 다른 Root Area에서는 Heap Area에 생성된 객체의 주소만 참조하는 형식으로 구성됨. 이 때 아무데서도 참조 되고 있지 않은 객체는 unreachable이라고 하며 참조 되고 있는 객체들은 reachable한 상태이다. unreachable의 상태에 있는 객체들을 gc가 주기적으로 제거해준다.

## **Mark And Sweep 알고리즘**

---

![Untitled](<JAVA(Garbage%20Collection)/Untitled%202.png>)

Mark and Sweep 알고리즘은 가비지 컬렉션이 동작하는 원리로 루트에서부터 해당 객체에 접근 가능한지에 대한 여부를 메모리 해제의 기준으로 삼는다. 그리고 3가지 과정으로 나뉜다.

1. Mark 과정: 먼저 Root로부터 그래프 순회를 통해 연결된 객체들을 찾아내어 각각 어떤 객체를 참조하고 있는지 찾아서 마킹한다.
2. Sweep 과정: 참조하고 있지 않은 객체, 즉 Unreachable 객체들을 Heap에서 제거한다.
3. Compact과정: Sweep 후에 분산된 객체들을 Heap의 시작 주소로 모아 메모리가 할당된 부분과 그렇지 않은 부분으로 압축한다.

## **GC의 대상이 되는 Heap 영역**

---

![Untitled](<JAVA(Garbage%20Collection)/Untitled%203.png>)

Heap Area는 효율적인 GC를 위해 위와 같이 Eden, Survivor, Old Generation으로 나뉜다.

Eden과 survival을 합쳐서 young generation 영역이라고 부른다. 객체가 최초로 생성되면 Eden영역으로 가고 이 영역에 데이터가 어느정도 쌓이게 되면 참조정도에 따라 survivor의 빈 공간으로 이동되거나 회수된다.

young generation영역이 차게 되면 또 참조정도에 따라 old영역으로 이동 되거나 회수된다. 이렇게 young generation과 tenured generration에서의 gc를 minor gc라고 한다.

Old 영역에서도 메모리가 허용치를 넘게 되면, Old 영역에 있는 모든 객체들을 전수조사하여 참조되지 않은 객체들을 한꺼번에 삭제하는 GC가 실행된다. 시간이 오래걸리고 GC를 제외한 모든 쓰레드는 작업을 멈춘다. 이를 ‘stop-the-world’ 이 때 실행되는 gc 를 major gc라고 한다.

permanent generation( JDK 1.7 이상에서는 Metaspace) 객체나 intern문자열 정보 등이 저장되는 곳, 일반적으로 meta 정보가 저장되는 공간이다.

## **동작 과정**

---

1. 첫번째 과정

![Untitled](<JAVA(Garbage%20Collection)/Untitled%204.png>)

- 객체가 처음 생성되고 Heap 영역의 Eden에 age-bit 0으로 할당된. 이 age-bit는 minor gc에서 살아남을 때마다 1씩 증가하게 된다.

1. 두번째 과정

![Untitled](<JAVA(Garbage%20Collection)/Untitled%205.png>)

- 시간이 지나 heap area의 eden 영역에 객체가 다 쌓이게 되면 minor gc가 한번 일어나게 되고 참조 정도에 따라 servivor()영역으로 이동시키거나 회수한다.

1. 세번째 과정

![Untitled](<JAVA(Garbage%20Collection)/Untitled%206.png>)

- 계속해서 Eden 영역에는 신규 객체들이 생성됨. 이렇게 또 Eden영역에 객체가 다 쌓이게 되면 young generation영역에 있는 객체들을 비어있는 survival1 영역에 이동하고 살아남은 모든 객체들은 age가 1씩 증가.

1. 네번째 과정

![Untitled](<JAVA(Garbage%20Collection)/Untitled%207.png>)

- 또다시 Eden 영역에 신규 객체들로 가득 차게 되면 다시한번 minor gc가 일어나고 young generation 영역에 있는 객체들을 비어있는 survival0으로 이동 시킨 뒤 age 1증가. 이과정을 계속 반복

1. 다섯 번째 과정

![Untitled](<JAVA(Garbage%20Collection)/Untitled%208.png>)

- 위 과정을 반복하다 보면 age bit가 특정 숫자 이상으로 되는 경우가 발생. 이때 JVM에서 설정해놓은 age bit에 도달하게 되면 오랫동안 쓰일 객체라고 판단하고 Old generation영역으로 이동시킵니다. 이 과정을 프로모션(promotion)이라고 한다.

1. 마지막 과정

![Untitled](<JAVA(Garbage%20Collection)/Untitled%209.png>)

- 시간이 지나 Old 영역에 할당된 메모리가 허용치를 넘게 되면, Old 영역에 있는 모든 객체들을 검사하여 참조되지 않는 객체들을 한꺼번에 삭제하는 GC가 실행된다. 이렇게 old generation영역의 메모리를 회수 하는 GC를 major gc라고 한다. 이때 위에서 잠깐 설명한 ‘stop - the -world’가 발생하고 이 작업이 잦으면 성능에 문제가 될 수 있다.

## **GC POLICY**

---

GC policy란 OLD 영역이 꽉 찼을 때 major gc가 작동하는데 어떤 방식으로 gc를 수행할 것인지가 결정되어 성능에 직접적이고 커다란 영향을 끼치게 되는 원리

1. **Serial GC( -XX:+UseSerialGC) 적은 메모리와 cpu 코어 개수가 적을 때 적합한 방식**

- mark & sweep 기준으로 GC를 수행하는 알고리즘 방식

1. **Parallel GC ( -XX:+UseParallelGC) Serial GC와 알고리즘은 같지만 여러개의 Thread가 나뉘어져 처리하는 방식**

- 메모리가 충분하고 코어의 개수가 많을 때 유리, Throughput GC라고 부른다.

1. **Parallel Old GC ( -XX:+UseParallelOldGC)**

- 위의 그냥 parallel GC와 비교하면 Old 영역의 GC 알고리즘만 차이가 있다. 기존의 mark & sweep & compaction 단계에서 parallel old gc는 mark & summary & compaction 단계를 사용한다. summary 단계는 앞서 GC를 수행한 영역에 대해서 별도의 살아있는 객체를 식별한다는 점에서 차이가 있음.

1. **CMS GC (-XX:+UseConcMarkSweepGC)**

- initial mark 단계에서는 클래스 로더에서 가장 가까운 객체 중 살아있는 객체만 찾아 냅니다. 따라서 초기에 STW가 발생되는 시간이 매우 짧게 형성되어 이점을 가져올 수 있다.
- Concurrent Mark 단계에서는 initial mark에서 확인된 객체에서 참조하고 있는 객체들을 따라가면서 확인함.
- remark 단계에서는 concurrent mark 단계에서 새로 추가되거나 참조가 끊어진 객체를 확인
- concurrent sweep 단계에선 gc를 수행하는 작업 실행
- 단점은
  - 다른 GC방식보다 메모리와 CPU를 더 많이 사용한다.
  - compaction 단계가 기본적으로 제공되지 않는다. 따라서 조각난 메모리가 많아질 수 있다.

1. **G1GC**

- G1GC는 메모리는 바둑판처럼 각각의 영역으로 구분하고 각 영역에 객체를 할당하여 GC를 실행합니다. 그러다가, 해당 영역이 꽉 차면 다른 영역에서 객체를 할당하고 GC를 실행한다. 즉 기존의 young, old 영역에서 진행하는 메모리 처리 방식이 한 영역에서 모두 담당한다고 보면 된다. G1GC는 장기적으로 문제가 야기될 가능성이 있는 CMS GC의 대체 방안으로 고안되었으며, 성능상 뛰어남
