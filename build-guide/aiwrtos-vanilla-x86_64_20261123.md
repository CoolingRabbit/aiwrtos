# aiWrtOS-Vanilla-X86_64_20261123

本文记录从空环境编译 OpenWrt x86_64 自定义固件的完整流程，内置 OpenClash、PassWall、PassWall2、Tailscale、iStore，以及 Glass、Argon、Aurora 三个主题。文档不附带预编译固件或第三方源码包，所有组件均从各自官方渠道获取。

版本表记录 2026-11-23 的核对结果。实际构建时应执行文中的版本查询命令，以官方当日发布为准。

## 一、构建目标与版本核对

截至 2026-11-23 核对时，各组件版本如下：

| 组件 | 核对版本 | 官方来源 |
|---|---:|---|
| OpenWrt | 25.12.6 | [稳定版目录](https://downloads.openwrt.org/releases/) |
| OpenClash | v0.48.002 | [Releases](https://github.com/vernesong/OpenClash/releases/latest) |
| Mihomo | v1.19.34 | [Releases](https://github.com/MetaCubeX/mihomo/releases/latest) |
| PassWall | 26.11.11 | [源码](https://github.com/Openwrt-Passwall/openwrt-passwall) |
| PassWall2 | 26.11.11 | [源码](https://github.com/Openwrt-Passwall/openwrt-passwall2) |
| PassWall 专用依赖 | 跟随主分支 | [源码](https://github.com/Openwrt-Passwall/openwrt-passwall-packages) |
| Tailscale | 1.104.1，随 OpenWrt 稳定版 feed | [packages/net/tailscale](https://github.com/openwrt/packages/tree/master/net/tailscale) |
| iStore | 0.2.1-r1 | [源码](https://github.com/istoreos/istore) |
| Glass | 1.1.6 | [源码](https://github.com/rchen14b/luci-theme-glass) |
| Argon | 3.1.4 | [源码](https://github.com/jerrykuku/luci-theme-argon) |
| Aurora | 1.4.3 | [源码](https://github.com/eamonxg/luci-theme-aurora) |

Tailscale 采用 OpenWrt 当前稳定版锁定的 packages feed，而不是强行替换为上游最新提交，这样更符合当前构建系统的兼容要求。

## 二、准备构建虚拟机

本次实测配置：

- 操作系统：Linux Mint 22.3；
- CPU：8 核；
- 内存：4 GiB；
- 磁盘：80 GiB（实际 62 GiB 可用磁盘）。

实测过程中曾出现内存不足和磁盘写满，因此新构建环境建议使用以下配置：

- 操作系统：Linux Mint 22.3 或 Ubuntu 24.04 LTS；
- CPU：8 核；
- 内存：推荐 16 GiB，最低 8 GiB；
- 磁盘：推荐 150 GiB，最低 120 GiB；
- 网络：能够访问 OpenWrt 官方 Git、GitHub 及软件源；
- 使用普通用户构建，不要直接使用 root。

OpenWrt 官方构建说明：[Build System Setup](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem)。

安装编译工具和版本查询工具：

```bash
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk \
  gcc-multilib gettext git libncurses-dev libssl-dev \
  python3-dev python3-setuptools rsync swig unzip zlib1g-dev \
  file wget curl ca-certificates libelf-dev libdw-dev \
  dwarves pahole qemu-utils time jq
```

国内环境可以将 Ubuntu 软件源切换为 USTC。Linux Mint 自有仓库不能直接改成不支持它的 Ubuntu 镜像。

## 三、查询当天的稳定版本

查询 OpenWrt 最新稳定版，排除 RC 和 Snapshot：

```bash
OPENWRT_VERSION=$(curl -fsSL https://downloads.openwrt.org/releases/ \
  | grep -oE 'href="[0-9]+\.[0-9]+\.[0-9]+/"' \
  | cut -d'"' -f2 | tr -d '/' | sort -V | tail -n 1)

test -n "$OPENWRT_VERSION" || { echo "未获取到 OpenWrt 版本"; exit 1; }
echo "$OPENWRT_VERSION"
```

查询 OpenClash 和 Mihomo 最新 Release：

```bash
OPENCLASH_TAG=$(curl -fsSL \
  https://api.github.com/repos/vernesong/OpenClash/releases/latest \
  | jq -r '.tag_name')

MIHOMO_TAG=$(curl -fsSL \
  https://api.github.com/repos/MetaCubeX/mihomo/releases/latest \
  | jq -r '.tag_name')

printf 'OpenClash: %s\nMihomo: %s\n' "$OPENCLASH_TAG" "$MIHOMO_TAG"
```

GitHub API 未认证请求有速率限制，失败时应先检查网络和返回内容，不要用一个猜测的版本号继续。

## 四、获取 OpenWrt 源码和 feeds

```bash
git clone --depth 1 --branch "v$OPENWRT_VERSION" \
  https://git.openwrt.org/openwrt/openwrt.git "$HOME/openwrt"
cd "$HOME/openwrt"

cp feeds.conf.default feeds.conf
sed -i 's|https://github.com/openwrt/video.git|https://git.openwrt.org/feed/video.git|' feeds.conf

./scripts/feeds update -a
./scripts/feeds install -a
```

核对源码：

```bash
git describe --tags --always
git rev-parse HEAD
grep '^src-git' feeds.conf
```

如果 GitHub 无法访问，OpenWrt 主仓库和 video feed 都使用上述官方 `git.openwrt.org` 地址，不要直接换成来源不明的镜像。

## 五、获取全部自定义组件

将第三方源码集中放在构建目录之外，便于核对和保留许可证：

```bash
SRC="$HOME/aiwrtos-sources"
mkdir -p "$SRC" package/custom

git clone --depth 1 --branch "$OPENCLASH_TAG" \
  https://github.com/vernesong/OpenClash.git "$SRC/OpenClash"
git clone --depth 1 \
  https://github.com/Openwrt-Passwall/openwrt-passwall.git "$SRC/openwrt-passwall"
git clone --depth 1 \
  https://github.com/Openwrt-Passwall/openwrt-passwall2.git "$SRC/openwrt-passwall2"
git clone --depth 1 \
  https://github.com/Openwrt-Passwall/openwrt-passwall-packages.git "$SRC/passwall-packages"
git clone --depth 1 \
  https://github.com/istoreos/istore.git "$SRC/istore"
git clone --depth 1 \
  https://github.com/rchen14b/luci-theme-glass.git "$SRC/luci-theme-glass"
git clone --depth 1 \
  https://github.com/jerrykuku/luci-theme-argon.git "$SRC/luci-theme-argon"
git clone --depth 1 \
  https://github.com/eamonxg/luci-theme-aurora.git "$SRC/luci-theme-aurora"
```

复制到 OpenWrt 的自定义 package 目录：

```bash
cp -a "$SRC/OpenClash/luci-app-openclash" package/custom/
cp -a "$SRC/OpenClash/LICENSE" package/custom/luci-app-openclash/LICENSE

cp -a "$SRC/openwrt-passwall/luci-app-passwall" package/custom/
cp -a "$SRC/openwrt-passwall/LICENSE" package/custom/luci-app-passwall/LICENSE
cp -a "$SRC/openwrt-passwall2/luci-app-passwall2" package/custom/
cp -a "$SRC/openwrt-passwall2/LICENSE" package/custom/luci-app-passwall2/LICENSE
cp -a "$SRC/passwall-packages" package/custom/passwall-packages

for pkg in luci-app-store luci-lib-taskd luci-lib-xterm taskd; do
  cp -a "$SRC/istore/luci/$pkg" package/custom/
  cp -a "$SRC/istore/LICENSE" "package/custom/$pkg/LICENSE"
done

for theme in luci-theme-glass luci-theme-argon luci-theme-aurora; do
  cp -a "$SRC/$theme" "package/custom/$theme"
done
```

## 六、处理 PassWall 专用依赖

PassWall 官方要求用其专用 packages 仓库替换一批同名包，避免核心版本与当前插件配置不匹配。官方说明以 [openwrt-passwall README](https://github.com/Openwrt-Passwall/openwrt-passwall) 为准。

当前上游列出的替换范围如下；未来执行时也应重新核对 README：

```bash
mkdir -p "$HOME/aiwrtos-feed-backup"
for pkg in xray-core v2ray-geodata sing-box chinadns-ng dns2socks \
  hysteria ipt2socks microsocks naiveproxy shadowsocks-rust \
  shadowsocksr-libev simple-obfs tcping v2ray-plugin xray-plugin \
  geoview shadow-tls smartdns; do
  official=$(find feeds/packages -path "*/$pkg/Makefile" -print -quit)
  if [ -n "$official" ]; then
    mv "$(dirname "$official")" "$HOME/aiwrtos-feed-backup/$pkg"
  fi
  rm -f "package/feeds/packages/$pkg"
done

official_passwall=$(find feeds/luci -path '*/luci-app-passwall/Makefile' -print -quit)
if [ -n "$official_passwall" ]; then
  mv "$(dirname "$official_passwall")" "$HOME/aiwrtos-feed-backup/luci-app-passwall"
fi
rm -f package/feeds/luci/luci-app-passwall
```

这里替换的是 Xray、Sing-Box、Hysteria、Shadowsocks 等代理核心及 DNS/分流工具，不是 Linux 内核或网卡驱动。

### Go 版本必须按源码要求选择

在 OpenWrt 25.12.6 中，默认 Go 为 1.26，而当前 Xray 的 `go.mod` 要求 Go 1.27。应只修改 Xray 的构建依赖：

```bash
sed -i 's|^PKG_BUILD_DEPENDS:=golang/host$|PKG_BUILD_DEPENDS:=golang1.27/host|' \
  package/custom/passwall-packages/xray-core/Makefile
```

未来版本不要机械照抄这一条。正确检查方式是：

```bash
find build_dir -name go.mod -exec grep -H '^go ' {} \;
ls feeds/packages/lang/golang/golang1.*
grep '^GO_DEFAULT_VERSION' feeds/packages/lang/golang/golang-values.mk
```

如果日志出现 `go.mod requires go >= X.Y`，就把对应软件包的 `PKG_BUILD_DEPENDS` 改成 feed 中满足要求的 `golangX.Y/host`，不要全局替换所有包的 Go 版本。

## 七、内置 Mihomo 内核

OpenClash 是 LuCI 管理界面，Mihomo 是实际运行的代理内核。下面下载最新 Release 的通用 amd64 版本，并校验 GitHub 提供的 SHA-256：

```bash
curl -fsSL https://api.github.com/repos/MetaCubeX/mihomo/releases/latest \
  -o /tmp/mihomo-release.json

MIHOMO_URL=$(jq -r '.assets[] \
  | select(.name | test("^mihomo-linux-amd64-compatible-.*\\.gz$")) \
  | .browser_download_url' /tmp/mihomo-release.json)
MIHOMO_SHA256=$(jq -r '.assets[] \
  | select(.name | test("^mihomo-linux-amd64-compatible-.*\\.gz$")) \
  | .digest | sub("^sha256:"; "")' /tmp/mihomo-release.json)

test -n "$MIHOMO_URL" && test -n "$MIHOMO_SHA256"
curl -fL "$MIHOMO_URL" -o /tmp/mihomo.gz
echo "$MIHOMO_SHA256  /tmp/mihomo.gz" | sha256sum -c -

mkdir -p files/etc/openclash/core
gzip -dc /tmp/mihomo.gz > files/etc/openclash/core/clash_meta
chmod 0755 files/etc/openclash/core/clash_meta
files/etc/openclash/core/clash_meta -v
```

`compatible` 版本适合不确定 CPU 指令集等级的通用 x86_64 软路由。确认设备支持更高指令集后，也可以选择相应的 v2/v3 构建。

## 八、配置默认及备用 APK 源

OpenWrt 25.12 使用 apk。默认源使用 USTC，官方源和 TUNA 作为备用，不同时启用多个内容相同的仓库：

```bash
mkdir -p files/etc/apk/repositories.d
cat > files/etc/apk/repositories.d/alternatives.list <<EOF
# To switch: comment out distfeeds.list, uncomment exactly one provider,
# then run: apk update. No custom-kernel modules are provided here.

# Official OpenWrt
# https://downloads.openwrt.org/releases/$OPENWRT_VERSION/targets/x86/64/packages/packages.adb
# https://downloads.openwrt.org/releases/$OPENWRT_VERSION/packages/x86_64/base/packages.adb
# https://downloads.openwrt.org/releases/$OPENWRT_VERSION/packages/x86_64/luci/packages.adb
# https://downloads.openwrt.org/releases/$OPENWRT_VERSION/packages/x86_64/packages/packages.adb
# https://downloads.openwrt.org/releases/$OPENWRT_VERSION/packages/x86_64/routing/packages.adb
# https://downloads.openwrt.org/releases/$OPENWRT_VERSION/packages/x86_64/telephony/packages.adb
# https://downloads.openwrt.org/releases/$OPENWRT_VERSION/packages/x86_64/video/packages.adb

# TUNA
# https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/$OPENWRT_VERSION/targets/x86/64/packages/packages.adb
# https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/$OPENWRT_VERSION/packages/x86_64/base/packages.adb
# https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/$OPENWRT_VERSION/packages/x86_64/luci/packages.adb
# https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/$OPENWRT_VERSION/packages/x86_64/packages/packages.adb
# https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/$OPENWRT_VERSION/packages/x86_64/routing/packages.adb
# https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/$OPENWRT_VERSION/packages/x86_64/telephony/packages.adb
# https://mirrors.tuna.tsinghua.edu.cn/openwrt/releases/$OPENWRT_VERSION/packages/x86_64/video/packages.adb
EOF
```

不要给自定义内核直接配置官方 kmod 仓库。即使 OpenWrt 版本号相同，官方模块也可能与自定义内核配置不匹配。

## 九、生成 x86_64 完整配置

下面的配置同时启用 BIOS、UEFI、1024 MiB 根分区、LuCI 中文翻译和全部目标组件：

```bash
cat > .config <<EOF
CONFIG_TARGET_x86=y
CONFIG_TARGET_x86_64=y
CONFIG_TARGET_x86_64_DEVICE_generic=y
CONFIG_GRUB_IMAGES=y
CONFIG_GRUB_EFI_IMAGES=y
CONFIG_TARGET_ROOTFS_PARTSIZE=1024
CONFIG_IMAGEOPT=y
CONFIG_VERSIONOPT=y
CONFIG_VERSION_REPO="https://mirrors.ustc.edu.cn/openwrt/releases/$OPENWRT_VERSION"

CONFIG_LUCI_LANG_zh_Hans=y
CONFIG_PACKAGE_luci=y
CONFIG_PACKAGE_luci-base=y
CONFIG_PACKAGE_luci-compat=y

CONFIG_PACKAGE_luci-app-openclash=y
CONFIG_PACKAGE_luci-app-passwall=y
CONFIG_PACKAGE_luci-app-passwall2=y
CONFIG_PACKAGE_tailscale=y
CONFIG_PACKAGE_luci-app-store=y
CONFIG_PACKAGE_luci-theme-glass=y
CONFIG_PACKAGE_luci-theme-argon=y
CONFIG_PACKAGE_luci-theme-aurora=y

# CONFIG_PACKAGE_dnsmasq is not set
CONFIG_PACKAGE_dnsmasq-full=y
CONFIG_PACKAGE_dnsmasq_full_nftset=y
CONFIG_PACKAGE_firewall4=y
CONFIG_PACKAGE_kmod-nft-tproxy=y
CONFIG_PACKAGE_kmod-nft-socket=y
CONFIG_PACKAGE_kmod-nft-nat=y
CONFIG_PACKAGE_kmod-tun=y
CONFIG_PACKAGE_kmod-inet-diag=y

CONFIG_PACKAGE_bash=y
CONFIG_PACKAGE_curl=y
CONFIG_PACKAGE_ca-bundle=y
CONFIG_PACKAGE_ip-full=y
CONFIG_PACKAGE_ruby=y
CONFIG_PACKAGE_ruby-yaml=y
CONFIG_PACKAGE_unzip=y
CONFIG_PACKAGE_wget-ssl=y
CONFIG_PACKAGE_jsonfilter=y
CONFIG_PACKAGE_ucode-mod-math=y
CONFIG_PACKAGE_flock=y

CONFIG_PACKAGE_chinadns-ng=y
CONFIG_PACKAGE_dns2socks=y
CONFIG_PACKAGE_microsocks=y
CONFIG_PACKAGE_resolveip=y
CONFIG_PACKAGE_tcping=y
CONFIG_PACKAGE_lyaml=y
CONFIG_PACKAGE_geoview=y
CONFIG_PACKAGE_v2ray-geoip=y
CONFIG_PACKAGE_v2ray-geosite=y
CONFIG_PACKAGE_libuci-lua=y
CONFIG_PACKAGE_lua=y
CONFIG_PACKAGE_luci-lib-jsonc=y

CONFIG_PACKAGE_coreutils=y
CONFIG_PACKAGE_coreutils-base64=y
CONFIG_PACKAGE_coreutils-nohup=y
CONFIG_PACKAGE_coreutils-timeout=y

CONFIG_PACKAGE_luci-lib-taskd=y
CONFIG_PACKAGE_taskd=y
CONFIG_PACKAGE_tar=y
CONFIG_PACKAGE_xz-utils=y
CONFIG_PACKAGE_mount-utils=y
EOF

make defconfig
```

检查关键包是否在展开配置后仍为 `y`：

```bash
for symbol in \
  CONFIG_TARGET_x86_64_DEVICE_generic \
  CONFIG_LUCI_LANG_zh_Hans \
  CONFIG_PACKAGE_luci-app-openclash \
  CONFIG_PACKAGE_luci-app-passwall \
  CONFIG_PACKAGE_luci-app-passwall2 \
  CONFIG_PACKAGE_tailscale \
  CONFIG_PACKAGE_luci-app-store \
  CONFIG_PACKAGE_luci-theme-glass \
  CONFIG_PACKAGE_luci-theme-argon \
  CONFIG_PACKAGE_luci-theme-aurora; do
  grep -qx "$symbol=y" .config || { echo "MISSING: $symbol"; exit 1; }
done
```

如果某个包消失，应先在 `tmp/.config-package.in` 检查其 `depends on`，不要盲目反复执行 `make defconfig`。

## 十、编译固件

国内 Go 模块使用可验证连通的镜像：

```bash
export GOPROXY="https://goproxy.cn,direct"
free -h
df -h .
```

8 GiB 内存使用：

```bash
make -j2 V=s > build.log 2>&1
```

16 GiB 内存可以使用 `make -j4`。首次构建需要编译工具链、Rust、Go、内核和全部软件包，耗时会明显长于后续增量构建。

### 常见错误的正确处理

- `cc1plus: fatal error: Killed signal`：通常是 OOM。降低并行度、增加内存或交换空间，不要先怀疑插件源码。
- `proxy.golang.org ... i/o timeout`：确认当前 shell 的 `GOPROXY=https://goproxy.cn,direct` 已生效。
- `go.mod requires go >= ...`：按第六节的说明给对应包选择满足要求的 Go 工具链。
- `No space left on device`：先检查 `df -h` 和 `du -sh build_dir/*`。只清理确定可重建且不会被当前构建立即重新要求的文件，不要删除 Rust 中间目录来“救急”，否则会触发数小时的重编。
- GitHub 连接失败：在能访问 GitHub 的机器上下载官方仓库或 `codeload.github.com` 源码包并上传，核对提交或 Release；不要换成不明第三方镜像。

后续更新时保留 `build_dir`、`staging_dir`、`dl` 和 `.config`，直接重新执行 `make -j2` 即可增量编译。`make dirclean` 会清除大量中间成果，不应作为常规更新步骤。

## 十一、验证产物

镜像输出目录：

```bash
cd "$HOME/openwrt"
ls -lh bin/targets/x86/64/
```

四个主要镜像代表两种启动方式和两种根文件系统：

| 启动方式 | 文件系统 | 文件名 | 用途 |
|---|---|---|---|
| UEFI | SquashFS | `openwrt-$OPENWRT_VERSION-x86-64-generic-squashfs-combined-efi.img.gz` | 推荐大多数现代 x86 软路由 |
| BIOS | SquashFS | `openwrt-$OPENWRT_VERSION-x86-64-generic-squashfs-combined.img.gz` | 老设备或传统 BIOS |
| UEFI | ext4 | `openwrt-$OPENWRT_VERSION-x86-64-generic-ext4-combined-efi.img.gz` | 需要可写 ext4 根分区的高级用户 |
| BIOS | ext4 | `openwrt-$OPENWRT_VERSION-x86-64-generic-ext4-combined.img.gz` | 传统 BIOS 加 ext4 |

验证压缩包、关键软件包和校验值：

```bash
for image in bin/targets/x86/64/*combined*.img.gz; do
  gzip -t "$image"
done

for pkg in luci-app-openclash luci-app-passwall luci-app-passwall2 \
  tailscale luci-app-store luci-theme-glass \
  luci-theme-argon luci-theme-aurora; do
  grep -E "^$pkg - " \
    bin/targets/x86/64/openwrt-*-x86-64-generic.manifest \
    || { echo "MISSING: $pkg"; exit 1; }
done

sha256sum bin/targets/x86/64/*combined*.img.gz
```

最后还应确认默认仓库和 Mihomo：

```bash
head -n 2 build_dir/target-*/root-x86/etc/apk/repositories.d/distfeeds.list
build_dir/target-*/root-x86/etc/openclash/core/clash_meta -v
```

编译成功只证明产物已生成并通过基本检查，不代表已经通过实际软路由的启动、网卡识别、代理和 Tailscale 功能验收。首次刷机前应备份原配置，并在备用设备或可恢复环境中测试。

## 十二、首次使用注意事项

- 默认 LuCI 地址通常为 `http://192.168.1.1`。
- 原版 OpenWrt 不自带系统级首次开机向导；OpenClash 当前版本提供插件首页初始引导。
- OpenClash、PassWall、PassWall2 可以同时装入固件，但不要同时启动，以免 DNS、防火墙和透明代理规则互相冲突。
- 刷机后仍需添加自己的订阅或节点配置，并单独登录 Tailscale。
- 三个主题只改变外观；插件中文是否完整取决于对应语言包和上游翻译覆盖情况。
