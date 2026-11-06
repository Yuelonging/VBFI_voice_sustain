# VBFI R-18 语音「消息改行时停止」修复补丁

**版本：v1.0.0**　适用：VenusBlood FRONTIER International（中文版 + R-18 补丁，Kirikiri/KAG3 引擎）

---

## 一、安装

1. 关闭游戏。
2. 把本文件夹里的 **`patch.xp3`** 复制到游戏根目录（和 `VBFI.exe`、`data.xp3` 同一个文件夹）。
3. 启动游戏，进设置把「消息改行时的音色设定」设为**停止**，进任意 H 场景点一行即可听到变化。

不需要解压、不需要改名、不需要删除或替换任何原有文件（`data.xp3` / `extra.xp3` / `extra2.xp3` 全部保持原样）。

**卸载**：删掉 `patch.xp3` 即恢复原状（存档不受影响）。

---

## 二、现象与根因

### 现象
设置里「消息改行时的音色设定」选**停止**，全年龄剧情正常（点一行语音就断），但 **R-18 部分的语音永远不停**，改选**不停止**也没有区别 —— 即该选项对 R-18 语音完全无效。

### 根因：R-18 的喘息声走的是另一条音频通道，而"停止"逻辑只管第一条

游戏把场景语音分成两套（`gamesystem/kag/userfunctions.tjs`）：

| 场景写法 | 调用 | 播放缓冲 | 循环 |
|---|---|---|---|
| `[voice storage="cv_xxx"]` | `playVoice()` → `playVoiceList(storage, 1, …)` | `kag.se[**1**]` | 否 |
| `[bgv storage="cv_xxx"]` | `playBackVoice()` → `playVoiceList(storage, 2, true, …)` | `kag.se[**2**]` | **是（loop=true）** |

而「消息改行时停止语音」的实现只有一行（`gamesystem/kag/userfunctions.tjs` 第 210 行）：

```tjs
function playVoiceStopCheck()
{
    if(kag.sflags.systemcore.option.voice.messcut) kag.se[1].stop();
}
```

它在 `[p2]`（点击等待）、`[lp]`、`[lp2]` 三个宏里被调用（`gamesystem/kag/macro.ks` 第 104/117/133 行），**只停 `se[1]`**。

于是：

* 普通对白语音（`[voice]`，se[1]）→ 点一行会停 ✔
* H 场景的喘息/娇声（`[bgv]`，se[2]，**循环播放**）→ 没有任何人停它 ✘

而且 `[bgv]` 是 `loop=true`，所以它一旦播放就会一直循环到被下一条 `[bgv]` 顶掉 —— 听感上就是"怎么设置都不停"。

### 数据佐证（实际解包统计）

* `extra.xp3`（R-18 资源包）里 287 个脚本中，**90 个脚本共 3266 处 `[bgv]`**，全部集中在 `za01~za91 / zb / zc / zd / ze / zh / zk / zl / zm` 系列，也就是 H 场景本体。
* `data.xp3`（全年龄线）里同名脚本也写着 `[bgv]`，但 `macro.ks` 的 bgv 宏带开关：

  ```tjs
  [macro name="bgv"]
  [if exp="sf.age_base!=0"]     ; ← 只有 R-18 模式(装过 extra/etc2)才真正播放
  ...
  [endif]
  [endmacro]
  ```

  所以这个 bug **只在 R-18 部分出现**，全年龄剧情一切正常 —— 与你的观察完全一致。

### 附：还有一个容易混淆的开关

「**背景语音**」（`option.voice.background`）控制 `[bgv]` 到底播不播；「**消息改行时的音色设定**」（`option.voice.messcut`）控制"换行时要不要掐掉"。原来的代码只让第二个开关作用于 `se[1]`，第一个开关作用于 `se[2]`，两者没接上。

---

## 三、改了什么

`patch.xp3` 里**只有一个文件**：`gamesystem/kag/userfunctions.tjs`
（与 `data.xp3` 内同名同路径，装载优先度更高，覆盖掉原版的那一份）

改动只有 1 个函数、共 2 行：

```tjs
// —— 原版 ——
function playVoiceStopCheck()
{
    if(kag.sflags.systemcore.option.voice.messcut) kag.se[1].stop();
}

// —— 补丁后 ——
function playVoiceStopCheck()
{
    if(kag.sflags.systemcore.option.voice.messcut){
        kag.se[1].stop();       // normal voice
        kag.se[2].stop();       // back voice (R-18 moan, se[2] loop)
    }
}
```

其他 8 万多字节内容逐字节保持原样（未重新排版、未转码，仍是 UTF-16LE + CRLF + BOM）。

