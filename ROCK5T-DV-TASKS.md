# RK3588 / Rock 5T Dolby Vision 待办与问题排查任务单

最后更新：2026-11-24（基线提交：`44c813e`）

---

## 任务 1：修复长期存在的 FD 泄漏问题（高优先级）— ✅ 已完成并实机验证

### 现象
运行/播放期间进程文件描述符（FD）持续增长，实机复现 15 分钟内泄漏 **2,359 个 `anon_inode:sync_file`**（总 FD 达 2,416），并诱发播放卡顿与耗尽风险（`EMFILE`）。

### 已定位并修复的代码级根因
1. **高频泄漏：`CDVTunnelDRMPRIME` 的 `m_overlayFence` / `frame.overlay_fd`（`0028` + `pi-dv-engine 0007`）**
   - `CDVTunnelDRMPRIME::DrawOverlay()` 在 `areas.empty()` 分支未关闭并重置 `m_overlayFence`。
   - `CDVTunnelDRMPRIME::Render()` 未判断 `frame.overlay != nullptr`，无字幕/OSD 时仍每帧 `dup(m_overlayFence)` 写入 `frame.overlay_fd`。
   - `pi_engine.c:3130` 在 `f->overlay == NULL` 时将 `drawn_fd` 置为 `-1`，导致传入的 `f->overlay_fd` 从不被 `close()`，以 24–60 个/秒的速率泄漏 `anon_inode:sync_file`。
2. **低频边界泄漏：`CDRMAtomic::m_videoInFenceFd`（`0028`）**
   - `CDRMAtomic::DestroyDrm()` 未关闭残留的 `m_videoInFenceFd`。

### 修复与实机验证 Checklist
- [x] 在 `CDVTunnelDRMPRIME::DrawOverlay()` 的 `areas.empty()` 分支关闭并重置 `m_overlayFence = -1`（`0028-fix-dv-overlay-fence-and-drm-fd-leaks.patch`）
- [x] 在 `CDVTunnelDRMPRIME::Render()` 仅当 `frame.overlay != nullptr && m_overlayFence >= 0` 时调用 `dup(m_overlayFence)`（`0028`）
- [x] 在 `pi_engine.c` 的 `pi_render()` 入口当 `f->overlay == NULL` 时也强制关闭传入的 `f->overlay_fd`（`0007-fix-overlay-fd-leak.patch`）
- [x] 在 `CDRMAtomic::DestroyDrm()` 关闭残留的 `m_videoInFenceFd`（`0028`）
- [x] 在构建机（`192.168.1.180`）完成 `make image` 编译验证，并在 Rock 5T 实机播放《变形金刚 2》（双层 P7 FEL + TrueHD 8ch HDMI 直通）复测：总 FD 稳定在 **122**（`anon_inode:sync_file` 仅 **6–7** 个，+5 秒采样零增长），且 TrueHD 直通不再卡顿

---

## 任务 2：修复《变形金刚》系列双层 DV ISO 特定帧瞬间闪绿问题（高优先级）— ✅ 已完成

### 现象与实机根因定位
- 双轨 Profile 7（蓝光 ISO）已可正常点亮杜比视界并播放，日志确认双轨 EL/BL 配对 100% 正常（`371/371` 零丢包、零失配）。
- 播放《变形金刚 2》2.5 秒时触发 `CDVTunnelDRMPRIME: MMR magnitudes too large for the split's frame bound: checked per sample`（命中 `p1_chroma.frag` 的 `SPLIT_GUARD` / `split_guarded()` 逐采样 64-bit 模拟路径）。
- **4 模式实机对照矩阵**：
  - `Float`：纯 32-bit 浮点 `mmr_float_mag()` → **无闪绿、HDMI TrueHD 8ch 直通流畅无卡顿**；
  - `Exact`（原默认）：极端帧触发 `SPLIT_GUARD` / `WIDE` 回落 64/128-bit 模拟 → **特定帧瞬间闪绿**；
  - `Split` / `Exact64`：全程跑 64/128-bit 模拟 → **闪绿且 GPU 超出 4K24 41.7 ms 帧预算（`OutputPicture - timeout` → `Audio stream stalled`）导致直通音画卡顿**。

