# Sulphur 2：4GB 显存本机路线研究

日期：2026-11-08（北京时间）。对应研究票：[确认 4GB 显存下 Sulphur 2 的模型与低内存部署路线](https://github.com/wozhendeshifwa/bendibushu/issues/2)。

## 结论

**先保留现有 coolthor 蒸馏 Q4_K_M，验证“独立进程预编码提示词 → 释放编码器 → 保留 GGUF mmap、按层卸载 → CPU VAE 解码”的本机路线。它是值得试验的首选，尚不是已证实能在 4GB 上完成的方案。** 更换大模型不是第一步：已有故障既涉及文本编码器内存峰值，也涉及 GGUF 首次加载的复制路径，单纯换量化不能证明能解决这些问题。

本次仅阅读模型作者说明、运行时代码、已有脚本与日志；没有下载模型、改配置、启动编码/生成或进行视频试跑。报告期间，先前由其他任务启动的预编码已自行结束，详见下文。

## 模型身份与证据强度

| 对象 | 已核实事实 | 对本机的含义 |
| --- | --- | --- |
| SulphurAI/Sulphur-2-base | 作者描述为基于 LTX-2.3、同时支持文生与图生的视频模型；作者提供 dev 模型与蒸馏 LoRA 的组合说明。[作者模型卡](https://huggingface.co/SulphurAI/Sulphur-2-base) | 这是上游身份；网页自动生成的 llama.cpp/聊天应用用法不能当作视频部署说明。 |
| 本地 coolthor 蒸馏 Q4_K_M | 作者明确它是修改后的量化分发，非官方发布；包含 DiT、辅助组件和完整视频 VAE，Gemma 另取。作者实测为 22GB RTX 2080 Ti，工作流峰值 18,050MiB。[固定版本 README](https://huggingface.co/coolthor/Sulphur-2-distilled-Q4_K_M-GGUF/blob/e9fbec9b86b677c7a72d75c8ffc1bb192095304c/README.md) | **作者没有证明本机 4GB 可用。** 但其 ComfyUI 0.28.0 / GGUF 6ea2651 / Torch 2.6.0+cu124 组合与本地相符，不应仅因版本号而先整体升级。 |
| 博客当前“蒸馏”入口的 10Eros GGUF | 量化作者指明其来自 TenStrip 的、基于 Sulphur 的图生合并模型。[量化作者说明](https://huggingface.co/vantagewithai/LTX2.3-10Eros-GGUF) | 与本地 coolthor 蒸馏包不能混称；更换后需重新核实辅助组件和采样方式。 |

本地下载来源由旧工作目录的 `download-manifest.json` 记录：固定 revision 为 `e9fbec9b86b677c7a72d75c8ffc1bb192095304c`。主 GGUF 14,194,463,072 bytes，辅助 checkpoint 2,676,981,980 bytes，视频 VAE 1,452,240,458 bytes，Gemma FP4 9,447,702,218 bytes。本次未对几十 GB 文件重新计算散列，因此清单中的 SHA256 是已有下载记录，不是本次完整性重验。

## 本地断点与新进展

- 旧 `outputs/test-result.json:255–289` 显示 `outputs={}`、`completed=false`，节点 `LTXAVTextEncoderLoader` 在 CPU 分配 1,541,406,720 bytes 时失败。缩小视频分辨率本身不能直接消除发生在文本编码加载阶段的分配失败。
- `work/comfyui-crash-20261108-214224.log:94–103` 是另一次加载故障：Windows access violation，栈经过 GGUF `nodes.py:85 pin_weight_to_device` 的设备搬运。**它说明故障位置，不足以单独证明根因为显存不足、驱动或 mmap 失效。**
- 工作区 `probe_gguf_transfer.log` 通过 4,444 次 CUDA 往返；`probe_gguf_model_load.log` 的保留 mmap 测试通过，private 约 2.245GiB，GPU peak 记录为 0。后者只说明该加载路径没有立即失败，不能充当 GPU 采样、速度或最终显存占用证据。
- 研究期间 `encode_prompt.log` 新增了编码完成记录：234.8 秒时写出 `conditioning/toy_car.pt`，实际文件存在，1,845,208 bytes，修改时间为 2026-11-08 22:38:35。该任务不是本次研究启动的。日志显示加载编码器时 private 16.63GiB、free RAM 0.16GiB、free commit 1.03GiB；结束前 private 16.96GiB。**预编码产物已生成，但内存/提交额度余量非常紧。**
- 现有缓存尚未在 ComfyUI 生成图中端到端消费；本次没有用 `torch.load` 加载它，也没有验证视频输出。缓存来自当前固定的小红色玩具车提示词，换提示词后必须重新预编码，不能只改工作流文字。

原目录基址：`C:/Users/27263/Documents/Codex/2026-11-08/https-www-youtube-com-watch-v/`。工作区基址：`C:/Users/27263/Desktop/本地部署/`。这些本地日志和文件是本节事实的直接来源，随后续运行可能变化。

## 为什么优先分阶段、保留 mmap

本地 ComfyUI HEAD 为 `700821e1364eaab0e8f21c538a2131719fec57bf`；GGUF HEAD 为 `6ea2651e7df66d7585f6ffee804b20e92fb38b8a`（通过只读 .git/HEAD 与 refs 文件核对）。

ComfyUI 的 LTXAV 文本加载器允许将加载与卸载设备都指定为 CPU，并同时读取 Gemma 与辅助 checkpoint。[ComfyUI 加载器源码](https://github.com/Comfy-Org/ComfyUI/blob/700821e1364eaab0e8f21c538a2131719fec57bf/comfy_extras/nodes_lt_audio.py#L166) 本地 `encode_prompt.py` 已据此生成正负条件，随后退出进程。因此后续采样可避免同时保留 Gemma 的大份内存；这项收益是生命周期上的推断，整体采样峰值尚未测得。

GGUF 原有 `pin_weight_to_device` 和首次 `load` 会为解除 mmap 关联而搬运模块，源码留有可能 OOM 的注释。[GGUF 源码](https://github.com/city96/ComfyUI-GGUF/blob/6ea2651e7df66d7585f6ffee804b20e92fb38b8a/nodes.py#L81) 本地 `SulphurMmapLoader` 设置 `mmap_released=True` 跳过这段首次复制，保留只读映射；这是实验性绕行，不能称为上游正式支持的优化。后续首测维持无 LoRA、无权重原地修改，并检查克隆、卸载、复跑及前向计算是否正常。

本地 `start_comfy_lowram.ps1` 已使用 `--novram`、CPU VAE、禁用 pinned memory / async offload / dynamic VRAM、`cache-none` 等参数，配合工作区节点路径。[对应 CLI 定义](https://github.com/Comfy-Org/ComfyUI/blob/700821e1364eaab0e8f21c538a2131719fec57bf/comfy/cli_args.py) 它们提供卸载控制，不能保证内存总量足够。GGUF 文件大小也不等于全部运行内存。

## 首选与备选

1. **首选：复用现有蒸馏 Q4_K_M + 已生成提示词缓存。** 先核对缓存来源/键/张量布局以及两种自定义节点已注册，再以现有小工作流完成真正的采样、解码、保存。256×256、9 帧、24fps、8 步、CFG1 是现有脚本参数，不是用户已确认的最终画质。9 帧约 0.375 秒，只适合粗略冒烟检查，未必足以看清运动。
2. **同模型的备选：采样显存不够时继续减小激活占用/分块解码，或评估 CPU 全量执行。** 依据原生 `--cpu` 与卸载控制路径逐项验证；CPU 运行仍需要足够系统内存和提交额度，耗时更久不能代替容量。CPU 全量生成的完整兼容性未实测，不能预先宣称必成。
3. **模型替换候选：Sulphur dev Q3_K_S（约 10.3GB）。** [量化发布者文件](https://huggingface.co/vantagewithai/Sulphur-2-Base-GGUF/blob/main/sulphur_dev-Q3_K_S.gguf) 比现有主文件小，是有来源的容量候选；但它是 dev，不是本地 distilled，不能原封不动沿用 8 步蒸馏配方。需核对 revision/哈希、辅助组件、LoRA 与采样方案后再决定，当前不推荐为了“更小”直接下载。它仍明显大于 4GB，也没有本机成功证据。
4. **文本编码的替换候选暂不拍板。** 本地 GGUF loader 识别 gemma3，并有双编码器载入路径，但不能据此断言任意 Gemma GGUF 都与 LTX 的 tokenizer、隐藏层及辅助投影兼容。当前预编码已成功，优先验证现有缓存，避免同时更换两个变量。

[LTX-2.3 官方说明](https://huggingface.co/Lightricks/LTX-2.3) 给出蒸馏版本的 8 步、CFG1，及尺寸按32、帧数按8n+1的约束。本地工作流满足这些整除条件；条件满足只说明形状合理，不证明画质或实际运行成功。

## 下一阶段操作清单（须先由用户确认进入阶段）

1. 用户确认首轮参数与是否要求音频。现有工作流包含 AV 联合采样和音频解码；“不保存音频”不自动等于模型采样阶段不处理音频，不应擅删 AV 节点。
2. 只读复核进程/8188、free RAM、free commit、磁盘余量；确认并发编码进程已经退出，避免双份模型同时常驻。读取缓存元信息、正负条件尺寸与模型标识；保留现有文件。
3. 以工作区已有启动脚本、`sulphur_lowram_test_api.json` 为候选验证；若须改配置/工作流，先做带时间戳可恢复备份。测试命名与正式验收分开，保存日志和 prompt ID。
4. 观察文本条件读取、DiT 前向采样、VAE/音频解码、保存四段；记录峰值内存与输出路径。首次生成不设人为时间上限；明确异常则记录栈与阶段并诊断，避免无证据重复重启。
5. 输出文件须实际探测/播放，确认尺寸、帧数、时长、画面有可见变化。短冒烟通过后再讨论足够验收的片段；随后另阶段验证图生输入与相关性。

## 尚未证实

4GB 全流程是否成功；现有 mmap 绕行的采样/复跑稳定性；缓存实际消费的正确性；长于9帧时峰值；最终视频/音频质量；任一替换量化在本机的吞吐与完整兼容性。本研究票可作为路线调查完成，**部署与两种视频验收仍未完成**。
