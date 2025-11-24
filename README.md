# Ryzen 7 8700G の APU と AMD ROCm による Stable Diffusion + ComfyUI の生成AI環境構築

- 生成AIと協同しながら環境構築し、SD1.5のサンプル画像の生成までできたので、改めて、その手順を整理する。
- 生成AIを気軽に試してみるかと思ったが、かなり手こずったので、ここに手順を整理した
- この文書の作成者はインフラ・サーバエンジニア＆バックエンドアプリエンジニアが書いている

# この文書の目的

- 画像生成AIをまずは低予算で触ってみてその動作やシステム構成を理解したいという方向けに構築手順を説明する
  - LinuxやDocker、githubなどの知識を一通り理解できるようになるので、アプリ開発者の学習教材にはなりそう

# 対象読者の前提知識

- Linux : Ubuntu で動かしているため。基本的なコマンドとCLIの使い方
- Docker : AMD から提供される ROCm は Docker イメージで提供されており、それを利用するのが推奨のため
‐ github : github リポジトリを複製して、これを使うため。基本的なコマンドとgithubのお作法

以上の知識が基本レベルでないと、環境構築に躓き、時間がかかります。

# 環境構築した環境
## ハードウェア環境

AMD ROCm による Stable Diffusion + ComfyUI が稼働している環境を下記に示す