### 修复 Checklist
- [x] 对照测试：确认 `Float` 彻底消除闪绿与卡顿，而 `Exact` / `Split` / `Exact64` 均命中 Mali-G610 PanVK 64/128-bit 模拟瓶颈与误算
- [x] 新增 `0029-rk3588-default-dv-mmr-float.patch`，将 RK3588 默认 Dolby Vision Chroma MMR 改为 `Float`（同时保留专家设置中的 `Exact` / `Split` / `Exact64` 手动切换选项）
- [x] 在构建机（`192.168.1.180`）完成 `make image` 编译验证并推送远端

---

## 任务 3：新增 DV → HDR10 / SDR（WCG-SDR BT.2020）动态色调映射输出（中优先级）— ✅ Phase 1 & Phase 2 已完成并实机验证

### 目标与背景
- 面向**不支持杜比视界的显示设备**（普通 HDR10 电视、SDR 显示器/投影仪），实时解码杜比视界（含 Profile 5 `IPTPQc2`、Profile 7 FEL/MEL、Profile 8.1 及 `0020` HDR Vivid 转 DV），通过逐帧动态元数据（RPU L1）进行色调映射，输出高质量 **HDR10** 或 **10-bit SDR / WCG-SDR（BT.2020 广色域 + Gamma）** 画面。
- 参考 `LibreELEC-Hades` 的 `DV → SDR` 能力，但不照搬其 `libplacebo`（29 个自定义补丁）+ `dvbridge` GLES 路线，避免维护成本高、与当前 `pi-dv-engine` 管线割裂以及丢弃 FEL 优势的问题。

### 技术路线：扩展 `pi-dv-engine` PanVK `P2` 着色器（路线 A）
直接复用当前 `pi-dv-engine` 的 `P0`（RPU 解析）/ `P1`（`NV15` 零拷贝、Profile 5 `IPTPQc2` 解码、Profile 7 FEL 12-bit 重建、MMR/Poly 映射）全部能力，仅扩展 **`P2` 打包与色调映射阶段**及 Kodi 输出控制：
1. **`pi-dv-engine` 输出模式与 `p2_pack.frag` 扩展**
   - 在 `pi_options` 新增输出模式枚举：`PI_OUT_DV_STANDARD`（现有 HDMI 隧道）、`PI_OUT_HDR10`、`PI_OUT_SDR`（含目标峰值亮度 `target_nits`、目标色域 `BT.709 / BT.2020`、目标传递函数 `Gamma 2.2 / 2.4` 参数）。
   - 在 `p2_pack.frag` 新增非隧道分支：12-bit 重建画面 → 线性 BT.2020 RGB → 按逐帧 RPU **L1**（`min_pq / avg_pq / max_pq`）执行 **BT.2390 EETF** 动态色调映射 → 编码为 ST.2084 PQ（HDR10）或 Gamma 2.2/2.4 + 可选 BT.2020→BT.709 矩阵（SDR/WCG-SDR）+ 有序抖动 → 输出 **10-bit `XRGB2101010` / `XBGR2101010`** 缓冲（不再打包顶部 DV 元数据行）。
