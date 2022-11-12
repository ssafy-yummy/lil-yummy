# Chapter10입출력시스템

# 01. 입출력 시스템

## 1). 입출력장치와 채널

- 저속 주변장치 : 메모리와 주변장치 사이에 오고 가는 데이터의 양이 적어 데이터 전송률이 낮은 장치 ex) 프린트, 키보드, 마우스
- 고속 주변장치 : 데이터 전송률이 높은 장치 ex) 그래픽카드,하드디스크

여러 주변장치는 메인보드 내의 버스로 연결 

채널 : 데이터가 지나다니는 하나의 통로

여러 버스를 묶어 사용 채널을 분리해 데이터 속도 향상

![Untitled](Chapter10(입출력시스템)/Untitled.png)

## 2). 입출력 버스의 구조

### 초기의 입출력 버스

![Untitled](Chapter10(입출력시스템)/Untitled%201.png)

### 입출력 제어기를 사용한  입출력 버스

![Untitled](Chapter10(입출력시스템)/Untitled%202.png)

### 입출력 버스의 분리

![Untitled](Chapter10(입출력시스템)/Untitled%203.png)

폴링 방식 사용 → 입출력 제어기 사용 → 입출력 버스의 분리

## 3). 직접 메모리 접근(Direct Memory Access)

입출력 제어기는 다양한 주변장치의 입출력을 대행하고 데이터를 메모리로 옮김

메모리는 cpu의 명령에 따라 작동하므로 DMA 권한을 주어 cpu의 효율성을 증가시킴

초기에 CPU가 사용하는 작업 공간인 메인 메모리가 DMA 제어기와 공유 → 오늘날 입출력 시스템에서는 메인 메모리에 공간을 CPU가 작업하는 공간과 DMA 제어기가 데이터를 옮기는 공간 분리해서 CPU와 DMA 제어기의 작업 공간이 겹치는 것을 막음

![Untitled](Chapter10(입출력시스템)/Untitled%204.png)

## 4). 인터럽트

- 주변장치의 입출력 요구나 하드웨어의 이상 현상을 CPU에 알려주는 역할을 하는 신호
- IRQ라는 고유 인터럽트 번호를 부여해 어떤 장치에서 발생했는지 파악
    
    ![Untitled](Chapter10(입출력시스템)/Untitled%205.png)
    

인터럽트 벡터라는 인터럽트를 처리할 수 있는 서비스 루틴들의 주소를 가진 공간을 통해 처리함

![Untitled](Chapter10(입출력시스템)/Untitled%206.png)

![Untitled](Chapter10(입출력시스템)/Untitled%207.png)

## 5). 버퍼링

속도가 다른 두 장치의 속도 차이를 완화하는 역할

![Untitled](Chapter10(입출력시스템)/Untitled%208.png)

# 02. 디스크 장치

## 1). 디스크 장치의 종류

하드 디스크

![Untitled](Chapter10(입출력시스템)/Untitled%209.png)

플래터 : 표면에 자성체가 발려 있어 자기를 이용해 0과 1 데이터 저장

섹터 : 하드디스크의 가장 작은 저장 단위

블록 : 데이터를 전송하는 논리적인 개념에서 가장 작은 단위, 여러 개의 섹터로 구성

트랙 : 동심원 상의 섹터 집합

실린더 : 트랙들의 집합

헤더 : 데이터를 읽거나 쓸때 사용

CD

표면에 미세한 홈이 파여 있어 레이저의 반사 여부로 인식 

![Untitled](Chapter10(입출력시스템)/Untitled%2010.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2011.png)

## 2). 디스크 장치의 데이터 전송 시간

![Untitled](Chapter10(입출력시스템)/Untitled%2012.png)

데이터 전송 시간 = 탐색 시간 + 회전 지연 시간+ 전송 시간

물리적으로 이동하는 시간이 상대적으로 매우 느리므로 탐색 시간이 가장 큰 비중을 차지

## 3). 디스크 장치 관리

파티션 : 디스크를 논리적으로 분할하는 작업

포매팅 : 디스크 표면을 초기화하는 작업 

조각모음 : 디스크에 파일을 저장했다가 지우기를 반복함으로써 중간중간에 생긴 빈 공간을 하나로 모으는 작업

## 4). 네트워크 저장 장치

DAS(Direct Attached Storage) : 서버와 같은 컴퓨터에 직접 연결된 저장장치

NAS(Network Attached Storage) : 기존의 저장 장치를 LAN이나 WAN에 붙여서 사용하는 방식

SAN(Storage Area Network) : 데이터 서버, 백업 서버, RAID 등의 장치를 네트워크로 묶고 데이터 접근을 위한 서버를 두는 형태

![Untitled](Chapter10(입출력시스템)/Untitled%2013.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2014.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2015.png)

# 03. 디스크 스케줄링

데이터 전송 시간중 탐색시간이 가장 느리기 때문에 탐색시간을 최소화 하기위해 다양한 디스크 스케줄링 기법이 나옴

