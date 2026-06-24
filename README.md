# Linux 部署中文语音识别：sherpa-onnx + RK3588 实战

本文记录一次从 **Linux 虚拟机验证** 到 **RK3588 边缘设备部署** 中文语音识别模型的完整流程。目标是在 Linux / RK3588 上部署本地离线中文 ASR，用于中文短指令识别、车载控制、设备控制等场景。

> 重点：本文不是做会议长文本转写，而是做中文短指令识别，例如“打开主屏”“关闭灯光”“升降杆上升”“切换四分屏”。

---

## 1. 部署目标

目标能力：

- 中文语音识别
- 本地离线运行
- 支持实时麦克风识别
- 运行内存尽量低
- 适合 RK3588 这类边缘设备
- 可后续接入业务控制系统

典型指令：

```text
打开主屏
关闭灯光
升降杆上升
升降杆下降
升降杆停止
切换四分屏
摄像头左转
摄像头右转
开始录播
停止录播
```

---

## 2. 模型选择建议

最开始可以用低内存模型验证：

```text
sherpa-onnx-streaming-zipformer-small-ctc-zh-int8-2025-04-01
```

这个模型优点是很小，`model.int8.onnx` 大约 25MB，适合低内存验证。缺点是识别率一般，尤其是对“升降杆”“四分屏”“录播”等业务词不够稳定。

更推荐正式部署使用：

```text
sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30
```

这个模型是 Transducer 结构，文件包括：

```text
encoder.int8.onnx
decoder.onnx
joiner.int8.onnx
tokens.txt
```

它比 small CTC 模型识别率更好，同时不会像 xlarge 模型那样占用过高内存，比较适合 4GB / 8GB RK3588。

模型下载页面：

```text
https://github.com/k2-fsa/sherpa-onnx/releases/tag/asr-models
```

---

## 3. 推荐部署路线

建议不要一开始直接上 RK3588。推荐顺序：

```text
Linux 虚拟机验证模型
→ 测试麦克风录音
→ 测试文件识别
→ 测试实时识别
→ 迁移到 RK3588
→ 加业务纠错和命令匹配
→ 封装成本地 ASR 服务
```

---

## 4. Linux 虚拟机环境准备

推荐环境：

```text
系统：Ubuntu 22.04 / Ubuntu 24.04
平台：VirtualBox / VMware
架构：x86_64
内存：2GB 以上
磁盘：5GB 以上
```

安装依赖：

```bash
sudo apt update

sudo apt install -y \
  git \
  wget \
  bzip2 \
  tar \
  unzip \
  alsa-utils \
  libasound2t64 \
  pulseaudio-utils \
  pavucontrol
```

Ubuntu 24.04 中 `libasound2` 可能会提示没有安装候选项，需要使用：

```bash
sudo apt install -y libasound2t64
```

---

## 5. VirtualBox 网络代理设置

如果虚拟机无法直接访问 GitHub，可以让 Ubuntu 走 Windows 宿主机代理。

VirtualBox 网络建议选择：

```text
连接方式：NAT
```

不要选择：

```text
NAT 网络
桥接到 VPN 虚拟网卡
FClash / Clash 虚拟网卡
VirtualBox Host-Only
```

如果 Windows 上 FClash / Clash 的代理端口是 `7890`，Ubuntu 中临时设置代理：

```bash
export http_proxy=http://10.0.2.2:7890
export https_proxy=http://10.0.2.2:7890
export all_proxy=socks5://10.0.2.2:7890
```

其中 `10.0.2.2` 是 VirtualBox NAT 模式下，Ubuntu 虚拟机访问 Windows 宿主机的特殊地址。

测试网络：

```bash
curl -I https://github.com
```

---

## 6. 下载 sherpa-onnx 运行程序

创建目录：

```bash
mkdir -p ~/asr
cd ~/asr
```

下载 x86_64 静态版本：

```bash
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/v1.13.2/sherpa-onnx-v1.13.2-linux-x64-static.tar.bz2
```

解压：

```bash
tar xvf sherpa-onnx-v1.13.2-linux-x64-static.tar.bz2
```

查看程序：

```bash
find . -name "sherpa-onnx*"
```

常用程序包括：

```text
sherpa-onnx
sherpa-onnx-microphone
sherpa-onnx-alsa
```

---

## 7. 下载推荐中文模型

下载更高识别率的中文实时模型：

```bash
cd ~/asr

wget https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30.tar.bz2
```

解压：

```bash
tar xvf sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30.tar.bz2
```

检查文件：

```bash
ls -lh sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30
```

应看到类似文件：

```text
encoder.int8.onnx
decoder.onnx
joiner.int8.onnx
tokens.txt
test_wavs
```

---

## 8. 先跑官方测试音频

先不要急着测试麦克风，先确认模型能正常推理：