2. **Kodi 解码/渲染门控与 DRM 元数据控制**
   - 放宽 `0004` / `0005`（`DVDVideoCodecDRMPRIME` / `RendererDRMPRIME`）对 `SupportsDolbyVision()` 的硬门控：当用户开启 DV 转 HDR10/SDR 时，即使 EDID 无 DV VSDB 也允许进入 `CDVTunnelDRMPRIME` 管线。
   - 按输出模式配置 DRM 连接器属性：
     - `DV → HDR10`：下发 `HDR_OUTPUT_METADATA`（ST.2084 + BT.2020 + MaxCLL/MaxFALL）并设置 HDMI `Colorspace = BT2020_RGB`；
     - `DV → WCG-SDR`：清空 `HDR_OUTPUT_METADATA`（SDR 模式）并设置 HDMI `Colorspace = BT2020_RGB`；
     - `DV → 标准 SDR`：清空 `HDR_OUTPUT_METADATA` 并设置 `Colorspace = BT709`。
   - 在 Kodi 播放设置中增加 DV 输出模式（自动 / 强制 DV / 转 HDR10 / 转 10-bit SDR）、目标峰值亮度与 SDR 色域选项。

### 硬件前置验证（Rock 5T 实机 `modetest` 已完成）
- [x] `modetest -M rockchip` 实机核查确认：VOP2 Cluster 平面（34/42/82/90）的 `XR30`/`XB30` 仅支持 AFBC 压缩，不支持 `LINEAR`；Kodi 视频平面 Esmart（50/58/66/74）仅支持 `LINEAR`，原生支持 `XR24`（`XRGB8888`）与线性 10-bit YUV（`NV15`/`NV20`/`NV30`）。
- [x] 已将 `p2_pack.frag` 非隧道输出统一为 **4×4 Bayer 有序抖动 `XRGB8888`（`XR24` 线性缓冲）**，兼容 Esmart 平面及 4K60 带宽，同时 HDMI 链路保持 `max bpc = 10`。

### 分阶段实施 Checklist
- [x] **Phase 1（DV → HDR10）**：`pi-dv-engine 0008` 支持 `PI_OUT_HDR10` + 有序抖动 `XRGB8888` 打包（YCbCr 零往返直通 + Profile 5 `IPTPQc2` 矩阵解码 + 可选 L1 BT.2390 峰值映射），Kodi `0031` 放宽非 DV 显示器门控与 Profile 7 EL 解码，实机验证 `p5-sample.mkv` 正确输出 `HDR10`、`Colorspace = BT2020_RGB (9)`、`max bpc = 10`、`Dolby Vision = Off (0)`、`plane 50` 活跃且 FD 稳定在 110
- [x] **Phase 2（DV → SDR / WCG-SDR + L1 BT.2390 DTM）**：在 `p2_pack.frag` 实现逐帧 RPU L1 BT.2390 动态色调映射、Gamma 2.2/2.4 与 BT.2020/BT.709 矩阵及 4×4 Bayer 有序抖动；实机验证 `DV P5 → SDR BT.709`（`Colorspace = 0`）与 `DV P5 → WCG-SDR BT.2020`（`Colorspace = 9`）均 `rendered in place` 正常播放，切回 `Auto` 正常恢复原生 DV（`Dolby Vision = 2`）
- [ ] **Phase 3（可选扩展）**：接入普通 HDR10 / HDR Vivid（`0020`）片源，统一通过 `pi-dv-engine` 输出到 SDR/WCG-SDR，补齐非 HDR 设备全片源画质增强

---

## 任务 4：新增 HDR10+ 实时转杜比视界 Profile 8.1（中优先级）— ✅ 已实现并编译验证（待 Rock 5T 实机验证）

### 背景与现状
- 当前 FFmpeg/Kodi 已能解析 `AV_FRAME_DATA_DYNAMIC_HDR_PLUS`（`0011` 播放信息面板已可读出 `Target / MaxSCL / Average / Curve`），但主线 RK3588 HDMI/DRM 驱动无原生 HDR10+ VSIF 下发，播放 HDR10+ 片源时自动回退为静态 HDR10。
- 已复用 `0020`（HDR Vivid → DV P8.1）架构，新增 `0032-videoplayer-hdr10plus-to-dolby-vision.patch`，实时将逐帧 HDR10+ ST 2094-40 动态元数据转为 Profile 8.1 RPU 喂给 `pi-dv-engine`。

