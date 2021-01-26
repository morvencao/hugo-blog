---
title: "MacBookPro 开启 HiDPI"
date: 2019-04-12
categories: ['note', 'tech']
draft: false
---

上周旧笔记本（2015-Mid MacBookPro）由于自己的疏忽导致背包内水杯漏水而浸液，导致开机黑屏，基本无法使用。因为是主力机，所以无奈只能硬着头皮换了最新款的 MacBookPro（2018-Mid）。不换不知道，原本以为很快的数据和配置迁移消耗了我一整晚的时间。可能是因为近些年来苹果品控的下降，对于最新 MacBookPro 没多少好感，拿到新本后令人发狂的蝶式键盘加上鸡肋的 TouchBar，让新旧本之间的过渡期再次延长，于是决定还是继续使用外接键盘和鼠标，并将自己之前的显示器作为主显示器。

将外接显示器连接上之后，很快就会发现整体显示模糊，即使4K显示屏也不能达到期望的 Retina 显示效果。网上搜索一番之后才发现原因是新本没有开启 HiDPI。

## 何为 HiDPI

我们知道，高分辨率意味着更小的字体和图标，而 HiDPI 可以用软件的方式实现单位面积内的高密度像素。通过开启 HiDPI 渲染，可以在保证分辨率不变的情况下，使得字体和图标变大。一句话概括就是就是：

```
高 PPI(硬件) + HiDPI 渲染(软件) = 更细腻的显示效果(Retina)
```

## 如何开启 HiDPI

关于如何开启 HiDPI，Google 搜索之后会有很多方案，但是因为系统的不断升级，有的不够全面，有的过于繁琐。在此针对我目前的笔记本（MacBookPro 2018-Mid）给出一个相对简洁的方案。主要包含三个步骤：

- 关闭 SIP
- 终端命令
- 开启 SIP

### 备份

实际的操作的过程会更改部分系统文件，因此在操作前要确保对文件进行备份：

1. 打开终端并进入到 `/System/Library/Displays/Contents/Resources`
2. 拷贝 `Overrides` 文件夹到其他目录一份

### 关闭 SIP

> Note: 关闭 SIP 有风险，确保所有操作完整之后再次打开 SIP，否则对系统文件的保护将不存在

1. 打开终端并输入 `csrutil status`，如果结果为 `enabled` 则表明 SIP 为开启状态
2. 关机之后再按电源键后长按 `command + R` 直至出现苹果 LOGO
3. 在 `Utils->Terminal` 打开终端，并输入 `csrutil disable`，之后关掉终端
4. 重启并正常开机
5. 打开终端并输入 `csrutil status`，确保结果是 `disable`

### 运行设置脚本

> Note: 为了简化配置过程，我使用了 Github 上开源的[自动脚本](https://github.com/syscl/Enable-HiDPI-OSX)来开启 HiDPI，如果想手动来配置，请访问：https://comsysto.github.io/Display-Override-PropertyList-File-Parser-and-Generator-with-HiDPI-Support-For-Scaled-Resolutions/

1. 输入 `curl -o ~/enable-HiDPI.sh https://raw.githubusercontent.com/syscl/Enable-HiDPI-OSX/master/enable-HiDPI.sh`
2. 输入 `chmod +x ~/enable-HiDPI.sh`
3. 输入 `~/enable-HiDPI.sh`
4. 输入你想设置的分辨率比如 `1920x1080`

> Note: 设置分辨率的时候请务必使用 `x`(乘号)，不能使用 `*`(星号)

### 开启 SIP

1. 关机之后再次按电源键开机，并长按 `Command + R` 直至出现苹果 LOGO
2. 在 `Utils->Terminal` 打开终端，并输入 `csrutil enable` 之后关掉终端
3. 重启并正常开机
4. 打开终端并输入 `csrutil status`，确保结果是 `enable`

### 使用 RDM 切换分辨率

[RDM](http://avi.alkalay.net/software/RDM/) 是一个用来切换分辨率的开源软件，使用方法也很简单，其中带⚡️标志的才是开启了 HiDPI 的分辨率。

## Tips

1. 开启 HiDPI 仅仅是改善显示效果并不能达到 MacBook Pro 本身屏幕的那种细腻程度，所以推荐新款的 MacBookPro 搭配使用4K或者5K显示器
2. 有时候不同的刷新率对于对线材和接口是有要求的，比如对于4K或者5K显示器，推荐使用 type-C 线缆，至少也得是 HDMI 或者 DP 电缆

> Note: 我目前在使用的[4K显示器](https://item.jd.com/1762516.html)在开启 HiDPI 之后发现 RDM 不能切换到理想的 Retina 显示效果，要么屏幕闪烁，要么黑屏，几经周折发现是使用的 HDMI 线缆有问题，更换了 HDMI 线缆之后，使用[绿联拓展坞](https://item.jd.com/5841257.html)转换可以可以完美呈现预期的显示效果。
