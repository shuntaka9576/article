---
title: "AI用ハードウェアの数字、いったん並べてみる"
description: "Mac Studio Ultra、DGX Spark、GeForce RTX 50〜30シリーズ、RTX PRO、H100〜B300を一覧比較。発売年、メモリ容量・帯域、コア数、ドル・円価格、Runpod料金と通信速度を並べます"
publish: true
tags:
  - "tech/llm"
  - "tech/nvidia"
  - "tech/mac"
  - "tech/hardware"
---

## ハードウェア比較

AI用途で気になるメモリ容量・帯域・コア数・価格を、MacからRTX、データセンター向けGPUまで並べました。**2026年9月15日確認。発売・提供開始年の新しい順、同年は米国参考価格の高い順**です。価格非公表の製品は各年の末尾に置いています。

Mac・Spark・Halo・EVOは本体1台、ほかはGPU 1基の仕様です。コア数はApple GPU、AMD CUと明記したもの以外はCUDAコア数です。HaloとEVOは、発表済みの最上位構成を各1台載せています。

| 発売・提供年 | 製品・構成 | メモリ容量 | 帯域（GB/s） | コア数 | FP32（TFLOPS）[^flops] | 米国参考価格 → 円換算 | 国内新品（税込） | 国内中古例（税込）[^used] | Runpod / 時間 |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 2026予定 | Mac Studio M5 Ultra・512GB[^mac512] | 512GB・共有 | 1,200 | Apple GPU 80 | 未確認 | 予想$17,700 → 約272.6万円 | 予想約314.6万円 | — | — |
| 2026予定 | Mac Studio M5 Ultra・256GB[^ssd] | 256GB・共有 | 1,200 | Apple GPU 64 | 未確認 | $9,999 → 約154.0万円 | 1,759,800円 | — | — |
| 2026予定 | AMD Ryzen AI Halo・PRO 495／192GB[^amd] | 192GB・共有 | 最大273※ | AMD CU 40 | 約30.7※ | 未公表 | 未公表 | — | — |
| 2026予定 | GMKtec EVO-X5 Pro・192GB[^amd] | 192GB・共有 | 273 | AMD CU 40 | 約30.7※ | 未公表 | 未公表 | — | — |
| 2025 | DGX Spark・GB10[^ssd] | 128GB・共有 | 273 | CUDA 6,144 | 未確認 | $4,699 → 約72.4万円 | 販売店による | — | — |
| 2025 | GeForce RTX 5090 | 32GB GDDR7 | 1,792 | 21,760 | 約104.9 | $1,999 → 約30.8万円 | 販売店による | — | $0.99（約152円） |
| 2025 | GeForce RTX 5080 | 16GB GDDR7 | 960 | 10,752 | 約56.3 | $999 → 約15.4万円 | 販売店による | — | — |
| 2025 | GeForce RTX 5070 Ti | 16GB GDDR7 | 896 | 8,960 | 約43.9 | $749 → 約11.5万円 | 販売店による | — | — |
| 2025 | GeForce RTX 5070 | 12GB GDDR7 | 672 | 6,144 | 約30.8 | $549 → 約8.5万円 | 販売店による | [124,980円](https://www.janpara.co.jp/sale/search/detail/?ITMCODE=363858) | — |
| 2025 | GeForce RTX 5060 Ti・16GB | 16GB GDDR7 | 448 | 4,608 | 約23.7 | $429 → 約6.6万円 | 販売店による | — | — |
| 2025 | GeForce RTX 5060 Ti・8GB | 8GB GDDR7 | 448 | 4,608 | 約23.7 | $379 → 約5.8万円 | 販売店による | — | — |
| 2025 | GeForce RTX 5060 | 8GB GDDR7 | 448 | 3,840 | 約19.2 | $299 → 約4.6万円 | 販売店による | [109,980円](https://www.janpara.co.jp/sale/search/detail/?ITMCODE=377147) | — |
| 2025 | GeForce RTX 5050 | 8GB GDDR6 | 320 | 2,560 | 約13.2 | $249 → 約3.8万円 | 販売店による | — | — |
| 2025 | B300 | 288GB HBM3e | 最大8,000 | 20,480※ | 75 | 非公表・見積もり | 構成別見積もり | — | $7.89（約1,215円） |
| 2025※ | B200 | 180GB HBM3e | 最大8,000 | 18,944 | 75 | 非公表・見積もり | 構成別見積もり | — | $6.79（約1,046円） |
| 2025 | RTX PRO 6000 Blackwell・Workstation | 96GB GDDR7 ECC | 1,792 | 24,064 | 125 | 非公表・見積もり | 販売店による | — | — |
| 2025 | RTX PRO 6000 Blackwell・Server | 96GB GDDR7 ECC | 1,597 | 24,064 | 120 | 非公表・見積もり | 構成別見積もり | — | $2.09（約322円） |
| 2025 | RTX PRO 5000 Blackwell・72GB | 72GB GDDR7 ECC | 1,344 | 14,080 | 65 | 非公表・見積もり | 販売店による | — | — |
| 2025 | RTX PRO 5000 Blackwell・48GB | 48GB GDDR7 ECC | 1,344 | 14,080 | 65 | 非公表・見積もり | 販売店による | — | — |
| 2025 | RTX PRO 4500 Blackwell・Workstation | 32GB GDDR7 ECC | 896 | 10,496 | 51 | 非公表・見積もり | 販売店による | — | — |
| 2025 | RTX PRO 4000 Blackwell | 24GB GDDR7 ECC | 672 | 8,960 | 40 | 非公表・見積もり | 販売店による | — | — |
| 2025 | RTX PRO 2000 Blackwell | 16GB GDDR7 ECC | 288 | 4,352 | 17 | 非公表・見積もり | 販売店による | — | — |
| 2024 | GeForce RTX 4080 SUPER | 16GB GDDR6X | 736 | 10,240 | 約52.2 | $999 → 約15.4万円 | 販売店による | — | — |
| 2024 | GeForce RTX 4070 Ti SUPER | 16GB GDDR6X | 672 | 8,448 | 約44.1 | $799 → 約12.3万円 | 販売店による | — | — |
| 2024 | GeForce RTX 4070 SUPER | 12GB GDDR6X | 504 | 7,168 | 約35.6 | $599 → 約9.2万円 | 販売店による | — | — |
| 2024 | H200 SXM | 141GB HBM3e | 4,800 | 16,896 | 67 | 非公表・見積もり | 構成別見積もり | — | $4.59（約707円） |
| 2023 | GeForce RTX 4060 Ti・16GB | 16GB GDDR6 | 288 | 4,352 | 約22.1 | $499 → 約7.7万円 | 販売店による | — | — |
| 2023 | GeForce RTX 4060 Ti・8GB | 8GB GDDR6 | 288 | 4,352 | 約22.1 | $399 → 約6.1万円 | 販売店による | [51,980円](https://www.janpara.co.jp/sale/search/detail/?ITMCODE=336279) | — |
| 2023 | GeForce RTX 4060 | 8GB GDDR6 | 272 | 3,072 | 約15.1 | $299 → 約4.6万円 | 販売店による | [47,980円](https://www.janpara.co.jp/sale/search/detail/?ITMCODE=338309) | — |
| 2022 | GeForce RTX 4090 | 24GB GDDR6X | 1,008 | 16,384 | 約82.6 | $1,599 → 約24.6万円 | 販売店による | — | $0.74（約114円） |
| 2022 | RTX 6000 Ada | 48GB GDDR6 ECC | 960 | 18,176 | 91.1 | 非公表・見積もり | 販売店による | — | $0.84（約129円） |
| 2022 | H100 SXM | 80GB HBM3 | 3,350 | 16,896 | 67 | 非公表・見積もり | 構成別見積もり | — | $3.49（約537円） |
| 2020 | GeForce RTX 3090 | 24GB GDDR6X | 936 | 10,496 | 約35.7 | $1,499 → 約23.1万円 | 販売店による | — | $0.50（約77円） |
| 2020 | GeForce RTX 3080・10GB | 10GB GDDR6X | 760 | 8,704 | 約29.8 | $699 → 約10.8万円 | 販売店による | — | — |
| 2020 | GeForce RTX 3070 | 8GB GDDR6 | 448 | 5,888 | 約20.4 | $499 → 約7.7万円 | 販売店による | [39,980円](https://www.janpara.co.jp/sale/search/detail/?ITMCODE=292603) | — |
| 2020 | GeForce RTX 3060 Ti・GDDR6 | 8GB GDDR6 | 448 | 4,864 | 約16.2 | $399 → 約6.1万円 | 販売店による | [37,980円](https://www.janpara.co.jp/sale/search/detail/?ITMCODE=336335) | — |
| 2020 | RTX A6000 | 48GB GDDR6 ECC | 768 | 10,752 | 38.7 | 非公表・見積もり | 販売店による | — | $0.53（約82円） |

- **Apple GPUコア・AMD CU・CUDAコアは個数で性能比較できません。** 共有メモリはOS・CPUとも共用します。
- GeForceの価格は**発売時の米国参考価格**で、現在の実売価格ではありません。Mac・Sparkは確認時の構成価格、512GB Macは予想です。国内販売店による値付けとは分けています。
- 円換算は**1ドル＝154.01円**（2026年9月11日終値）。税・送料は含みません。[為替データ](https://www.investing.com/currencies/usd-jpy-historical-data)
- Runpodは**Pods / Secure Cloud・GPU 1基の時間単価**。ストレージ料金は別、`—`は今回の料金表で掲載価格を確認していない製品です。[Runpod公式料金](https://www.runpod.io/pricing)

[^flops]: **GPUのFP32理論性能**。1 TFLOPSは1秒あたり1兆回の浮動小数点演算で、FMAは2演算と数えます。CPU・NPU・Tensorコアの性能は加算しません。GeForceの「約」は公式比較表のCUDAコア数×公開ブーストクロック（GHz）×2÷1,000で算出。クロックが小数第2位に丸められているため、ほかの公称値と僅差が出ます。AMDの「※」はチップ仕様からの計算値。Mac・Sparkは確認した公式資料にFP32値がなく「未確認」としています。
[^used]: **じゃんぱらの中古販売例を2026年9月15日に確認**。送料別、ポイント・クーポン適用前。GPU単体・同じVRAM容量の在庫から1個体を採用した価格で、市場全体の最安値・中央値・落札相場ではありません。新品・未使用品、ジャンク、ファン異常や腐食などの記載がある個体は除外。`—`は今回、条件を満たす販売価格を確認できていないものです。型番・個体番号・状態は補足に記載しています。
[^amd]: **発表済みの最上位、Ryzen AI Max+ PRO 495／Radeon 8065S／192GBを選定**。Haloは近日登場、EVO-X5 Proは2026年9月28日発売予定で、両構成の価格は未確認。SSDは比較条件として2TBを想定していますが、発売構成・価格は確定していません。Haloの帯域はチップの上限仕様（256-bit LPDDR5X-8533）から約273GB/s、FP32は両機とも40 CU×256 FLOP/CU/clock×3GHz＝30.72 TFLOPSと計算。実機の持続性能ではありません。

[^ssd]: MacはSSD **2TB**構成。256GBモデルは30コアCPU・64コアGPUです。DGX SparkはFounders Editionの**4TB固定構成**。GPU単体の価格にはホストPC・SSDを含みません。
[^mac512]: **価格未確認のため筆者予想。SSD 2TB、36コアCPU・80コアGPUを想定**。米国約$17,700、日本税込約314.6万円。発売は2026年10月後半予定。予想の計算根拠は下の補足に記載しています。

:::details 仕様・価格の出典と補足

**中古価格の取得元と採用個体**

今回は取得元を**じゃんぱらの直販サイト**にそろえ、商品詳細で販売価格・在庫・状態を確認しました。買取価格や検索エンジンの抜粋だけでは採用していません。中古店の提示する売値なので、個人売買の成約価格とは区別します。更新時も同じ取得元・条件で取り直し、在庫が消えた価格を現行相場として残さない方針です。

- RTX 5070：ZOTAC SOLID ZT-B50700D-10P、商品番号65330586。124,980円。箱など付属、高負荷時の軽いコイル鳴きあり。
- RTX 5060：Palit Infinity 2 OC 8GB、商品番号39254040。109,980円。状態良好、箱・シール付属。
- RTX 4060 Ti 8GB：Palit NE6406T019P1-1060F、商品番号216060495。51,980円。箱付属。
- RTX 4060：MSI VENTUS 2X BLACK 8G OC、商品番号119212600。47,980円。本体のみ、ファンの汚れ・PCIe端子部の微細キズあり。
- RTX 3070：ASUS DUAL-RTX3070-8G、商品番号71188686。39,980円。本体のみ、外装に若干のスレあり。35,980円の別個体はファン軸ブレの記載があり除外。
- RTX 3060 Ti：ASUS DUAL-RTX3060TI-8G-MINI-V2、商品番号72365033。37,980円。本体のみ。

各価格のリンクは比較表内にあります。メーカー・冷却器・付属品・状態による差があるため、新品の発売時参考価格との差を、そのまま値下がり率として読むことはできません。

**FLOPSの読み方と出典**

FP32をそろえると演算能力の一面を比較できますが、LLMの生成速度はメモリ帯域や量子化形式にも左右されます。FP32の大小だけで、FP4・FP8を使う推論の速さは決まりません。

GeForceの計算には[NVIDIA公式比較表](https://www.nvidia.com/en-us/geforce/graphics-cards/compare/)のコア数とブーストクロックを使用。PROは各製品データシートの単精度性能を使用しています。PRO 4000は[現行製品ページの40 TFLOPS](https://www.nvidia.com/en-us/products/workstations/professional-desktop-gpus/rtx-pro-4000/)、A6000は[現行データシート](https://www.nvidia.com/content/dam/en-zz/Solutions/products/workstations/nvidia-rtx-a6000-datasheet.pdf)を参照。H100・H200は[H100仕様](https://www.nvidia.com/en-us/data-center/h100/)・[H200仕様](https://www.nvidia.com/en-us/data-center/h200/)、B200・B300は[HGXの8GPU合計600 TFLOPS](https://www.nvidia.com/en-au/data-center/hgx/)を8で割った1GPUあたり75 TFLOPSです。TF32やTensorコアによるFP32相当の行列演算は含めていません。

**AMD Ryzen AI Halo・GMKtec EVO-X5 Pro**

[AMD公式Haloページ](https://www.amd.com/en/products/processors/desktops/ryzen/ryzen-ai-halo.html)は、PRO 495・192GB版を近日登場として予告しています。販売中の395・128GB版の$3,999は、今回の192GB構成には使っていません。[EVO-X5 Proの公式発表](https://www.gmktec.com/blogs/news/gmktec-unveils-evo-x5-pro-at-ifa-2026-with-amd-introducing-the-worlds-first-desktop-ai-supercomputer-capable-of-running-a-300b-parameter-llm-fully-offline)で192GB・273GB/sを確認し、[製品ページ](https://www.gmktec.com/products/gmktec-evo-x5-pro-amd-ryzen-ai-max-pro-495-ai-mini-pc)の9月28日発売予定を記載しています。

GPUは両構成とも[PRO 495のRadeon 8065S・40 CU・最大3GHz](https://www.amd.com/en/products/processors/laptop/ryzen-pro/ai-max-pro-400-series/amd-ryzen-ai-max-plus-pro-495.html)。FP32の計算は[AMDのRDNA 3.5演算上限の説明](https://rocm.docs.amd.com/projects/rocprofiler-compute/en/develop/conceptual/rdna/system-speed-of-light.html)に従い、デュアルイシュー成立時の256 FLOP/CU/clockを使っています。EVOは最大160GBをGPUメモリに割り当て可能としていますが、共有メモリ全量をVRAMとして使えるという意味ではありません。

**Macの構成価格と512GBの予想**

256GB・64コアGPU・2TBは[米国Apple Store](https://www.apple.com/shop/buy-mac/mac-studio/m5-ultra-chip-30-core-cpu-64-core-gpu-256gb-memory-2tb-storage)で$9,999、[日本Apple Store](https://www.apple.com/jp/shop/buy-mac/mac-studio/m5-ultra-チップ-30コアcpu-64コアgpu-256gb-のメモリ-2tb-のストレージ)で1,759,800円。仕様と発売予定日は[Apple仕様](https://www.apple.com/jp/mac-studio/specs/)と[発表記事](https://www.apple.com/jp/newsroom/2026/08/apple-introduces-new-mac-studio-with-m5-max-and-m5-ultra/)を参照しています。

512GBは80コアGPUが必要です。256GB・80コアGPU・2TBは[米国$11,299](https://www.apple.com/shop/buy-mac/mac-studio/m5-ultra-chip-36-core-cpu-80-core-gpu-256gb-memory-2tb-storage)、[日本1,993,800円](https://www.apple.com/jp/shop/buy-mac/mac-studio/m5-ultra-チップ-36コアcpu-80コアgpu-256gb-のメモリ-2tb-のストレージ)。96GB→256GBの増額（$4,000 / 720,000円）を追加1GBあたりに直し、256GB→512GBにも同じ単価を仮定しました。

`米国: $11,299 + ($4,000 ÷ 160GB × 256GB) = $17,699 ≒ $17,700`

`日本: 1,993,800円 + (720,000円 ÷ 160GB × 256GB) = 3,145,800円 ≒ 314.6万円`

これは容量に比例すると仮定した試算で、Appleの予告価格ではありません。米国価格の円換算と日本販売価格の予想は別々に算出しています。

**DGX Spark・データセンター向けGPU**

Spark：[仕様](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)、[販売価格](https://marketplace.nvidia.com/en-us/enterprise/personal-ai-supercomputers/dgx-spark/)、[2025年の発売案内](https://blogs.nvidia.com/blog/live-dgx-spark-delivery/)。

H/B系：[H100 / H200 / B200仕様](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory-h100-h200-b200/latest/components.html)、[B300仕様](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/components.html)、[Lenovoのコア数比較](https://lenovopress.lenovo.com/lp2265-lenovo-thinksystem-gpu-comparison)、[RunpodのB300仕様](https://www.runpod.io/articles/guides/nvidia-b300)。

※B300のコア数はRunpodが20,480、LenovoのHGX B300欄が18,944と資料間に差があるため参考値です。B200はHGX向けの180GB構成を採用し、帯域はNVIDIAの最大8TB/s表記に合わせています。[LenovoのB200仕様](https://lenovopress.lenovo.com/lp2226-thinksystem-nvidia-b200-180gb-1000w-gpu)では7.7TB/sです。

H100の提供開始は[2022年](https://nvidianews.nvidia.com/news/nvidia-hopper-in-full-production)、H200は[2024年第2四半期](https://s201.q4cdn.com/141608511/files/doc_presentations/2023/11/nvda-f3q24-investor-presentation-final.pdf)。B200は2024年発表ですが、表では広い商用提供の目安として[2025年](https://developer.nvidia.com/blog/nvidia-blackwell-delivers-massive-performance-leaps-in-mlperf-inference-v5-0/)を採用しています。B300は[2025年のBlackwell Ultra](https://developer.nvidia.com/blog/inside-nvidia-blackwell-ultra-the-chip-powering-the-ai-factory-era/)です。

**GeForce**

コア数・メモリ容量は[NVIDIA比較表](https://www.nvidia.com/en-us/geforce/graphics-cards/compare/)。帯域は同表、[Ada白書](https://images.nvidia.com/aem-dam/Solutions/geforce/ada/nvidia-ada-gpu-architecture.pdf)、[Ampere白書](https://images.nvidia.com/aem-dam/en-zz/Solutions/geforce/ampere/pdf/NVIDIA-ampere-GA102-GPU-Architecture-Whitepaper-V1.pdf)、[RTX 4060 Tiのメモリ解説](https://www.nvidia.com/en-us/geforce/news/rtx-40-series-vram-video-memory-explained/)を参照。4080 SUPER・4070 Ti SUPER・4060はメーカー仕様のメモリ速度×バス幅÷8で算出しています（[23Gb/s](https://www.msi.com/Graphics-Card/GeForce-RTX-4080-SUPER-16G-GAMING-X-SLIM-WHITE)、[21Gb/s](https://www.msi.com/Graphics-Card/GeForce-RTX-4070-Ti-SUPER-16G-GAMING-SLIM)、[17Gb/s](https://www.msi.com/Graphics-Card/GeForce-RTX-4060-GAMING-8G)）。

発売年・米国参考価格：[RTX 50上位](https://nvidianews.nvidia.com/news/nvidia-blackwell-geforce-rtx-50-series-opens-new-world-of-ai-computer-graphics)、[RTX 5060系](https://www.nvidia.com/ja-jp/geforce/news/rtx-5060-desktop-family-laptop-5060-coming-soon/)、[RTX 5050](https://www.nvidia.com/en-au/geforce/news/rtx-5050-desktop-gpu-and-laptops/)、[RTX 40 SUPER](https://www.nvidia.com/en-au/geforce/news/geforce-rtx-4080-4070-ti-4070-super-gpu/)、[RTX 4060系](https://nvidianews.nvidia.com/news/geforce-rtx-4060-family-is-here-nvidias-revolutionary-ada-lovelace-architecture-comes-to-core-gamers-everywhere-starting-at-299)、[RTX 4090](https://www.nvidia.com/en-ph/geforce/news/rtx-40-series-graphics-cards-announcements/)、[RTX 30上位](https://nvidianews.nvidia.com/news/nvidia-delivers-greatest-ever-generational-leap-in-performance-with-geforce-rtx-30-series-gpus)、[RTX 3060 Ti](https://www.nvidia.com/en-us/geforce/news/gfecnt/202012/geforce-rtx-3060-ti-out-december-2/)。3060 Tiは発売時のGDDR6版、3080は10GB版です。

**RTX PRO・旧世代の48GBモデル**

仕様：[PRO 6000 Workstation](https://www.nvidia.com/content/dam/en-zz/Solutions/data-center/rtx-pro-6000-blackwell-workstation-edition/workstation-blackwell-rtx-pro-6000-workstation-edition-nvidia-us-3519208-web.pdf)、[PRO 6000 Server](https://www.nvidia.com/en-us/data-center/rtx-pro-6000-blackwell-server-edition/)、[PRO 5000・48/72GB](https://www.nvidia.com/content/dam/en-zz/Solutions/products/workstations/professional-desktop-gpus/rtx-pro-5000-blackwell/workstation-datasheet-blackwell-rtx-pro-5000-5488550-nvidia.pdf)、[PRO 4500](https://www.nvidia.com/content/dam/en-zz/Solutions/data-center/rtx-pro-4500-blackwell/workstation-datasheet-blackwell-rtx-pro-4500-we-nvidia-us-5108623-web.pdf)、[PRO 4000](https://www.nvidia.cn/content/dam/en-zz/Solutions/design-visualization/quadro-product-literature/workstation-datasheet-blackwell-rtx-pro-4000-nvidia-3662515.pdf)、[PRO 2000](https://www.nvidia.com/content/dam/en-zz/Solutions/products/workstations/professional-desktop-gpus/rtx-pro-2000/workstation-datasheet-blackwell-rtx-pro-2000-nvidia-us-4016661.pdf)、[RTX 6000 Ada](https://www.nvidia.com/content/dam/en-zz/Solutions/design-visualization/rtx-6000/proviz-print-rtx6000-datasheet-web-2504660.pdf)、[RTX A6000](https://www.nvidia.com/content/dam/en-zz/Solutions/design-visualization/quadro-product-literature/proviz-print-nvidia-rtx-a6000-datasheet-us-nvidia-1454980-r9-web%20%281%29.pdf)。

登場時期：[Blackwell PRO上位製品](https://nvidianews.nvidia.com/news/nvidia-blackwell-rtx-pro-workstations-servers-agentic-ai)、[PRO 2000](https://blogs.nvidia.com/blog/blackwell-ai-acceleration-workstation-rtx-pro/)、[RTX 6000 Ada](https://nvidianews.nvidia.com/news/nvidias-new-ada-lovelace-rtx-gpu-arrives-for-designers-and-creators)、[RTX A6000](https://www.nvidia.com/en-gb/geforce/news/december-2020-nvidia-studio-driver/)。RunpodのPRO 6000は[Server Edition](https://docs.runpod.io/flash/configuration/gpu-types)です。

:::

## 通信速度の比較

**8Gb/s＝1GB/s**。通信は一方向にそろえた公称値・換算値で、実効速度ではありません。

| 接続 | 搭載・用途の例 | 公称速度・換算（Gb/s） | GB/s換算 | 集計単位 |
| --- | --- | ---: | ---: | --- |
| 10Gb Ethernet | Mac Studio・DGX Spark | 10 | 1.25 | 1ポート |
| Thunderbolt 5 | Mac Studio | 80 | 10 | 1リンク・各方向 |
| ConnectX-7・Spark構成 | DGX Spark | 200 | 25 | 1ポート |
| ConnectX-7・400G構成 | B200サーバーなど | 400 | 50 | 1アダプター |
| ConnectX-8 | HGX B300など | 800 | 100 | 1アダプター |
| ConnectX-9・800Gポート | データセンター向け | 800 | 100 | 1ポート |
| ConnectX-9・Vera Rubin構成 | データセンター向け | 1,600 | 200 | GPU 1基あたり |
| NVLink 第4世代 | H100 / H200のGPU間接続 | 3,600 | 450 | GPU 1基の全リンク・一方向換算 |
| NVLink 第5世代 | B200 / B300のGPU間接続 | 7,200 | 900 | GPU 1基の全リンク・一方向換算 |

- Thunderbolt 5の「最大120Gb/s」は主にディスプレイ向けの非対称モードです。通常リンクの80Gb/sも、そのままPC間通信の実効速度にはなりません。
- Sparkは200Gb/sポートを2つ搭載。SoCとのPCIe接続も制約になるため、実効400Gb/sとは限りません。
- NVLinkの公式表記は**双方向合計で900 / 1,800GB/s**。上表は半分に換算しており、特定のGPUペア間の速度保証ではありません。

出典：[Thunderbolt 5](https://newsroom.intel.com/client-computing/intel-introduces-thunderbolt-5-standard)、[SparkのConnectX-7](https://docs.nvidia.com/dgx/dgx-spark/spark-clustering.html)、[B200の400G構成](https://lenovopress.lenovo.com/lp2226-thinksystem-nvidia-b200-180gb-1000w-gpu)、[ConnectX-8搭載構成](https://docs.nvidia.com/enterprise-reference-architectures/hgx-ai-factory/latest/components.html)、[ConnectX-9仕様](https://networking-docs.nvidia.com/connectx9hw/specifications)、[Vera Rubin構成](https://blogs.nvidia.com/blog/vera-rubin-lpx-spectrum-x-nvlink-fusion/)、[NVLink仕様](https://developer.nvidia.com/blog/inside-nvidia-blackwell-ultra-the-chip-powering-the-ai-factory-era/)。

## 追加で見る指標

| 指標 | 比較するときのポイント |
| --- | --- |
| Tensor演算性能 | BF16 / FP8 / FP4、dense / sparseをそろえる |
| TTFT | 最初のトークンが出るまでの待ち時間 |
| 出力tokens/s | 1ユーザーの生成速度と、同時実行時の合計を分ける |
| 実効通信帯域・遅延 | 分散推論での転送・同期の待ち時間。RDMA対応も確認 |
| 実消費電力・tokens/J | 同じモデル・負荷での電力と処理量 |
| ソフトウェア対応 | 推論エンジン、モデル、量子化形式、対応カーネル |

たとえばSparkの「1PFLOPS」はFP4・sparseの値です。演算精度や条件が異なるFLOPSとは直接比較できません。[DGX Spark性能仕様](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)