### 实施 Checklist
- [x] 实现 `AVDynamicHDRPlus`（`maxscl` / `average_maxrgb` / `distribution_maxrgb` 场景峰值）→ DV Profile 8.1 RPU（L1 动态元数据 + 场景切换检测 + seek 缺帧回退）转换器（`HDRVividDovi.c` `hdr10plus_dovi_next()`）
- [x] 在 `DVDDemuxFFmpeg`（支持容器 `HDR_TYPE_HDR10PLUS` 与带内 HEVC ST 2094-40 SEI 探测）与 `DVDVideoCodecDRMPRIME` 接入实时转换开关（`videoplayer.dolbyvisionfromhdr10plus`）并添加 Kodi 设置项，已在 `192.168.1.180` 完成 `make image` 编译验证
- [ ] Rock 5T 实机验证：开启后在 DV 电视上正确点亮杜比视界且亮度/暗部过渡优于静态 HDR10

---

## 任务 5：新增 AV1 杜比视界 Profile 10（P10.0 / P10.1 / P10.4）支持（中优先级）— ✅ 已完成并实机验证

### 背景与现状
- RK3588 内置 VPU981 AV1 10-bit 硬件解码器（`rockchip_vpu_hw.c`）默认优先输出线性 `V4L2_PIX_FMT_NV15`，与 `pi-dv-engine` 现有 `NV15` 零拷贝管线完全兼容；FFmpeg 9.0.2 `av1dec.c` 已在所有输出帧（含 `v4l2_request`）原生解析 AV1 OBU `ITU-T T.35` DV RPU 并挂载 `AV_FRAME_DATA_DOVI_METADATA` / `AV_FRAME_DATA_DOVI_RPU_BUFFER`。
- 本质上 **P10.0 = AV1 版 P5、P10.1 = AV1 版 P8.1、P10.4 = AV1 版 P8.4**，已通过 `0033-videoplayer-av1-dolby-vision-profile10.patch` 放行并完善信息展示。

### 实施 Checklist
- [x] 核对 RK3588 VPU981 AV1 硬解输出的 10-bit DRM 像素格式（`rockchip_vpu981_postproc_fmts` 默认优先 `V4L2_PIX_FMT_NV15`）及 `pi-dv-engine` 兼容性
- [x] 核对 FFmpeg 9.0.2 V4L2-request AV1 硬解已通过 `export_metadata()` / `export_itut_t35()` 原生挂载 `AV_FRAME_DATA_DOVI_METADATA` 与 `AV_FRAME_DATA_DOVI_RPU_BUFFER`
- [x] 在 `CDVTunnelDRMPRIME::Supports()` / `Create()` 放行 Profile 10（按 `dv_bl_signal_compatibility_id` 映射 P10.0→P5、P10.1/10.4→P8），同步完善 `DRMPRIMEPlayerInfo`、`DVDStreamInfo`、`DVDFileInfo` 与 `VideoPlayer` 的 Profile 10.x 识别，已在 `192.168.1.180` 完成 `make image` 编译验证
- [x] Rock 5T 实机验证：实机播放《倩女幽魂》4K AV1 DV P10.1 正常走 `NV15` 零拷贝与 `pi-dv-engine`（`profile 10 at 3840x2160: Mali-G610 MC4, MMR float, output rendered in place`），点亮杜比视界且色彩正常

---

## 任务 6：后续可选优化项（低优先级）

- [ ] 简体中文显示设置与 Estuary 皮肤本地化（参考 Hades `25ad00489d`，按当前 `strings.po` ID 对齐）
- [ ] 评估是否需要基于 `pi-dv-engine` 独立实现 DV Level 5 三态设置（Hades `6b5d1539f2` 强绑定 `dvbridge`，不可直接移植）
- [ ] 可选探索：在 `linux.aarch64.conf` 启用 `CONFIG_VIDEO_SYNOPSYS_HDMIRX=m` 测试 Rock 5T HDMI-IN（流光溢彩 Hyperion / V4L2 预览）