```bash
cd ~/asr

./sherpa-onnx-v1.13.2-linux-x64-static/bin/sherpa-onnx \
  --tokens=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/tokens.txt \
  --encoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/encoder.int8.onnx \
  --decoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/decoder.onnx \
  --joiner=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/joiner.int8.onnx \
  ./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/test_wavs/0.wav
```

如果能输出中文，说明：

```text
程序正常
模型正常
推理链路正常
```

---

## 9. VirtualBox 麦克风设置

VirtualBox 声音设置建议：

```text
启用音频：勾选
主机音频驱动：Windows DirectSound
控制芯片：Intel HD 音频
启用声音输出：勾选
启用声音输入：勾选
```

注意：VirtualBox 中通常不能直接选择具体麦克风。它默认使用 Windows 当前默认输入设备。

因此需要在 Windows 中设置默认麦克风：

```text
设置 → 系统 → 声音 → 输入
```

并确认：

```text
麦克风访问权限：开启
允许桌面应用访问麦克风：开启
```

---

## 10. Ubuntu 中测试麦克风

查看录音设备：

```bash
arecord -l
```

查看 PulseAudio / PipeWire 输入源：

```bash
pactl list sources short
```

如果看到类似：

```text
alsa_input.pci-0000_00_05.0.analog-stereo
```

说明 Ubuntu 已经识别到输入源。

设置默认输入源：

```bash
pactl set-default-source alsa_input.pci-0000_00_05.0.analog-stereo
pactl set-source-mute alsa_input.pci-0000_00_05.0.analog-stereo 0
pactl set-source-volume alsa_input.pci-0000_00_05.0.analog-stereo 150%
```

录音测试：

```bash
arecord -D pulse -f S16_LE -r 16000 -c 1 -d 8 -vv /tmp/test.wav
```

录音时观察音量条是否跳动。如果一直是 `00%`，说明虚拟机还没有真正拿到麦克风声音。

---

## 11. 识别自己的录音

录音时可以说：

```text
打开主屏
关闭灯光
升降杆上升
切换四分屏
```

然后识别：

```bash
cd ~/asr

./sherpa-onnx-v1.13.2-linux-x64-static/bin/sherpa-onnx \
  --tokens=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/tokens.txt \
  --encoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/encoder.int8.onnx \
  --decoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/decoder.onnx \
  --joiner=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/joiner.int8.onnx \
  /tmp/test.wav
```

如果能输出中文，说明：

```text
麦克风正常
录音正常
模型识别正常
```

---

## 12. 实时麦克风识别

### 方式一：使用默认麦克风

```bash
cd ~/asr

./sherpa-onnx-v1.13.2-linux-x64-static/bin/sherpa-onnx-microphone \
  --tokens=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/tokens.txt \
  --encoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/encoder.int8.onnx \
  --decoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/decoder.onnx \
  --joiner=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/joiner.int8.onnx \
  --num-threads=2
```

说一句后停顿 1 到 2 秒：

```text
打开主屏
```

### 方式二：使用 ALSA 指定麦克风设备

如果 `sherpa-onnx-microphone` 不出结果，但文件识别正常，通常是默认输入源没有选对。此时建议使用 `sherpa-onnx-alsa` 指定设备。

```bash
cd ~/asr

./sherpa-onnx-v1.13.2-linux-x64-static/bin/sherpa-onnx-alsa \
  --tokens=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/tokens.txt \
  --encoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/encoder.int8.onnx \
  --decoder=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/decoder.onnx \
  --joiner=./sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/joiner.int8.onnx \
  --num-threads=2 \
  plughw:0,1
```

如果设备不是 `plughw:0,1`，用下面命令确认：

```bash
arecord -l
```

---

## 13. 识别率优化建议

### 13.1 保证音频质量

录音音量不要太小，也不要爆音。理想状态是说话时音量条明显跳动，但不要长期顶到 100%。

建议：

```text
距离麦克风 20cm 以内
环境尽量安静
一句话说完整
说完后停顿 1 到 2 秒
```

### 13.2 不要只看虚拟机识别率

VirtualBox 中音频链路较长：

```text
Windows 麦克风
→ VirtualBox
→ Ubuntu
→ PulseAudio / PipeWire
→ ALSA
→ sherpa-onnx
```

中间可能出现重采样、增益、噪声和延迟。因此虚拟机只适合验证部署流程，最终准确率应在 RK3588 真机 + USB 麦克风上测试。

### 13.3 加业务层纠错

对于车载控制 / 设备控制场景，建议在识别结果后加一层纠错词典：

```json
{
  "生降杆": "升降杆",
  "升降竿": "升降杆",
  "四屏": "四分屏",
  "是分屏": "四分屏",
  "路播": "录播",
  "主萍": "主屏",
  "朱屏": "主屏",
  "关闭登光": "关闭灯光",
  "摄像机左转": "摄像头左转"
}
```

### 13.4 使用命令候选匹配

设备控制不是自由听写，应该限制命令集合，例如：

