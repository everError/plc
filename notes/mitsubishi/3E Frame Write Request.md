## Mitsubishi 3E Frame Write Request

### 개요

Mitsubishi MELSEC Q/L/R/FX5 시리즈 PLC와 통신하기 위한 3E Frame 프로토콜의 Batch Write 요청 구조체입니다. Binary 통신 방식을 사용합니다.

### 프레임 구조

| Offset | 필드명               | 크기 | 기본값 | 설명                          |
| ------ | -------------------- | ---- | ------ | ----------------------------- |
| 0x00   | Header               | 2    | 50 00  | 3E Frame 식별자               |
| 0x02   | NetworkNo            | 1    | 00     | 네트워크 번호                 |
| 0x03   | PcNo                 | 1    | FF     | PC 번호                       |
| 0x04   | DestModuleIONo       | 2    | FF 03  | 목적지 모듈 I/O 번호          |
| 0x06   | DestModuleStationNo  | 1    | 00     | 목적지 국번                   |
| 0x07   | RequestLength        | 2    | 가변   | 이후 데이터 길이              |
| 0x09   | CpuTimer             | 2    | 10 00  | CPU 감시 타이머               |
| 0x0B   | Command              | 2    | 01 14  | 명령어 (0x1401 = Batch Write) |
| 0x0D   | SubCommand           | 2    | 가변   | 00 00(워드) / 01 00(비트)     |
| 0x0F   | HeadDeviceNumber     | 3    | 가변   | 시작 주소 (Little-endian)     |
| 0x12   | DeviceCode           | 1    | 가변   | 디바이스 코드                 |
| 0x13   | NumberOfDevicePoints | 2    | 가변   | 쓸 포인트 수                  |
| 0x15   | WriteData            | 가변 | 가변   | 쓸 데이터                     |

### 쓰기 제한

| 모드      | SubCommand | 최대 포인트 |
| --------- | ---------- | ----------- |
| 워드 단위 | 00 00      | 960         |
| 비트 단위 | 01 00      | 3584        |

### 데이터 형식

| 모드      | 데이터 단위     | 설명                                     |
| --------- | --------------- | ---------------------------------------- |
| 워드 쓰기 | 2바이트/포인트  | Little-endian 16비트 값                  |
| 비트 쓰기 | 1바이트/2포인트 | 상위 4비트 + 하위 4비트 (0x00 또는 0x01) |

### 사용 예시

```csharp
// D100부터 3워드 쓰기
var data1 = new ushort[] { 100, 200, 300 };
var req1 = new WriteRequest_3E_Frames(Device.D, "100", data1);

// W0x1A0부터 2워드 쓰기
var data2 = new ushort[] { 0x1234, 0x5678 };
var req2 = new WriteRequest_3E_Frames(Device.W, "1A0", data2);

// M0부터 4비트 쓰기 (ON, OFF, ON, ON)
var bits = new bool[] { true, false, true, true };
var req3 = new WriteRequest_3E_Frames(Device.M, "0", bits);

// M0부터 워드 단위로 쓰기 (16비트를 한번에)
var wordData = new ushort[] { 0b1010_1010_1010_1010 };
var req4 = new WriteRequest_3E_Frames(Device.M, "0", wordData, writeAsBit: false);
```

### 주의사항

- 필드 순서 변경 금지 (`StructLayout.Sequential` 사용)
- 모든 멀티바이트 값은 Little-endian
- 비트 쓰기 시 홀수 개의 비트는 마지막 바이트의 상위 4비트가 0x00으로 패딩됨
- RequestLength는 데이터 크기에 따라 동적으로 계산됨

### 구현 코드

