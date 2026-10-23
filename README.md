# Waveshare ESP32-S3-RLCD-4.2

本仓库是从 xiaozhi-esp32 精简出的单板固件工程，只保留
Waveshare ESP32-S3-RLCD-4.2 所需的代码、资源、依赖和构建配置。

## 构建

需要 ESP-IDF 5.5.2 或更高版本：

```bash
idf.py set-target esp32s3
idf.py build
```

烧录：

```bash
idf.py -p <PORT> flash monitor
```

也可以直接运行 `idf.py flash`，构建系统会自动把字体、唤醒词和表情资源写入
`assets` 分区。

## 板卡说明

板卡引脚、功能、界面和故障排查请查看
[Waveshare ESP32-S3-RLCD-4.2 说明](main/boards/waveshare-s3-rlcd-4.2/README.md)。
