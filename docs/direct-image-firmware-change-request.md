# 固件支持 Direct Image URL 修改说明

## 目标

让 ESP32 固件支持一个可选配置 `direct_image_url`。当该配置存在时，设备刷新图片时不再请求后端 `/api/render`，而是直接从该 URL 下载图片并显示；为空时保持现有后端逻辑不变。

## 需要改的模块

- `firmware/src/storage.h`
- `firmware/src/storage.cpp`
- `firmware/src/portal.cpp`
- `firmware/data/portal_html.h`
- `firmware/src/network.cpp`
- `firmware/src/main.cpp`
- `firmware/platformio.ini`

## 配置存储

新增运行时变量：

```cpp
extern String cfgDirectImageUrl;
```

在 `storage.cpp` 中：

- 定义 `String cfgDirectImageUrl;`
- `loadConfig()` 从 NVS 读取 key：`direct_img_url`
- 默认值为空字符串
- URL 长度建议限制为 300
- 新增保存函数：

```cpp
void saveDirectImageUrl(const String &url);
```

保存时：

- 非空：`prefs.putString("direct_img_url", url)`
- 空：`prefs.remove("direct_img_url")`

## Portal 配网页面

在配网页面新增输入框：

```text
直接图片 URL（可选）
```

表单提交 `/save_wifi` 时追加字段：

```text
direct_image_url
```

在 `portal.cpp` 的 `/save_wifi` 处理里：

- 读取 `webServer.arg("direct_image_url")`
- sanitize，最大长度 300
- 非空时校验必须以 `http://` 或 `https://` 开头
- 调用 `saveDirectImageUrl(directImageUrl)`
- 如果提交为空且已有旧值，则清空 `direct_img_url`

`/info` 接口也返回：

```json
{
  "direct_image_url": "..."
}
```

这样再次进入 Portal 时能回填。

## 启动条件

原逻辑可能要求 `cfgServer` 非空。需要改成：

```cpp
if (cfgServer.length() == 0 && cfgDirectImageUrl.length() == 0) {
    enterPortalMode();
}
```

也就是说：有 Wi-Fi 且有 `direct_image_url`，即使没有后端 server，也允许正常运行。

## 联网 Token 逻辑

`connectWiFi()` 里，如果 `cfgDirectImageUrl` 非空，可以跳过后端 token 初始化：

```cpp
if (cfgDirectImageUrl.length() > 0) {
    return true;
}
```

`ensureDeviceToken()` 也要避免 `cfgServer` 空时拼后端 URL：

```cpp
if (cfgServer.length() == 0) return false;
```

## 图片下载逻辑

在 `network.cpp` 的 `fetchBMP()` 开头加优先分支：

```cpp
if (cfgDirectImageUrl.length() > 0) {
    return fetchDirectImageUrl(cfgDirectImageUrl);
}
```

新增 `fetchDirectImageUrl(const String &url)`：

- 支持 HTTP/HTTPS
- `http.setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS)`
- `Accept-Encoding: identity`
- 读取 HTTP 200 响应
- 如果 `EPD_BPP >= 2` 且 `Content-Length == COLOR_BUF_LEN`，按 raw 2bpp 读取到 `colorBuf`
- 否则按 BMP 读取到 `imgBuf`
- BMP 逻辑可复用现有 `/api/render` 的 BMP 解码逻辑
- 不需要加 `X-Device-Token`

## 三色屏注意

如果屏幕是中景园 4.2 寸黑白红，要使用三色环境，不要用黑白环境。

### 已验证可用配置

当前已在中景园 V1.4 4.2 寸黑白红屏 + ESP32-C3 std 板上验证可正常显示：

```bash
epd_42_zhongjingyuan_bwr_gxepd2_gdey042z98_c3_std
```

对应关键配置：

```ini
-DEPD_PANEL_42_GXEPD2_GDEY042Z98
-DEPD_BPP=2
-DEPD_GXEPD2_SPI_HZ=2000000
```

该屏使用 GxEPD2 的 `GxEPD2_420c_GDEY042Z98` 三色驱动。实测 `GxEPD2_420c` 和 `GxEPD2_420c_Z21` 与这套屏/转接板组合不匹配，可能出现刷新很快结束但屏幕不动。

Direct `.2bpp` 数据是 packed 2bpp，不是两个 1bpp plane。设备端需要解包为 GxEPD2 需要的黑色 plane 和红色 plane：

```text
00 = 黑
01 = 红
10 = 红/兼容映射
11 = 白
```

GxEPD2 3C bitmap plane 的位语义是：

```text
1 = 白
0 = 有色
```

因此解码时：

- 黑色 plane 默认 `0xFF`，遇到 `00` 清 bit
- 红色 plane 默认 `0xFF`，遇到 `01` 或 `10` 清 bit
- `11` 保持两个 plane 都为白

新增或使用类似环境：

```ini
[env:epd_42_zhongjingyuan_bwr_wft0cz15_c3_std]
extends = common
board = esp32-c3-devkitm-1
upload_speed = 460800
board_build.flash_mode = dio
board_build.f_flash = 40000000L
build_flags =
    -DBOARD_PROFILE_ESP32_C3
    -DARDUINO_USB_MODE=0
    -DARDUINO_USB_CDC_ON_BOOT=0
    -DEPD_WIDTH=400
    -DEPD_HEIGHT=300
    -DEPD_PANEL_42_WFT
    -DEPD_BPP=2
    -DALLOW_INSECURE_FALLBACK=0
```

三色 raw 2bpp 文件要求：

- 尺寸：`400x300`
- 文件大小：`400 * 300 / 4 = 30000 bytes`
- 每 4 像素 1 byte
- 当前 WFT 三色驱动颜色编码：

```text
00 = 黑
01 = 红
11 = 白
10 = 红/兼容映射，不建议主动使用
```

## 兼容性要求

- `direct_image_url` 为空时，现有 `/api/render`、设备绑定、配置、后端统计逻辑不变。
- 只有配置了 `direct_image_url` 时才绕过后端渲染。
- Portal 应允许用户清空 `direct_image_url`，恢复后端模式。