**为什么安全**：在 ADV 场景里 `se[2]` 只被 `[bgv]` 使用；战斗里虽然有 `[se buf=2]`，但战斗没有"消息改行"，`playVoiceStopCheck()` 不会被调用。H 场景收尾处本身也有 `[se_fade buf=2]` 用来淡出这条通道，说明它本来就是"背景喘声"专用道。

**行为差异**：
* 设为**停止** → H 场景点一行，喘息声立即停；下一行的 `[bgv]` 再起新的。与全年龄语音表现一致。
* 设为**不停止** → 行为与打补丁前**完全相同**（代码路径不变）。

---

## 四、为什么用 patch.xp3，而不是直接替换 data.xp3

`system/Initialize.tjs` 内建了补丁归档机制：

```tjs
useArchiveIfExists("patch.xp3");
for(var i = 2; ; i++){ ... patch2.xp3, patch3.xp3 ... }
```

吉里吉里的 `Storages.addAutoPath` 遵循**后注册者优先**（脚本注释原话：「patch2.xp3, patch3.xp3 … がある場合はそちらを**優先**して読み込むように」），而 `patch.xp3` 在 `Initialize.tjs` 第 92 行注册，晚于 `data.xp3`（引擎启动时自动挂载）和 `extra.xp3`（第 75 行 `addPath.tjs` 里注册），因此：

```
patch.xp3  >  extra.xp3  >  data.xp3
```

好处：不用碰 49MB 的 `data.xp3`，也不用碰 584MB 的 `extra.xp3` / 648MB 的 `extra2.xp3`，出问题删一个 6.6KB 的小文件就回滚。

---

## 五、兼容性与注意

* 与官方 R-18 补丁（`extra.xp3` / `extra2.xp3`）**共存**，不需要重装，也不需要调整文件顺序。
* 不修改存档（`文档\dualtail\venusblood_frontier_cn\`），老存档直接可用，不需要重开。
* 若以后再出现**同样改 `gamesystem/kag/userfunctions.tjs` 的 mod**，会与本补丁互相覆盖（同名文件只能有一个生效），届时需要合并。
* 本补丁只改语音停止逻辑，**不影响音量、角色音量、背景语音开关、历史回顾语音**（历史回顾走 `historyVoice()`，仍是 `se[1]`）。

---

## 六、文件与校验

| 文件 | 大小 | MD5 |
|---|---|---|
| `patch.xp3` | 7,237 字节 | `c1705d2086d0ea322c460cefad7da1ca` |

归档内条目：

1. `$$$ This is a protected archive. $$$ …`（157 字节，原归档自带的占位警告条目，照抄以保持结构一致）
2. `gamesystem/kag/userfunctions.tjs`（81,088 字节，adler32 `d5c4d4a5`）

---

## 七、如果装完没有任何变化（排查）

`patch.xp3` 依靠 `system/Initialize.tjs` 的补丁机制装载。正常情况放进根目录就生效。若毫无变化，按顺序排查：

1. 确认文件名确实是 `patch.xp3`（不是 `patch.xp3.xp3`，也不是被浏览器加了 `(1)`）。
2. 确认放在 `VBFI.exe` 同一层，而不是子文件夹里。
3. 确认设置里「消息改行时的音色设定」= **停止**，且 H 场景里确实有 `[bgv]` 的喘息声（za/zb 系列）。
4. 仍然无效时，用散文件方案（优先级同样高于 `data.xp3`）：

   ```
   游戏目录\gamesystem\kag\userfunctions.tjs   ← 把 src\v1.0.0\gamesystem\kag\userfunctions.tjs 放进去
   ```

   这是 `addPath.tjs` 里 `Storages.addAutoPath("gamesystem/kag/")` 提供的目录搜索路径，不需要改任何原文件，删掉即回滚。

---

## 八、自行重建 / 追查

工作区（非游戏目录）内：

* `tools/vbfi_xp3.py` —— 本游戏 xp3 的读取器（这版 xp3 是本地化组重打包过的：索引区被 zlib 压缩后放在文件末尾，偏移记录在头部 `0x20`，条目里没有标准 `segm` 的 kind 字节，所以 GARbro / KrkrExtract 一类工具会直接失败）
* `tools/vbfi_xp3_pack.py` —— 同格式打包器
* `tools/build_patch.py` —— 从游戏原始 `data.xp3` 取文件 → 打补丁 → 重新打包 → 回读自检
* `src/v1.0.0/gamesystem/kag/userfunctions.tjs` —— 补丁后的源码（可直接比对）

重建命令：

```bash
python tools/build_patch.py v1.0.0
```

脚本会打印回读自检结果与 `patch.xp3` 的 MD5。
