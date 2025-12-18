## Mitsubishi 3E Frame Read Request

### 개요

Mitsubishi MELSEC Q/L/R/FX5 시리즈 PLC와 통신하기 위한 3E Frame 프로토콜의 Batch Read 요청 구조체입니다. Binary 통신 방식을 사용합니다.

### 프레임 구조 (총 21바이트)

| Offset | 필드명               | 크기 | 기본값 | 설명                         |
| ------ | -------------------- | ---- | ------ | ---------------------------- |
| 0x00   | Header               | 2    | 50 00  | 3E Frame 식별자              |
| 0x02   | NetworkNo            | 1    | 00     | 네트워크 번호                |
| 0x03   | PcNo                 | 1    | FF     | PC 번호                      |
| 0x04   | DestModuleIONo       | 2    | FF 03  | 목적지 모듈 I/O 번호         |
| 0x06   | DestModuleStationNo  | 1    | 00     | 목적지 국번                  |
| 0x07   | RequestLength        | 2    | 0C 00  | 이후 데이터 길이 (12바이트)  |
| 0x09   | CpuTimer             | 2    | 10 00  | CPU 감시 타이머              |
| 0x0B   | Command              | 2    | 01 04  | 명령어 (0x0401 = Batch Read) |
| 0x0D   | SubCommand           | 2    | 가변   | 00 00(워드) / 01 00(비트)    |
| 0x0F   | HeadDeviceNumber     | 3    | 가변   | 시작 주소 (Little-endian)    |
| 0x12   | DeviceCode           | 1    | 가변   | 디바이스 코드                |
| 0x13   | NumberOfDevicePoints | 2    | 가변   | 읽을 포인트 수               |

### 디바이스 코드

| 디바이스            | 코드 | 주소 형식 | 타입 |
| ------------------- | ---- | --------- | ---- |
| D (데이터 레지스터) | 0xA8 | 10진수    | Word |
| W (링크 레지스터)   | 0xB4 | 16진수    | Word |
| R (파일 레지스터)   | 0xAF | 10진수    | Word |
| M (내부 릴레이)     | 0x90 | 10진수    | Bit  |
| L (래치 릴레이)     | 0x92 | 10진수    | Bit  |
| B (링크 릴레이)     | 0xA0 | 16진수    | Bit  |

### 읽기 제한

| 모드      | SubCommand | 최대 포인트 |
| --------- | ---------- | ----------- |
| 워드 단위 | 00 00      | 960         |
| 비트 단위 | 01 00      | 3584        |

### 사용 예시

```csharp
// D100부터 10워드 읽기
var req1 = new ReadRequest_3E_Frames(Device.D, "100", 10);

// W0x1A0부터 5워드 읽기
var req2 = new ReadRequest_3E_Frames(Device.W, "1A0", 5);

// M0부터 32비트 읽기
var req3 = new ReadRequest_3E_Frames(Device.M, "0", 32);

// M0부터 워드 단위로 16포인트 읽기
var req4 = new ReadRequest_3E_Frames(Device.M, "0", 16, readAsBit: false);
```

### 주의사항

- 필드 순서 변경 금지 (`StructLayout.Sequential` 사용)
- 모든 멀티바이트 값은 Little-endian
- 비트 디바이스도 `readAsBit: false`로 워드 단위 읽기 가능

### 구현 코드

