---
name: fbox-apk-modding
description: FBox(com.zuimo.fbox 3.3.3) 逆向改包：设置页「开发者」区块结构与 MT MCP 改包/出包流程、容器访问不到宿主存储的限制
---
# FBox (com.zuimo.fbox 3.3.3) 改包笔记

## 已核实结构
- 设置页「开发者」区块 = `Lcom/zuimo/fbox/ui/settings/SettingsSectionsKt;->DeveloperSection(Function0 onOpenLogcat, Function0 onOpenChangelog, Composer, I)V`（SettingsSections.kt:1058 起）。
  - 标题：`SectionTitle("开发者", modifier, Icons.Rounded.Code, ColorKt.getCyan(), composer, 6, 2)`（`.line 1060`）。
  - 卡片：`GlassCard(Modifier.fillMaxWidth(), null, rememberComposableLambda(DeveloperSection$1), composer, 0x186, 2)`（`.line 1061`）。
  - 卡片内容 lambda 类：`SettingsSectionsKt$DeveloperSection$1`（约 1716 行 smali；第一行「运行日志」→ onOpenLogcat，第二行「更新内容」→ onOpenChangelog，中间一条 Divider）。
  - **唯一调用点**：`Lcom/zuimo/fbox/ui/MainScreenKt$MainScreen$3$1$5$1$21$2;->invoke(Composer;I)V`（MainScreen.kt:328-332），该 lambda 只是 remember 两个 nav lambda（"logcat"/"changelog"）后调用 DeveloperSection。
- 「整块移除」最省事的做法：删掉 `DeveloperSection` 里 `const-string v1, "开发者"` 到 `GlassCard(...)V` 之间的整段调用（内部无标签、无组不平衡问题），再把失去引用的 `DeveloperSection$1` 整类删除。
- Compose smali 注意事项：删调用要保证 `startReplaceGroup/endReplaceGroup`、`startRestartGroup/endRestartGroup` 成对；删掉某行的寄存器定义后，后续行会引用未定义寄存器（如 `move-object/from16 v11, v28`）导致校验/运行异常——整体删区块时优先整块摘除，别只删一半。

## 上下文 token 预算上限（200K→改大）
- 共 **3 处**写死 200000(=0x30d40)，改大必须全改，否则会出现「滑块能拉大但保存被夹回」「设置生效但主界面顶部 x/200.0k 不变」：
  1. 滑块范围：`SettingsSectionsKt$AiProviderSection$1$1$15->invoke` 第 604 行附近 `rangeTo(8000.0f, 200000.0f)` = `const v2, 0x48435000`(200000.0f) → 改 `0x49f42400`(2000000.0f)。注意 8000.0f=`const/high16 0x45fa0000`、200000.0f=`0x48435000`、1000000.0f=`0x49742400`、2000000.0f=`0x49f42400`。
  2. 保存钳制：`LinuxViewModel->updateAiMaxCtxTokens(I)` 的 `coerceIn(0x1f40, 0x30d40)` → 上限改 `0x1e8480`(2000000)。
  3. **主界面顶部显示/实际使用**：`LinuxViewModel->resolveCtxWindow()I`。manual 分支（getHasManualCtxTokens 为真时）读 pref 后 `coerceIn(0x1f40, 0x30d40)` → 上限改 `0x1e8480`。顶部「≈x/ymax」的分母就是 resolveCtxWindow()。auto 分支（按模型名/MODEL_CONTEXT_MAP/正则 \dm \dk 推断）本就 coerceIn 到 0x1e8480(2M)，无需改。
- 滑块 `steps`(第5个 int 参数，原 `const/16 v34, 0x17`=23)：原设计 min8000/max200000/steps23 → 每格正好 8000、24 个刻度点。**扩大 max 后若想保持同样 ~24 点的点状样式必须保持 steps=23**，但每格变 (max-8000)/24（2M→83000，非整），停靠值不圆整；若改 steps 让每格仍=8000(2M→steps=248=0xf8)则刻度点过密、视觉变成实线（用户不接受）。二者不可兼得（8000 的 min 导致任何圆整步长都无法整除）。

## 自动匹配模型上下文（已验证的正确做法：改 resolveCtxWindow，别碰 <init>）
- **血泪坑（已真机验证）：绝不要重写 `LinuxViewModel-><init>` 里的 MODEL_CONTEXT_MAP 构建块**。那个 <init> 是超大手写字节码、寄存器高度复用，改 map 构建会连锁崩溃：
  1. 循环用的寄存器留下的类型与原版不一致 → 启动即 `VerifyError: Invalid reg type for array index (Conflict)`。
  2. 为修①做 `const/4 vX,0x0` 复位又改了后续代码依赖的寄存器值 → `NullPointerException: Pair.component1() on null`（下游另一处 mapOf 拿到 null）。
  edit_check / build 都查不出这类 ART 校验/运行期错误，只有真机启动才暴露。
