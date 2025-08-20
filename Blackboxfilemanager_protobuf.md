# MPU MONA - 车载多模块平台系统

![Version](https://img.shields.io/badge/version-v2.0-blue)
![Platform](https://img.shields.io/badge/platform-AUTOSAR-green)
![License](https://img.shields.io/badge/license-Proprietary-red)

MPU MONA 是一个基于 AUTOSAR Adaptive Platform 的车载多模块平台系统，集成了智能空调控制、电池热管理和黑盒文件管理三个核心模块。

## 🏗️ 系统架构

```
MPU_MONA/
├── intelligentaircondition/     # 智能空调控制模块
│   └── impl/src/main.cpp        # 空调服务主程序
├── batterythermalmanagement/    # 电池热管理模块
│   └── impl/src/main.cpp        # 电池管理服务
└── blackboxfilemanager/         # 黑盒文件管理模块
    ├── src/main.cpp              # TCP服务器主程序
    ├── include/                  # 头文件目录
    └── doc/FileTransfer.proto    # 通信协议定义
```

## 📦 核心模块

### 1. 智能空调控制 (Intelligent Air Condition)
- **端口**: AUTOSAR 通信接口
- **实例ID**: `IntelligentAirConditioning_EthernetECUInterface105`
- **功能**: 智能空调系统控制和监控
- **架构**: 基于 AUTOSAR Adaptive Platform

### 2. 电池热管理 (Battery Thermal Management)
- **端口**: AUTOSAR 通信接口  
- **实例ID**: `BatteryThermalManagement_EthernetECUInterface105`
- **功能**: 电池温度监控和热管理策略执行
- **架构**: 基于 AUTOSAR Adaptive Platform

### 3. 黑盒文件管理器 (BlackBox File Manager) ⭐
- **端口**: `32990` (TCP)
- **功能**: 车载数据采集、存储和远程传输
- **协议**: 基于 Protocol Buffers 的自定义协议
- **支持数据**: SOME/IP 日志、CAN 总线数据

## 🚀 黑盒文件管理器详解

### 核心功能

#### 📊 数据采集与存储
- **SOME/IP 协议日志**: 车载以太网通信数据
- **CAN 总线数据**: 传统车载总线通信记录
- **时间戳索引**: 基于时间范围的高效数据检索
- **自动压缩**: `.tar.gz` 格式压缩存储

#### 🌐 远程数据传输
- **TCP 服务器**: 监听端口 `32990`
- **多线程处理**: 每个连接独立线程处理
- **分块传输**: 支持大文件分块下载 (50KB/块)
- **完整性校验**: MD5 + XOR 双重校验

### 通信协议

基于 **Protocol Buffers** 的高效二进制协议：

#### PDU 数据包格式
```
┌─────────────────────────────────────────────┐
│ Header(2) | SeqNum(2) | Length(4) | Type(1) │
│   0x11EE  |   0x0000  |   Length  |   0x03  │
└─────────────────────────────────────────────┘
│          ProtoBuf Data Content              │
└─────────────────────────────────────────────┘
│                Checksum(1)                  │
└─────────────────────────────────────────────┘
```

#### 消息类型

| 消息类型 | 功能 | 方向 |
|---------|------|-----|
| `TimeStamp` | 时间范围查询请求 | Client → Server |
| `FileList` | 文件列表响应 | Server → Client |
| `FileData` | 文件数据请求/响应 | 双向 |
| `AckData` | 操作确认应答 | Server → Client |

### 📡 API 使用示例

#### 1. 连接服务器
```cpp
#include <sys/socket.h>
#include <netinet/in.h>

int socketfd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in serverAddr;
serverAddr.sin_family = AF_INET;
inet_pton(AF_INET, "192.168.1.100", &serverAddr.sin_addr);
serverAddr.sin_port = htons(32990);

connect(socketfd, (struct sockaddr*)&serverAddr, sizeof(serverAddr));
```

#### 2. 查询时间范围内的文件
```cpp
#include "FileTransfer.pb.h"

// 创建时间范围查询
datatranst::v1::TimeStamp timeStamp;
timeStamp.set_starttimestamp(1634567890);  // 开始时间 (Unix时间戳)
timeStamp.set_endtimestamp(1634654290);    // 结束时间

// 序列化为二进制数据
auto size = timeStamp.ByteSize();
uint8_t buffer[size];
timeStamp.SerializeToArray(buffer, size);

// 构造 PDU 数据包
std::vector<uint8_t> pdu = {
    0x11, 0xEE,                    // 包头
    0x00, 0x00,                    // 序列号
    0x00, 0x00, 0x00, length,      // 长度
    0x03                           // 序列化类型
};
pdu.insert(pdu.end(), buffer, buffer + size);
pdu.push_back(checksum);           // XOR 校验和

// 发送请求
send(socketfd, pdu.data(), pdu.size(), 0);
```

#### 3. 接收文件列表
```cpp
uint8_t recvBuffer[50*1024];
int bytesReceived = recv(socketfd, recvBuffer, sizeof(recvBuffer), 0);

// 解析 PDU 获取 ProtoBuf 数据
datatranst::v1::FileList fileList;
fileList.ParseFromArray(protobuf_data, protobuf_size);

// 遍历文件列表
for(int i = 0; i < fileList.valuedata_size(); i++) {
    const auto& file = fileList.valuedata(i);
    std::cout << "文件名: " << file.filename() << std::endl;
    std::cout << "大小: " << file.filelength() << " bytes" << std::endl;
    std::cout << "MD5: " << file.md5() << std::endl;
}
```

#### 4. 下载文件内容
```cpp
datatranst::v1::FileData fileRequest;
fileRequest.set_filename("2023-10-18_2023-10-19.tar.gz");
fileRequest.set_filepoint(0);         // 起始位置
fileRequest.set_datalength(51200);    // 请求 50KB

// 发送文件数据请求...
// 接收文件数据响应...

datatranst::v1::FileData fileResponse;
fileResponse.ParseFromArray(response_data, response_size);

// 保存文件数据
std::ofstream file(fileResponse.filename(), std::ios::binary | std::ios::app);
file.write(fileResponse.msg().data(), fileResponse.msg().size());
```

### 🔧 技术特性

#### 多线程架构
- **连接处理**: 每个 TCP 连接分配独立线程
- **观察者模式**: `PDU → DataSubject → FileManager` 数据流处理
- **线程安全**: 使用 RAII 模式管理资源

#### 数据处理流程
```cpp
// TcpSocket.cpp:71 - 为每个连接创建处理线程
ProjectController projCtrller;
mThreadpool.push_back(std::thread{
    &ProjectController::threadFunc, &projCtrller, connfd
});

// ProjectController.cpp:63 - 消息分发处理
pduSubject.notify(pduContentVec);
```

#### Protocol Buffers 优势
- **高效性能**: 比 JSON 小 20-100 倍，速度快 20-100 倍
- **跨语言支持**: C++, Java, Python 等多语言兼容
- **向前兼容**: 支持协议版本升级而不破坏兼容性
- **类型安全**: 编译期类型检查，避免运行时错误

### 🛠️ 编译和部署

#### 依赖库
- **Protocol Buffers**: `lib/libprotobuf.a`
- **OpenSSL**: `lib/libcrypto.a`, `lib/libssl.a` (用于 MD5)
- **标准库**: C++11 及以上

#### 编译步骤
```bash
cd blackboxfilemanager
mkdir build && cd build
cmake ..
make

# 运行服务器
./fileManager.out
```

#### 生成 Protocol Buffers 代码
```bash
protoc --cpp_out=. doc/FileTransfer.proto
```

### 📈 使用场景

1. **远程诊断**: 技术人员远程提取车辆运行数据进行故障分析
2. **质量监控**: 自动收集指定时间段的通信日志用于质量分析
3. **开发调试**: 开发阶段实时获取车载网络通信数据
4. **数据备份**: 批量传输重要的车载黑盒数据到云端存储
5. **合规审计**: 提取特定时间范围的数据用于监管合规检查

### ⚠️ 注意事项

1. **网络安全**: 建议在安全的车载网络环境中使用，避免外部网络直接访问
2. **存储空间**: 定期清理临时文件，避免存储空间耗尽
3. **并发限制**: 根据系统资源合理控制并发连接数量
4. **数据隐私**: 传输的车载数据可能包含敏感信息，需要适当的加密保护

### 🤝 贡献

本项目为内部开发项目，如需贡献或报告问题，请联系开发团队。

### 📄 许可证

版权所有 © 。保留所有权利。

---