## 1). FCFS(First Come, First Service)

![Untitled](Chapter10(입출력시스템)/Untitled%2016.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2017.png)

## 2). SSTF(Shortest Seek Time First)

효울성은 좋지만 아사 현상 가능성이 높다.

![Untitled](Chapter10(입출력시스템)/Untitled%2018.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2019.png)

## 3). 블록 SSTF

공평성을 보장하지만 효울성 떨어짐

![Untitled](Chapter10(입출력시스템)/Untitled%2020.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2021.png)

## 4). SCAN

헤드가 한 방향으로만 움직이며 트랙을 탐색

동일한 트랙에 요청이 연속적으로 발생하면 바깥 트랙에 아사 현상 발생

![Untitled](Chapter10(입출력시스템)/Untitled%2022.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2023.png)

## 5). C-SCAN(Circular SCAN)

바깥 트랙에 방문하면 반대 방향으로 이동 후 탐색

동일한 트랙에 요청이 연속적으로 발생하면 바깥 트랙에 아사 현상 발생

![Untitled](Chapter10(입출력시스템)/Untitled%2024.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2025.png)

## 6). LOOK

![Untitled](Chapter10(입출력시스템)/Untitled%2026.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2027.png)

## 7). C-LOOK

![Untitled](Chapter10(입출력시스템)/Untitled%2028.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2029.png)

## N-STEP SCAN

디스크 헤더가 n개의 요청만 받아들이고 이들만 수행한다.

![Untitled](Chapter10(입출력시스템)/Untitled%2030.png)

## 8). SLTF(Shortest Latency Time First)

작업 요청이 들어온 섹터의 순서를 디스크가 회전하는 방향에 맞추어 다시 정렬

![Untitled](Chapter10(입출력시스템)/Untitled%2031.png)

### 에션바흐(Eschenbach)

요청이 오래 걸리는 작업에 사용

한 섹터당 한개의 요청만 처리

헤드는 C-SCAN으로 처리

회전지연 시간 최적화 

![Untitled](Chapter10(입출력시스템)/Untitled%2032.png)

# 04. RAID(Redundant Array of Independent Disks)

자동으로 백업을 하고 장애가 발생하면 이를 복구하는 시스템 

하나의 원본 디스크와 같은 크기의 백업 디스크에 같은 내용을 동시에 저장하고 고장시 백업 디스크를 사용하여 복구

미러링 : 하나의 디스크에 똑같은 내용을 저장

스트라이핑 : 데이터를 여러 조각으로 나누어 저장 

패리티 비트 : 

### RAID 0

스트라이핑만 존재

입출력 속도는 빠르나 장애 발생 시 복구하는 기능이 없다.

![Untitled](Chapter10(입출력시스템)/Untitled%2033.png)

### RAID 1

미러링만 존재

데이터 Read시에는 같은 데이터가 두 군데의 디스크에 저장되어서 빠르게 읽어 오는 게 가능하지만, 데이터 Write할 경우에는 두 군데에 써야 하기에 성능 향상은 없다고 본다.

![Untitled](Chapter10(입출력시스템)/Untitled%2034.png)

### RAID 2

비트 단위로 나누어 저장

허밍코드를 사용하여 오류를 찾고 교정한다.

RAID 1보다 적은 저장 공간을 요구하지만 오류 교정 코드를 계산하는데 많은 시간 소비

![Untitled](Chapter10(입출력시스템)/Untitled%2035.png)

### RAID 3

하나의 디스크에 패리티 비트를 사용하여 오류를 검출

바이트 단위로 처리하여 데이터를 읽을 때 모든 디스크를 읽어야 함

![Untitled](Chapter10(입출력시스템)/Untitled%2036.png)

### RAID 4

RAID 3과 같은 방식이지만 블록단위로 처리

하나의 디스크에 패리티 비트가 저장되여 패리티 비트 디스크에 병목 현상 발생

두개의 디스크에서 장애 발생시 복구 불가

![Untitled](Chapter10(입출력시스템)/Untitled%2037.png)

### RAID 5

패리티 비트를 분산 저장

RAID 4의 단점 해결 -병목 현상 제거

![Untitled](Chapter10(입출력시스템)/Untitled%2038.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2039.png)

![Untitled](Chapter10(입출력시스템)/Untitled%2040.png)

### RAID 6

패리티 비트를 2개 운용

디스크 2개의 장애 복구 가능

패리티 비트가 2개라 계산량이 많고 추가 디스크 필요

![Untitled](Chapter10(입출력시스템)/Untitled%2041.png)

### RAID 0+1 , RAID10

미러링 기능의 RAID1과 스트라이핑 기능의 RAID0 결합

복구의 효율성 문제로 RAID10이 더 많이 사용

![Untitled](Chapter10(입출력시스템)/Untitled%2042.png)

### RAID 60

RAID6에 RAID0을 결합

![Untitled](Chapter10(입출력시스템)/Untitled%2043.png)