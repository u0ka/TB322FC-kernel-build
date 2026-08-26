# Android Kernel GKI Builder

使用 GitHub Actions 自动编译 Android Common Kernel (GKI)，支持自定义内核参数。

## 支持的分支

- `common-android15-6.6` (Android 15, Kernel 6.6)
- `common-android14-6.1`
- `common-android14-5.15`
- `common-android13-5.15`
- 以及其他 `common-androidXX-X.XX` 格式的分支

## 快速开始

### 1. Fork 本仓库

将本仓库 Fork 到你自己的 GitHub 账号下。

### 2. 触发构建

1. 进入仓库的 **Actions** 标签页
2. 左侧选择 **Build Android Kernel (GKI)**
3. 点击 **Run workflow**
4. 填写参数后点击运行

### 3. 下载产物

构建完成后，在工作流运行页面的 **Artifacts** 区域下载编译好的内核包。

## 构建参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `manifest_branch` | 内核 manifest 分支名 | `common-android15-6.6` |
| `arch` | 目标架构 (`aarch64` / `x86_64`) | `aarch64` |
| `custom_config` | 自定义内核配置（每行一个） | 空 |
| `local_version` | 内核版本后缀（如 `-mykernel`） | 空 |
| `build_boot_img` | 是否构建 GKI boot 镜像 | `true` |
| `debug_build` | 是否为 Debug 构建 | `false` |
| `enable_ccache` | 是否启用 ccache 加速 | `true` |

## 自定义内核参数

在 `custom_config` 输入框中，每行填写一个内核配置项，例如：

```
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBE_EVENTS=y
CONFIG_UPROBES=y
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
# CONFIG_MODULE_SIG_FORCE is not set
CONFIG_TCP_CONG_BBR=y
CONFIG_NETFILTER_XT_MATCH_BPF=m
```

### 工作原理

1. 你输入的配置会被写入 `common/custom_defconfig` 文件
2. 自动在 `common/BUILD.bazel` 中注册该文件
3. 构建时通过 Bazel 的 `--defconfig_fragment` 参数应用
4. Bazel 会在 `.config` 生成后校验所有 fragment 是否生效

### 常用自定义配置示例

#### 启用 KernelSU 基础支持
```
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBE_EVENTS=y
CONFIG_UPROBES=y
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_HAVE_EBPF_JIT=y
```

#### 启用 Docker / 容器支持
```
CONFIG_NAMESPACES=y
CONFIG_NET_NS=y
CONFIG_PID_NS=y
CONFIG_IPC_NS=y
CONFIG_UTS_NS=y
CONFIG_USER_NS=y
CONFIG_CGROUPS=y
CONFIG_CGROUP_CPUACCT=y
CONFIG_CGROUP_DEVICE=y
CONFIG_CGROUP_FREEZER=y
CONFIG_CGROUP_SCHED=y
CONFIG_CPUSETS=y
CONFIG_MEMCG=y
CONFIG_KEYS=y
CONFIG_VETH=y
CONFIG_BRIDGE=y
CONFIG_BRIDGE_NETFILTER=y
CONFIG_IP_NF_FILTER=y
CONFIG_IP_NF_TARGET_MASQUERADE=y
CONFIG_NETFILTER_XT_MATCH_ADDRTYPE=y
CONFIG_NETFILTER_XT_MATCH_CONNTRACK=y
CONFIG_NETFILTER_XT_MATCH_IPVS=y
CONFIG_NETFILTER_XT_MARK=y
CONFIG_IP_NF_NAT=y
CONFIG_NF_NAT=y
CONFIG_POSIX_MQUEUE=y
CONFIG_NF_NAT_IPV4=y
CONFIG_NF_NAT_NEEDED=y
```

#### 启用 WireGuard
```
CONFIG_WIREGUARD=y
```

#### 启用 BBR 拥塞控制
```
CONFIG_TCP_CONG_BBR=y
CONFIG_NET_SCH_FQ=y
```

## 构建产物说明

编译完成后，`dist` 目录包含：

| 文件 | 说明 |
|------|------|
| `Image` | 未压缩的内核镜像 (arm64) |
| `Image.gz` | gzip 压缩的内核镜像 |
| `vmlinux` | 带调试信息的内核 ELF |
| `System.map` | 内核符号表 |
| `*.ko` | 内核模块文件 |
| `modules.load` | 模块加载列表 |
| `boot.img` | GKI boot 镜像（如启用） |
| `init_boot.img` | init_boot 镜像 |

## 本地构建

如需在本地构建，执行以下命令：

```bash
# 安装 repo
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH="$HOME/bin:$PATH"

# 配置 git
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# 初始化并同步源码
mkdir android-kernel && cd android-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b common-android15-6.6 --depth=1
repo sync -c -j$(nproc)

# （可选）添加自定义配置
cat > common/custom_defconfig << 'EOF'
CONFIG_KPROBES=y
EOF
# 在 common/BUILD.bazel 中添加 exports_files(["custom_defconfig"])

# 构建
mkdir -p dist
tools/bazel run //common:kernel_aarch64_dist -- --destdir=dist
# 带自定义配置:
# tools/bazel run //common:kernel_aarch64_dist -- --destdir=dist --defconfig_fragment=//common:custom_defconfig
```

## 注意事项

1. **构建时间**：首次构建约需 30-60 分钟，启用缓存后后续构建更快
2. **磁盘空间**：需要约 30-50GB 可用空间，工作流会自动清理 runner
3. **配置冲突**：如果自定义配置与 GKI 基础配置冲突，Bazel 会在构建时报错提示
4. **ABI 检查**：GKI 内核有严格的 ABI 检查，修改核心配置可能导致 ABI 检查失败
5. **模块签名**：GKI 默认启用模块签名，自定义模块需要正确签名才能加载

## 故障排查

### Bazel 找不到 custom_defconfig

确保 `common/BUILD.bazel` 中有：
```python
exports_files(["custom_defconfig"])
```

### 配置未生效

检查构建日志中是否有类似警告：
```
warning: option CONFIG_XXX not applied
```
这通常意味着该配置依赖的其他配置未启用，或者配置名拼写错误。

### 磁盘空间不足

工作流已包含磁盘清理步骤，如仍不足，可以：
- 减少并发编译数
- 启用 ccache 减少重复编译
- 选择更小的内核配置
