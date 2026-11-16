# 让 AI 帮你画数电电路，还能自动验证：Logisim CLI + 真值表闭环实战

> 学数字电路时最烦两件事：画图费劲，验算更费劲。这篇文章介绍一套我在用的流程——**AI 写电路文件，Logisim 无界面命令行自动跑真值表，逐行比对通过才算交付**。全程不需要打开 GUI 点鼠标，也不需要装任何额外工具。

## 一、痛点与思路

让 AI（ZCode、Codex、Claude 之类）直接生成 Logisim 电路图，最大的问题不是"画不出来"，而是**画错了你不知道**：一根导线端点差 10px 就是断线，AI 自己看不出来，你打开 GUI 逐个元件检查要花半小时。

突破口是 Logisim-evolution 自带的无界面（headless）模式：一条命令就能把电路的**完整真值表**打印到终端。真值表一出来，验证就从"人眼看图"变成了"文本比对"——这正是 AI 最擅长的事。

整个闭环就三步：

```
推导预期真值表 → AI 写 .circ 文件 → CLI 跑真值表逐行比对 →（不一致就改，直到全对）
```

## 二、环境准备

只需要两样东西：Java 和 Logisim-evolution 的 jar 包。

**1. 安装 Java 21**

推荐 Eclipse Temurin（Adoptium）发行版，装完确认 `java -version` 能用。

**2. 下载 Logisim-evolution**