```csharp
using System.Runtime.InteropServices;

namespace ADOT.PLC.Mitsubishi.Protocols.Binary.Request;

/// <summary>
/// Mitsubishi 3E Frames protocol
/// Communication to the Q/QnA/L/R and FX5 Series CPUs.
/// </summary>
[Serializable]
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct ReadRequest_3E_Frames
{
    // field 순서 변경 금지
    // Request-frame format
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] Header = new byte[2] { 0x50, 0x00 };              // 50 00

    // Q header
    [MarshalAs(UnmanagedType.U1)]
    public byte NetworkNo = 0x00;                                   // 00

    [MarshalAs(UnmanagedType.U1)]
    public byte PcNo = 0xFF;                                        // FF

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] DestModuleIONo = new byte[2] { 0xFF, 0x03 };      // FF 03

    [MarshalAs(UnmanagedType.U1)]
    public byte DestModuleStationNo = 0x00;                         // 00

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] RequestLength = new byte[2] { 0x0C, 0x00 };       // 0C 00 (12 bytes)
    // 해당 필드 이후 프레임의 바이트 크기

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] CpuTimer = new byte[2] { 0x10, 0x00 };            // 10 00

    // Text portion of the 3E/3C frame
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] Command = new byte[2] { 0x01, 0x04 };             // 01 04 (0x0401: Batch Read)

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] SubCommand;                                       // 00 00 - word, 01 00 - bit

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 3)]
    public byte[] HeadDeviceNumber;

    [MarshalAs(UnmanagedType.U1)]
    public byte DeviceCode;

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] NumberOfDevicePoints;

    // 디바이스별 최대 읽기 포인트 수
    private const int MaxBitDevicePoints = 3584;
    private const int MaxWordDevicePoints = 960;

    /// <summary>
    /// 3E Frame Read Request 생성
    /// </summary>
    /// <param name="device">D, W, M, R, L, B</param>
    /// <param name="headAddress">DeviceCode를 제외한 주소번지만으로 구성</param>
    /// <param name="size">읽을 디바이스 포인트 수</param>
    /// <param name="readAsBit">비트 디바이스를 비트 단위로 읽을지 여부 (기본값: true)</param>
    /// <exception cref="ArgumentException">지원하지 않는 디바이스 타입</exception>
    /// <exception cref="ArgumentOutOfRangeException">주소 또는 크기 범위 초과</exception>
    public ReadRequest_3E_Frames(Device device, string headAddress, int size, bool readAsBit = true)
    {
        // 비트 디바이스 여부 확인
        bool isBitDevice = device is Device.M or Device.L or Device.B;

        // SubCommand 결정 (비트/워드 읽기 모드)
        bool useBitMode = isBitDevice && readAsBit;
        SubCommand = useBitMode
            ? new byte[] { 0x01, 0x00 }  // 비트 단위 읽기
            : new byte[] { 0x00, 0x00 }; // 워드 단위 읽기

        // size 범위 검증
        int maxPoints = useBitMode ? MaxBitDevicePoints : MaxWordDevicePoints;
        if (size < 1 || size > maxPoints)
        {
            throw new ArgumentOutOfRangeException(nameof(size),
                $"Size must be between 1 and {maxPoints} for {(useBitMode ? "bit" : "word")} read mode.");
        }

        // HeadDeviceNumber 계산
        int address = device switch
        {
            Device.W => Convert.ToInt32(headAddress, 16),   // Base HEX
            Device.B => Convert.ToInt32(headAddress, 16),   // Base HEX
            Device.D => int.Parse(headAddress),             // Base Decimal
            Device.M => int.Parse(headAddress),             // Base Decimal
            Device.R => int.Parse(headAddress),             // Base Decimal
            Device.L => int.Parse(headAddress),             // Base Decimal
            _ => throw new ArgumentException($"Unsupported Device Type: {device}", nameof(device))
        };

        // 주소 범위 검증 (3바이트 = 24비트)
        if (address < 0 || address > 0xFFFFFF)
        {
            throw new ArgumentOutOfRangeException(nameof(headAddress),
                "Address must be between 0 and 16777215 (0xFFFFFF).");
        }

        HeadDeviceNumber = new byte[]
        {
            (byte)address,          // 하위 8비트
            (byte)(address >> 8),   // 중간 8비트
            (byte)(address >> 16)   // 상위 8비트
        };

        // DeviceCode 설정
        DeviceCode = device switch
        {
            Device.D => 0xA8,
            Device.W => 0xB4,
            Device.M => 0x90,
            Device.R => 0xAF,
            Device.L => 0x92,
            Device.B => 0xA0,
            _ => throw new ArgumentException($"Unsupported Device Type: {device}", nameof(device))
        };

        // NumberOfDevicePoints 설정
        NumberOfDevicePoints = new byte[2]
        {
            (byte)size,         // 하위 8비트
            (byte)(size >> 8)   // 상위 8비트
        };
    }
}
```
