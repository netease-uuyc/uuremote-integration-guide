# uuremote-integration-guide
把 UU 远程远控能力接入你的 App —— 设备绑定与发起远控的 URL Scheme 协议说明，覆盖全屏与画中画模式，含 iOS / Android 示例代码与完整错误码。

## 概述

本文档描述第三方 App 与 UU 远程客户端之间的跨 App 交互协议。第三方 App 可以运行在 iOS 或 Android：iOS 通过 URL Scheme 拉起 UU 远程，Android 通过 Deeplink / Intent 拉起 UU 远程。

本文中的平台范围需要区分：

- **调用端平台**：第三方 App 支持 iOS 和 Android，调用方式分别见对应平台示例。
- **目标设备平台**：设备绑定和发起远控目前仅支持 Windows 与 macOS 被控设备。Android、iOS、TV 等其他平台设备不会出现在绑定选择列表中，也不属于本协议支持的远控目标。

协议提供两个核心能力：

1. **设备绑定**：第三方 App 拉起 UU 远程，用户从当前账号下的 Windows/macOS 设备中选择一台进行绑定，UU 远程返回加密后的设备标识。
2. **发起远控**：第三方 App 使用已绑定的设备标识发起远控连接，支持全屏和画中画小窗两种模式。

**UU 远程 URL Scheme**：`uuremote`

---

## 1. 安装检测

### 1.1 iOS 前置配置

iOS 第三方 App 需在 `Info.plist` 中添加 `LSApplicationQueriesSchemes` 白名单，否则 `canOpenURL` 将始终返回 `false`。

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>uuremote</string>
</array>
```

### 1.2 iOS 检测方法

```swift
func isUURemoteInstalled() -> Bool {
    guard let url = URL(string: "uuremote://external") else { return false }
    return UIApplication.shared.canOpenURL(url)
}
```

### 1.3 iOS 未安装处理

若检测结果为 `false`，应停止发起绑定或远控请求，并引导用户前往 App Store 安装 UU 远程。

### 1.4 Android 前置配置

Android 11（API 30）及以上的第三方 App 需在 `AndroidManifest.xml` 中声明 `<queries>`，否则应用可见性限制可能导致安装检测失败。

```xml
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="uuremote" />
    </intent>
</queries>
```

### 1.5 Android 检测方法

```kotlin
fun isUURemoteInstalled(context: Context): Boolean {
    val intent = Intent(
        Intent.ACTION_VIEW,
        Uri.parse("uuremote://external")
    )
    return intent.resolveActivity(context.packageManager) != null
}
```

UU 远程 Android 正式包名为 `com.netease.uuremote`。包名可用于安装引导或应用市场跳转，不建议仅通过包名查询代替 Deeplink 可用性检测。

### 1.6 Android 未安装处理

若 Intent 无法解析，应停止发起绑定或远控请求，并按接入方既有分发渠道引导用户安装 UU 远程。

---

## 2. 设备绑定

### 2.1 触发方式

第三方 App 通过以下 URL Scheme 拉起 UU 远程的设备绑定流程：

```text
uuremote://external/device/authorize?callback=<回调URL>&nonce=<随机ID>
```

### 2.2 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `callback` | String | 是 | 第三方 App 的回调 URL（含 scheme 和 host）。UU 远程完成操作后通过此 URL 回跳。参数值需进行 URL 编码。 |
| `nonce` | String | 否 | 用于匹配请求和回调的随机唯一标识符，建议使用 UUID。传入后，UU 远程会在回调中原样返回。 |

### 2.3 iOS 调用示例

```swift
let callback = "myapp://bindresult"
let nonce = UUID().uuidString
let encodedCallback = callback.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? callback
let urlString = "uuremote://external/device/authorize?callback=\(encodedCallback)&nonce=\(nonce)"