- MB: Asrock Steel Legend B650E
- CPU: Ryzen 7 8700G ( RDNA3 の GPUを搭載する, gfx1103 という型番らしいが、gfx1100 でないと動かなかった)
　- [Compatibility matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html#compatibility-matrix) で対応しているROCmバージョンを事前に確認しておくとよい
- MEM: 128GB ( DDR-5600 64GB x 2 )
- GPU: なし. 上記の APU の 内蔵GPUのみで動かす。
- SSD: 1TB NVME Gen 4

## ソフトウェア環境

AMD ROCm による Stable Diffusion + ComfyUI が稼働している環境を下記に示す

- OS: Ubuntu 24.04.03 LTS ※ AMDの公式も、 Ubuntu 24.04を記載するようになったので、ROCmの最新版(7.1, 2025/11時点)を試すなら、今あえて、22.04にする必要ないと思う
  ‐ Windows 11 Pro 25H2 + WSL2 + Ubuntu 24.04で最初は環境構築しようとしたが、AMD所定のカーネルモジュールをカーネルに読み込ませたが、GPUとして認識できず、2,3日ほど格闘したが諦めて、素のLinux環境にした
  - AMD ROCmの開発も、Ubuntu環境で開発されているようなので、Linux ディスクトリビューションに拘りがなければ、Ubuntuを進める
  - Kernel: 6.14.0-36-generic
  - kernel modules:
  ```bash
  hnishi@rz8700g:~/workdir/build-stbdff-with-rocm-ryz8700g$ lsmod | sort
Module                  Size  Used by
aesni_intel           122880  5
af_alg                 32768  6 algif_hash,algif_skcipher
ahci                   49152  0
algif_hash             16384  1
algif_skcipher         16384  1
amd_atl                69632  1
amd_sched              61440  1 amdgpu
amddrm_buddy           24576  1 amdgpu
amddrm_exec            12288  1 amdgpu
amddrm_ttm_helper      12288  1 amdgpu
amdgpu              19824640  35
amdkcl                 36864  4 amd_sched,amdttm,amddrm_exec,amdgpu
amdttm                131072  2 amdgpu,amddrm_ttm_helper
amdxcp                 16384  1 amdgpu
amdxdna               139264  0
autofs4                57344  2
binfmt_misc            24576  1
bluetooth            1011712  49 btrtl,btmtk,btintel,btbcm,bnep,btusb,rfcomm
bnep                   32768  2
bridge                421888  0
btbcm                  24576  1 btusb
btintel                69632  1 btusb
btmtk                  36864  1 btusb
btrtl                  32768  1 btusb
btusb                  73728  0
ccp                   155648  1 kvm_amd
cec                    94208  2 drm_display_helper,amdgpu
cfg80211             1437696  4 mt76,mac80211,mt7921_common,mt76_connac_lib
cmac                   12288  4
cryptd                 24576  3 crypto_simd,ghash_clmulni_intel
crypto_simd            16384  1 aesni_intel
dmi_sysfs              24576  0
drm_display_helper    278528  1 amdgpu
drm_panel_backlight_quirks    12288  1 amdgpu
drm_suballoc_helper    20480  1 amdgpu
drm_ttm_helper         16384  1 amdgpu
edac_mce_amd           28672  0
efi_pstore             12288  0
ghash_clmulni_intel    16384  0
gpio_amdpt             16384  0
gpu_sched              61440  1 amdxdna
hid                   262144  2 usbhid,hid_generic
hid_generic            12288  0
i2c_algo_bit           16384  1 amdgpu
i2c_piix4              32768  0
i2c_smbus              20480  1 i2c_piix4
input_leds             12288  0
intel_rapl_common      53248  1 intel_rapl_msr
intel_rapl_msr         20480  0
ip6_udp_tunnel         16384  1 vxlan
ip_set                 61440  1 xt_set
ip_tables              32768  0
irqbypass              12288  1 kvm
joydev                 32768  0
k10temp                16384  0
kvm                  1425408  1 kvm_amd
kvm_amd               245760  0
libahci                53248  1 ahci
libarc4                12288  1 mac80211
llc                    16384  2 bridge,stp
lp                     28672  0
mac80211             1814528  4 mt792x_lib,mt76,mt7921_common,mt76_connac_lib
mac_hid                12288  0
msr                    12288  0
mt76                  155648  4 mt792x_lib,mt7921e,mt7921_common,mt76_connac_lib
mt76_connac_lib       106496  3 mt792x_lib,mt7921e,mt7921_common
mt7921_common          90112  1 mt7921e
mt7921e                20480  0
mt792x_lib             69632  2 mt7921e,mt7921_common
nf_conntrack          200704  5 xt_conntrack,nf_nat,xt_nat,nf_conntrack_netlink,xt_MASQUERADE
nf_conntrack_netlink    57344  0
nf_defrag_ipv4         12288  1 nf_conntrack
nf_defrag_ipv6         24576  1 nf_conntrack
nf_nat                 61440  3 xt_nat,nft_chain_nat,xt_MASQUERADE
nf_tables             385024  96 nft_compat,nft_chain_nat
nfnetlink              20480  6 nft_compat,nf_conntrack_netlink,nf_tables,ip_set
nft_chain_nat          12288  5
nft_compat             20480  7
nls_iso8859_1          12288  1
nvme                   61440  2
nvme_auth              28672  1 nvme_core
nvme_core             225280  3 nvme
overlay               217088  1
parport                73728  3 parport_pc,lp,ppdev
parport_pc             53248  0
polyval_clmulni        12288  0
polyval_generic        12288  1 polyval_clmulni
ppdev                  24576  0
qrtr                   53248  2
r8169                 126976  0
rapl                   20480  0
rc_core                73728  1 cec
realtek                49152  1
rfcomm                102400  0
sch_fq_codel           24576  2
sha1_ssse3             32768  0
sha256_ssse3           32768  0
snd                   143360  17 snd_hda_codec_generic,snd_seq,snd_seq_device,snd_hda_codec_hdmi,snd_hwdep,snd_hda_intel,snd_hda_codec,snd_hda_codec_realtek,snd_timer,snd_pcm,snd_rawmidi
snd_hda_codec         204800  4 snd_hda_codec_generic,snd_hda_codec_hdmi,snd_hda_intel,snd_hda_codec_realtek
snd_hda_codec_generic   122880  1 snd_hda_codec_realtek
snd_hda_codec_hdmi     98304  1
snd_hda_codec_realtek   212992  1
snd_hda_core          147456  5 snd_hda_codec_generic,snd_hda_codec_hdmi,snd_hda_intel,snd_hda_codec,snd_hda_codec_realtek
snd_hda_intel          61440  2
snd_hda_scodec_component    20480  1 snd_hda_codec_realtek
snd_hrtimer            12288  1
snd_hwdep              20480  1 snd_hda_codec
snd_intel_dspcfg       45056  1 snd_hda_intel
snd_intel_sdw_acpi     16384  1 snd_intel_dspcfg
snd_pcm               192512  4 snd_hda_codec_hdmi,snd_hda_intel,snd_hda_codec,snd_hda_core
snd_rawmidi            57344  1 snd_seq_midi
snd_seq               122880  9 snd_seq_midi,snd_seq_midi_event,snd_seq_dummy
snd_seq_device         16384  3 snd_seq,snd_seq_midi,snd_rawmidi
snd_seq_dummy          12288  0
snd_seq_midi           24576  0
snd_seq_midi_event     16384  1 snd_seq_midi
snd_timer              53248  3 snd_seq,snd_hrtimer,snd_pcm
soundcore              16384  1 snd
spd5118                12288  0
stp                    12288  1 bridge
ttm                   118784  1 drm_ttm_helper
udp_tunnel             32768  1 vxlan
usbhid                 77824  0
veth                   45056  0
video                  77824  1 amdgpu
vxlan                 159744  0
wmi                    28672  2 video,wmi_bmof
wmi_bmof               12288  0
x_tables               65536  8 xt_conntrack,nft_compat,xt_tcpudp,xt_addrtype,xt_nat,xt_set,ip_tables,xt_MASQUERADE
xfrm_algo              16384  1 xfrm_user
xfrm_user              65536  1
xt_MASQUERADE          16384  1
xt_addrtype            12288  4
xt_conntrack           12288  1
xt_nat                 12288  1
xt_set                 20480  0
xt_tcpudp              16384  0
hnishi@rz8700g:~/workdir/build-stbdff-with-rocm-ryz8700g$ 
  ```
- Docker: Ubuntu で提供されているものをそのまま利用する
  - Docker IO: docker.io/noble-updates,now 28.2.2-0ubuntu1~24.04.1 amd64
  - docker-compose: docker-compose-v2/noble-updates,now 2.37.1+ds1-0ubuntu2~24.04.1
- AMD GPU Driver: 後の手順で説明する
  - amdgpu-core/noble,noble,now 1:7.1.70100-2238427.24.04
  - amdgpu-dkms-firmware/noble,noble,now 30.20.0.0.30200000-2238411.24.04
  - amdgpu-dkms/noble,noble,now 1:6.16.6.30200000-2238411.24.04
  - amdgpu-install/noble,noble,now 30.20.0.0.30200000-2238411.24.04
  - etc
- ROCm: 後の手順で説明する
  - rocm-core/noble,now 7.1.0.70100-20~24.04

# 環境構築手順
## 概要

1. Ubuntu 24.04.03 LTSのインストール ※説明省略
2. Docker のインストール ※説明省略
3. github CLIのインストールとgitrhubアカウントの登録 ※説明省略
4. AMD GPU Driver と ROCm のインストール
5. Pytorch のインストール
6. Stable Diffusion のインストール
7. ComfyUI のインストールとセットアップ
8. 生成AIモデルのインストールとセットアップ
9. 動かしてみる

## 4. AMD GPU Driver と ROCm のインストール


# 参考
- [AMD ROCm documentation](https://rocm.docs.amd.com/en/latest/)