```text
打开主屏
关闭主屏
打开辅屏
切换四分屏
切换全屏
打开灯光
关闭灯光
升降杆上升
升降杆下降
升降杆停止
摄像头左转
摄像头右转
开始录播
停止录播
```

识别结果出来后，再做编辑距离、拼音相似度或关键词匹配。这样即使模型识别有少量错误，也可以纠正到正确指令。

---

## 14. RK3588 部署建议

RK3588 上建议不要把模型放到根目录 `/`，因为很多系统根目录很小。

查看磁盘：

```bash
df -h
```

查看内存：

```bash
free -h
```

如果看到类似：

```text
Mem: 3.8Gi
available: 2.9Gi
/userdata: 4.7G available
```

说明这是 4GB 内存版本，适合部署普通 `zh-int8` 模型。

建议部署目录：

```bash
mkdir -p /userdata/asr
cd /userdata/asr
```

推荐目录结构：

```text
/userdata/asr/
├── bin/
│   └── sherpa-onnx-alsa
├── models/
│   └── sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30/
│       ├── encoder.int8.onnx
│       ├── decoder.onnx
│       ├── joiner.int8.onnx
│       └── tokens.txt
└── start_asr.sh
```

不要放到：

```text
/root
/opt
/usr/local
/tmp
```

尤其不要放 `/tmp`，因为 `/tmp` 可能是 tmpfs，既占内存，也可能重启丢失。

---

## 15. RK3588 内存评估

对于 `sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30`：

```text
模型本体：约 160MB
运行内存预估：400MB ~ 800MB
加服务 / VAD 后：500MB ~ 1GB
推荐设备：RK3588 4GB 起步，8GB 更稳
```

建议：

```text
2GB 内存：偏紧
4GB 内存：可以部署
8GB 内存：比较稳
```

不建议 4GB 设备一开始上 xlarge 模型。

---

## 16. RK3588 启动脚本示例

创建脚本：

```bash
nano /userdata/asr/start_asr.sh
```

写入：

```bash
#!/bin/sh

BASE_DIR=/userdata/asr
MODEL_DIR=$BASE_DIR/models/sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30
DEVICE=plughw:0,0

$BASE_DIR/bin/sherpa-onnx-alsa \
  --tokens=$MODEL_DIR/tokens.txt \
  --encoder=$MODEL_DIR/encoder.int8.onnx \
  --decoder=$MODEL_DIR/decoder.onnx \
  --joiner=$MODEL_DIR/joiner.int8.onnx \
  --num-threads=2 \
  $DEVICE
```

授权：

```bash
chmod +x /userdata/asr/start_asr.sh
```

运行：

```bash
/userdata/asr/start_asr.sh
```

如果麦克风不是 `plughw:0,0`，用下面命令确认：

```bash
arecord -l
```

然后修改脚本中的：

```bash
DEVICE=plughw:0,0
```

---

## 17. 查看运行内存

运行 ASR 后，另开终端：

```bash
ps -o pid,comm,rss,vsz,%mem -C sherpa-onnx-alsa
```

或者：

```bash
top -p $(pidof sherpa-onnx-alsa)
```

其中 `RSS` 是实际物理内存占用，单位是 KB。

例如：

```text
RSS = 620000
```

大约就是：

```text
620MB
```

---

## 18. 常见问题

### 18.1 `libasound2` 没有安装候选项

Ubuntu 24.04 使用：

```bash
sudo apt install -y libasound2t64
```

### 18.2 `pactl` 拒绝连接

不要用 root 用户执行 `pactl`。PulseAudio / PipeWire 通常跟随桌面普通用户运行。

先退出 root：

```bash
exit
```

确认当前用户：

```bash
whoami
```

应该是普通用户，例如：

```text
yi
```

### 18.3 录音文件有大小，但识别为空

先确认录音时音量条是否跳动：

```bash
arecord -D pulse -f S16_LE -r 16000 -c 1 -d 8 -vv /tmp/test.wav
```

如果一直是 `00%`，说明没有真正录到声音。

### 18.4 实时识别启动了但没有输出

说完一句话后需要停顿 1 到 2 秒。

如果仍无输出，优先改用 `sherpa-onnx-alsa` 指定设备：

```bash
arecord -l
```

找到麦克风设备后，用：

```bash
plughw:card,device
```

---

## 19. 总结

如果目标是 RK3588 上做中文设备控制语音识别，推荐路线是：

```text
虚拟机验证流程
→ RK3588 真机部署
→ USB 麦克风测试
→ streaming-zipformer-zh-int8 模型
→ 业务层纠错
→ 命令候选匹配
→ 封装成本地 ASR 服务
```

最终推荐模型：

```text
sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30
```

一句话总结：

```text
small CTC 适合低内存验证；
streaming-zipformer-zh-int8 适合正式部署；
xlarge 和 SenseVoice 适合准确率优先但内存更充足的场景。
```
