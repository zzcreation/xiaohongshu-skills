# 小红书图文发布自动化技术 Note

更新时间：2026-05-15

## 背景

调试小红书创作服务平台图文发布链路时，发现原流程在以下位置不稳定：

1. 进入发布页后偶尔停在「上传视频」而不是「上传图文」。
2. Windows Chrome 无法直接读取 WSL/Linux 路径上传图片。
3. 图片上传后，预览数量检测在部分扩展权限状态下不可靠。
4. 底部「发布」按钮不是普通 DOM button，普通选择器和 `element.click()` 不稳定。

## 关键发现

### 1. 图文页入口

直接访问带 `target=image` 的发布 URL 比模拟点击顶部 Tab 稳定：

```text
https://creator.xiaohongshu.com/publish/publish?source=official&target=image
```

进入后应使用 DOM 状态校验，而不是截图判断：

- active tab 为「上传图文」
- 存在「上传图片」入口
- 存在图片上传 input
- input `accept` 包含 `.jpg,.jpeg,.png,.webp`

### 2. WSL 到 Windows Chrome 的文件路径

CLI 在 WSL/Linux 中运行，而浏览器是 Windows Chrome 时，Chrome 不能读取：

```text
/home/zzc/...
```

需要转换成 Windows 可访问的 UNC 路径：

```text
\\wsl.localhost\Ubuntu\home\zzc\...
```

否则可能出现上传失败或页面崩溃，例如 `RESULT_CODE_KILLED_BAD_MESSAGE`。

### 3. 上传完成检测

当前实现先使用固定等待兼容上传后 DOM 预览数量不可读的问题：首张图等待 6 秒，后续图等待 3 秒。

这是一个务实 workaround；后续如果扩展权限/页面结构稳定，建议恢复为基于预览数量或上传完成状态的显式检测。

### 4. 发布按钮结构

新版底部发布区是 custom element：

```html
<xhs-publish-btn submit-text="发布" save-text="暂存离开" ...></xhs-publish-btn>
```

该 host 内部没有普通子节点，真实按钮渲染在 closed ShadowRoot 中；但页面对象暴露了私有引用：

```js
const host = document.querySelector('xhs-publish-btn[submit-text="发布"]')
const sr = host._sr
```

ShadowRoot 内真实结构：

```html
<button type="button" class="ce-btn white">暂存离开</button>
<button type="button" class="ce-btn bg-red">发布</button>
```

因此最稳定的点击方式是直接点击 shadow 内红色按钮：

```js
const host = document.querySelector('xhs-publish-btn[submit-text="发布"]')
const btn = host._sr.querySelector('button.bg-red')
btn.click()
```

不要在 `xhs-publish-btn` host 上按比例盲点。host 同时覆盖「暂存离开」和「发布」两个区域，错误点位可能触发暂存并回到上传页。

### 5. 成功验证

点击发布后应显式等待并验证：

- URL 包含 `/publish/success`
- 或页面文本包含 `发布成功`

不要只依赖点击函数是否返回。

## 当前实现要点

- `fill_publish_form()`：直接导航 `target=image`，并调用 `_ensure_image_publish_page()` 做 DOM 校验。
- `BridgePage.set_file_input()`：将 WSL Linux 路径转换为 `\\wsl.localhost\Ubuntu\...` 后传给 Windows Chrome。
- `_wait_for_upload_complete()`：暂用固定等待兼容预览数量检测不稳定。
- `click_publish_button()`：优先点击 `xhs-publish-btn._sr.querySelector('button.bg-red')`，然后验证成功页。

## 已验证

- 连续 3 次稳定进入「上传图文」页。
- 完整发布测试通过：上传图片、填写标题正文、点击 ShadowRoot 内真实发布按钮，2 秒内检测到 `发布成功`。