去 [logisim-evolution 的 GitHub Releases](https://github.com/logisim-evolution/logisim-evolution/releases) 下载 `all.jar` 版本（无需安装，一个文件就是全部）。本文以 v5.0.0 为例，实测约 54MB。

> 国内网络提示：GitHub 直连不稳时，可以在 release 下载链接前拼镜像前缀（如 `https://ghfast.top/` 或 `https://gh-proxy.com/`）加速下载。

**3. 一个启动脚本（可选但推荐）**

Windows 下建个 `logisim.bat`，以后双击 .circ 文件就能打开 GUI：

```bat
@echo off
rem 用法: logisim.bat [file.circ]
start "" "C:\path\to\jdk21\bin\javaw.exe" -jar "D:\EDA\logisim\logisim.jar" %*
```

目录结构建议：

```
D:\EDA\logisim\
├── logisim.jar
├── logisim.bat
└── circuits\          ← 电路文件都放这里
    ├── full_adder.circ
    ├── and_or.circ
    └── ...
```

## 三、核心命令：一条命令打印真值表

```bash
java -jar logisim.jar -t table 你的电路.circ
```

`-t` 是 tty（无界面输出）选项，`table` 表示枚举**全部输入组合**并打印真值表。比如对一个两输入异或电路：

```
A B Y
0 0 0
0 1 1
1 0 1
1 1 0
```

配合两个有用的变体：

```bash
java -jar logisim.jar -t stats 你的电路.circ   # 打印元件统计（排查"元件被静默丢弃"神器）
java -jar logisim.jar 你的电路.circ             # 不带 -t 就是正常打开 GUI
```

**判读规则（重要）：**

- 输出里出现 `U`（unknown）= 有导线没连上或悬空；
- 输出**全 0** 也可能是断线——不能只看"命令能跑"，必须和预期真值表逐行比对；
- `-t table` 只对**组合逻辑**有效（后面"限制"一节细说）。

## 四、.circ 文件格式：骨架与三个坑

`.circ` 就是 XML，UTF-8 编码，可以直接文本编辑。最小骨架：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="1.0" source="5.0.0">
  <lib desc="#Wiring" name="0"/>
  <lib desc="#Gates" name="1"/>
  <lib desc="#Plexers" name="2"/>
  <lib desc="#Arithmetic" name="3"/>
  <lib desc="#Memory" name="4"/>
  <lib desc="#I/O" name="5"/>
  <main name="main"/>
  <circuit name="main">
    <comp .../>   <!-- 元件 -->
    <wire from="(x1,y1)" to="(x2,y2)"/>   <!-- 导线 -->
  </circuit>
</project>
```

三个实测踩过的坑，**每一个都会让元件静默消失或断线，而且不报错**：

**坑 1：`<lib>` 的 `name` 属性是序号字符串，不是显示名。**
`name="0"` 对应 Wiring、`name="1"` 对应 Gates……元件里写 `lib="0"`、`lib="1"` 就是按这个序号引用的。写错（比如写成显示名）会导致**所有元件被静默丢弃**——GUI 打开是空的。用 `-t stats` 一看元件数是 0 就知道中招了。

**坑 2：门的 `loc` 是"输出端点"，不是中心。**
`facing="east"` 的门，输出朝右，输入在左边。以 2 输入、`size="50"`（中等宽度）为例，两个输入引脚的位置是 `(loc.x-50, loc.y±20)`。而 Pin（输入/输出引脚）的连接点就是 `loc` 本身——**导线端点必须精确落在引脚 loc 上，差 1px 都算断线**。

**坑 3：XOR/XNOR 的输入比 AND/OR 多偏移 10px。**
同样是 2 输入 size=50，XOR 的输入在 `(loc.x-60, loc.y±20)`（源码里有个 `bonusWidth=10`）。这个不实测根本发现不了。

另外两条纪律：

- **不要用门的取反属性**（negateOutput 之类），它会改变端口偏移导致几何全错；要取反就显式加一个 NOT 门。
- **所有坐标对齐 10px 网格**，导线只画水平/垂直线。

## 五、几何规则速查表

| 元件 | loc 含义 | 2输入(size=50)输入位置 |
|---|---|---|
| Pin | 连接点=loc | — |
| AND/OR/NAND/NOR | 输出端点 | `(loc.x-50, loc.y±20)` |
| XOR/XNOR | 输出端点 | `(loc.x-60, loc.y±20)` ← 多10px |
| NOT(size=30) | 输出端点 | `(loc.x-30, loc.y)` |

连线规则（Logisim 的"节点"判定）：

- 两条导线**共享端点** = 相连（可以多线共点）；
- 一条导线的**端点落在另一条导线中间** = T 型连接，也相连；
- **十字交叉不相连**——需要交叉的地方要么错开，要么有意利用这一点；
- 三输入门（size=50）的三个输入 y 偏移是 `-20 / 0 / +20`。

常用元件片段：

```xml
<!-- 输入引脚 -->
<comp lib="0" loc="(140,140)" name="Pin">
  <a name="facing" val="east"/><a name="output" val="false"/><a name="label" val="A"/>
</comp>
<!-- 输出引脚 -->
<comp lib="0" loc="(700,140)" name="Pin">
  <a name="facing" val="east"/><a name="output" val="true"/><a name="label" val="Y"/>
</comp>
<!-- 与门 -->
<comp lib="1" loc="(420,160)" name="AND Gate">
  <a name="facing" val="east"/><a name="size" val="50"/><a name="inputs" val="2"/>
</comp>
```

> 引脚的 `label` 会成为真值表的列名，务必都写上。

## 六、完整实战：与或门 Y = A·B + C·D

从零走一遍全流程。

**第 1 步：先推导预期真值表**（这一步不能省，它是验收标准）：

```
A B C D | Y
0 0 0 0 | 0
0 0 0 1 | 0
0 0 1 0 | 0
0 0 1 1 | 1   ← C、D 同时为 1
0 1 0 0 | 0
0 1 0 1 | 0
0 1 1 0 | 0
0 1 1 1 | 1   ← C、D 同时为 1
1 0 0 0 | 0
1 0 0 1 | 0
1 0 1 0 | 0
1 0 1 1 | 1   ← C、D 同时为 1
1 1 0 0 | 1   ← A、B 同时为 1
1 1 0 1 | 1   ← A、B 同时为 1
1 1 1 0 | 1   ← A、B 同时为 1
1 1 1 1 | 1
```

**第 2 步：写电路文件** `and_or.circ`。布局习惯：输入引脚 x=140 竖排，门放中部，输出引脚 x=700：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="1.0" source="5.0.0">
  <lib desc="#Wiring" name="0"/>
  <lib desc="#Gates" name="1"/>
  <lib desc="#Plexers" name="2"/>
  <lib desc="#Arithmetic" name="3"/>
  <lib desc="#Memory" name="4"/>
  <lib desc="#I/O" name="5"/>
  <main name="main"/>
  <circuit name="main">
    <comp lib="0" loc="(140,100)" name="Pin">
      <a name="facing" val="east"/><a name="output" val="false"/><a name="label" val="A"/>
    </comp>
    <comp lib="0" loc="(140,180)" name="Pin">
      <a name="facing" val="east"/><a name="output" val="false"/><a name="label" val="B"/>
    </comp>
    <comp lib="0" loc="(140,260)" name="Pin">
      <a name="facing" val="east"/><a name="output" val="false"/><a name="label" val="C"/>
    </comp>
    <comp lib="0" loc="(140,340)" name="Pin">
      <a name="facing" val="east"/><a name="output" val="false"/><a name="label" val="D"/>
    </comp>
    <comp lib="1" loc="(420,140)" name="AND Gate">
      <a name="facing" val="east"/><a name="size" val="50"/><a name="inputs" val="2"/>
    </comp>
    <comp lib="1" loc="(420,300)" name="AND Gate">
      <a name="facing" val="east"/><a name="size" val="50"/><a name="inputs" val="2"/>
    </comp>
    <comp lib="1" loc="(600,220)" name="OR Gate">
      <a name="facing" val="east"/><a name="size" val="50"/><a name="inputs" val="2"/>
    </comp>
    <comp lib="0" loc="(700,220)" name="Pin">
      <a name="facing" val="east"/><a name="output" val="true"/><a name="label" val="Y"/>
    </comp>
    <wire from="(140,100)" to="(320,100)"/>
    <wire from="(320,100)" to="(320,120)"/>
    <wire from="(320,120)" to="(370,120)"/>
    <wire from="(140,180)" to="(320,180)"/>
    <wire from="(320,180)" to="(320,160)"/>
    <wire from="(320,160)" to="(370,160)"/>
    <wire from="(140,260)" to="(320,260)"/>
    <wire from="(320,260)" to="(320,280)"/>
    <wire from="(320,280)" to="(370,280)"/>
    <wire from="(140,340)" to="(320,340)"/>
    <wire from="(320,340)" to="(320,320)"/>
    <wire from="(320,320)" to="(370,320)"/>
    <wire from="(420,140)" to="(480,140)"/>
    <wire from="(480,140)" to="(480,200)"/>
    <wire from="(480,200)" to="(550,200)"/>
    <wire from="(420,300)" to="(480,300)"/>
    <wire from="(480,300)" to="(480,240)"/>
    <wire from="(480,240)" to="(550,240)"/>
    <wire from="(600,220)" to="(700,220)"/>
  </circuit>
</project>
```

**第 3 步：CLI 验证**

```bash
java -jar logisim.jar -t table and_or.circ
```

实测输出：

```
A B C D Y
0 0 0 0 0
0 0 0 1 0
0 0 1 0 0
0 0 1 1 1
0 1 0 0 0
0 1 0 1 0
0 1 1 0 0
0 1 1 1 1
1 0 0 0 0
1 0 0 1 0
1 0 1 0 0
1 0 1 1 1
1 1 0 0 1
1 1 0 1 1
1 1 1 0 1
1 1 1 1 1
```

16 行与预期**完全一致**——这个电路就算交付了。GUI 打开（`logisim.bat and_or.circ`）就是一张规整的两级与或图，用手型工具拨输入引脚可以看到导线变绿（高电平）、输出灯点亮。

**顺手写个自动比对脚本**（可选，交给 AI 生成也行）：

```python
import subprocess

circ = "and_or.circ"
actual = subprocess.run(
    ["java", "-jar", "logisim.jar", "-t", "table", circ],
    capture_output=True, text=True
).stdout.strip().splitlines()

expected = """A B C D Y
0 0 0 0 0
0 0 0 1 0
0 0 1 0 0
0 0 1 1 1
0 1 0 0 0
0 1 0 1 0
0 1 1 0 0
0 1 1 1 1
1 0 0 0 0
1 0 0 1 0
1 0 1 0 0
1 0 1 1 1
1 1 0 0 1
1 1 0 1 1
1 1 1 0 1
1 1 1 1 1""".strip().splitlines()

print("PASS" if actual == expected else "FAIL")
for a, e in zip(actual, expected):
    if a != e:
        print(f"实测 {a!r} != 预期 {e!r}")
```

## 七、把这套规则变成给 AI 的指令模板

上面所有规则，直接作为提示词交给 AI 编程助手即可复现整条流水线。模板：

```text
请帮我用 Logisim 画数电电路，严格遵守以下规则：

1. 先从需求推导预期真值表给我看，作为验收标准；
2. 电路输出为 .circ 文件（XML，UTF-8）。项目骨架中 <lib> 的 name 属性
   是序号字符串（0=Wiring, 1=Gates…），元件用 lib="0"/"1" 按序号引用；
3. 所有坐标对齐 10px 网格，导线只水平/垂直，端点必须精确落在引脚 loc 上；
4. 门的 loc 是输出端点。2输入 AND/OR（size=50）输入在 (loc.x-50, loc.y±20)；
   XOR/XNOR 输入在 (loc.x-60, loc.y±20)，比 AND/OR 多 10px；NOT(size=30)
   输入在 (loc.x-30, loc.y)；Pin 连接点=loc；
5. 不用门的取反属性，需要取反就加显式 NOT 门；每个引脚都写 label；
6. 写完后运行：java -jar logisim.jar -t table <文件>，
   把输出与第 1 步的预期真值表逐行比对。出现 U、全 0 或任何一行不一致，
   都视为断线/错线，修改几何后重跑，直到完全一致才交付。
```

亲测这套模板可以让 AI 稳定产出**能通过全真值表验证**的电路：全加器（8 行）、三人表决器（8 行）、4 输入与或（16 行）这类组合逻辑基本都是一次过或一次返工内通过。

## 八、已知限制（别踩）

- **`-t table` 只验组合逻辑**。D 触发器、计数器、寄存器这类时序电路验不出来，要么 GUI 仿真，要么用测试向量模式（`-w` 参数，输入一个 vector 文件逐拍比对）。
- 多位总线（width>1）、Splitter、MUX 的端口几何我没有逐一实测，**先用小电路跑真值表验证几何，再拼大电路**。
- 真值表列的顺序跟引脚在文件里出现的顺序有关，比对时留意列名。

## 九、结语

这套流程的本质是把"画电路"从图形操作降维成文本问题：**电路即代码，真值表即测试用例**。数电教材里的经典电路——加法器、编码器、译码器、MUX、比较器、奇偶校验——都可以用这个闭环一个个啃下来，每画一个都附带一份机器验证过的真值表，比抄书印象深刻得多。

（完）
