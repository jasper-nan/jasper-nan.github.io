---
title: "CAN 总线日志分析入门：自动驾驶测试必备技能"
description: "自动驾驶测试工程师的核心技能之一。从CAN总线基础到日志解析，一文讲透。"
date: 2026-03-25
tags:
  - CAN总线
  - 自动驾驶
  - 测试
  - Python
---
> 如果你从事自动驾驶或智能网联汽车测试，CAN 总线日志分析是绕不开的技能。这篇从基础概念到实战解析，帮你快速上手。

* * *

## 一、什么是 CAN 总线？ 

CAN（Controller Area Network）是汽车内部最主流的通信协议。简单说：

> 车辆的各个 ECU（电子控制单元）通过 CAN 总线互相"说话"，比如发动机告诉仪表盘转速、方向盘告诉转向灯亮灯。

### 核心概念 

概念 | 说明  
---|---  
**报文（Frame）** | CAN 总线上传输的最小数据单元  
**ID** | 报文的"身份证"，决定优先级，ID 越小优先级越高  
**DLC** | 数据长度（0-8 字节）  
**Payload** | 实际数据内容  
**周期** | 某些报文按固定周期发送（如 10ms、100ms）  
  
### 报文结构 
    
    
    ┌─────────┬───────┬─────┬──────────┬────────┬─────────┐
    │ Start   │ ID    │ DLC │ Payload  │ CRC    │ ACK     │
    │ of Frame│ 11bit │     │ 0-8 bytes│ Check  │         │
    └─────────┴───────┴─────┴──────────┴────────┴─────────┘
    

* * *

## 二、CAN DBC 文件 

DBC 是 CAN 数据的"字典文件"，定义了每个 ID 对应什么信号、如何解析。

### DBC 文件长什么样 
    
    
    VERSION ""
    
    NS_ :
    
    BS_:
    
    BU_: ECU1 ECU2 ECU3
    
    BO_ 256 SteeringAngle: 7 ECU1
     SG_ SteeringAngle_Value : 0|15@1+ (0.1,0) [0|6553.5] "deg" Vector__XXX
     SG_ SteeringAngle_Speed : 16|15@1+ (1,0) [0|32767] "deg/s" Vector__XXX
    

**解读：**

  * `BO_ 256` → CAN ID 为 0x100 (256)
  * `SteeringAngle` → 信号名：方向盘转角
  * `0|15@1+` → 从第 0 位开始，长度 15 位，小端序，无符号
  * `(0.1,0)` → 精度 0.1，偏移 0（实际值 = 原始值 × 0.1 + 0）
  * `[0|6553.5]` → 物理值范围 0 到 6553.5
  * `"deg"` → 单位：度



**没有 DBC 文件，你看到的只是十六进制数据；有了 DBC，你才能知道这些数据代表什么。**

* * *

## 三、日志格式 

### 常见日志格式 

格式 | 说明 | 工具  
---|---|---  
**ASC** | Vector CANalyzer/CANoe 格式 | CANalyzer、CANoe  
**BLF** | Binary Log Format | Vector 工具  
**PCAP** | 通用网络抓包格式 | Wireshark  
**CSV** | 逗号分隔，通用性强 | Python/Excel  
**MDF** | 测量数据格式 | CANape、DIAdem  
  
### ASC 格式示例 
    
    
    date Mon Jan 01 10:00:00 am 2026
    base hex  timestamps absolute
       0.000000 1  100             Tx   d 8 01 02 03 04 05 06 07 08
       0.001000 1  1A0             Rx   d 8 AA BB CC DD EE FF 00 11
       0.002500 1  200             Rx   d 4 12 34 56 78
    

**每列含义：** 时间戳 | 通道 | ID | 方向 | 帧类型 | DLC | 数据字节

* * *

## 四、Python 解析实战 

### 1\. 解析 ASC 文件 
    
    
    import re
    
    def parse_asc(filepath):
        """解析 ASC 格式的 CAN 日志"""
        frames = []
        pattern = re.compile(
            r'(\d+\.\d+)\s+\d+\s+([0-9A-Fa-f]+)\s+\w+\s+d\s+(\d+)\s+(.*)'
        )
        
        with open(filepath, 'r') as f:
            for line in f:
                match = pattern.search(line)
                if match:
                    timestamp = float(match.group(1))
                    can_id = int(match.group(2), 16)
                    dlc = int(match.group(3))
                    data = [int(x, 16) for x in match.group(4).split()]
                    frames.append({
                        'timestamp': timestamp,
                        'id': can_id,
                        'dlc': dlc,
                        'data': data
                    })
        
        return frames
    

### 2\. 使用 cantools 解析 DBC 
    
    
    import cantools
    
    # 加载 DBC 文件
    db = cantools.database.load_file('vehicle.dbc')
    
    # 查看所有报文
    for msg in db.messages:
        print(f"ID: 0x{msg.frame_id:03X}  Name: {msg.name}  "
              f"Signals: {len(msg.signals)}")
    
    # 解析一条报文
    msg = db.get_message_by_frame_id(0x100)
    decoded = msg.decode(bytes([0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08]))
    print(decoded)
    # {'SteeringAngle_Value': 15.3, 'SteeringAngle_Speed': 259.0}
    

