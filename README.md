# emoeem 的 Linux 字体配置 (fontconfig)

一套以 **MiSans** 为核心的 Linux fontconfig 配置：全局默认字体、**按语言自动切换中日韩（CJK）地区字形**、常见网页字体映射，以及适合高分屏的渲染参数。

在 CachyOS / Arch Linux + Wayland（niri）+ fontconfig 2.18 下验证，任何使用 fontconfig 的发行版均可参考。

## 效果一览

| 请求 | 结果 |
|---|---|
| `sans-serif`（默认界面/网页） | **MiSans** |
| `serif`（衬线） | **霞鹜臻楷 GB**（LXGW ZhenKai GB）→ 霞鹜文楷 Screen |
| `monospace`（等宽/终端） | **Maple Mono Normal NF CN**（自带 Nerd Font 图标 + 中文） |
| 日文内容（`lang=ja`） | Noto Sans CJK **JP** / Noto Serif CJK JP |
| 韩文内容（`lang=ko`） | Noto Sans CJK **KR** / Noto Serif CJK KR |
| 台湾繁中（`lang=zh-tw`） | Noto Sans CJK **TC** / LXGW WenKai TC |
| 香港繁中（`lang=zh-hk`） | Noto Sans CJK **HK** |
| `Arial`、`Segoe UI`、`Liberation Sans` 等网页字体 | 映射到上面的 sans-serif 偏好链（即 MiSans） |
| Emoji | Noto Color Emoji |

日文网页的「直、门、关」等汉字会正确显示日文字形，繁中网页显示繁体字形——这就是按语言切换地区字形的意义。

## 快速使用

### 第 1 步：安装字体

Arch / CachyOS：

```bash
# 官方仓库
sudo pacman -S --needed otf-misans otf-noto-sans-cjk noto-fonts-emoji

# AUR
yay -S --needed ttf-lxgw-zhenkai ttf-lxgw-wenkai-screen ttf-maplemononormal-nf-cn
```

其他发行版可以从这些字体的 GitHub Releases 手动下载，放到 `~/.local/share/fonts/` 后执行 `fc-cache -f`：

