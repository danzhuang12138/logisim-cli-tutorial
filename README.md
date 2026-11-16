# Logisim CLI 真值表闭环工作流

让 AI 写 Logisim 电路文件，用 Logisim-evolution 自带的无界面命令行（`-t table`）自动跑真值表、逐行比对——断线错线一眼现形，不用打开 GUI 逐个元件检查。本仓库是配套博客教程的示例仓库。

## 仓库内容

| 文件 | 内容 | 验证状态 |
|---|---|---|
| [logisim-cli-tutorial.md](logisim-cli-tutorial.md) | 完整教程：环境准备、CLI 用法、.circ 格式三坑、几何规则速查、与或门实战、给 AI 的提示词模板 | — |
| circuits/full_adder.circ | 全加器：Sum = A⊕B⊕Cin，Cout = AB + Cin(A⊕B) | 真值表 8 行全对 ✅ |
| circuits/and_or.circ | 与或门：Y = AB + CD | 真值表 16 行全对 ✅ |
| circuits/xor.circ | 异或门：Y = A⊕B | 真值表 4 行全对 ✅ |
| circuits/majority_voter.circ | 三人表决器：Y = AB + AC + BC | 真值表 8 行全对 ✅ |

## 快速开始

1. 安装 Java 21（推荐 [Eclipse Temurin](https://adoptium.net/)）；
2. 下载 [Logisim-evolution Releases](https://github.com/logisim-evolution/logisim-evolution/releases) 里的 `all.jar`，放到任意目录（约 54MB，无需安装）；
   > 国内下载加速：在 release 链接前拼 `https://ghfast.top/` 前缀。
3. 无界面跑真值表：

   ```bash
   java -jar logisim.jar -t table circuits/xor.circ
   ```

4. 打开 GUI，用手型工具（快捷键 1）拨输入引脚看仿真效果：

   ```bash
   java -jar logisim.jar circuits/xor.circ
   ```

   启动脚本模板见 [logisim.bat.example](logisim.bat.example)。

## 工作流一句话

```
推导预期真值表 → AI 写 .circ 文件 → CLI 跑真值表 → 逐行比对 → 不一致改几何重跑 → 全对交付
```

核心思想：**电路即代码，真值表即测试用例**。详细规则、踩坑记录和可直接粘贴给 AI 的指令模板见 [logisim-cli-tutorial.md](logisim-cli-tutorial.md)。
