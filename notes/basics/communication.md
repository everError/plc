## 3. communication.md

### 통신 방식 개요

PLC와 외부 장치(PC, HMI, 다른 PLC)는 여러 방식으로 통신합니다.

### 물리 계층

| 방식     | 특징           | 거리      |
| -------- | -------------- | --------- |
| RS-232C  | 1:1 통신, 단순 | ~15m      |
| RS-422   | 1:N, 차동 신호 | ~1.2km    |
| RS-485   | N:N, 멀티드롭  | ~1.2km    |
| Ethernet | 고속, 범용     | 제한 없음 |

### 프로토콜 계층

| 프로토콜             | 제조사       | 전송 매체         |
| -------------------- | ------------ | ----------------- |
| MC Protocol (MELSEC) | Mitsubishi   | Serial / Ethernet |
| SLMP                 | CC-Link 협회 | Ethernet          |
| Modbus RTU           | 공개 표준    | Serial            |
| Modbus TCP           | 공개 표준    | Ethernet          |
| EtherNet/IP          | ODVA         | Ethernet          |

### MC Protocol 프레임 종류

| 프레임 | 전송 매체 | 대상 기종    |
| ------ | --------- | ------------ |
| 1E     | Ethernet  | A 시리즈     |
| 3E     | Ethernet  | Q/L/R/FX5    |
| 4E     | Ethernet  | Q/L/R (확장) |
| 1C     | Serial    | A 시리즈     |
| 3C     | Serial    | Q/L/R        |
| 4C     | Serial    | Q/L/R (확장) |

### Binary vs ASCII

| 모드   | 특징                | 데이터 크기 |
| ------ | ------------------- | ----------- |
| Binary | 효율적, 파싱 쉬움   | 작음        |
| ASCII  | 디버깅 쉬움, 가독성 | 약 2배      |

### 통신 흐름

```
[Client/Master]              [PLC]
      |                        |
      |--- Request Frame ----->|
      |                        |
      |<-- Response Frame -----|
      |                        |
```