```csharp
/// <summary>
/// Mitsubishi 3E Frames protocol - Batch Write
/// Communication to the Q/QnA/L/R and FX5 Series CPUs.
/// </summary>
[Serializable]
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct WriteRequest_3E_Frames
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
    public byte[] RequestLength;                                    // 가변 (12 + 데이터 길이)

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] CpuTimer = new byte[2] { 0x10, 0x00 };            // 10 00

    // Text portion of the 3E/3C frame
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] Command = new byte[2] { 0x01, 0x14 };             // 01 14 (0x1401: Batch Write)

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] SubCommand;                                       // 00 00 - word, 01 00 - bit

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 3)]
    public byte[] HeadDeviceNumber;

    [MarshalAs(UnmanagedType.U1)]
    public byte DeviceCode;

    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2)]
    public byte[] NumberOfDevicePoints;

    public byte[] WriteData;

    // 디바이스별 최대 쓰기 포인트 수
    private const int MaxBitDevicePoints = 3584;
    private const int MaxWordDevicePoints = 960;

    /// <summary>
    /// 3E Frame Write Request 생성 (워드 데이터)
    /// </summary>
    /// <param name="device">D, W, M, R, L, B</param>
    /// <param name="headAddress">DeviceCode를 제외한 주소번지만으로 구성</param>
    /// <param name="data">쓸 워드 데이터 배열</param>
    /// <param name="writeAsBit">비트 디바이스를 비트 단위로 쓸지 여부 (기본값: false)</param>
    public WriteRequest_3E_Frames(Device device, string headAddress, ushort[] data, bool writeAsBit = false)
    {
        bool isBitDevice = device is Device.M or Device.L or Device.B;
        bool useBitMode = isBitDevice && writeAsBit;

        // 워드 데이터로 비트 쓰기는 허용하지 않음
        if (useBitMode)
        {
            throw new ArgumentException(
                "Use bool[] overload for bit write mode.", nameof(writeAsBit));
        }

        InitializeCommon(device, headAddress, data.Length, useBitMode: false);

        // 워드 데이터를 바이트 배열로 변환 (Little-endian)
        WriteData = new byte[data.Length * 2];
        for (int i = 0; i < data.Length; i++)
        {
            WriteData[i * 2] = (byte)data[i];           // 하위 8비트
            WriteData[i * 2 + 1] = (byte)(data[i] >> 8); // 상위 8비트
        }

        // RequestLength 계산: 12 (고정 헤더) + 데이터 길이
        int requestLength = 12 + WriteData.Length;
        RequestLength = new byte[]
        {
            (byte)requestLength,
            (byte)(requestLength >> 8)
        };
    }

    /// <summary>
    /// 3E Frame Write Request 생성 (비트 데이터)
    /// </summary>
    /// <param name="device">M, L, B (비트 디바이스만)</param>
    /// <param name="headAddress">DeviceCode를 제외한 주소번지만으로 구성</param>
    /// <param name="data">쓸 비트 데이터 배열</param>
    public WriteRequest_3E_Frames(Device device, string headAddress, bool[] data)
    {
        bool isBitDevice = device is Device.M or Device.L or Device.B;
        if (!isBitDevice)
        {
            throw new ArgumentException(
                $"Device {device} is not a bit device. Use ushort[] overload.", nameof(device));
        }

        InitializeCommon(device, headAddress, data.Length, useBitMode: true);

        // 비트 데이터를 바이트 배열로 변환 (2비트/바이트, 상위+하위 4비트)
        int byteLength = (data.Length + 1) / 2;
        WriteData = new byte[byteLength];

        for (int i = 0; i < data.Length; i++)
        {
            byte bitValue = data[i] ? (byte)0x01 : (byte)0x00;
            if (i % 2 == 0)
            {
                WriteData[i / 2] = (byte)(bitValue << 4); // 상위 4비트
            }
            else
            {
                WriteData[i / 2] |= bitValue;             // 하위 4비트
            }
        }

        // RequestLength 계산: 12 (고정 헤더) + 데이터 길이
        int requestLength = 12 + WriteData.Length;
        RequestLength = new byte[]
        {
            (byte)requestLength,
            (byte)(requestLength >> 8)
        };
    }

    /// <summary>
    /// 공통 필드 초기화
    /// </summary>
    private void InitializeCommon(Device device, string headAddress, int pointCount, bool useBitMode)
    {
        // size 범위 검증
        int maxPoints = useBitMode ? MaxBitDevicePoints : MaxWordDevicePoints;
        if (pointCount < 1 || pointCount > maxPoints)
        {
            throw new ArgumentOutOfRangeException(nameof(pointCount),
                $"Point count must be between 1 and {maxPoints} for {(useBitMode ? "bit" : "word")} write mode.");
        }

        // SubCommand 설정
        SubCommand = useBitMode
            ? new byte[] { 0x01, 0x00 }  // 비트 단위 쓰기
            : new byte[] { 0x00, 0x00 }; // 워드 단위 쓰기

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
            (byte)address,
            (byte)(address >> 8),
            (byte)(address >> 16)
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
        NumberOfDevicePoints = new byte[]
        {
            (byte)pointCount,
            (byte)(pointCount >> 8)
        };
    }

    /// <summary>
    /// 전체 프레임을 바이트 배열로 반환
    /// </summary>
    public byte[] ToBytes()
    {
        int totalLength = 21 + WriteData.Length;
        byte[] result = new byte[totalLength];

        int offset = 0;

        Buffer.BlockCopy(Header, 0, result, offset, 2); offset += 2;
        result[offset++] = NetworkNo;
        result[offset++] = PcNo;
        Buffer.BlockCopy(DestModuleIONo, 0, result, offset, 2); offset += 2;
        result[offset++] = DestModuleStationNo;
        Buffer.BlockCopy(RequestLength, 0, result, offset, 2); offset += 2;
        Buffer.BlockCopy(CpuTimer, 0, result, offset, 2); offset += 2;
        Buffer.BlockCopy(Command, 0, result, offset, 2); offset += 2;
        Buffer.BlockCopy(SubCommand, 0, result, offset, 2); offset += 2;
        Buffer.BlockCopy(HeadDeviceNumber, 0, result, offset, 3); offset += 3;
        result[offset++] = DeviceCode;
        Buffer.BlockCopy(NumberOfDevicePoints, 0, result, offset, 2); offset += 2;
        Buffer.BlockCopy(WriteData, 0, result, offset, WriteData.Length);

        return result;
    }
}
```