| 字体 | 用途 |
|---|---|
| [MiSans](https://hyperos.mi.com/font) | 默认无衬线 |
| [Noto Sans/Serif CJK](https://github.com/notofonts/noto-cjk/releases) | 各地区 CJK 字形 |
| [LXGW ZhenKai GB（霞鹜臻楷）](https://github.com/lxgw/LxgwZhenKaiGB/releases) | 衬线首选 |
| [LXGW WenKai（霞鹜文楷）](https://github.com/lxgw/LxgwWenkai/releases) | 衬线回退 / 繁中楷体 |
| [Maple Mono NF CN](https://github.com/subframe7536/maple-font/releases) | 等宽 |
| Noto Color Emoji | Emoji |

### 第 2 步：应用配置

```bash
git clone https://github.com/emoeem/fontconfig.git
cd fontconfig

# 备份已有配置
mv ~/.config/fontconfig/fonts.conf ~/.config/fontconfig/fonts.conf.bak 2>/dev/null

# 复制（或用软链接，方便以后 git pull 同步更新）
cp fonts.conf ~/.config/fontconfig/fonts.conf
# ln -sf "$(pwd)/fonts.conf" ~/.config/fontconfig/fonts.conf

fc-cache -f
```

然后**重启浏览器和正在运行的应用**（fontconfig 的新规则对已运行的程序不生效）。

> 单文件配置，不需要 `conf.d/`。fontconfig 通过 `50-user.conf` 自动读取 `~/.config/fontconfig/fonts.conf`。

### 第 3 步：验证

```bash
fc-match sans-serif              # 期望: MiSans
fc-match serif                   # 期望: LXGW ZhenKai GB
fc-match monospace               # 期望: Maple Mono Normal NF CN

# 按语言切换（核心功能）
fc-match 'sans-serif:lang=ja'    # 期望: Noto Sans CJK JP
fc-match 'sans-serif:lang=ko'    # 期望: Noto Sans CJK KR
fc-match 'sans-serif:lang=zh-tw' # 期望: Noto Sans CJK TC
fc-match 'sans-serif:lang=zh-cn' # 期望: MiSans

# 网页字体映射
fc-match Arial                   # 期望: MiSans
fc-match 'Segoe UI'              # 期望: MiSans

# Emoji
fc-match emoji                   # 期望: Noto Color Emoji
```

打开一个日文网站（如 <https://ja.wikipedia.org>）和一个繁中网站对比汉字字形，效果一目了然。

## 配置是怎么工作的

- **通用族偏好**：用 `match + prepend`（而不是 `alias/prefer`）把 MiSans / 霞鹜臻楷 / Maple Mono 放到 sans-serif / serif / monospace 的最前面，压过发行版自带的偏好列表。
- **按语言切换地区字形**：通过 `<test name="lang" compare="contains">` 匹配语言。浏览器按页面 `<html lang>`、GTK/Qt 按 `LC_*` 环境变量传入 lang；没有 lang 的请求不受影响，保持简中默认。
- **网页字体映射**：把 `Arial`、`Segoe UI`、`Liberation Sans/Mono` 等 `assign` 到通用族，让 Windows 风格的网页字体栈统一走 MiSans。
- **渲染参数**：hinting=slight + 亚像素（rgb）抗锯齿，适合 4K 以下、整数缩放的屏幕。
- **屏蔽 Nimbus Sans**：这个字体会导致 GitHub 部分页面排版异常，直接 `rejectfont`。

### ⚠️ 关键机制：规则顺序就是优先级

fontconfig **按规则执行顺序排列 prepend 的值：先执行的规则，其字体排得越靠前**（和「prepend 插到最前」的直觉相反）。所以本配置的顺序是：

```
网页字体映射 → 语言地区规则 → 默认字体规则
```

语言规则在前，日文/繁中字体才能排到 MiSans 前面；默认规则在后，MiSans 作为回退仍排在发行版偏好之前。修改配置时不要打乱这个顺序，改完可以用下面的命令直观检查最终的 family 列表：

```bash
fc-pattern --config 'sans-serif:lang=ja' | grep -m1 family
```

## 自定义

- **换默认无衬线字体**：把默认规则和映射规则里的 `MiSans` 改成你喜欢的字体（如 `HarmonyOS Sans SC`、`Sarasa Gothic SC`）。
- **想要标准字重（MiSans Regular）**：MiSans 官方 OTF 的字重元数据有误（Regular 被标为 53，正常应为 80），默认请求会匹配到 Medium（偏粗）。本配置保持原样；如果你想要标准 Regular，加入下面的扫描期规则并执行 `fc-cache -f`：

  ```xml
  <match target="scan">
    <test name="file" compare="contains">
      <string>MiSans-Regular</string>
    </test>
    <edit name="weight" mode="assign">
      <const>regular</const>
    </edit>
  </match>
  ```

- **加一种语言**：照抄一段 lang 规则，改语言代码和字体即可，注意放在默认规则**之前**。
- **衬线想用宋体风格**：把默认衬线规则里的 `Noto Serif CJK SC` 放到第一位。

## 注意事项

- **全局生效**：fontconfig 对所有应用生效。Arial → sans-serif 的映射会让依赖 Arial 度量的文档（如 LibreOffice 排版）产生偏移，介意的话删掉对应的映射规则。
- **分数缩放**：Wayland 下如果使用分数缩放（如 125%、150%），`rgba=rgb` 亚像素抗锯齿可能出现彩色边缘，把渲染设置里的 `rgba` 改为 `none`（灰度）即可。
- **HiDPI**：4K 以上屏幕通常建议 hinting 关闭或保持 hintslight + 灰度。