- **正确做法**：完全不动 <init>，把模型→窗口映射注入到隔离的小方法 `LinuxViewModel->resolveCtxWindow()I` 的 auto 分支。在 `:cond_34`（模型名非空、v0=小写名 CharSequence）之后、原 `iget-object v3, ...MODEL_CONTEXT_MAP` 之前，注入一串 `contains$default` 判断：命中就 `const v6, <窗口>` + `return v6`，全不命中 `goto :ctx_orig` 落回原 MODEL_CONTEXT_MAP+正则逻辑。
- 注入写法（照抄原方法已有的 contains$default 调用，寄存器全用新建常量，不碰 v0/v1/v2）：开头 `const/4 v3,0x0`(ignoreCase=false) `const/4 v4,0x2`(mask) `const/4 v5,0x0`(null marker)；每条 `const-string v6,"key"` + `invoke-static {v0,v6,v3,v4,v5}, StringsKt;->contains$default(...)Z` + `move-result v6` + `if-nez v6, :ctx_xxx`；末尾用值标签 `:ctx_1m/:ctx_400k/:ctx_256k/:ctx_200k/:ctx_128k` 各 `const v6,值` + `return v6`，并在它们前面放 `goto :ctx_orig` 避免 fall-through。标签名别和原有 :cond_/:goto_ 冲突。
- 顺序同样「具体 key 在泛化 key 前」（gpt-4.1/gpt-5 在 gpt-4 前、deepseek-v4 在 deepseek 前、qwen3.x-max 在 qwen 前、claude-opus/sonnet/fable 在 claude 前、glm-5.x 在 glm 前、minimax-m3 在 minimax 前）。
- 已内置 **62 条**（数据源=models.dev api.json 权威值，2026-10）。生成脚本与成品在 ~/workspace/.aicode/attachments/ 下：gen.pl、inject.smali（完整注入块，可直接复用）。用 context 总窗口值：gpt-6/5.6/5.5/5.4=1.05M(0x100590),gpt-4.1=0xffc18,gpt-5=400K(0x61a80),gpt-4o/4-turbo/4&deepseek泛=128K(0x1f400),o1/o3/o4&claude-opus-4.5&haiku&claude泛&glm-5.1=200K(0x30d40),claude-opus/sonnet/fable&deepseek-v4&glm-5.2/5.3&qwen3.5~3.8多款&qwen-plus/flash/turbo&minimax-m3=1M(0xf4240),glm-5/4.7/4.6&minimax泛=204800(0x32000),glm-4.5&glm/qwen/grok/mistral泛=131072(0x20000),qwen3-max/coder&qwen-max&kimi-k2/kimi&mistral-large/medium&devstral=262144(0x40000),kimi-k3&gemini全=1048576(0x100000),doubao&step&mistral-small&codestral&hunyuan=256000(0x3e800),grok-4=500000(0x7a120)。
- AiCode(1.11.0) 设计参考：模型上下文来自外部 models.dev（ModelMetadataService 拉 api.json+内置 asset 兑底，24h 缓存）按 id 匹配；ModelApiService.fetchModels 打中转站 /v1/models 只读 id 不读上下文。所以 FBox 显示真实上下文只能同样靠 id 匹配。
- 注：一旦用户手动拖过「上下文 token 预算」滑块（prefs 写 ai_max_ctx_tokens），hasManualCtxTokens 恒真，走 manual 分支，上面 auto 注入不执行；自动匹配仅对「从未手动设过」的模型生效。
- **名称匹配固有缺陷 + 定稿做法（2026-10）**：同名不同厂（[商汤]glm-5.2=128K vs 智谱 glm-5.2=1M）无法用名字区分。把匹配从 contains 改成 java.lang.String.startsWith（纯 JDK、无 Kotlin stdlib 风险）：注入开头 check-cast v0, Ljava/lang/String; ；每条 const-string v6 + invoke-virtual {v0,v6} String->startsWith(String)Z + move-result v6 + if-nez v6,:ctxX。这样 [商汤]glm-5.2 不以 key 开头→不命中→落默认；纯 glm-5.2→1M。表尾 goto :ctxmiss 设 v5=2/v6=0/v7=0 后 fall-through 进原正则分支 :cond_71（跳过原 MODEL_CONTEXT_MAP 迭代）。默认值 const/16 v2,0x6590(26000) 改成 const v2,0x1f400(128000)，未匹配/空名回 128K（对齐 AiCode）。生成脚本 gen2.pl / inject2.smali 在 ~/workspace/.aicode/attachments/。

## 工具/流程
- MT MCP（工作区 + editSession）改包：`mt_apk_edit_open` → `mt_apk_edit_text`(replace_match 删除) → `mt_apk_edit_check(runBuildChecks=true)` → `mt_apk_build(sign=true)`。
- MT MCP 的 Home（出包目录）= `/storage/emulated/0/MT2/mcp`；出包名带 `_sign` 后缀（或自定义名）。
- **AiCode 容器（proot）读不到宿主存储**：`/storage/emulated/0`、`/sdcard`、`/data/media/0` 均不存在 → 构建产物无法用 sendFile 发到聊天区，只能在回复里给出宿主路径，由用户用 MT/文件管理器自取。
- 出包用 MT 的签名密钥（cert sha256 165a14eb…），与原包签名不同，安装前通常需先卸载原应用。