if let url = URL(string: urlString) {
    UIApplication.shared.open(url)
}
```

### 2.4 UU 远程内部流程

```text
拉起 UU 远程
    |
    +-- 未登录或首页未就绪 -> 排队等待，最长 180 秒
    |       +-- 期间被新的绑定请求取代，或同批只保留最新一条 -> 回调 error=superseded
    |       +-- 排队期间用户退出登录 -> 回调 error=logout
    |       +-- 窗口内仍未完成登录 -> 回调 error=timeout
    |
    +-- 登录态与首页条件满足 -> 进入绑定流程
    |
    +-- 首页设备列表加载中 -> 等待加载完成，最长 15 秒 -> 超时回调 error=timeout
    |
    +-- 过滤设备平台，仅保留 Windows/macOS 设备
    |
    +-- 过滤后设备列表为空 -> 展示“当前无可关联设备”提示 -> 用户点击确认后回调 error=no_device
    |
    +-- 过滤后设备列表非空 -> 弹出设备选择页面
        +-- 用户选择设备并确认 -> 回调 anonymous_device_id
        +-- 用户取消（关闭按钮 / 下拉关闭 / 页面被关闭）-> 回调 error=user_cancel
    |
    +-- 页面栈缺失或无可用展示宿主 -> 回调 error=internal_error
```

Android、iOS、TV 等非 Windows/macOS 设备不展示在绑定列表中，不能通过本协议生成 `anonymous_device_id`。

### 2.5 回调参数

UU 远程通过打开第三方 App 的 `callback` URL 返回结果。

#### 绑定成功

```text
<callback>?anonymous_device_id=<加密设备ID>&nonce=<原始nonce>
```

| 参数 | 说明 |
|------|------|
| `anonymous_device_id` | 加密后的 Windows/macOS 设备标识。第三方 App 应持久化存储此值，后续发起远控时使用。 |
| `nonce` | 请求中传入 `nonce` 时原样返回；未传入时回传空值（字段仍会出现）。 |

#### 绑定失败

```text
<callback>?error=<错误码>&nonce=<原始nonce>
```

| `error` 值 | 说明 |
|-------------|------|
| `no_device` | 当前 UU 账号下没有可关联的 Windows/macOS 设备 |
| `user_cancel` | 用户主动取消，包括点击关闭按钮或下拉关闭 |
| `timeout` | 排队等待登录（180 秒）或设备列表加载（15 秒）超时 |
| `not_logined` | 排队被触发时已不满足登录态或首页条件，无法继续 |
| `superseded` | 该绑定请求被后发的同类请求取代，或同批请求中未被选中 |
| `logout` | 排队期间用户退出登录，绑定请求被清理 |
| `internal_error` | UU 远程内部异常，无法继续处理（页面栈缺失、无可用展示宿主等），不再细分内部原因 |

绑定成功与失败互斥：成功返回 `anonymous_device_id`，失败返回 `error`。`internal_error` 为兜底错误码，UU 远程不向第三方暴露具体内部原因。

### 2.6 iOS 回调接收示例

第三方 App 需注册自己的 URL Scheme，并在 `AppDelegate` 或 `SceneDelegate` 中处理回调：

```swift
func application(
    _ app: UIApplication,
    open url: URL,
    options: [UIApplication.OpenURLOptionsKey: Any] = [:]
) -> Bool {
    guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else {
        return false
    }
    let params = Dictionary(uniqueKeysWithValues:
        (components.queryItems ?? []).compactMap { item in
            item.value.map { (item.name, $0) }
        }
    )

    if url.host == "bindresult" {
        if let deviceId = params["anonymous_device_id"] {
            UserDefaults.standard.set(deviceId, forKey: "bound_device_id")
        } else if let error = params["error"] {
            print("绑定失败: \(error)")
        }
        return true
    }
    return false
}
```

### 2.7 Android 调用与回调接收

Android 第三方 App 使用 Intent 拉起 UU 远程。这里的 Android 指第三方调用端平台，不表示支持绑定 Android 被控设备。

```kotlin
val callback = "myapp://bindresult"
val nonce = UUID.randomUUID().toString()
val url = "uuremote://external/device/authorize" +
        "?callback=${URLEncoder.encode(callback, "UTF-8")}" +
        "&nonce=$nonce"
startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(url)))
```

第三方 App 需注册自己的 scheme，并在 Activity 中读取 `intent.data` 的 `anonymous_device_id`、`nonce` 或 `error` 参数。

---

## 3. 发起远控

### 3.1 触发方式

第三方 App 通过以下 URL Scheme 发起远控连接：

```text
uuremote://external/device/control?anonymous_device_id=<加密设备ID>&window_type=<窗口模式>[&callback=<回调URL>]
```

本接口仅支持通过设备绑定流程获得的 Windows/macOS 设备标识。

### 3.2 请求参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `anonymous_device_id` | String | 是 | 绑定流程返回的加密设备标识。参数值需进行 URL 编码。 |
| `window_type` | Int | 否 | 窗口模式：`0` = 全屏，`1` = 画中画小窗；默认值为 `0`。 |
| `callback` | String | 否 | 第三方 App 的回调 URL。仅小窗模式且传入该参数时，UU 远程才会通过该 URL 回调最终结果；不传不影响发起远控。全屏模式不回调第三方 App。参数值需进行 URL 编码。 |

### 3.3 窗口模式说明

#### 全屏模式（`window_type=0`）

- UU 远程以全屏方式展示远控画面。
- 远端首帧成功显示后，本次请求判定为成功。
- 无论远控成功或失败，全屏模式都不通过 `callback` 回调第三方 App。

#### 小窗模式（`window_type=1`）

- UU 远程在连接成功后尝试进入画中画（PiP）小窗模式。
- 只有 PiP 实际启动成功，本次请求才判定为成功；若传入 `callback`，UU 远程回调 `status=success`。
- PiP 不可用或启动失败时不降级为成功；若传入 `callback`，UU 远程回调 `status=failure` 及对应 `reason`。
- iOS 客户端使用小窗模式要求 iOS 18.0 或更高版本，且当前设备支持画中画；不满足要求时返回 `pip_unsupported`。
- 用户可点击小窗恢复到 UU 远程全屏查看。

### 3.4 调用示例

以下示例演示需要接收小窗远控结果时的调用方式；不需要回调时，可省略 `callback` 参数。

#### iOS

```swift
let deviceId = UserDefaults.standard.string(forKey: "bound_device_id") ?? ""
let callback = "myapp://controlresult"
let windowType = 1

let encodedDeviceId = deviceId.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? deviceId
let encodedCallback = callback.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? callback
let urlString = "uuremote://external/device/control?anonymous_device_id=\(encodedDeviceId)&window_type=\(windowType)&callback=\(encodedCallback)"

if let url = URL(string: urlString) {
    UIApplication.shared.open(url)
}
```

#### Android

Android 第三方 App 使用 Intent 发起请求。这里的 Android 指第三方调用端平台，目标设备仍仅支持 Windows/macOS。

```kotlin
val deviceId = URLEncoder.encode(savedDeviceId, "UTF-8")
val callback = URLEncoder.encode("myapp://controlresult", "UTF-8")
val url = "uuremote://external/device/control" +
        "?anonymous_device_id=$deviceId&window_type=1&callback=$callback"