### 3\. 信号提取 + 时序分析 
    
    
    import pandas as pd
    import matplotlib.pyplot as plt
    
    def extract_signal(frames, db, message_name, signal_name):
        """从日志中提取指定信号的时间序列"""
        msg = db.get_message_by_name(message_name)
        timestamps = []
        values = []
        
        for frame in frames:
            if frame['id'] == msg.frame_id:
                decoded = msg.decode(bytes(frame['data']))
                timestamps.append(frame['timestamp'])
                values.append(decoded[signal_name])
        
        return pd.DataFrame({'timestamp': timestamps, signal_name: values})
    
    # 提取方向盘转角
    df = extract_signal(frames, db, 'SteeringAngle', 'SteeringAngle_Value')
    
    # 画图
    plt.figure(figsize=(12, 4))
    plt.plot(df['timestamp'], df['SteeringAngle_Value'])
    plt.xlabel('Time (s)')
    plt.ylabel('Steering Angle (deg)')
    plt.title('Steering Angle Time Series')
    plt.grid(True)
    plt.tight_layout()
    plt.savefig('steering_angle.png', dpi=150)
    

### 4\. 报文频率统计 
    
    
    from collections import Counter
    
    def message_frequency(frames):
        """统计每个 CAN ID 的发送频率"""
        id_counts = Counter(f['id'] for f in frames)
        total_time = frames[-1]['timestamp'] - frames[0]['timestamp']
        
        freq = {}
        for can_id, count in sorted(id_counts.items()):
            freq[can_id] = {
                'count': count,
                'frequency': count / total_time if total_time > 0 else 0
            }
        
        return freq
    
    # 输出前 10 个最频繁的报文
    freq = message_frequency(frames)
    for can_id, info in sorted(freq.items(), key=lambda x: x[1]['count'], reverse=True)[:10]:
        print(f"ID: 0x{can_id:03X}  Count: {info['count']}  "
              f"Freq: {info['frequency']:.1f} Hz")
    

* * *

## 五、常见分析场景 

### 场景 1：验证报文周期 
    
    
    def check_period(frames, target_id, expected_period_ms, tolerance=0.1):
        """检查报文发送周期是否符合预期"""
        target_frames = [f for f in frames if f['id'] == target_id]
        periods = []
        
        for i in range(1, len(target_frames)):
            period_ms = (target_frames[i]['timestamp'] - target_frames[i-1]['timestamp']) * 1000
            periods.append(period_ms)
        
        issues = []
        for i, p in enumerate(periods):
            if abs(p - expected_period_ms) / expected_period_ms > tolerance:
                issues.append((i, p))
        
        print(f"平均周期: {sum(periods)/len(periods):.2f} ms")
        print(f"最大偏差: {max(periods):.2f} ms")
        print(f"超差次数: {len(issues)}/{len(periods)}")
        
        return issues
    

### 场景 2：检测总线负载率 
    
    
    def bus_load(frames, bitrate=500000, total_time=None):
        """计算 CAN 总线负载率"""
        if total_time is None:
            total_time = frames[-1]['timestamp'] - frames[0]['timestamp']
        
        # 每帧占用的 bit 数（标准帧）
        bits_per_frame = 47 + 8 * max(f['dlc'] for f in frames) + 3  # 简化估算
        total_bits = len(frames) * bits_per_frame
        
        load = total_bits / (bitrate * total_time) * 100
        print(f"总线负载率: {load:.2f}%")
        return load
    

### 场景 3：错误帧检测 
    
    
    def find_error_frames(filepath):
        """查找日志中的错误帧"""
        errors = []
        with open(filepath, 'r') as f:
            for i, line in enumerate(f, 1):
                if 'Error' in line or 'error' in line.lower():
                    errors.append((i, line.strip()))
        
        print(f"发现 {len(errors)} 个错误帧")
        for ln, content in errors[:10]:
            print(f"  Line {ln}: {content[:100]}")
        
        return errors
    

* * *

## 六、常用工具 

工具 | 用途 | 价格  
---|---|---  
**Vector CANalyzer** | CAN 日志录制与分析 | 商业（贵）  
**Vector CANoe** | 仿真 + 测试 + 分析 | 商业（很贵）  
**PCAN-View** | 简单的 CAN 监控 | 硬件附赠  
**Wireshark (CAN) plugin** | 开源抓包分析 | 免费  
**can-utils (Linux)** | 命令行 CAN 工具 | 免费  
**cantools (Python)** | DBC 解析 + 信号提取 | 免费  
**SavvyCAN** | 开源 CAN 分析工具 | 免费  
  
**新手推荐：** Python + cantools 免费强大，配合 pandas/matplotlib 做可视化。

* * *

## 七、调试技巧 

  1. **先确认 ID 是否正确** — 用十六进制和十进制都要检查一遍
  2. **检查字节序** — 大端序（Motorola）和小端序（Intel）搞混是常见错误
  3. **关注时间戳** — 确保时间戳是连续的，有没有断点
  4. **先看一个信号** — 不要一次性分析所有信号，先从一个简单的开始
  5. **对齐视频和日志** — 问题复现时，视频时间戳和日志时间戳对齐是关键



* * *

_CAN 日志分析是自动驾驶测试工程师的核心竞争力。掌握 Python 解析 + DBC 解码 + 可视化分析，你的测试效率会提升一个量级。_