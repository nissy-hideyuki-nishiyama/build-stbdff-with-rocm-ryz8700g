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
6. 画像生成AIのユーザー環境の構築
7. Stable Diffusion のインストール
8. ComfyUI のインストールとセットアップ
9. 起動確認と Docker イメージへの保存
10. 動かしてみる
11. Dockerfile と docker-compose.yml の保存

## 4. AMD GPU Driver と ROCm のインストール

- Ubuntu を動かしている Dockerホストマシンに AMD GPU Driver と ROCm をインストールする
- [Quick start installation guide](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/quick-start.html#quick-start-installation-guide) の通りにコマンドを実行する

1. リポジトリの追加と ROCm のインストール

```bash
# AMD公式パッケージのダウンロードとインストール
wget https://repo.radeon.com/amdgpu-install/7.1/ubuntu/noble/amdgpu-install_7.1.70100-1_all.deb
sudo apt install ./amdgpu-install_7.1.70100-1_all.deb
# apt パッケージ情報の更新
sudo apt update
# Python3 のセットアップツールと wheel パッケージのインストール
sudo apt install python3-setuptools python3-wheel
# ユーザーのグループ設定の更新
sudo usermod -a -G render,video $LOGNAME # Add the current user to the render and video groups
# 
sudo apt install rocm
```

2. AMD GPU Driver のインストール

```bash
# AMD公式パッケージのダウンロードとインストール ※前に実施しているので不要
# wget https://repo.radeon.com/amdgpu-install/7.1/ubuntu/noble/amdgpu-install_7.1.70100-1_all.deb
# sudo apt install ./amdgpu-install_7.1.70100-1_all.deb
# apt パッケージ情報の更新
sudo apt update
# カーネルヘッダーとカーネルモジュールエクストラのインストール
sudo apt install "linux-headers-$(uname -r)" "linux-modules-extra-$(uname -r)"
sudo apt install amdgpu-dkms
```

3. システムの再起動

```bash
sudo reboot
```

4. AMD GPU Driver と ROCm の動作確認

```bash
# AMD GPU Driver の動作確認
# rocminfo 
ROCk module version 6.16.6 is loaded
=====================    
HSA System Attributes    
=====================    
Runtime Version:         1.18
Runtime Ext Version:     1.14
System Timestamp Freq.:  1000.000000MHz
Sig. Max Wait Duration:  18446744073709551615 (0xFFFFFFFFFFFFFFFF) (timestamp count)
Machine Model:           LARGE                              
System Endianness:       LITTLE                             
Mwaitx:                  DISABLED
XNACK enabled:           NO
DMAbuf Support:          YES
VMM Support:             YES

==========               
HSA Agents               
==========               
*******                  
Agent 1                  
*******                  
  Name:                    AMD Ryzen 7 8700G w/ Radeon 780M Graphics
  Uuid:                    CPU-XX                             
  Marketing Name:          AMD Ryzen 7 8700G w/ Radeon 780M Graphics
  Vendor Name:             CPU                                
  Feature:                 None specified                     
  Profile:                 FULL_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        0(0x0)                             
  Queue Min Size:          0(0x0)                             
  Queue Max Size:          0(0x0)                             
  Queue Type:              MULTI                              
  Node:                    0                                  
  Device Type:             CPU                                
  Cache Info:              
    L1:                      32768(0x8000) KB                   
  Chip ID:                 0(0x0)                             
  ASIC Revision:           0(0x0)                             
  Cacheline Size:          64(0x40)                           
  Max Clock Freq. (MHz):   5177                               
  BDFID:                   0                                  
  Internal Node ID:        0                                  
  Compute Unit:            16                                 
  SIMDs per CU:            0                                  
  Shader Engines:          0                                  
  Shader Arrs. per Eng.:   0                                  
  WatchPts on Addr. Ranges:1                                  
  Memory Properties:       
  Features:                None
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: FINE GRAINED        
      Size:                    114961888(0x6da2de0) KB            
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 2                   
      Segment:                 GLOBAL; FLAGS: EXTENDED FINE GRAINED
      Size:                    114961888(0x6da2de0) KB            
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 3                   
      Segment:                 GLOBAL; FLAGS: KERNARG, FINE GRAINED
      Size:                    114961888(0x6da2de0) KB            
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 4                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    114961888(0x6da2de0) KB            
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
  ISA Info:                
*******                  
Agent 2                  
*******                  
  Name:                    gfx1103                            # ※ OSにAPUの内蔵GPUが認識されていると、このように表示される
  Uuid:                    GPU-XX                             
  Marketing Name:          AMD Radeon Graphics                
  Vendor Name:             AMD                                
  Feature:                 KERNEL_DISPATCH                    
  Profile:                 BASE_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        128(0x80)                          
  Queue Min Size:          64(0x40)                           
  Queue Max Size:          131072(0x20000)                    
  Queue Type:              MULTI                              
  Node:                    1                                  
  Device Type:             GPU                                # ※
  Cache Info:              
    L1:                      32(0x20) KB                        
    L2:                      2048(0x800) KB                     
  Chip ID:                 5567(0x15bf)                       
  ASIC Revision:           12(0xc)                            
  Cacheline Size:          128(0x80)                          
  Max Clock Freq. (MHz):   2900                               
  BDFID:                   3072                               
  Internal Node ID:        1                                  
  Compute Unit:            12                                 
  SIMDs per CU:            2                                  
  Shader Engines:          1                                  
  Shader Arrs. per Eng.:   2                                  
  WatchPts on Addr. Ranges:4                                  
  Coherent Host Access:    FALSE                              
  Memory Properties:       APU
  Features:                KERNEL_DISPATCH 
  Fast F16 Operation:      TRUE                               
  Wavefront Size:          32(0x20)                           
  Workgroup Max Size:      1024(0x400)                        
  Workgroup Max Size per Dimension:
    x                        1024(0x400)                        
    y                        1024(0x400)                        
    z                        1024(0x400)                        
  Max Waves Per CU:        32(0x20)                           
  Max Work-item Per CU:    1024(0x400)                        
  Grid Max Size:           4294967295(0xffffffff)             
  Grid Max Size per Dimension:
    x                        2147483647(0x7fffffff)             
    y                        65535(0xffff)                      
    z                        65535(0xffff)                      
  Max fbarriers/Workgrp:   32                                 
  Packet Processor uCode:: 67                                 
  SDMA engine uCode::      23                                 
  IOMMU Support::          None                               
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    57480944(0x36d16f0) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:2048KB                             
      Alloc Alignment:         4KB                                
      Accessible by all:       FALSE                              
    Pool 2                   
      Segment:                 GLOBAL; FLAGS: EXTENDED FINE GRAINED
      Size:                    57480944(0x36d16f0) KB             
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:2048KB                             
      Alloc Alignment:         4KB                                
      Accessible by all:       FALSE                              
    Pool 3                   
      Segment:                 GROUP                              
      Size:                    64(0x40) KB                        
      Allocatable:             FALSE                              
      Alloc Granule:           0KB                                
      Alloc Recommended Granule:0KB                                
      Alloc Alignment:         0KB                                
      Accessible by all:       FALSE                              
  ISA Info:                
    ISA 1                    
      Name:                    amdgcn-amd-amdhsa--gfx1103         
      Machine Models:          HSA_MACHINE_MODEL_LARGE            
      Profiles:                HSA_PROFILE_BASE                   
      Default Rounding Mode:   NEAR                               
      Default Rounding Mode:   NEAR                               
      Fast f16:                TRUE                               
      Workgroup Max Size:      1024(0x400)                        
      Workgroup Max Size per Dimension:
        x                        1024(0x400)                        
        y                        1024(0x400)                        
        z                        1024(0x400)                        
      Grid Max Size:           4294967295(0xffffffff)             
      Grid Max Size per Dimension:
        x                        2147483647(0x7fffffff)             
        y                        65535(0xffff)                      
        z                        65535(0xffff)                      
      FBarrier Max Size:       32                                 
    ISA 2                    
      Name:                    amdgcn-amd-amdhsa--gfx11-generic   
      Machine Models:          HSA_MACHINE_MODEL_LARGE            
      Profiles:                HSA_PROFILE_BASE                   
      Default Rounding Mode:   NEAR                               
      Default Rounding Mode:   NEAR                               
      Fast f16:                TRUE                               
      Workgroup Max Size:      1024(0x400)                        
      Workgroup Max Size per Dimension:
        x                        1024(0x400)                        
        y                        1024(0x400)                        
        z                        1024(0x400)                        
      Grid Max Size:           4294967295(0xffffffff)             
      Grid Max Size per Dimension:
        x                        2147483647(0x7fffffff)             
        y                        65535(0xffff)                      
        z                        65535(0xffff)                      
      FBarrier Max Size:       32                                 
*******                  
Agent 3                  
*******                  
  Name:                    aie2                               
  Uuid:                    AIE-XX                             
  Marketing Name:          AIE-ML                             
  Vendor Name:             AMD                                
  Feature:                 AGENT_DISPATCH                     
  Profile:                 BASE_PROFILE                       
  Float Round Mode:        NEAR                               
  Max Queue Number:        1(0x1)                             
  Queue Min Size:          64(0x40)                           
  Queue Max Size:          64(0x40)                           
  Queue Type:              SINGLE                             
  Node:                    0                                  
  Device Type:             DSP                                
  Cache Info:              
    L2:                      2048(0x800) KB                     
  Chip ID:                 0(0x0)                             
  ASIC Revision:           0(0x0)                             
  Cacheline Size:          0(0x0)                             
  Max Clock Freq. (MHz):   0                                  
  BDFID:                   0                                  
  Internal Node ID:        0                                  
  Compute Unit:            0                                  
  SIMDs per CU:            0                                  
  Shader Engines:          0                                  
  Shader Arrs. per Eng.:   0                                  
  WatchPts on Addr. Ranges:0                                  
  Memory Properties:       
  Features:                AGENT_DISPATCH
  Pool Info:               
    Pool 1                   
      Segment:                 GLOBAL; FLAGS: KERNARG, COARSE GRAINED
      Size:                    114961888(0x6da2de0) KB            
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 2                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    65536(0x10000) KB                  
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:0KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
    Pool 3                   
      Segment:                 GLOBAL; FLAGS: COARSE GRAINED      
      Size:                    114961888(0x6da2de0) KB            
      Allocatable:             TRUE                               
      Alloc Granule:           4KB                                
      Alloc Recommended Granule:4KB                                
      Alloc Alignment:         4KB                                
      Accessible by all:       TRUE                               
  ISA Info:                
*** Done ***             
# 
```

## 5. Pytorch のインストール


- AMD がビルドした Docker イメージを利用して、Pytorch 環境を構築する。
  - Pytorch の Docker イメージから Docker コンテナを起動する
    - ROCm x ubuntu x python x pytotch のそれぞれのバージョンの組み合わせで複数のイメージが提供されている
    - 例えば、ROCm 7.1, Ubuntu 24.04, Python 3.12, Pytorch 2.8.0 の組み合わせのイメージは以下の通り
      - docker.io/rocm/pytorch:rocm7.1_ubuntu24.04_py3.12_pytorch_release_2.8.0 このイメージを今回は利用し、これは rocm/pytorch:latest としても参照できる。各自のソフトウェア環境に合わせて、適宜、イメージを選択する
  - Stable Diffusion + ComfyUI はこの Docker コンテナにインストールする
- [PyTorch on ROCm installation](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/3rd-party/pytorch-install.html) を参照して、Pytorch が事前にインストールされたDockerイメージを利用する手順(Use a prebuilt Docker image with PyTorch pre-installed)を実施する

1. Pytorch の Docker イメージのダウンロードとコンテナの起動

```bash
# Pytorch の Docker イメージをダウンロードする
# docker pull rocm/pytorch:rocm7.1_ubuntu24.04_py3.12_pytorch_release_2.8.0 下記コマンドと同じ
docker pull rocm/pytorch:latest
# Pytorch の Docker コンテナを起動する
docker run -it \
    --cap-add=SYS_PTRACE \
    --security-opt seccomp=unconfined \
    --device=/dev/kfd \
    --device=/dev/dri \
    --group-add video \
    --ipc=host \
    --shm-size 8G \
    rocm/pytorch:latest
```

2. Pytorch の動作確認をする

```bash
# Docker コンテナ内で Pytorch の動作確認をする
## Docker コンテナにログインする
docker exec -it <コンテナIDまたはコンテナ名> /bin/bash

[以降は、Docker コンテナ内で実行するコマンド]
## Pytorch のバージョンと GPU デバイスの認識を確認する
python3 -c 'import torch' 2> /dev/null && echo 'Success' || echo 'Failure'
Success ♯ と表示されれば、pythorch が正常にインストールされている
## Pytorch がCUDAを認識を確認する
python3 -c 'import torch; print(torch.cuda.is_available())'
True ♯ と表示されれば、GPU デバイスが認識されている
# Docker コンテナからGPU デバイスの認識を確認する
rocminfo | grep gfx
  Name:                    gfx1103                            # ※ OSにAPUの内蔵GPUが認識されていると、このように表示される
  Device Type:             GPU                                # ※ 

# Docker コンテナの終了
exit

[以降は、Dockerホストマシンで実行するコマンド]
docker container list --all  # コンテナIDまたはコンテナ名を確認する
docker container stop <コンテナIDまたはコンテナ名>  # コンテナを停止する
```

## 6. 画像生成AIのユーザー環境の構築

- 画像生成AIのユーザー環境を構築するために、ユーザーのホームディレクトリ以下に作業ディレクトリを作成する
  - 主に画像生成モデルのデータや、生成した画像データを保存するためのディレクトリを作成し、これを Docker コンテナにマウントする
  - 以下の例では、~/workdir ディレクトリを作成し、その下に rocm ディレクトリを作成する
  
```bash
# ホームディレクトリ以下に作業ディレクトリを作成する
mkdir -p ~/workdir/rocm/confy_data/models/checkpoints
mkdir -p ~/workdir/rocm/confy_data/output
mkdir -p ~/workdir/rocm/confy_data/custom_nodes

# Stable Diffusion 1.5 モデルファイルをダウンロードして、checkpoints ディレクトリに保存する
# 例: v1-5-pruned-emaonly.safetensors をダウンロードする場合
cd ~/workdir/rocm/confy_data/models/checkpoints
wget https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.safetensors
```
- ディレクトリ構成の例を以下に示す

```bash
[ディレクトリ構成]
~/workdir/rocm
└── comfy_data        # ConfyUI 用データ格納ディレクトリ
    ├── custom_nodes  # ConfyUI 用カスタムノード格納ディレクトリ
    ├── models    # 画像生成モデル格納ディレクトリ
    │   └── checkpoints # Stable Diffusion モデル格納ディレクトリ
    │       └── v1-5-pruned-emaonly.safetensors   # Stable Diffusion 1.5 モデルファイル例
    └── output     # 生成画像格納ディレクトリ
```

## 7. Stable Diffusion のインストール

- Docker コンテナ内に Stable Diffusion をインストールする
- 以降の作業は、下記で起動した Docker コンテナに環境構築していく

1. Pytorch の Docker イメージから、新しい Dockerコンテナ を起動する

```bash
docker run -it \
    --cap-add=SYS_PTRACE \
    --security-opt seccomp=unconfined \
    --device=/dev/kfd \
    --device=/dev/dri \
    --group-add video \
    --ipc=host \
    --shm-size 8G \
    rocm/pytorch:latest
```
2. Docker コンテナ内で Stable Diffusion をインストールする

```bash
## Docker コンテナにログインする
docker exec -it <コンテナIDまたはコンテナ名> /bin/bash

[以降は、Docker コンテナ内で実行するコマンド]
# Docker コンテナ内で Stable Diffusion をインストールする
pip install \
  huggingface-hub \
  diffusers \
  accelerate \
  transformers \
  safetensors
# Docker コンテナ内で Stable Diffusion のインストールを確認する
python3 -c 'import diffusers' 2> /dev/null && echo 'Success' || echo 'Failure'
Success  # と表示されれば、Stable Diffusion が正常に
```

このまま、Docker コンテナ内に ComfyUI をインストールしていく


## 8. ComfyUI のインストールとセットアップ

- ConfyUI の github リポジトリをクローンして、Docker コンテナ内に ComfyUI をインストールする
- [ComfyUI GitHub リポジトリ](https://github.com/comfyanonymous/ComfyUI)
- requirements.txt に記載されている Python パッケージの内、pythorch 関連のパッケージは既に Pytorch の Docker イメージにインストールされているため、これらを除外してインストールする

1. Docker コンテナ内で ComfyUI をインストールする

```bash
[以降は、Docker コンテナ内で実行するコマンド]
# ComfyUI をインストールするディレクトリを作成する
mkdir -p /app
# ComfyUI の github リポジトリをクローンする
cd /app
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
# requirements.txt から Pytorch 関連のパッケージを除外してインストールする
grep -vE 'torch$|torchvision$|torchaudio$' requirements.txt > temp_requirements.txt
# ComfyUI の 依存パッケージをインストールする。上記で作成した temp_requirements.txt を利用する
pip install -r temp_requirements.txt

```

## 9. 起動確認と Docker イメージへの保存

- ComfyUI が正常に動作することを確認し、動作確認ができた Docker コンテナを Docker イメージとして保存する

1. Docker コンテナ内で ComfyUI を起動して動作確認をする

```bash
[以降は、Docker コンテナ内で実行するコマンド]
cd /app/ComfyUI
# ComfyUI を起動する
python3 main.py --listen 0.0.0 --port 8188

# 下記のように表示されれば、ComfyUI が正常に起動している
Checkpoint files will always be loaded safely.
Total VRAM 56134 MB, total RAM 112267 MB
pytorch version: 2.8.0+rocm7.1.0.git7a520360
Set: torch.backends.cudnn.enabled = False for better AMD performance.
AMD arch: gfx1103
ROCm version: (7, 1)
Set vram state to: NORMAL_VRAM
Device: cuda:0 AMD Radeon Graphics : native
Enabled pinned memory 106654.0
Using sub quadratic optimization for attention, if you have memory or speed issues try using: --use-split-cross-attention
Python version: 3.12.3 (main, Aug 14 2025, 17:47:21) [GCC 13.3.0]
ComfyUI version: 0.3.71
****** User settings have been changed to be stored on the server instead of browser storage. ******
****** For multi-user setups add the --multi-user CLI argument to enable multiple user profiles. ******
ComfyUI frontend version: 1.30.6
[Prompt Server] web root: /opt/venv/lib/python3.12/site-packages/comfyui_frontend_package/static
Total VRAM 56134 MB, total RAM 112267 MB
pytorch version: 2.8.0+rocm7.1.0.git7a520360
Set: torch.backends.cudnn.enabled = False for better AMD performance.
AMD arch: gfx1103
ROCm version: (7, 1)
Set vram state to: NORMAL_VRAM
Device: cuda:0 AMD Radeon Graphics : native
Enabled pinned memory 106654.0

Import times for custom nodes:
   0.0 seconds: /home/ubuntu/workdir/comfyui/ComfyUI/custom_nodes/websocket_image_save.py

Context impl SQLiteImpl.
Will assume non-transactional DDL.
No target revision found.
Starting server

To see the GUI go to: http://0.0.0.0:8188
```

2. Docker コンテナを停止して、Docker イメージとして保存する

```bash
[以降は、Docker コンテナ内で実行するコマンド]
# Docker コンテナを停止する
exit  

[以降は、Dockerホストマシンで実行するコマンド]
# コンテナIDまたはコンテナ名を確認する
docker container list --all
# Docker コンテナを Docker イメージとして保存する
docker commit <コンテナIDまたはコンテナ名> rocm/comfyui_${USER}:latest
```

3. 保存した Docker イメージを確認する

```bash
# 保存した Docker イメージを確認する
docker image list | grep comfyui
rocm/comfyui          latest              <イメージID>        <作成日時>        <サイズ>
```

4. 保存した Docker イメージから Docker コンテナを起動して、ComfyUI が動作することを確認する

```bash
# 保存した Docker イメージから Docker コンテナを起動する
docker run -it -d --name rocm_comfyui_${USER} -p 8188:8188 \
  --device=/dev/kfd --device=/dev/dri \
  -v ${HOME}/workdir/rocm/comfy_data/models/checkpoints:/app/ComfyUI/models/checkpoints \
  -e HSA_OVERRIDE_GFX_VERSION=11.0.0 \
  -e TORCH_COMMAND_LINE="--opt-sdp-attention" \
  -e PYTORCH_HIP_ALLOC_CONF="garbage_collection_threshold:0.9,max_split_size_mb:512" \
  rocm/comfyui:latest \
  python /app/ComfyUI/main.py --listen 0.0.0.0 --port 8188 --lowvram --cpu-vae

```

5. ブラウザで ComfyUI (http://127.0.0.1:8188/ または http://[DockerホストのUbuntuのIP]:8188/) にアクセスして、ComfyUI の GUI が表示されることを確認する

## 10. 動かしてみる

- ComfyUI の GUI から Stable Diffusion 1.5 モデルを利用して、画像生成を試してみる
- 生成した画像は、~/workdir/rocm/comfy_data/output ディ レクトリに保存される

1. ComfyUI の GUI でノードを配置して、Stable Diffusion 1.5 モデルを利用した画像生成のワークフローを作成する
  - 例: テキストプロンプトから画像を生成するワークフロー
    - Text Input ノード → Stable Diffusion ノード → Image Save ノード
2. Text Input ノードに生成したい画像のテキストプロンプトを入力する
3. Stable Diffusion ノードで、Model Path に /app/ComfyUI/models/checkpoints/v1-5-pruned-emaonly.safetensors を指定する
4. Image Save ノードで、Save Directory に /app/comfy_data/output を指定する
5. ワークフローを実行して、画像生成を行う
6. 生成された画像が ~/workdir/rocm/comfy_data/output ディレクトリに保存されていることを確認する
7. 生成された画像を確認する
8. 必要に応じて、ワークフローやテキストプロンプトを変更して、様々な画像生成を試してみる

# 参考
- [AMD ROCm documentation](https://rocm.docs.amd.com/en/latest/)
- [Quick start installation guide](https://rocm.docs.amd.com/en/latest/install/quick-start.html#quick-start-installation-guide)
- [PyTorch on ROCm installation](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/3rd-party/pytorch-install.html)
- 