startActivity(Intent(Intent.ACTION_VIEW, Uri.parse(url)))
```

### 3.5 回调参数

`callback` 为可选参数。仅小窗模式且请求中传入有效的 `callback` 时，UU 远程才会回调第三方 App；未传入 `callback` 时不回调，也不影响远控流程。全屏模式无论是否传入 `callback`、远控成功或失败都不回调。小窗模式发生回调时，UU 远程在第三方传入的 `callback` URL 上追加结果参数。若 `callback` 已包含查询参数，UU 远程会保留原参数，并追加或覆盖同名的结果字段；成功回调时会同时移除 `callback` 中原有的 `reason`，避免第三方读到上一次的失败原因。

#### 小窗模式连接成功

```text
<callback>?status=success
```

#### 小窗模式连接失败

```text
<callback>?status=failure&reason=<失败原因>
```

| 参数 | 说明 |
|------|------|
| `status` | 小窗远控的最终状态：`success` 或 `failure`。 |
| `reason` | 失败原因。仅在 `status=failure` 时返回；成功时不返回。 |

#### `reason` 取值

| `reason` | 含义 | 第三方 App 建议处理 |
|----------|------|--------------------|
| `busy` | UU 远程正在远控其他设备，或当前处于不允许发起远控的互斥业务流程 | 提示用户先在 UU 远程结束当前远控或互斥业务后重试 |
| `device_offline` | 目标设备存在，但当前离线 | 提示用户开启或唤醒目标设备后重试 |
| `device_not_found` | 设备不存在、已解绑，或切换 UU 账号导致 `anonymous_device_id` 失效 | 清除旧设备标识并引导用户重新绑定 |
| `pip_unsupported` | 请求小窗，但当前系统、设备或运行状态不支持 PiP | 提示无法使用小窗，可改用全屏模式重试 |
| `login_timeout` | 用户未在规定时间内完成 UU 远程登录，或首页初始化超时 | 结束等待并允许用户重新发起 |
| `network_unavailable` | 本机无可用网络，或请求明确因断网、DNS、网络不可达而失败 | 提示用户检查本机网络后重试 |
| `not_allowed` | 设备未开启允许被控，或目标设备平台不受支持 | 引导用户检查被控端设置和设备平台 |
| `connect_timeout` | 建连、等待首帧或启动 PiP 超时 | 提示用户重试 |
| `connect_failed` | 无法进一步区分的连接失败 | 展示通用连接失败提示并允许重试 |
| `user_cancel` | 用户取消登录或主动终止本次远控流程 | 结束等待 |
| `invalid_request` | 必填参数缺失或参数值不合法 | 检查请求参数 |
| `internal_error` | UU 远程内部状态异常，无法继续处理请求 | 展示通用失败提示并允许重试 |

### 3.6 回调约定

- 回调仅用于传入有效 `callback` 的小窗模式；`status=success` 与 `status=failure` 互斥，成功时不返回 `reason`，失败时必须返回非空的 `reason`。
- 不再使用旧版 `success=0/1` 字段。
- 全屏模式无论远控成功、失败、用户取消或超时都不回调第三方 App。
- 小窗模式传入有效 `callback` 时，在成功、失败、用户取消或超时后最终且只回调一次；未传入时不回调。
- 小窗模式传入有效 `callback` 时，前置拦截类失败（设备不允许被控、设备不存在、设备离线、当前忙碌）会先在 UU 远程内弹窗提示用户，**用户点击确认后才回调**对应的 `reason`；该路径不启用本地兜底超时，第三方 App 需按用户确认时机处理，不应据此判定回调丢失。若提示弹窗无法展示，UU 远程立即以对应 `reason` 回调。
- 小窗模式传入有效 `callback` 时，无需用户确认的失败（参数非法、PiP 启动失败等）会立即回调。
- 全屏远控以远端首帧成功显示为成功，但不回调结果；小窗远控以 PiP 实际启动成功为成功，并在传入有效 `callback` 时回调 `status=success`。
- 第三方 App 收到未知 `reason` 时，应按 `connect_failed` 处理，不应忽略回调。
- 第三方 App 不应在全屏模式或未传入 `callback` 时等待回调。传入有效 `callback` 的小窗模式可保留本地总超时作为异常容灾，但不应依赖它代替正常失败回调；前置拦截路径的等待时长取决于用户在 UU 远程内的确认操作，本地超时应留出足够余量。

### 3.7 iOS 回调接收示例

以下示例处理小窗模式传入 `callback` 后收到的成功和失败回调。全屏模式不会进入该回调处理。

```swift
if url.host == "controlresult" {
    switch params["status"] {
    case "success":
        print("远控连接成功")
    case "failure":
        let reason = params["reason"] ?? "connect_failed"
        if reason == "device_not_found" {
            UserDefaults.standard.removeObject(forKey: "bound_device_id")
            // 引导用户重新执行设备绑定
        } else {
            print("远控连接失败: \(reason)")
        }
    default:
        break
    }
}
```

---

## 4. 异常和错误处理

### 4.1 通用异常

`device/control` 的回调结果仅适用于传入有效 `callback` 的小窗模式。全屏模式的异常由 UU 远程内部展示和处理，不回调第三方 App。

| 场景 | 表现 | 建议处理 |
|------|------|----------|
| iOS 未安装 UU 远程 | `canOpenURL` 返回 `false` | 引导用户前往 App Store 安装 |
| Android 未安装 UU 远程 | Intent 无法解析 | 按接入方既有分发渠道引导用户安装 |
| `device/control` 携带非法 `callback` | 请求参数非法，无法回调 | 如需回调，发起请求前校验并正确编码 `callback`；无需回调时不要传入该参数 |
| `device/authorize` 缺少或携带非法 `callback` | 同上，绑定流程不执行且无法回调 | 发起请求前校验并正确编码 `callback` |
| `device/control` 缺少 `anonymous_device_id` | `status=failure&reason=invalid_request` | 确保必填参数完整 |
| `window_type` 不是 `0` 或 `1` | `status=failure&reason=invalid_request` | 修正窗口模式参数 |

### 4.2 绑定流程异常

| 场景 | 回调 | 说明 |
|------|------|------|
| 未登录或首页未就绪，排队等待超时 | `error=timeout` | 排队窗口 180 秒，结束等待并允许用户重试 |
| 设备列表加载超时 | `error=timeout` | 15 秒窗口内未拿到设备列表 |
| 排队被触发时已不满足登录态或首页条件 | `error=not_logined` | 结束等待，用户需重新发起绑定 |
| 绑定请求被新的同类请求取代，或同批未被选中 | `error=superseded` | 同一时刻仅处理最新一次请求，被取代的请求也会收到一次失败回调 |
| 排队期间用户退出登录 | `error=logout` | 请求随登录态失效被清理 |
| 过滤后没有 Windows/macOS 设备 | `error=no_device` | Android、iOS、TV 等设备不计入可关联设备；须用户点击提示弹窗确认后才回调 |
| 用户主动取消 | `error=user_cancel` | 用户点击关闭按钮、下拉关闭，或绑定页面被关闭 |
| UU 远程内部异常 | `error=internal_error` | 页面栈缺失、无可用展示宿主、设备标识加密失败等统一归入此类，不暴露内部细节 |

### 4.3 远控连接异常

下表中的回调仅适用于传入有效 `callback` 的小窗模式。未传入 `callback` 或全屏模式遇到相同异常时不回调第三方 App。

| 场景 | 回调 | 建议处理 |
|------|------|----------|
| UU 远程正在远控其他设备 | `status=failure&reason=busy` | 引导用户先结束当前远控 |
| 目标设备离线 | `status=failure&reason=device_offline` | 引导用户开启或唤醒设备 |
| 设备标识对应旧 UU 账号，或设备已解绑/不存在 | `status=failure&reason=device_not_found` | 清除旧标识并引导用户重新绑定 |
| 本机网络不可用 | `status=failure&reason=network_unavailable` | 检查网络后重试 |
| 连接或首帧等待超时 | `status=failure&reason=connect_timeout` | 结束等待并允许用户重试 |
| 连接失败但无法进一步分类 | `status=failure&reason=connect_failed` | 展示通用连接失败提示 |
| PiP 不可用 | `status=failure&reason=pip_unsupported` | 改用全屏模式重试 |
| 登录或首页初始化超时 | `status=failure&reason=login_timeout` | 结束等待并允许用户重新发起 |
| 设备不允许被控，或目标平台不受支持 | `status=failure&reason=not_allowed` | 引导用户检查被控端允许被控设置与设备平台；须用户点击提示弹窗确认后才回调 |
| 用户取消登录或主动终止本次远控 | `status=failure&reason=user_cancel` | 结束等待 |
| 必填参数缺失或取值非法 | `status=failure&reason=invalid_request` | 修正请求参数后重试 |
| UU 远程内部状态异常 | `status=failure&reason=internal_error` | 展示通用失败提示并允许重试 |

### 4.4 超时机制

远控超时仅在传入有效 `callback` 的小窗模式下回调第三方 App；未传入 `callback` 或全屏模式不回调。设备绑定流程的超时回调不受此规则影响。

| 超时项 | 时长 | 回调 |
|--------|------|------|
| 登录/首页初始化等待 | 180 秒 | `status=failure&reason=login_timeout` |
| 首页设备列表等待 | 15 秒 | `status=failure&reason=connect_failed` |
| 远控连接与首帧等待（本地兜底，前置拦截弹窗路径除外） | 30 秒 | `status=failure&reason=connect_timeout` |
| Scene 激活或 PiP 启动等待 | 由 UU 远程内部控制 | `connect_timeout`、`pip_unsupported` 或对应连接错误 |
| 绑定排队等待（未登录 / 首页未就绪） | 180 秒 | `error=timeout` |
| 绑定设备列表等待 | 15 秒 | `error=timeout` |
| 绑定「当前无可关联设备」提示弹窗 | 无本地超时 | 用户点击提示后回调 `error=no_device` |
| 远控前置拦截提示弹窗（忙碌、设备不存在/离线、不允许被控） | 无本地超时 | 用户点击确认后回调对应的 `status=failure&reason=busy/device_not_found/device_offline/not_allowed` |

### 4.5 注意事项

1. **目标设备范围**：仅支持绑定和远控 Windows/macOS 设备，不支持 Android、iOS、TV 等其他平台的被控设备。
2. **账号绑定关系**：`anonymous_device_id` 与生成该标识时登录的 UU 账号绑定。切换 UU 账号后，旧标识会返回 `device_not_found`。
3. **回调 URL 编码**：传入 `callback` 时，`callback` 和 `anonymous_device_id` 可能包含 `://`、`+`、`=` 等 URL 特殊字符，必须进行 URL 编码后再拼接。
4. **回调原参数**：`device/control`（以及 `device/authorize`）发生回调时会保留 `callback` 原有查询参数，并追加本次结果；同名的结果字段以本次结果为准，且小窗模式远控成功时会移除原有 `reason`。
5. **重复绑定**：同一第三方 App 可多次调用绑定接口。第三方 App 应保存最新的 `anonymous_device_id`。被后发请求取代的绑定请求会收到 `error=superseded`。
6. **并发请求**：请勿同时发起多个绑定或远控请求。UU 远程同一时刻仅处理一个第三方请求；传入有效 `callback` 的小窗远控请求被替换或取消时会收到一次失败回调，未传入 `callback` 的小窗请求和全屏远控请求不回调；绑定请求被取代或落选时回调 `error=superseded`。
7. **错误码粒度**：`internal_error` 是端内异常的兜底错误码，具体内部原因（加密失败、页面栈缺失、无可用展示宿主等）不会透出给第三方，第三方只需按通用失败处理。

---

## 附录：协议速查

### 绑定协议

```text
-> uuremote://external/device/authorize?callback={url}&nonce={uuid}
<- {callback}?anonymous_device_id={encrypted_id}&nonce={uuid}
<- {callback}?error={error_code}&nonce={uuid}
```

`{error_code}` 取值：`no_device`、`user_cancel`、`timeout`、`not_logined`、`superseded`、`logout`、`internal_error`（详见 2.5）。

绑定目标设备：仅 Windows/macOS。

### 远控协议

```text
-> uuremote://external/device/control?anonymous_device_id={encrypted_id}&window_type={0|1}[&callback={url}]

window_type=0（全屏）：
<- 不回调

window_type=1（小窗，传入 callback 时）：
<- {callback}?status=success
<- {callback}?status=failure&reason={reason}

window_type=1（小窗，未传入 callback 时）：
<- 不回调
```

远控目标设备：仅 Windows/macOS。
