## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الملف `dwmac-sun8i.c` جزء من **STMMAC Ethernet Driver** — الـ subsystem اللي بيدير كارت الشبكة المبني على core بتاع Synopsys DesignWare GMAC. الكود موجود في:

```
drivers/net/ethernet/stmicro/stmmac/
```

وده subsystem بيشغّل عشرات الـ SoCs المختلفة (Allwinner, Rockchip, Intel, Qualcomm, ...) كلها بتستخدم نفس الـ core من Synopsys بس كل واحد ليه glue layer خاص بيه. ملف `dwmac-sun8i.c` هو الـ glue layer بتاع Allwinner sun8i/sun50i SoCs.

---

### الصورة الكبيرة — ELI5

#### القصة

تخيل عندك شركة بتصنع محرك سيارة (Synopsys GMAC Core) وبتبيعه لشركات تانية (Allwinner, Rockchip, ...). كل شركة بتاخد نفس المحرك وبتحطه في سيارتها، بس كل سيارة بيبقى ليها طريقة مختلفة في:
- **تشغيل المحرك** (system control registers)
- **توصيل الوقود** (clocks and resets)
- **اختيار الإطارات** (PHY interface: MII / RMII / RGMII)

الـ Linux kernel عنده framework اسمه **stmmac** بيقول: "أنا بعرف أشغّل المحرك دا بشكل عام، بس أنت قولي ازاي تشغّله على سيارتك بالذات." الـ glue layer (`dwmac-sun8i.c`) هو اللي بيجاوب السؤال ده لـ Allwinner H3 / A64 / H6 / V3s / A83T / R40.

---

#### الـ Hardware اللي بنتكلم عنه

**الـ Allwinner EMAC** — كارت شبكة Ethernet مدمج جوا SoC بتاع Allwinner (أرخص SoCs شائعة في Orange Pi, NanoPi, ...). الـ core نفسه مش Synopsys الكامل، فيه اختلافات في الـ register map.

```
┌─────────────────────────────────────┐
│           Allwinner SoC             │
│                                     │
│  ┌──────────────┐  ┌─────────────┐  │
│  │  System CTL  │  │    CCU      │  │  ← syscon / CCU registers
│  │  (syscon)    │  │  (R40 only) │  │     تحدد PHY interface type
│  └──────┬───────┘  └──────┬──────┘  │
│         │                 │          │
│  ┌──────▼─────────────────▼───────┐  │
│  │         EMAC Core              │  │  ← الـ MAC نفسه
│  │  (DMA + MAC + MDIO + EPHY)    │  │
│  └────────────────┬───────────────┘  │
│                   │                  │
│  ┌────────────────▼───────────────┐  │
│  │   Internal EPHY (H3/V3s)       │  │  ← PHY مدمج (H3 فقط)
│  │   أو External PHY عبر MDIO     │  │
│  └────────────────────────────────┘  │
└─────────────────────────────────────┘
```

---

#### ليه الملف ده موجود؟

لما الـ stmmac framework بيشتغل على أي SoC تاني (زي Rockchip)، بيستخدم **Synopsys DW GMAC standard register map**. لكن Allwinner كتبوا EMAC بتاعهم بـ register map مختلف كليًا — مثلًا:

| Feature | DW GMAC Standard | Allwinner EMAC |
|---------|-----------------|----------------|
| TX DMA start | reg DMA_CONTROL | `EMAC_TX_CTL1` bit 31 |
| RX desc base | DMA_RCVBASEADDR | `EMAC_RX_DESC_LIST` @ 0x34 |
| MAC address | MAC_ADDR_HIGH/LOW | `EMAC_MACADDR_HI/LO` @ 0x50+ |
| Interrupts | DMA_INTR_ENA | `EMAC_INT_EN` @ 0x0C |

لو تركنا الـ framework يستخدم الـ standard addresses، هيكتب على registers غلط وهيعطل الجهاز. الـ glue layer بيعمل **override** لكل العمليات دي بـ ops tables:
- `sun8i_dwmac_ops` — MAC-level operations
- `sun8i_dwmac_dma_ops` — DMA-level operations

---

#### المشكلة الأصعب: الـ PHY والـ Syscon

الـ Allwinner H3 عنده **internal PHY** (EPHY) مدمج جوا الـ SoC نفسه. لكن المستخدم ممكن يختار:
1. يستخدم الـ internal PHY (أسرع/أسهل)
2. يستخدم external PHY عبر MDIO

اللي بيحدد الاختيار ده هو **syscon register** (Control Register @ 0x30 في System Control). الـ glue layer لازم:
1. يقرأ الـ device tree عشان يعرف المستخدم عايز إيه
2. يكتب في الـ syscon register الـ PHY interface type (MII / RMII / RGMII)
3. لو internal PHY: يمسّك الـ clock والـ reset controller بتاعه ويشغّله

#### الـ MDIO Mux

الـ H3 عنده **MDIO bus مشترك** بين الـ internal و external PHY. الـ driver بيعمل **software mux** عبر `mdio-mux` framework يختار أي PHY يكلم دلوقتي ويـ power up/down الـ internal PHY عند الاختيار.

```
         MDIO Bus
            │
     ┌──────┴──────┐
     │  MDIO Mux   │  ← mdio_mux_syscon_switch_fn()
     └──────┬──────┘
     ┌──────┴──────┐
     │             │
  Internal      External
   EPHY          PHY
```

---

#### الـ SoC Variants

الـ driver بيدعم 6 variants مختلفة، كل واحد له قدرات مختلفة:

| SoC | Internal PHY | RGMII | RMII | TX Delay | RX Delay |
|-----|-------------|-------|------|----------|----------|
| H3 | نعم | نعم | نعم | 7 steps | 31 steps |
| V3s | نعم | لا | لا | — | — |
| A83T | لا | نعم | لا | 7 steps | 31 steps |
| R40 | لا | نعم | لا | — | 7 steps |
| A64 | لا | نعم | نعم | 7 steps | 31 steps |
| H6 | لا* | نعم | نعم | 7 steps | 31 steps |

*الـ H6 عنده PHY خارجي على chip AC200 منفصل.

---

#### الـ TX/RX Delay — ليه موجود؟

في RGMII، الـ clock والـ data لازم يوصلوا بـ توقيت دقيق. المسافة الفيزيائية على الـ PCB بتسبب **تأخير** (propagation delay). الـ syscon register بيتيح ضبط تأخير رقمي للـ TX و RX عشان يعوّض الـ PCB layout. المستخدم بيحدده في الـ device tree بـ `allwinner,tx-delay-ps` و `allwinner,rx-delay-ps`.

---

#### ملخص ما يعمله الملف

```
probe()
  ├── اقرأ variant من device tree (H3? A64? ...)
  ├── احجز PHY regulator (اختياري)
  ├── احصل على syscon regmap
  ├── اعمل set_syscon() ← اكتب PHY interface في System Control
  ├── stmmac_pltfr_probe() ← سجّل مع framework
  └── لو internal PHY:
        ├── get_ephy_nodes() ← جيب clock + reset
        └── register_mdio_mux() ← سجّل الـ mux

sun8i_dwmac_setup()
  └── سجّل sun8i_dwmac_ops + sun8i_dwmac_dma_ops
      (override كل operations بـ implementations خاصة بـ Allwinner)
```

---

### الملفات المرتبطة اللي المطوّر لازم يعرفها

#### الـ Core (نفس الـ directory)

| الملف | الدور |
|-------|-------|
| `stmmac_main.c` | الـ main driver — بيشغّل الـ net_device وبيستدعي الـ glue |
| `stmmac_platform.c` | الـ platform probe/remove المشترك — `stmmac_pltfr_probe()` |
| `stmmac_mdio.c` | إدارة الـ MDIO bus والـ PHY |
| `dwmac1000_dma.c` | الـ standard DW GMAC DMA ops (اللي sun8i بيـ override عليها) |
| `dwmac1000_core.c` | الـ standard DW GMAC MAC ops (اللي sun8i بيـ override عليها) |
| `hwif.c` / `hwif.h` | الـ hardware interface abstraction — بيربط الـ ops tables |

#### الـ Headers المهمة

| الملف | الدور |
|-------|-------|
| `include/linux/stmmac.h` | `plat_stmmacenet_data` — الـ platform data الرئيسي |
| `drivers/net/ethernet/stmicro/stmmac/common.h` | `mac_device_info`, `stmmac_dma_ops`, `stmmac_ops` |
| `include/linux/mdio-mux.h` | الـ MDIO mux framework API |

#### Device Tree Bindings

| الملف |
|-------|
| `Documentation/devicetree/bindings/net/allwinner,sun8i-h3-emac.yaml` |

#### ملفات Allwinner قريبة

| الملف | الدور |
|-------|-------|
| `dwmac-sunxi.c` | الـ glue layer للـ A20 (sun7i) — الجيل السابق |
| `dwmac-sun55i.c` | الـ glue layer للـ H616/T507 — الجيل الأحدث |
## Phase 2: شرح الـ STMMAC / dwmac-sun8i Framework

---

### المشكلة اللي الـ Subsystem بيحلها

**الـ DesignWare MAC (DWMAC)** من Synopsys هو IP core بيتلقى في تقريباً كل SoC بيحتاج Ethernet — من STMicro لـ Allwinner لـ Rockchip لـ Amlogic. المشكلة إن الـ IP core نفسه بيتغير شوية بين كل vendor وكل SoC generation:

- الـ clock والـ reset setup مختلف من SoC للتاني.
- الـ PHY interface selection (MII / RMII / RGMII) بيتعمل عن طريق syscon register مش جوه الـ MAC نفسه.
- بعض الـ SoCs زي H3 فيها internal EPHY جوه الـ SoC، محتاجة clock وreset خاصين.
- الـ DMA engine مش standard Synopsys DMA — Allwinner عملوا DMA engine مختلف خالص.

لو كتبت driver لكل variant لوحده هتكرر آلاف الأسطر. المطلوب: framework بيعزل الجزء المشترك (network stack, TSO, checksum offload, NAPI) عن الجزء الـ SoC-specific (clocks, resets, syscon registers, DMA quirks).

---

### الحل: الـ STMMAC Framework

الـ kernel بيحل المشكلة بطريقة classic layering:

```
┌─────────────────────────────────────────────────────┐
│              Network Stack (net core)               │
├─────────────────────────────────────────────────────┤
│         stmmac_main.c  — Generic STMMAC Driver      │
│  (NAPI, TSO, checksum offload, ethtool, phylink)    │
├──────────────────────┬──────────────────────────────┤
│   stmmac_ops         │      stmmac_dma_ops          │
│  (MAC control ops)   │   (DMA engine ops)           │
├──────────────────────┴──────────────────────────────┤
│          Glue Layer (dwmac-sun8i.c)                 │
│  Allwinner-specific: syscon, EPHY, MDIO mux, clocks │
├─────────────────────────────────────────────────────┤
│          Hardware: Allwinner H3/A64/H6 EMAC         │
└─────────────────────────────────────────────────────┘
```

الـ stmmac_main.c بيعمل كل حاجة shared: بيـregister الـ net_device، بيتعامل مع الـ packet path، بيـhandle الـ phylink. الـ glue layer (dwmac-sun8i.c) بيـprovide فقط الـ ops structs اللي بتوصف hardware الـ Allwinner.

---

### تشبيه من الواقع

تخيل إن عندك **شركة فنادق عالمية** زي Marriott. كل فرع (H3، A64، H6) بيشتغل بنفس السياسة العامة: نفس standard الاستقبال، نفس نظام الحجوزات، نفس التدريب. لكن كل فرع عنده:

- مفاتيح الإضاءة في أماكن مختلفة (syscon register مختلف).
- بعض الفروع فيها مطعم داخلي (internal EPHY)، وبعضها بيوصلك لمطعم برة (external PHY).
- نظام التكييف (clock tree) مختلف في كل مبنى.

الـ **stmmac_main.c** = إدارة Marriott المركزية (السياسات الثابتة).
الـ **dwmac-sun8i.c** = مدير الفرع (عارف كيف يشغّل الأجهزة المحلية).
الـ **stmmac_ops / stmmac_dma_ops** = الـ job description اللي بيقول لمدير الفرع "أنت مسؤول عن إيه".
الـ **plat_stmmacenet_data** = العقد اللي بيوقع عليه مدير الفرع مع الإدارة المركزية (hardware features + callbacks).
الـ **emac_variant** = الـ building specs لكل فرع (عنده حمام سباحة ولا لأ = soc_has_internal_phy).
الـ **syscon** = لوحة التحكم الكهربائية المركزية للمبنى اللي مش تبعت الفندق بس بيحتاج يوصل ليها.

---

### الـ Big Picture Architecture

```
Device Tree (allwinner,sun8i-h3-emac)
           │
           ▼
   sun8i_dwmac_probe()
           │
           ├─── stmmac_get_platform_resources()    ← IRQ, IOMEM base address
           │
           ├─── sun8i_dwmac_get_syscon_from_dev()  ← regmap لـ System Control
           │         │
           │         └── devm_regmap_field_alloc() ← بيـlock على register field محدد
           │
           ├─── devm_stmmac_probe_config_dt()      ← parse DT → plat_stmmacenet_data
           │
           ├─── sun8i_dwmac_set_syscon()           ← يكتب PHY interface في syscon
           │         │
           │         ├── RGMII → SYSCON_EPIT | SYSCON_ETCS_INT_GMII
           │         ├── RMII  → SYSCON_RMII_EN | SYSCON_ETCS_EXT_GMII
           │         └── MII   → default (0)
           │
           ├─── stmmac_pltfr_probe()               ← ينشئ net_device ويـregister كل حاجة
           │         │
           │         └── stmmac_dvr_probe()
           │                   │
           │                   ├── mac_setup() → sun8i_dwmac_setup()
           │                   │     └── يربط sun8i_dwmac_ops + sun8i_dwmac_dma_ops
           │                   │
           │                   └── stmmac_pltfr_init() → sun8i_dwmac_init()
           │                         └── regulator_enable() + power_internal_phy()
           │
           └─── sun8i_dwmac_register_mdio_mux()    ← لو H3 (internal PHY)
                     │
                     └── mdio_mux_init()
                           └── mdio_mux_syscon_switch_fn() كـswitch callback
```

---

### الـ Core Abstraction: الـ ops structs

الـ STMMAC framework بيقوم على فكرة إن الـ MAC hardware بيتوصف عبر **جدولين من function pointers**:

#### 1. `struct stmmac_ops` — MAC-level operations

```c
/* sun8i_dwmac_ops: الـ Allwinner implementation لـ MAC control */
static const struct stmmac_ops sun8i_dwmac_ops = {
    .core_init      = sun8i_dwmac_core_init,   /* burst len إعداد الـ DMA */
    .set_mac        = sun8i_dwmac_set_mac,     /* enable/disable TX+RX */
    .dump_regs      = sun8i_dwmac_dump_mac_regs,
    .rx_ipc         = sun8i_dwmac_rx_ipc_enable, /* enable checksum verify */
    .set_filter     = sun8i_dwmac_set_filter,  /* multicast/promisc */
    .flow_ctrl      = sun8i_dwmac_flow_ctrl,   /* pause frames */
    .set_umac_addr  = sun8i_dwmac_set_umac_addr,
    .get_umac_addr  = sun8i_dwmac_get_umac_addr,
    .set_mac_loopback = sun8i_dwmac_set_mac_loopback,
};
```

#### 2. `struct stmmac_dma_ops` — DMA engine operations

```c
/* sun8i_dwmac_dma_ops: الـ Allwinner DMA implementation */
static const struct stmmac_dma_ops sun8i_dwmac_dma_ops = {
    .reset              = sun8i_dwmac_dma_reset,
    .init               = sun8i_dwmac_dma_init,
    .init_rx_chan       = sun8i_dwmac_dma_init_rx,  /* يكتب RX descriptor base addr */
    .init_tx_chan       = sun8i_dwmac_dma_init_tx,  /* يكتب TX descriptor base addr */
    .dump_regs          = sun8i_dwmac_dump_regs,
    .dma_rx_mode        = sun8i_dwmac_dma_operation_mode_rx,
    .dma_tx_mode        = sun8i_dwmac_dma_operation_mode_tx,
    .enable_dma_irq     = sun8i_dwmac_enable_dma_irq,
    .disable_dma_irq    = sun8i_dwmac_disable_dma_irq,
    .start_tx           = sun8i_dwmac_dma_start_tx,
    .stop_tx            = sun8i_dwmac_dma_stop_tx,
    .start_rx           = sun8i_dwmac_dma_start_rx,
    .stop_rx            = sun8i_dwmac_dma_stop_rx,
    .dma_interrupt      = sun8i_dwmac_dma_interrupt,
};
```

---

### كيف الـ Structs بتترابط مع بعض

```
plat_stmmacenet_data                    stmmac_priv
┌──────────────────────┐               ┌────────────────────────┐
│ bsp_priv ────────────┼──────────────▶│ sunxi_priv_data        │
│ init() ──────────────┼──┐            │  ├── ephy_clk          │
│ exit() ──────────────┼──┤            │  ├── regulator         │
│ mac_setup() ─────────┼──┤            │  ├── rst_ephy          │
│ rx_coe               │  │            │  ├── variant ──────────┼──▶ emac_variant
│ tx_coe               │  │            │  ├── regmap_field      │
│ flags (HAS_SUN8I)    │  │            │  ├── internal_phy_pow  │
└──────────────────────┘  │            │  └── mux_handle        │
                          │            ├────────────────────────┤
                          │            │ hw ────────────────────┼──▶ mac_device_info
                          │            │  ├── mac ──────────────┼──▶ stmmac_ops
                          │            │  ├── dma ──────────────┼──▶ stmmac_dma_ops
                          │            │  ├── pcsr (ioaddr)     │
                          │            │  └── mii (MDIO regs)   │
                          │            ├────────────────────────┤
                          │            │ ioaddr                 │
                          │            │ mii (mii_bus)          │
                          │            │ phylink                │
                          └───────────▶│ plat ──────────────────┼──▶ (circular ref)
                                       └────────────────────────┘

emac_variant
┌─────────────────────────┐
│ syscon_field ───────────┼──▶ reg_field { .reg=0x30, .lsb=0, .msb=31 }
│ soc_has_internal_phy    │    (H3/V3S) أو { .reg=0x164 } (R40)
│ support_mii             │
│ support_rmii            │
│ support_rgmii           │
│ rx_delay_max            │
│ tx_delay_max            │
└─────────────────────────┘
```

---

### تفاصيل المكونات الأساسية

#### الـ `emac_variant` — Hardware Descriptor

كل SoC variant بيتوصف بـ`emac_variant` struct ثابت في compile-time. الـ `of_device_get_match_data()` بيجيب الـ pointer المناسب من الـ `of_device_id` table:

```c
static const struct of_device_id sun8i_dwmac_match[] = {
    { .compatible = "allwinner,sun8i-h3-emac",  .data = &emac_variant_h3  },
    { .compatible = "allwinner,sun8i-v3s-emac", .data = &emac_variant_v3s },
    { .compatible = "allwinner,sun8i-a83t-emac",.data = &emac_variant_a83t},
    { .compatible = "allwinner,sun8i-r40-gmac", .data = &emac_variant_r40 },
    { .compatible = "allwinner,sun50i-a64-emac",.data = &emac_variant_a64 },
    { .compatible = "allwinner,sun50i-h6-emac", .data = &emac_variant_h6  },
    { }
};
```

الجدول ده بيوضح الفروق:

| Variant | Internal PHY | MII | RMII | RGMII | syscon reg |
|---------|:---:|:---:|:----:|:-----:|:----------:|
| H3      | ✓  | ✓  | ✓   | ✓    | 0x30       |
| V3s     | ✓  | ✓  | ✗   | ✗    | 0x30       |
| A83T    | ✗  | ✓  | ✗   | ✓    | 0x30       |
| R40     | ✗  | ✓  | ✗   | ✓    | 0x164 (CCU)|
| A64     | ✗  | ✓  | ✓   | ✓    | 0x30       |
| H6      | ✗  | ✓  | ✓   | ✓    | 0x30       |

ملاحظة مهمة: H6 فيه "AC200" chip مباشرة على نفس package بس مش على نفس die — الـ comment في الكود بيوضح ده صراحة. الفرق مهم لأنه بيأثر على كيفية الـ power sequence.

#### الـ Syscon / Regmap — Shared Register Access

الـ **syscon** هو subsystem في الـ kernel (محتاج تعرف عنه) بيسمح لـ multiple drivers إنهم يـaccess نفس register block عن طريق **regmap** موحد. ده بيمنع conflict لأن كل driver بيـread-modify-write في نفس الـ register block.

بدل ما الـ driver يعمل `ioremap` مباشرة على الـ system control registers (اللي فيها registers تانية لـ drivers تانية)، بيعمل:

```c
/* 1. يجيب الـ regmap من الـ syscon device */
regmap = sun8i_dwmac_get_syscon_from_dev(pdev->dev.of_node);

/* 2. يخصص field محدد جوه الـ regmap */
gmac->regmap_field = devm_regmap_field_alloc(dev, regmap,
                                              *gmac->variant->syscon_field);

/* 3. يكتب بشكل آمن في الـ field ده بس */
regmap_field_write(gmac->regmap_field, reg);
```

الـ `reg_field` struct بيحدد `{reg, lsb, msb}` — يعني بيـmask الكتابة على bits محددة بس، مش الـ register كله.

#### الـ PHY Interface — الـ Syscon Register Layout

للـ H3 مثلاً، الـ EMAC_CLK register عند offset 0x30 في System Control:

```
Bit 31..21  20..18     17        16       15      13      10..8   4..0
           [EPHY_ADDR][CLK_SEL][LED_POL][SHUTDOWN][SELECT][RMII_EN][TX_DELAY][EPIT][TX_CLK_SRC][RX_DELAY]
```

الـ `sun8i_dwmac_set_syscon()` بيبني الـ value ده بناءً على:
1. الـ `phy_interface` من الـ DT (RGMII/RMII/MII).
2. الـ `allwinner,tx-delay-ps` و `allwinner,rx-delay-ps` properties.
3. الـ PHY MDIO address (لو internal PHY).

#### الـ Internal PHY (EPHY) — مشكلة الـ MDIO Mux

الـ H3 SoC فيه internal 10/100 PHY. المشكلة إن الـ MDIO bus واحدة بس، والـ MAC محتاج يتكلم مع إما الـ internal PHY أو external PHY عبر نفس الـ bus. الحل هو **MDIO Multiplexer** (subsystem مستقل في الـ kernel: `mdio-mux`):

```
          MDIO Bus (واحدة)
               │
        ┌──────┴──────┐
        │  mdio-mux   │  (kernel mdio-mux subsystem)
        │  (syscon)   │
        └──────┬──────┘
        ┌──────┴──────┐
        │             │
   Internal MDIO  External MDIO
   (EPHY, addr=1) (connector, addr=2)
```

الـ switch function:
```c
static int mdio_mux_syscon_switch_fn(int current_child, int desired_child,
                                      void *data)
{
    /* لو عايز internal PHY: */
    val = (reg & ~H3_EPHY_MUX_MASK) | H3_EPHY_SELECT;
    /* لو عايز external PHY: */
    val = (reg & ~H3_EPHY_MUX_MASK) | H3_EPHY_SHUTDOWN;

    regmap_field_write(gmac->regmap_field, val);

    /* بعد تغيير الـ syscon لازم MAC reset */
    ret = sun8i_dwmac_reset(priv);
}
```

ملاحظة critical: بعد أي تغيير في الـ syscon register، الـ MAC لازم يتعمله reset عشان يـpick up الـ new value. ده موضح في comment الكود.

#### الـ DMA Engine — مختلف عن الـ Standard Synopsys

الـ standard DWMAC بيستخدم Synopsys DMA engine مع register map معين. الـ Allwinner EMAC بيستخدم DMA مختلف — عشان كده الـ `sun8i_dwmac_dma_ops` بيـoverride كل الـ DMA functions.

الـ store-and-forward vs threshold modes:
```c
/* RX: اختيار threshold */
if (mode == SF_DMA_MODE) {
    v |= EMAC_RX_MD;          /* store-and-forward */
} else {
    /* threshold mode: 32/64/96/128 bytes */
    if (mode < 32)       v |= EMAC_RX_TH_32;
    else if (mode < 64)  v |= EMAC_RX_TH_64;
    ...
}
```

في الـ **store-and-forward mode**: الـ DMA بيستنى الـ frame يتكتب كاملاً في الـ FIFO قبل ما يبدأ يـforward للـ host. أبطأ بس أكثر reliability وبيسمح بـchecksum verification.

في الـ **threshold mode**: الـ DMA يبدأ يـforward لما يوصل عدد معين من الـ bytes — أسرع بس ممكن يـforward frame ناقص لو حصل error.

---

### الـ Probe Sequence — رحلة الـ Driver من Power-on لـ Network-Up

```
1. sun8i_dwmac_probe()
   │
   ├─► stmmac_get_platform_resources()
   │     └── يجيب IRQ number + MMIO base من DT
   │
   ├─► devm_kzalloc(sunxi_priv_data)
   │     └── الـ private state بتاع الـ glue layer
   │
   ├─► of_device_get_match_data()
   │     └── يجيب emac_variant الصح (H3, A64, etc.)
   │
   ├─► devm_regulator_get_optional("phy")
   │     └── optional power supply للـ external PHY
   │
   ├─► sun8i_dwmac_get_syscon_from_dev()
   │     └── يجيب regmap من الـ syscon platform device
   │
   ├─► devm_regmap_field_alloc()
   │     └── يخصص field في الـ regmap
   │
   ├─► devm_stmmac_probe_config_dt()
   │     └── يـparse DT → plat_stmmacenet_data
   │           ├── phy_interface (RGMII/RMII/MII)
   │           └── phy_node, mdio_node, clocks, resets
   │
   ├─► [إعداد plat_dat manually]
   │     ├── rx_coe = STMMAC_RX_COE_TYPE2
   │     ├── tx_coe = 1
   │     ├── flags |= STMMAC_FLAG_HAS_SUN8I
   │     ├── bsp_priv = gmac
   │     ├── init = sun8i_dwmac_init
   │     ├── exit = sun8i_dwmac_exit
   │     ├── mac_setup = sun8i_dwmac_setup
   │     ├── tx_fifo_size = 4096
   │     └── rx_fifo_size = 16384
   │
   ├─► sun8i_dwmac_set_syscon()
   │     └── يكتب PHY interface config في syscon register
   │
   ├─► stmmac_pltfr_probe()
   │     └── يـcall stmmac_dvr_probe()
   │           ├── mac_setup() → sun8i_dwmac_setup()
   │           │     └── يربط ops structs + يعمل setup للـ MII regs
   │           └── stmmac_pltfr_init() → sun8i_dwmac_init()
   │                 ├── regulator_enable()
   │                 └── sun8i_dwmac_power_internal_phy()
   │                       ├── clk_prepare_enable(ephy_clk)
   │                       └── reset_control_reset(rst_ephy)
   │
   └─► [لو soc_has_internal_phy]
         ├── get_ephy_nodes()
         │     └── يجيب ephy_clk + rst_ephy من DT
         └── sun8i_dwmac_register_mdio_mux()
               └── mdio_mux_init()
```

---

### الـ MAC Setup — ربط الـ Hardware Description

الـ `sun8i_dwmac_setup()` هو أهم function في الـ glue layer لأنه بيـfill الـ `mac_device_info` struct اللي الـ stmmac core بيستخدمه:

```c
static int sun8i_dwmac_setup(void *ppriv, struct mac_device_info *mac)
{
    struct stmmac_priv *priv = ppriv;

    mac->pcsr = priv->ioaddr;          /* register base */
    mac->mac  = &sun8i_dwmac_ops;     /* MAC ops table */
    mac->dma  = &sun8i_dwmac_dma_ops; /* DMA ops table */

    priv->dev->priv_flags |= IFF_UNICAST_FLT; /* hardware filter support */

    /* ما يعرفه الـ hardware من speeds */
    mac->link.caps = MAC_ASYM_PAUSE | MAC_SYM_PAUSE |
                     MAC_10 | MAC_100 | MAC_1000;

    /* Speed bits في EMAC_BASIC_CTL0 */
    mac->link.speed10   = EMAC_SPEED_10;   /* 0x02 << 2 */
    mac->link.speed100  = EMAC_SPEED_100;  /* 0x03 << 2 */
    mac->link.speed1000 = EMAC_SPEED_1000; /* 0x00 */
    mac->link.duplex    = EMAC_DUPLEX_FULL;/* BIT(0) */

    /* MDIO register layout للـ bit-bang MDIO */
    mac->mii.addr          = EMAC_MDIO_CMD;
    mac->mii.data          = EMAC_MDIO_DATA;
    mac->mii.reg_shift     = 4;
    mac->mii.reg_mask      = GENMASK(8, 4);
    mac->mii.addr_shift    = 12;
    mac->mii.addr_mask     = GENMASK(16, 12);
    mac->mii.clk_csr_shift = 20;
    mac->mii.clk_csr_mask  = GENMASK(22, 20);

    mac->unicast_filter_entries = 8; /* 8 hardware MAC address slots */

    /* مفيش Synopsys ID register في الـ Allwinner EMAC */
    priv->synopsys_id = 0;

    return 0;
}
```

---

### إيه اللي الـ Framework بيعمله هو vs إيه اللي بيـdelegate للـ Glue Layer

| المسؤولية | stmmac_main.c (Framework) | dwmac-sun8i.c (Glue) |
|---|---|---|
| إنشاء الـ `net_device` | ✓ | ✗ |
| NAPI setup | ✓ | ✗ |
| TSO / checksum offload logic | ✓ | ✗ |
| phylink integration | ✓ | ✗ |
| ethtool ops | ✓ | ✗ |
| TX/RX packet path | ✓ | ✗ |
| DMA descriptor management | ✓ | ✗ |
| Clock / Reset initialization | ✗ | ✓ |
| Syscon register setup | ✗ | ✓ |
| Internal PHY power control | ✗ | ✓ |
| MDIO Mux registration | ✗ | ✓ |
| DMA engine reset/start/stop | ✗ | ✓ (non-standard DMA) |
| IRQ register bits | ✗ | ✓ |
| MAC address register layout | ✗ | ✓ |
| Speed/duplex register bits | ✗ | ✓ |
| Flow control register bits | ✗ | ✓ |

---

### الـ Interrupt Handling Flow

```
Hardware IRQ fires
      │
      ▼
stmmac_interrupt()  [stmmac_main.c]
      │
      └──▶ stmmac_napi_check() per channel
                 │
                 └──▶ sun8i_dwmac_dma_interrupt()  [glue layer]
                            │
                            ├── reads EMAC_INT_STA
                            ├── checks TX/RX bits
                            ├── updates stats (u64_stats)
                            ├── clears interrupts (write-1-to-clear)
                            └── returns flags: handle_rx | handle_tx | tx_hard_error
                                          │
                                          ▼
                              stmmac_main.c يقرر يـschedule NAPI أو يـhandle error
```

الـ `EMAC_INT_STA` register هو write-1-to-clear — الـ driver بيكتب نفس الـ value اللي قرأه عشان يـclear bits المتفاعل معها.

---

### الـ RX Filter Logic

الـ `sun8i_dwmac_set_filter()` بيـimplement الـ hardware MAC filter بـ8 slots:

```
Slot 0:  MAC الأساسي (primary unicast)
Slot 1-7: للـ additional unicast + multicast

لو عدد العناوين > 8:   → promiscuous mode (EMAC_FRM_FLT_RXALL)
لو IFF_ALLMULTI:        → multicast pass-all (EMAC_FRM_FLT_MULTICAST)
لو IFF_PROMISC:         → all frames (EMAC_FRM_FLT_RXALL)
```

الـ stmmac core بيـcall `.set_filter` كل ما الـ network stack يطلب تغيير في الـ address list (e.g., لما تضيف interface لـ bridge).

---

### ملاحظات للـ Embedded Engineer

**الـ TX/RX delay tuning** مهم جداً للـ RGMII: الـ EMAC بيدعم hardware delay chains للـ TX وRX clocks. الـ delay ده بيعوض الـ PCB trace delay بين الـ SoC والـ PHY chip. لو غلط — الـ link بيتحتل عند 1Gbps بس بيشتغل عند 100Mbps. الـ `allwinner,tx-delay-ps` و `allwinner,rx-delay-ps` في DT هم اللي بيتحكموا في ده.

**الـ pm_runtime** في الـ probe: بعد `stmmac_pltfr_probe()`، الـ MAC بيكون runtime suspended. الـ driver بيعمل `pm_runtime_get_sync()` عشان يـwake up الـ MAC قبل أي operation زي الـ MDIO mux registration.

**الـ EPHY reset quirk**: الكود بيـforce reset على الـ internal PHY حتى لو U-Boot خلاه في deasserted state — لأن الـ EMAC reset بيفشل لو الـ EPHY مش في clean state.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### EMAC Register Offsets — Cheatsheet

| Macro | Offset | الوظيفة |
|---|---|---|
| `EMAC_BASIC_CTL0` | `0x00` | Speed, duplex, loopback |
| `EMAC_BASIC_CTL1` | `0x04` | DMA burst length + software reset |
| `EMAC_INT_STA` | `0x08` | Interrupt status (W1C) |
| `EMAC_INT_EN` | `0x0C` | Interrupt enable mask |
| `EMAC_TX_CTL0` | `0x10` | TX enable |
| `EMAC_TX_CTL1` | `0x14` | TX DMA mode/threshold/start |
| `EMAC_TX_FLOW_CTL` | `0x1C` | TX flow control enable |
| `EMAC_TX_DESC_LIST` | `0x20` | Physical addr of TX descriptor ring |
| `EMAC_RX_CTL0` | `0x24` | RX enable, CRC check, flow ctrl |
| `EMAC_RX_CTL1` | `0x28` | RX DMA mode/threshold/start |
| `EMAC_RX_DESC_LIST` | `0x34` | Physical addr of RX descriptor ring |
| `EMAC_RX_FRM_FLT` | `0x38` | Frame filter (promisc/multicast) |
| `EMAC_MDIO_CMD` | `0x48` | MDIO command register |
| `EMAC_MDIO_DATA` | `0x4C` | MDIO data register |
| `EMAC_MACADDR_HI(n)` | `0x50 + n*8` | MAC address slot n (high 16 bits) |
| `EMAC_MACADDR_LO(n)` | `0x54 + n*8` | MAC address slot n (low 32 bits) |
| `EMAC_TX_DMA_STA` | `0xB0` | TX DMA status |
| `EMAC_RX_DMA_STA` | `0xC0` | RX DMA status |

---

#### EMAC_BASIC_CTL0 Bits

| Bit/Field | Macro | المعنى |
|---|---|---|
| `[0]` | `EMAC_DUPLEX_FULL` | 1 = full-duplex |
| `[1]` | `EMAC_LOOPBACK` | 1 = MAC loopback mode |
| `[3:2]` | `EMAC_SPEED_1000/100/10` | Speed select (0=1G, 2=10M, 3=100M) |

#### EMAC_INT_STA / EMAC_INT_EN Bits

| Bit | Macro | المعنى |
|---|---|---|
| 0 | `EMAC_TX_INT` | TX complete |
| 1 | `EMAC_TX_DMA_STOP_INT` | TX DMA stopped |
| 2 | `EMAC_TX_BUF_UA_INT` | TX buffer unavailable |
| 3 | `EMAC_TX_TIMEOUT_INT` | TX timeout → hard error |
| 4 | `EMAC_TX_UNDERFLOW_INT` | TX underflow → hard error |
| 5 | `EMAC_TX_EARLY_INT` | TX early interrupt |
| 8 | `EMAC_RX_INT` | RX complete |
| 9 | `EMAC_RX_BUF_UA_INT` | RX buffer unavailable |
| 10 | `EMAC_RX_DMA_STOP_INT` | RX DMA stopped |
| 11 | `EMAC_RX_TIMEOUT_INT` | RX timeout → hard error |
| 12 | `EMAC_RX_OVERFLOW_INT` | RX FIFO overflow → hard error |
| 13 | `EMAC_RX_EARLY_INT` | RX early interrupt |
| 16 | `EMAC_RGMII_STA_INT` | RGMII link status change |

#### EMAC_RX_CTL1 Threshold Bits

| Macro | Value | المعنى |
|---|---|---|
| `EMAC_RX_MD` | BIT(1) | 1 = Store-and-Forward mode |
| `EMAC_RX_TH_32` | `0<<4` | Threshold 32 bytes |
| `EMAC_RX_TH_64` | `1<<4` | Threshold 64 bytes |
| `EMAC_RX_TH_96` | `2<<4` | Threshold 96 bytes |
| `EMAC_RX_TH_128` | `3<<4` | Threshold 128 bytes |
| `EMAC_RX_DMA_EN` | BIT(30) | Enable RX DMA |
| `EMAC_RX_DMA_START` | BIT(31) | Start/restart RX DMA |

#### EMAC_TX_CTL1 Threshold Bits

| Macro | Value | المعنى |
|---|---|---|
| `EMAC_TX_MD` | BIT(1) | 1 = Store-and-Forward mode |
| `EMAC_TX_NEXT_FRM` | BIT(2) | Pipeline second frame (undocumented perf bit) |
| `EMAC_TX_TH_64` | `0<<8` | Threshold 64 bytes |
| `EMAC_TX_TH_128` | `1<<8` | Threshold 128 bytes |
| `EMAC_TX_TH_192` | `2<<8` | Threshold 192 bytes |
| `EMAC_TX_TH_256` | `3<<8` | Threshold 256 bytes |
| `EMAC_TX_DMA_EN` | BIT(30) | Enable TX DMA |
| `EMAC_TX_DMA_START` | BIT(31) | Start/restart TX DMA |

#### Syscon EMAC_CLK Register Bits (H3/A64)

| Macro | Bits | المعنى |
|---|---|---|
| `SYSCON_ETCS_MASK` | `[1:0]` | Clock source: 0=MII, 1=ExtGMII, 2=IntGMII |
| `SYSCON_EPIT` | BIT(2) | PHY interface: 1=RGMII, 0=MII |
| `SYSCON_ERXDC_SHIFT` | 5 | RX delay chain field start bit |
| `SYSCON_ETXDC_SHIFT` | 10 | TX delay chain field start bit |
| `SYSCON_RMII_EN` | BIT(13) | Enable RMII mode (overrides EPIT) |
| `H3_EPHY_SELECT` | BIT(15) | 1 = internal PHY selected |
| `H3_EPHY_SHUTDOWN` | BIT(16) | 1 = shutdown internal PHY |
| `H3_EPHY_LED_POL` | BIT(17) | 1 = LED active-low |
| `H3_EPHY_CLK_SEL` | BIT(18) | 1 = 24 MHz crystal (0 = 25 MHz) |
| `H3_EPHY_ADDR_SHIFT` | 20 | PHY MDIO address field start bit |

#### STMMAC_FLAG Bits (في `plat_dat->flags`)

| Macro | Bit | المعنى |
|---|---|---|
| `STMMAC_FLAG_HAS_SUN8I` | BIT(3) | يخبر الـ core إن الـ glue هو sun8i |
| `STMMAC_FLAG_TSO_EN` | BIT(4) | TCP Segmentation Offload |
| `STMMAC_FLAG_SPH_DISABLE` | BIT(1) | تعطيل split-header |

---

### الـ Structs الأساسية

#### 1. `struct emac_variant`

**الغرض:** بيوصف الفروق بين SoC variants مختلفة (H3, A64, R40 ...) — بيعمل "hardware capability table" ثابتة في الـ ROM.

```c
struct emac_variant {
    const struct reg_field *syscon_field;  // مكان register الـ EMAC_CLK في الـ syscon/CCU
    bool soc_has_internal_phy;             // في PHY داخلي؟ (H3/V3s فقط)
    bool support_mii;                      // يدعم MII؟
    bool support_rmii;                     // يدعم RMII؟
    bool support_rgmii;                    // يدعم RGMII؟
    u8 rx_delay_max;                       // أقصى قيمة raw لـ RX delay chain (0 = مش مدعوم)
    u8 tx_delay_max;                       // أقصى قيمة raw لـ TX delay chain (0 = مش مدعوم)
};
```

**الـ instances الثابتة (static const):**

| Instance | Internal PHY | MII | RMII | RGMII | RX delay max | TX delay max |
|---|---|---|---|---|---|---|
| `emac_variant_h3` | Yes | Yes | Yes | Yes | 31 | 7 |
| `emac_variant_v3s` | Yes | Yes | No | No | 0 | 0 |
| `emac_variant_a83t` | No | Yes | No | Yes | 31 | 7 |
| `emac_variant_r40` | No | Yes | No | Yes | 7 | 0 |
| `emac_variant_a64` | No | Yes | Yes | Yes | 31 | 7 |
| `emac_variant_h6` | No* | Yes | Yes | Yes | 31 | 7 |

*الـ H6 فيه AC200 chip على نفس الـ package بس مش على الـ die.

**الـ syscon_field:**
- معظم SoCs → `sun8i_syscon_reg_field` (offset 0x30 في system control)
- R40 فقط → `sun8i_ccu_reg_field` (offset 0x164 في CCU)

---

#### 2. `struct sunxi_priv_data`

**الغرض:** الـ private data بتاع الـ glue layer — بيتخزن في `plat_dat->bsp_priv` وبيعدي لكل callback.

```c
struct sunxi_priv_data {
    struct clk *ephy_clk;              // clock الـ internal PHY (from DT)
    struct regulator *regulator;       // optional voltage regulator لـ external PHY
    struct reset_control *rst_ephy;    // reset line الـ internal PHY
    const struct emac_variant *variant; // pointer للـ variant الخاص بالـ SoC
    struct regmap_field *regmap_field; // mapped field في الـ syscon/CCU register
    bool internal_phy_powered;         // state flag: هل الـ internal PHY شغال؟
    bool use_internal_phy;             // هل الـ mux اختار الـ internal PHY؟
    void *mux_handle;                  // opaque handle من mdio-mux library
};
```

**أهم الـ fields:**
- **الـ `regmap_field`** هو أهم field — بيعمل abstraction فوق الـ syscon register، بيسمح بقراءة/كتابة bits محددة بدون ما تلمس باقي البتات.
- **الـ `internal_phy_powered`** flag بيمنع double-enable للـ clock والـ reset.
- **الـ `mux_handle`** بيستخدمه `mdio_mux_uninit()` وقت الـ remove.

---

#### 3. `struct stmmac_dma_ops` (sun8i implementation)

**الـ `sun8i_dwmac_dma_ops`** بيملي الـ ops table اللي بيستخدمها الـ stmmac core لعمليات الـ DMA:

| Op | Implementation | الوظيفة |
|---|---|---|
| `.reset` | `sun8i_dwmac_dma_reset` | Reset الـ DMA: صفّر كل RX/TX regs |
| `.init` | `sun8i_dwmac_dma_init` | Enable RX+TX interrupts |
| `.init_rx_chan` | `sun8i_dwmac_dma_init_rx` | اكتب base address لـ RX descriptor ring |
| `.init_tx_chan` | `sun8i_dwmac_dma_init_tx` | اكتب base address لـ TX descriptor ring |
| `.dump_regs` | `sun8i_dwmac_dump_regs` | dump كل الـ EMAC regs لـ ethtool |
| `.dma_rx_mode` | `sun8i_dwmac_dma_operation_mode_rx` | Set SF mode أو threshold mode |
| `.dma_tx_mode` | `sun8i_dwmac_dma_operation_mode_tx` | Set SF mode أو threshold mode |
| `.enable_dma_transmission` | `sun8i_dwmac_enable_dma_transmission` | Set TX DMA_START + DMA_EN |
| `.enable_dma_irq` | `sun8i_dwmac_enable_dma_irq` | Enable RX/TX interrupt bits |
| `.disable_dma_irq` | `sun8i_dwmac_disable_dma_irq` | Disable RX/TX interrupt bits |
| `.start_tx` | `sun8i_dwmac_dma_start_tx` | Set TX DMA_START + DMA_EN |
| `.stop_tx` | `sun8i_dwmac_dma_stop_tx` | Clear TX DMA_EN |
| `.start_rx` | `sun8i_dwmac_dma_start_rx` | Set RX DMA_START + DMA_EN |
| `.stop_rx` | `sun8i_dwmac_dma_stop_rx` | Clear RX DMA_EN |
| `.dma_interrupt` | `sun8i_dwmac_dma_interrupt` | Handle interrupt, return handle_rx/tx flags |

---

#### 4. `struct stmmac_ops` (sun8i implementation)

**الـ `sun8i_dwmac_ops`** بيغطي MAC-level operations:

| Op | Implementation | الوظيفة |
|---|---|---|
| `.core_init` | `sun8i_dwmac_core_init` | Set burst length=8 في CTL1 |
| `.set_mac` | `sun8i_dwmac_set_mac` | Enable/disable TX+RX paths |
| `.dump_regs` | `sun8i_dwmac_dump_mac_regs` | dump EMAC regs via hw->pcsr |
| `.rx_ipc` | `sun8i_dwmac_rx_ipc_enable` | Enable CRC check في RX_CTL0 |
| `.set_filter` | `sun8i_dwmac_set_filter` | Set promiscuous / multicast / unicast filters |
| `.flow_ctrl` | `sun8i_dwmac_flow_ctrl` | Enable/disable RX+TX flow control |
| `.set_umac_addr` | `sun8i_dwmac_set_umac_addr` | اكتب MAC address في slot n |
| `.get_umac_addr` | `sun8i_dwmac_get_umac_addr` | اقرأ MAC address من slot n |
| `.set_mac_loopback` | `sun8i_dwmac_set_mac_loopback` | Toggle EMAC_LOOPBACK bit |

---

#### 5. `struct plat_stmmacenet_data` (أهم الـ fields المستخدمة هنا)

**الغرض:** الـ platform data اللي بيعبي الـ glue layer وبيديه للـ stmmac core.

| Field | القيمة في sun8i | الوظيفة |
|---|---|---|
| `rx_coe` | `STMMAC_RX_COE_TYPE2` | RX checksum offload type 2 |
| `tx_coe` | `1` | TX checksum offload enabled |
| `flags` | `STMMAC_FLAG_HAS_SUN8I` | Signal للـ core إنه sun8i variant |
| `bsp_priv` | `gmac` (sunxi_priv_data) | Pointer للـ private data |
| `init` | `sun8i_dwmac_init` | Called عند network up |
| `exit` | `sun8i_dwmac_exit` | Called عند network down |
| `mac_setup` | `sun8i_dwmac_setup` | Setup الـ mac_device_info |
| `tx_fifo_size` | `4096` | TX FIFO = 4KB |
| `rx_fifo_size` | `16384` | RX FIFO = 16KB |

---

#### 6. `struct mac_device_info` (الـ fields المستخدمة في sun8i_dwmac_setup)

**الغرض:** بيجمع كل المعلومات عن الـ MAC hardware في مكان واحد — بيعبيه `sun8i_dwmac_setup()`.

| Field | القيمة | الوظيفة |
|---|---|---|
| `pcsr` | `priv->ioaddr` | Base address للـ EMAC registers |
| `mac` | `&sun8i_dwmac_ops` | MAC operations table |
| `dma` | `&sun8i_dwmac_dma_ops` | DMA operations table |
| `link.caps` | `MAC_ASYM_PAUSE \| MAC_SYM_PAUSE \| MAC_10 \| MAC_100 \| MAC_1000` | PHY link capabilities |
| `link.speed_mask` | `GENMASK(3,2) \| EMAC_LOOPBACK` | Bits للـ speed في CTL0 |
| `link.speed10/100/1000` | `EMAC_SPEED_10/100/1000` | Speed bit values |
| `link.duplex` | `EMAC_DUPLEX_FULL` | Duplex bit value |
| `mii.addr` | `EMAC_MDIO_CMD` | MDIO command register offset |
| `mii.data` | `EMAC_MDIO_DATA` | MDIO data register offset |
| `mii.reg_shift` | 4 | PHY register field shift |
| `mii.addr_shift` | 12 | PHY address field shift |
| `mii.clk_csr_shift` | 20 | MDC clock divisor field shift |
| `unicast_filter_entries` | 8 | عدد MAC address slots |

---

### Struct Relationship Diagram

```
platform_device (pdev)
       |
       | of_device_get_match_data()
       v
emac_variant (static const) ─────────────────────────────────────┐
  .syscon_field ──→ reg_field (sun8i_syscon_reg_field /           |
                               sun8i_ccu_reg_field)               |
  .soc_has_internal_phy                                           |
  .support_mii/rmii/rgmii                                         |
  .rx_delay_max / .tx_delay_max                                   |
                                                                   |
sunxi_priv_data (gmac) ←── devm_kzalloc()                        |
  .variant ──────────────────────────────────────────────────────-┘
  .regmap_field ──→ regmap_field
                      |
                      └──→ regmap ──→ syscon/CCU registers
  .ephy_clk ──→ clk (internal PHY clock)
  .rst_ephy ──→ reset_control (internal PHY reset)
  .regulator ──→ regulator (external PHY power, optional)
  .mux_handle ──→ opaque mdio-mux handle
  .internal_phy_powered (bool)
  .use_internal_phy (bool)
       |
       | stored in plat_dat->bsp_priv
       v
plat_stmmacenet_data (plat_dat)
  .bsp_priv ──→ sunxi_priv_data
  .init ──→ sun8i_dwmac_init()
  .exit ──→ sun8i_dwmac_exit()
  .mac_setup ──→ sun8i_dwmac_setup()
       |
       | stmmac_pltfr_probe()
       v
stmmac_priv (priv)
  .plat ──→ plat_stmmacenet_data
  .ioaddr ──→ EMAC MMIO base
  .device ──→ &pdev->dev
  .mii ──→ mii_bus (parent MDIO bus)
       |
       | sun8i_dwmac_setup()
       v
mac_device_info (mac)
  .pcsr ──→ priv->ioaddr (EMAC MMIO base)
  .mac ──→ sun8i_dwmac_ops (stmmac_ops)
              .core_init / .set_mac / .set_filter / ...
  .dma ──→ sun8i_dwmac_dma_ops (stmmac_dma_ops)
              .reset / .init / .start_tx / .start_rx / ...
  .link ──→ { speed10/100/1000, duplex, caps, speed_mask }
  .mii  ──→ { addr, data, addr_shift, reg_shift, clk_csr_* }
```

---

### Lifecycle Diagram

#### Driver Probe → Ready

```
sun8i_dwmac_probe(pdev)
  │
  ├─ stmmac_get_platform_resources()   → stmmac_resources { ioaddr, irq, mac[] }
  │
  ├─ devm_kzalloc() → sunxi_priv_data *gmac
  │
  ├─ of_device_get_match_data()        → gmac->variant (emac_variant_h3/a64/...)
  │
  ├─ devm_regulator_get_optional()     → gmac->regulator (optional)
  │
  ├─ sun8i_dwmac_get_syscon_from_dev() → regmap
  │    └─ fallback: syscon_regmap_lookup_by_phandle()
  │
  ├─ devm_regmap_field_alloc()         → gmac->regmap_field
  │
  ├─ devm_stmmac_probe_config_dt()     → plat_dat (DT parsing)
  │
  ├─ Setup plat_dat:
  │    plat_dat->bsp_priv = gmac
  │    plat_dat->init     = sun8i_dwmac_init
  │    plat_dat->exit     = sun8i_dwmac_exit
  │    plat_dat->mac_setup = sun8i_dwmac_setup
  │    plat_dat->rx_coe   = STMMAC_RX_COE_TYPE2
  │    plat_dat->flags   |= STMMAC_FLAG_HAS_SUN8I
  │
  ├─ sun8i_dwmac_set_syscon()          → كتابة PHY interface bits في syscon/CCU
  │
  ├─ stmmac_pltfr_probe()              → stmmac_priv, net_device, MDIO bus
  │
  ├─ pm_runtime_get_sync()             → ensure MAC is awake
  │
  ├─ [if soc_has_internal_phy]:
  │    ├─ get_ephy_nodes()             → gmac->ephy_clk, gmac->rst_ephy
  │    └─ sun8i_dwmac_register_mdio_mux()
  │         └─ mdio_mux_init(mdio_mux_syscon_switch_fn)
  │
  ├─ [else]:
  │    └─ sun8i_dwmac_reset()          → software reset الـ EMAC
  │
  └─ pm_runtime_put()                  → allow suspend
```

#### Network Interface Up (init callback)

```
sun8i_dwmac_init(dev, priv=gmac)
  │
  ├─ regulator_enable(gmac->regulator)   [optional]
  │
  └─ [if use_internal_phy]:
       sun8i_dwmac_power_internal_phy(priv)
         │
         ├─ clk_prepare_enable(gmac->ephy_clk)
         └─ reset_control_reset(gmac->rst_ephy)
              → gmac->internal_phy_powered = true
```

#### MAC Setup (mac_setup callback)

```
sun8i_dwmac_setup(ppriv, mac)
  │
  ├─ mac->pcsr = priv->ioaddr          (EMAC base address)
  ├─ mac->mac  = &sun8i_dwmac_ops      (MAC ops)
  ├─ mac->dma  = &sun8i_dwmac_dma_ops  (DMA ops)
  ├─ mac->link.{speed10/100/1000, duplex, caps, speed_mask}
  ├─ mac->mii.{addr=EMAC_MDIO_CMD, data=EMAC_MDIO_DATA, shifts/masks}
  ├─ mac->unicast_filter_entries = 8
  └─ priv->synopsys_id = 0             (no Synopsys ID register)
```

#### Network Interface Down (exit callback)

```
sun8i_dwmac_exit(dev, priv=gmac)
  │
  ├─ [if soc_has_internal_phy]:
  │    sun8i_dwmac_unpower_internal_phy(gmac)
  │      ├─ clk_disable_unprepare(gmac->ephy_clk)
  │      ├─ reset_control_assert(gmac->rst_ephy)
  │      └─ gmac->internal_phy_powered = false
  │
  └─ regulator_disable(gmac->regulator)  [if exists]
```

#### Driver Remove

```
sun8i_dwmac_remove(pdev)
  │
  ├─ [if soc_has_internal_phy]:
  │    ├─ mdio_mux_uninit(gmac->mux_handle)
  │    ├─ sun8i_dwmac_unpower_internal_phy(gmac)
  │    ├─ reset_control_put(gmac->rst_ephy)
  │    └─ clk_put(gmac->ephy_clk)
  │
  ├─ stmmac_pltfr_remove(pdev)
  └─ sun8i_dwmac_unset_syscon(gmac)
       └─ [if soc_has_internal_phy]: write H3_EPHY_SHUTDOWN|H3_EPHY_SELECT
```

---

### Call Flow Diagrams

#### TX Path

```
kernel sends packet (net_device->ndo_start_xmit)
  └─→ stmmac_xmit()                         [stmmac core]
        └─→ stmmac_enable_dma_transmission() [stmmac core]
              └─→ mac->dma->enable_dma_transmission(ioaddr, chan)
                    └─→ sun8i_dwmac_enable_dma_transmission()
                          └─→ EMAC_TX_CTL1 |= TX_DMA_START | TX_DMA_EN
                                → EMAC hardware picks up TX descriptor ring
                                  → DMA fetches buffer from RAM
                                    → EMAC transmits frame on wire
                                      → TX interrupt fires
                                        → sun8i_dwmac_dma_interrupt()
                                              sets handle_tx flag
                                          → stmmac core frees SKB
```

#### RX Path

```
Frame arrives on wire
  → EMAC DMA writes to RX descriptor ring (base: EMAC_RX_DESC_LIST)
    → RX interrupt fires
      → sun8i_dwmac_dma_interrupt()
            reads EMAC_INT_STA
            sets handle_rx flag
            writes back to EMAC_INT_STA (W1C clear)
        → stmmac core NAPI poll
          → processes RX descriptors
            → passes SKB up to network stack
```

#### MDIO Mux Switch (H3/V3s internal PHY only)

```
phylink or ethtool accesses MDIO bus
  └─→ mdio-mux layer detects mux switch needed
        └─→ mdio_mux_syscon_switch_fn(current_child, desired_child, data=priv)
              │
              ├─ regmap_field_read(gmac->regmap_field, &reg)
              │
              ├─ [desired = INTERNAL (id=1)]:
              │    val = (reg & ~H3_EPHY_MUX_MASK) | H3_EPHY_SELECT
              │    gmac->use_internal_phy = true
              │    regmap_field_write(gmac->regmap_field, val)
              │    sun8i_dwmac_power_internal_phy()
              │      → clk_prepare_enable(ephy_clk)
              │      → reset_control_reset(rst_ephy)
              │
              ├─ [desired = EXTERNAL (id=2)]:
              │    val = (reg & ~H3_EPHY_MUX_MASK) | H3_EPHY_SHUTDOWN
              │    gmac->use_internal_phy = false
              │    regmap_field_write(gmac->regmap_field, val)
              │    sun8i_dwmac_unpower_internal_phy()
              │      → clk_disable_unprepare(ephy_clk)
              │      → reset_control_assert(rst_ephy)
              │
              └─ sun8i_dwmac_reset()
                   → EMAC_BASIC_CTL1 |= 0x01 (software reset bit)
                   → readl_poll_timeout() wait up to 100ms
```

#### DMA Interrupt Handling

```
IRQ fires
  └─→ stmmac_interrupt()                    [stmmac core]
        └─→ mac->dma->dma_interrupt(priv, ioaddr, x, chan, dir)
              └─→ sun8i_dwmac_dma_interrupt()
                    v = readl(EMAC_INT_STA)
                    │
                    ├─ v & EMAC_TX_INT       → ret |= handle_tx
                    │                           stats->tx_normal_irq_n++
                    ├─ v & EMAC_TX_TIMEOUT   → ret |= tx_hard_error
                    ├─ v & EMAC_TX_UNDERFLOW → ret |= tx_hard_error
                    │                           x->tx_undeflow_irq++
                    ├─ v & EMAC_RX_INT       → ret |= handle_rx
                    │                           stats->rx_normal_irq_n++
                    ├─ v & EMAC_RX_OVERFLOW  → ret |= tx_hard_error
                    │                           x->rx_overflow_irq++
                    ├─ v & EMAC_RGMII_STA   → x->irq_rgmii_n++
                    │
                    └─ writel(v, EMAC_INT_STA)   (W1C: clear handled bits)
```

#### DMA Operation Mode Setup

```
stmmac core sets DMA mode
  └─→ mac->dma->dma_rx_mode(priv, ioaddr, mode, chan, fifosz, qmode)
        └─→ sun8i_dwmac_dma_operation_mode_rx()
              v = readl(EMAC_RX_CTL1)
              │
              ├─ [mode == SF_DMA_MODE]:
              │    v |= EMAC_RX_MD        (store-and-forward)
              │
              └─ [threshold mode]:
                   v &= ~EMAC_RX_MD
                   v &= ~EMAC_RX_TH_MASK
                   if mode < 32  → v |= EMAC_RX_TH_32
                   if mode < 64  → v |= EMAC_RX_TH_64
                   if mode < 96  → v |= EMAC_RX_TH_96
                   if mode < 128 → v |= EMAC_RX_TH_128
              writel(v, EMAC_RX_CTL1)
```

#### Syscon Configuration Flow (probe time)

```
sun8i_dwmac_set_syscon(dev, plat_dat)
  │
  ├─ [soc_has_internal_phy]:
  │    reg |= H3_EPHY_CLK_SEL           (force 24 MHz crystal)
  │    reg |= H3_EPHY_LED_POL           (if DT: allwinner,leds-active-low)
  │    reg |= (phy_addr << H3_EPHY_ADDR_SHIFT)
  │
  ├─ [allwinner,tx-delay-ps in DT]:
  │    val = tx_delay_ps / 100
  │    reg |= (val << SYSCON_ETXDC_SHIFT)
  │
  ├─ [allwinner,rx-delay-ps in DT]:
  │    val = rx_delay_ps / 100
  │    reg |= (val << SYSCON_ERXDC_SHIFT)
  │
  ├─ [phy_interface]:
  │    MII      → nothing (default)
  │    RGMII*   → reg |= SYSCON_EPIT | SYSCON_ETCS_INT_GMII
  │    RMII     → reg |= SYSCON_RMII_EN | SYSCON_ETCS_EXT_GMII
  │
  └─ regmap_field_write(gmac->regmap_field, reg)
       → atomically updates EMAC_CLK bits in syscon/CCU
```

---

### Locking Strategy

#### القاعدة العامة

**التعليق في بداية الملف صريح:**

```c
/* Locking: no locking is necessary in this file because all necessary
 *          locking is done in the "stmmac files"
 */
```

الـ glue layer نفسه **مش محتاج locks** — كل الـ synchronization بيتعمل في الـ stmmac core.

#### توزيع الـ Locking في المنظومة الكاملة

| Resource | Lock | المسؤول | الملف |
|---|---|---|---|
| RX/TX descriptor rings | `stmmac_channel.lock` (spinlock) | stmmac core | `stmmac_main.c` |
| TX queue state | per-queue spinlock | stmmac core | `stmmac_main.c` |
| MDIO bus access | `mii_bus->mdio_lock` (mutex) | mdio-bus layer | `drivers/net/mdio/...` |
| EMAC registers (normal) | لا يوجد — single-threaded context | glue layer | `dwmac-sun8i.c` |
| syscon regmap | regmap internal lock | regmap framework | `drivers/base/regmap/` |
| `internal_phy_powered` flag | لا يوجد — بيتغير فقط من mux switch fn | mdio-mux (serialized) | `dwmac-sun8i.c` |
| `use_internal_phy` flag | لا يوجد — نفس السبب | mdio-mux (serialized) | `dwmac-sun8i.c` |

#### ملاحظات مهمة على الـ Locking

1. **الـ `sun8i_dwmac_dma_interrupt()`** بيشتغل في interrupt context — بيستخدم `u64_stats_update_begin/end` اللي بيعمل `local_irq_save` لحماية الـ per-CPU stats.

2. **الـ `mdio_mux_syscon_switch_fn()`** بيشتغل من سياق الـ phylink/mdio اللي هو serialized من mdio-mux layer — مفيش حاجة لـ explicit lock.

3. **الـ `regmap_field_read/write()`** — الـ regmap framework نفسه فيه internal lock (`regmap->lock`) بيحميه من concurrent access.

4. **الـ `sun8i_dwmac_set_filter()`** بيتكلم من `ndo_set_rx_mode` اللي بيتكلم من `rtnl_lock` context — مفيش حاجة لـ explicit lock في الـ glue.

#### Lock Ordering (في حالة الـ internal PHY)

```
rtnl_lock
  └─→ mii_bus->mdio_lock
        └─→ mdio_mux_syscon_switch_fn()
              └─→ regmap internal lock
                    → (no further locks)
```

مفيش risk لـ deadlock لأن الـ sun8i glue مش بياخد أي lock من جانبه.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs (Cheatsheet)

#### DMA Operations Group

| Function | Prototype المختصر | الغرض |
|---|---|---|
| `sun8i_dwmac_dma_reset` | `int (void __iomem *)` | Reset كل ريجستراتالـ DMA وكلير الـ interrupts |
| `sun8i_dwmac_dma_init` | `void (ioaddr, dma_cfg)` | تفعيل الـ RX/TX interrupts الأساسية |
| `sun8i_dwmac_dma_init_rx` | `void (priv, ioaddr, cfg, phy, chan)` | كتابة عنوان الـ RX descriptor ring |
| `sun8i_dwmac_dma_init_tx` | `void (priv, ioaddr, cfg, phy, chan)` | كتابة عنوان الـ TX descriptor ring |
| `sun8i_dwmac_dma_start_tx` | `void (priv, ioaddr, chan)` | تشغيل الـ TX DMA engine |
| `sun8i_dwmac_dma_stop_tx` | `void (priv, ioaddr, chan)` | إيقاف الـ TX DMA engine |
| `sun8i_dwmac_dma_start_rx` | `void (priv, ioaddr, chan)` | تشغيل الـ RX DMA engine |
| `sun8i_dwmac_dma_stop_rx` | `void (priv, ioaddr, chan)` | إيقاف الـ RX DMA engine |
| `sun8i_dwmac_enable_dma_irq` | `void (priv, ioaddr, chan, rx, tx)` | تفعيل الـ RX و/أو TX interrupt |
| `sun8i_dwmac_disable_dma_irq` | `void (priv, ioaddr, chan, rx, tx)` | تعطيل الـ RX و/أو TX interrupt |
| `sun8i_dwmac_dma_interrupt` | `int (priv, ioaddr, x, chan, dir)` | معالجة الـ DMA interrupt وتصنيف الأحداث |
| `sun8i_dwmac_dma_operation_mode_rx` | `void (priv, ioaddr, mode, chan, fifosz, qmode)` | ضبط threshold أو SF mode للـ RX FIFO |
| `sun8i_dwmac_dma_operation_mode_tx` | `void (priv, ioaddr, mode, chan, fifosz, qmode)` | ضبط threshold أو SF mode للـ TX FIFO |
| `sun8i_dwmac_enable_dma_transmission` | `void (ioaddr, chan)` | كِك الـ TX DMA بعد إضافة descriptor |
| `sun8i_dwmac_dump_regs` | `void (priv, ioaddr, reg_space)` | dump كل ريجستراتالـ EMAC للـ ethtool |

#### MAC Operations Group

| Function | Prototype المختصر | الغرض |
|---|---|---|
| `sun8i_dwmac_core_init` | `void (hw, dev)` | ضبط burst length في `EMAC_BASIC_CTL1` |
| `sun8i_dwmac_set_mac` | `void (ioaddr, enable)` | تشغيل/إيقاف الـ TX و RX path |
| `sun8i_dwmac_set_umac_addr` | `void (hw, addr, reg_n)` | برمجة unicast MAC address في slot محدد |
| `sun8i_dwmac_get_umac_addr` | `void (hw, addr, reg_n)` | قراءة MAC address من slot محدد |
| `sun8i_dwmac_rx_ipc_enable` | `int (hw)` | تفعيل CRC check على الـ RX path |
| `sun8i_dwmac_set_filter` | `void (hw, dev)` | ضبط frame filter (promisc/multicast/unicast) |
| `sun8i_dwmac_flow_ctrl` | `void (hw, duplex, fc, pause, tx_cnt)` | تفعيل/تعطيل RX/TX flow control |
| `sun8i_dwmac_set_mac_loopback` | `void (ioaddr, enable)` | تشغيل/إيقاف loopback mode في `EMAC_BASIC_CTL0` |
| `sun8i_dwmac_dump_mac_regs` | `void (hw, reg_space)` | dump MAC registers عبر `stmmac_ops->dump_regs` |
| `sun8i_dwmac_reset` | `int (priv)` | software reset للـ EMAC مع polling على انتهاء الـ reset |

#### Platform / Glue Group

| Function | الغرض |
|---|---|
| `sun8i_dwmac_probe` | نقطة دخول الـ driver — تهيئة كل شيء |
| `sun8i_dwmac_remove` | تنظيف كل الموارد عند إزالة الـ driver |
| `sun8i_dwmac_shutdown` | إيقاف الـ PHY والـ regulator عند shutdown |
| `sun8i_dwmac_init` | callback من الـ stmmac core: تشغيل الـ regulator والـ PHY |
| `sun8i_dwmac_exit` | callback من الـ stmmac core: إيقاف الـ PHY والـ regulator |
| `sun8i_dwmac_setup` | ربط الـ ops structs وضبط MAC capabilities |
| `sun8i_dwmac_set_syscon` | برمجة SYSCON بناءً على PHY interface mode والـ DT properties |
| `sun8i_dwmac_unset_syscon` | إعادة الـ SYSCON لحالة shutdown للـ internal PHY |
| `sun8i_dwmac_get_syscon_from_dev` | جلب الـ regmap من الـ syscon device مباشرة |

#### Internal PHY Group

| Function | الغرض |
|---|---|
| `get_ephy_nodes` | البحث في الـ DT عن الـ internal PHY node وجلب clk وreset |
| `sun8i_dwmac_power_internal_phy` | تشغيل clock وreset الـ internal PHY |
| `sun8i_dwmac_unpower_internal_phy` | إيقاف clock وassert reset للـ internal PHY |
| `mdio_mux_syscon_switch_fn` | switch الـ MDIO mux بين الـ internal وexternal PHY |
| `sun8i_dwmac_register_mdio_mux` | تسجيل الـ mdio-mux مع الـ mdio_mux subsystem |

---

### Group 1: DMA Initialization & Reset

هذه المجموعة مسؤولة عن إعداد الـ DMA engine من الصفر وتوفير نقطة reset نظيفة قبل أي عملية. الـ sun8i EMAC لا يستخدم الـ Synopsys DMA descriptor format المعتاد في الـ GMAC4، بل له ريجستراتخاصة بيه.

---

#### `sun8i_dwmac_dma_reset`

```c
static int sun8i_dwmac_dma_reset(void __iomem *ioaddr)
```

بتعمل software reset كامل للـ DMA engine عن طريق تصفير كل الريجستراتالمتعلقة بالـ RX وTX، وكمان بتكلير كل الـ pending interrupts عن طريق كتابة `0x1FFFFFF` على `EMAC_INT_STA` (write-1-to-clear).

**Parameters:**
- `ioaddr` — الـ base address المُعيّنة للـ EMAC register space

**Return:** دايماً صفر (الـ function مش بتفشل)

**Key details:**
- بتتكلم مباشرة على الريجستراتمن غير locking — الـ locking بيتم في الـ stmmac core قبل ما تتكلم على أي DMA ops
- بتصفر: `EMAC_RX_CTL1`, `EMAC_TX_CTL1`, `EMAC_RX_FRM_FLT`, `EMAC_RX_DESC_LIST`, `EMAC_TX_DESC_LIST`, `EMAC_INT_EN`
- الـ caller: `stmmac_dma_ops->reset` — بيتكلمها الـ stmmac core أثناء `stmmac_hw_setup()`

---

#### `sun8i_dwmac_dma_init`

```c
static void sun8i_dwmac_dma_init(void __iomem *ioaddr,
                                 struct stmmac_dma_cfg *dma_cfg)
```

بتفعل الـ basic interrupts بس (RX وTX)، وبتكلير أي pending status. الـ sun8i EMAC ما بيدعمش الـ DMA burst length configuration اللي في الـ Synopsys GMAC4، فالـ `dma_cfg` struct بيتجاهل خالص.

**Parameters:**
- `ioaddr` — الـ EMAC I/O base
- `dma_cfg` — struct الـ DMA configuration (بيتجاهل في الـ sun8i implementation)

**Return:** void

**Key details:** بتكتب `EMAC_RX_INT | EMAC_TX_INT` على `EMAC_INT_EN` — ده الـ minimum اللازم لأي operation

---

#### `sun8i_dwmac_dma_init_rx` و `sun8i_dwmac_dma_init_tx`

```c
static void sun8i_dwmac_dma_init_rx(struct stmmac_priv *priv,
                                    void __iomem *ioaddr,
                                    struct stmmac_dma_cfg *dma_cfg,
                                    dma_addr_t dma_rx_phy, u32 chan)

static void sun8i_dwmac_dma_init_tx(struct stmmac_priv *priv,
                                    void __iomem *ioaddr,
                                    struct stmmac_dma_cfg *dma_cfg,
                                    dma_addr_t dma_tx_phy, u32 chan)
```

كل function منهم بتكتب الـ physical address للـ descriptor ring في الريجستر المناسب. الـ sun8i EMAC بيعتمد على عنوان واحد فقط (32-bit) للـ descriptor ring ومش بيدعم الـ multi-channel DMA.

**Parameters:**
- `priv` — الـ stmmac private data
- `ioaddr` — الـ EMAC I/O base
- `dma_cfg` — بيتجاهل
- `dma_rx_phy` / `dma_tx_phy` — الـ DMA physical address للـ descriptor ring
- `chan` — رقم الـ channel (الـ sun8i بيعمل single channel فقط)

**Key details:** بتستخدم `lower_32_bits()` لأن الـ sun8i EMAC بيدعم 32-bit addresses فقط — مش بيدعم الـ 64-bit DMA addressing

---

### Group 2: DMA Runtime Control

هذه المجموعة بتتحكم في تشغيل وإيقاف الـ DMA engines، وتفعيل/تعطيل الـ interrupts، ومعالجة الـ interrupt events أثناء التشغيل الفعلي.

---

#### `sun8i_dwmac_dma_start_tx` و `sun8i_dwmac_enable_dma_transmission`

```c
static void sun8i_dwmac_dma_start_tx(struct stmmac_priv *priv,
                                     void __iomem *ioaddr, u32 chan)

static void sun8i_dwmac_enable_dma_transmission(void __iomem *ioaddr, u32 chan)
```

الاتنين بيعملوا نفس الشيء بالظبط: بيسيتوا `EMAC_TX_DMA_START` و`EMAC_TX_DMA_EN` في ريجستر `EMAC_TX_CTL1`. الفرق الوحيد إن `sun8i_dwmac_enable_dma_transmission` ما بياخدش `priv` وبيتكلمها الـ stmmac core بعد ما يضيف descriptor جديد، بينما `sun8i_dwmac_dma_start_tx` هي الـ ops callback الرسمية.

**Key details:**
- `EMAC_TX_DMA_START` (BIT 31) = kick للـ DMA إنه يبدأ من أول descriptor
- `EMAC_TX_DMA_EN` (BIT 30) = تفعيل الـ TX DMA بشكل عام
- الفرق العملي بين الـ function دول بسيط — واضح إن الـ implementation جه من refactoring ناقص

---

#### `sun8i_dwmac_dma_stop_tx` و `sun8i_dwmac_dma_stop_rx`

```c
static void sun8i_dwmac_dma_stop_tx(struct stmmac_priv *priv,
                                    void __iomem *ioaddr, u32 chan)

static void sun8i_dwmac_dma_stop_rx(struct stmmac_priv *priv,
                                    void __iomem *ioaddr, u32 chan)
```

بيكليروا الـ `EMAC_TX_DMA_EN` / `EMAC_RX_DMA_EN` بس من غير ما ينتظروا الـ DMA يخلص الـ in-flight transactions. ده graceful stop مش forced abort — الـ stmmac core هو اللي بيضمن flush الـ pending frames قبل الـ stop.

---

#### `sun8i_dwmac_enable_dma_irq` و `sun8i_dwmac_disable_dma_irq`

```c
static void sun8i_dwmac_enable_dma_irq(struct stmmac_priv *priv,
                                       void __iomem *ioaddr, u32 chan,
                                       bool rx, bool tx)

static void sun8i_dwmac_disable_dma_irq(struct stmmac_priv *priv,
                                        void __iomem *ioaddr, u32 chan,
                                        bool rx, bool tx)
```

بيعملوا read-modify-write على `EMAC_INT_EN` لتفعيل أو تعطيل الـ `EMAC_RX_INT` و/أو `EMAC_TX_INT` بشكل مستقل. الـ `chan` parameter بيتجاهل لأن الـ sun8i EMAC ما بيدعمش multi-channel interrupts.

**Parameters:**
- `rx` / `tx` — flags لتحديد أي direction تتأثر

---

#### `sun8i_dwmac_dma_interrupt`

```c
static int sun8i_dwmac_dma_interrupt(struct stmmac_priv *priv,
                                     void __iomem *ioaddr,
                                     struct stmmac_extra_stats *x, u32 chan,
                                     u32 dir)
```

الـ IRQ handler الرئيسي للـ DMA. بيقرا ريجستر الـ interrupt status، بيفلتر الأحداث حسب الـ direction المطلوبة، بيحدث الـ per-CPU stats، وبيعيد bitmask بيحدد الـ action المطلوب من الـ stmmac core.

**Parameters:**
- `priv` — الـ stmmac private data (للوصول للـ per-CPU stats)
- `ioaddr` — الـ EMAC I/O base
- `x` — struct الـ extra stats لتسجيل أحداث زي underflow وoverflow
- `chan` — رقم الـ channel (بيتجاهل)
- `dir` — `DMA_DIR_RX` أو `DMA_DIR_TX` أو الاتنين

**Return values (bitmask):**
- `handle_rx` — في الـ RX interrupt عادي
- `handle_tx` — في الـ TX interrupt عادي
- `tx_hard_error` — في حالة timeout أو underflow أو overflow

**Pseudocode flow:**
```
v = readl(EMAC_INT_STA)
if dir == RX: v &= EMAC_INT_MSK_RX
if dir == TX: v &= EMAC_INT_MSK_TX

for each bit in v:
    if TX_INT:       ret |= handle_tx; update tx_normal_irq_n
    if TX_TIMEOUT:   ret |= tx_hard_error
    if TX_UNDERFLOW: ret |= tx_hard_error; x->tx_undeflow_irq++
    if RX_INT:       ret |= handle_rx; update rx_normal_irq_n
    if RX_TIMEOUT:   ret |= tx_hard_error  /* note: reuses TX error flag */
    if RX_OVERFLOW:  ret |= tx_hard_error; x->rx_overflow_irq++
    if RGMII_STA:    x->irq_rgmii_n++

writel(v, EMAC_INT_STA)   /* write-1-to-clear */
return ret
```

**Key details:**
- الـ `EMAC_RX_TIMEOUT_INT` بيرجع `tx_hard_error` — ده quirk في الـ implementation مش error في الفهم
- الـ per-CPU stats بتتحدث باستخدام `u64_stats_update_begin/end` لضمان consistency على الـ SMP
- write-1-to-clear pattern: ممنوع تكتب كل الـ bits، بتكتب بس اللي اتشالت

---

#### `sun8i_dwmac_dma_operation_mode_rx` و `sun8i_dwmac_dma_operation_mode_tx`

```c
static void sun8i_dwmac_dma_operation_mode_rx(struct stmmac_priv *priv,
                                              void __iomem *ioaddr, int mode,
                                              u32 channel, int fifosz, u8 qmode)

static void sun8i_dwmac_dma_operation_mode_tx(struct stmmac_priv *priv,
                                              void __iomem *ioaddr, int mode,
                                              u32 channel, int fifosz, u8 qmode)
```

بيضبطوا الـ FIFO operation mode — إما **Store-and-Forward (SF)** أو **Threshold mode**. في الـ SF mode، الـ DMA ما بيبدأش الـ transfer إلا بعد ما الـ frame يكمل في الـ FIFO كامل. في الـ threshold mode، الـ transfer بيبدأ بعد ما عدد معين من الـ bytes يتجمع.

**Parameters:**
- `mode` — إما `SF_DMA_MODE` أو قيمة threshold بالـ bytes
- `fifosz` و `qmode` — بيتجاهلوا في الـ sun8i implementation

**RX Thresholds:** 32 / 64 / 96 / 128 bytes
**TX Thresholds:** 64 / 128 / 192 / 256 bytes

**Key detail للـ TX:** في الـ SF mode، بيسيت `EMAC_TX_NEXT_FRM` (bit غير موثق في الـ Allwinner datasheet، موجود في الـ BSP بس) اللي بيحسن الأداء عن طريق السماح بمعالجة الـ frame الجاي قبل ما الـ current ينتهي.

---

### Group 3: MAC Core Operations

هذه المجموعة مسؤولة عن ضبط سلوك الـ MAC نفسه: تهيئته، التحكم في الـ filtering، الـ addressing، وتفعيل أو تعطيل الـ data path.

---

#### `sun8i_dwmac_core_init`

```c
static void sun8i_dwmac_core_init(struct mac_device_info *hw,
                                  struct net_device *dev)
```

أبسط function في الـ MAC group. بتكتب burst length = 8 في الـ `EMAC_BASIC_CTL1`. ده الإعداد الوحيد اللي بيعمله الـ core init لأن الـ sun8i EMAC مش بيدعم features زي الـ checksum offload configuration في الـ core init stage.

**Key details:**
- `EMAC_BURSTLEN_SHIFT = 24` — الـ burst length بيتحط في bits [31:24]
- القيمة 8 معناها burst length = 8 × 4 bytes = 32 bytes على الـ AHB bus

---

#### `sun8i_dwmac_set_mac`

```c
static void sun8i_dwmac_set_mac(void __iomem *ioaddr, bool enable)
```

بتشغل أو بتوقف الـ TX transmitter وRX receiver في نفس الوقت عن طريق `EMAC_TX_TRANSMITTER_EN` (BIT 31 في `EMAC_TX_CTL0`) و`EMAC_RX_RECEIVER_EN` (BIT 31 في `EMAC_RX_CTL0`).

**Caller context:** بيتكلمها الـ stmmac core عند `ndo_open()` وعند `ndo_stop()` — دايماً من process context

---

#### `sun8i_dwmac_set_umac_addr`

```c
static void sun8i_dwmac_set_umac_addr(struct mac_device_info *hw,
                                      const unsigned char *addr,
                                      unsigned int reg_n)
```

بتبرمج MAC address في أي من الـ 8 slots المتاحة في الـ EMAC. لو `addr == NULL`، بتكلير الـ slot. الـ slot 0 بيتعامل معه بطريقة مختلفة — الـ slots من 1 لـ 7 محتاجة `MAC_ADDR_TYPE_DST` (BIT 31) يتسيت في الـ high register لعشان يتفعلوا كـ destination address filters.

**Parameters:**
- `hw` — pointer على الـ `mac_device_info` اللي فيه `pcsr` = ioaddr
- `addr` — الـ 6-byte MAC address، أو NULL لمسح الـ slot
- `reg_n` — رقم الـ slot من 0 لـ 7

**Key detail:** بتستخدم `stmmac_set_mac_addr()` من الـ stmmac core library لأنها بتتعامل مع الـ byte ordering بشكل صح (الـ MAC أفضل طريقة لكتابة الـ 6 bytes في ريجسترين).

---

#### `sun8i_dwmac_get_umac_addr`

```c
static void sun8i_dwmac_get_umac_addr(struct mac_device_info *hw,
                                      unsigned char *addr,
                                      unsigned int reg_n)
```

عكس `set_umac_addr` — بتقرا الـ MAC address من الـ slot المحدد. بيستخدم `stmmac_get_mac_addr()`.

---

#### `sun8i_dwmac_rx_ipc_enable`

```c
static int sun8i_dwmac_rx_ipc_enable(struct mac_device_info *hw)
```

بتفعل RX CRC checking عن طريق سيت `EMAC_RX_DO_CRC` (BIT 27) في `EMAC_RX_CTL0`.

**Return:** دايماً `1` — ده مش error code! الـ stmmac core بيستخدم الـ return value مباشرة كـ boolean للـ `hw_cap.rx_coe` flag. لو رجعت صفر، الـ stmmac هيفكر إن الـ IPC مش متاح ومش هيتفعل.

---

#### `sun8i_dwmac_set_filter`

```c
static void sun8i_dwmac_set_filter(struct mac_device_info *hw,
                                   struct net_device *dev)
```

الأكثر تعقيداً في الـ MAC group. بتضبط frame reception filter بناءً على حالة الـ network device.

**Logic flow:**

```
if IFF_PROMISC:
    v = EMAC_FRM_FLT_RXALL  /* accept everything */
elif IFF_ALLMULTI:
    v |= EMAC_FRM_FLT_MULTICAST  /* accept all multicast */
elif (total_addrs <= 8):  /* hw has 8 unicast filter slots */
    program each MC and UC addr into slots 1..7
    v = EMAC_FRM_FLT_CTL  /* enable control frame filter */
else:
    /* too many addresses, fall back to promiscuous */
    v = EMAC_FRM_FLT_RXALL

/* clear unused slots */
while i < 8: set_umac_addr(NULL, i++)

writel(v, EMAC_RX_FRM_FLT)
```

**Key details:**
- الـ `hw->unicast_filter_entries = 8` — الـ EMAC عنده 8 address slots (0 للـ primary MAC)
- عند fallback للـ promiscuous، بيطبع warning مرة واحدة بس (لو الريجستر مش كان RXALL من قبل)
- الـ `EMAC_FRM_FLT_CTL` بيفلتر الـ control frames زي PAUSE frames

---

#### `sun8i_dwmac_flow_ctrl`

```c
static void sun8i_dwmac_flow_ctrl(struct mac_device_info *hw,
                                  unsigned int duplex, unsigned int fc,
                                  unsigned int pause_time, u32 tx_cnt)
```

بتفعل أو بتعطل الـ IEEE 802.3x flow control على الـ RX وTX بشكل مستقل.

**Parameters:**
- `fc` — `FLOW_AUTO` = تفعيل، أي قيمة تانية = تعطيل
- `duplex`, `pause_time`, `tx_cnt` — بيتجاهلوا (الـ sun8i ما بيدعمش asymmetric pause)

**Key detail:** بتعمل RMW على `EMAC_RX_CTL0` (للـ RX flow control) وعلى `EMAC_TX_FLOW_CTL` (للـ TX). الـ EMAC بيدعم بس `FLOW_AUTO` — ما فيش control على الـ pause time.

---

#### `sun8i_dwmac_reset`

```c
static int sun8i_dwmac_reset(struct stmmac_priv *priv)
```

Software reset للـ EMAC كامل (مش بس الـ DMA) عن طريق سيت BIT 0 في `EMAC_BASIC_CTL1` والانتظار لحد ما يتكلير.

**Key details:**
- بيستخدم `readl_poll_timeout()` مع timeout = 100ms وinterval = 100µs
- الـ timeout اتختار بحرص: بعض الـ boards زي OrangePI Zero محتاجة أكتر من 10ms لو ما فيش cable متوصل
- **Caller:** بيتكلمها `sun8i_dwmac_probe()` للـ external PHY variants، وبيتكلمها `mdio_mux_syscon_switch_fn()` بعد كل switch للـ SYSCON — لأن الـ MAC بيحتاج reset عشان يعرف الـ PHY interface الجديد

---

#### `sun8i_dwmac_set_mac_loopback`

```c
static void sun8i_dwmac_set_mac_loopback(void __iomem *ioaddr, bool enable)
```

بتسيت أو بتكلير `EMAC_LOOPBACK` (BIT 1) في `EMAC_BASIC_CTL0`. الـ loopback bit بيتـ reset تلقائياً عند كل link change، فالـ stmmac core بيكلمها في كل مرة speed تتغير (ولده `mac->link.speed_mask` بيشمل `EMAC_LOOPBACK` عشان الـ phylink يـ mask اللي بتعمله).

---

### Group 4: Platform Initialization & Probe

هذه المجموعة هي الـ glue الحقيقية بين الـ Allwinner hardware والـ stmmac framework. بتتعامل مع الـ Device Tree، الـ regmap، الـ SYSCON، والـ power management.

---

#### `sun8i_dwmac_get_syscon_from_dev`

```c
static struct regmap *sun8i_dwmac_get_syscon_from_dev(struct device_node *node)
```

بتحاول تجيب الـ regmap من الـ syscon device مباشرة عن طريق الـ `phandle` في الـ DT (`syscon` property). الفرق عن `syscon_regmap_lookup_by_phandle()` إن دي بتستخدم `dev_get_regmap()` على الـ platform_device نفسه، مش الـ syscon generic mechanism.

**Pseudocode flow:**
```
syscon_node = of_parse_phandle(node, "syscon", 0)
syscon_pdev = of_find_device_by_node(syscon_node)
if !syscon_pdev: return ERR_PTR(-EPROBE_DEFER)

regmap = dev_get_regmap(&syscon_pdev->dev, NULL)
if !regmap: return ERR_PTR(-EINVAL)

platform_device_put(syscon_pdev)
of_node_put(syscon_node)
return regmap
```

**Return:** `struct regmap *` أو `ERR_PTR()`:
- `-ENODEV` — ما فيش `syscon` phandle
- `-EPROBE_DEFER` — الـ syscon device لسه ما اتـ probe
- `-EINVAL` — الـ device ما عندوش regmap

**Key detail:** ده موجود عشان الـ R40 SoC بيعمل expose للـ GMAC clock control register عبر الـ CCU driver مش عبر الـ syscon — فمحتاج approach مختلف. الـ probe بتعمل fallback لـ `syscon_regmap_lookup_by_phandle()` لو الـ function دي فشلت.

---

#### `sun8i_dwmac_set_syscon`

```c
static int sun8i_dwmac_set_syscon(struct device *dev,
                                  struct plat_stmmacenet_data *plat)
```

أهم function في الـ glue layer. بتبرمج الـ SYSCON/CCU register اللي بيتحكم في الـ PHY interface mode وتوقيت الـ clock.

**Pseudocode flow:**
```
if soc_has_internal_phy:
    if "allwinner,leds-active-low" in DT: reg |= H3_EPHY_LED_POL
    reg |= H3_EPHY_CLK_SEL          /* force 24MHz xtal */
    phy_addr = of_mdio_parse_addr()
    reg |= (phy_addr << H3_EPHY_ADDR_SHIFT)

if "allwinner,tx-delay-ps" in DT:
    val = tx_delay_ps / 100
    validate(val <= variant->tx_delay_max)
    reg |= (val << SYSCON_ETXDC_SHIFT)

if "allwinner,rx-delay-ps" in DT:
    val = rx_delay_ps / 100
    validate(val <= variant->rx_delay_max)
    reg |= (val << SYSCON_ERXDC_SHIFT)

switch phy_interface:
    case MII:    /* nothing, default */
    case RGMII*: reg |= SYSCON_EPIT | SYSCON_ETCS_INT_GMII
    case RMII:   reg |= SYSCON_RMII_EN | SYSCON_ETCS_EXT_GMII
    default:     return -EINVAL

regmap_field_write(gmac->regmap_field, reg)
```

**Key details:**
- الـ TX/RX delay بيتحسب بـ picoseconds وبيتحول لـ raw register value عن طريق قسمة على 100
- الـ RGMII mode بيستخدم `SYSCON_ETCS_INT_GMII` (= 0x2) مش `EXT_GMII` — ده تسمية Allwinner للـ internal GMII clock
- **Error path:** بيرجع `-EINVAL` لو الـ delay قيمته أكبر من `variant->*_delay_max` أو لو الـ interface غير مدعوم

---

#### `sun8i_dwmac_unset_syscon`

```c
static void sun8i_dwmac_unset_syscon(struct sunxi_priv_data *gmac)
```

بتعيد الـ SYSCON register لحالة آمنة: `H3_EPHY_SHUTDOWN | H3_EPHY_SELECT`. ده بيحط الـ internal PHY في shutdown mode. بس بتعمل حاجة لو الـ SoC عنده internal PHY — على الـ variants التانية الـ function دي no-op.

---

#### `sun8i_dwmac_setup`

```c
static int sun8i_dwmac_setup(void *ppriv, struct mac_device_info *mac)
```

الـ `mac_setup` callback — بيتكلمها الـ stmmac core بعد الـ probe مباشرة. بتربط كل الـ ops structs وبتضبط الـ MAC capabilities.

**ما بتعمله:**
1. بتسيت `mac->pcsr = priv->ioaddr` — ده الـ I/O base اللي كل MAC ops هتستخدمه
2. بتربط `mac->mac = &sun8i_dwmac_ops` و`mac->dma = &sun8i_dwmac_dma_ops`
3. بتسيت `IFF_UNICAST_FLT` flag على الـ net_device
4. بتضبط `mac->link` struct للـ phylink integration:
   - Speed bits في `EMAC_BASIC_CTL0[3:2]`
   - Duplex bit في `EMAC_BASIC_CTL0[0]`
5. بتضبط الـ MDIO register offsets في `mac->mii` struct
6. بتسيت `priv->synopsys_id = 0` — لأن الـ sun8i EMAC مش Synopsys IP حقيقي

**Key detail للـ MII:** الـ `mac->mii.addr = EMAC_MDIO_CMD` و`mac->mii.data = EMAC_MDIO_DATA` — ده بيعلم الـ stmmac core مكان ريجستراتالـ MDIO عشان يقدر يعمل bitbanging أو hardware MDIO access.

---

#### `sun8i_dwmac_probe`

```c
static int sun8i_dwmac_probe(struct platform_device *pdev)
```

نقطة الدخول الرئيسية. أطول function في الـ file وأكثرها تعقيداً.

**Pseudocode flow:**

```
1. stmmac_get_platform_resources()       /* جيب IRQ و I/O memory */
2. alloc sunxi_priv_data (gmac)
3. gmac->variant = of_device_get_match_data()   /* H3/V3s/A83T/R40/A64/H6 */
4. gmac->regulator = devm_regulator_get_optional("phy")
5. regmap = sun8i_dwmac_get_syscon_from_dev()
   if failed: regmap = syscon_regmap_lookup_by_phandle("syscon")
6. gmac->regmap_field = devm_regmap_field_alloc(variant->syscon_field)
7. plat_dat = devm_stmmac_probe_config_dt()     /* parse DT */
8. set plat_dat callbacks: init/exit/mac_setup, fifo sizes, COE flags
9. sun8i_dwmac_set_syscon()       /* برمج PHY interface في SYSCON */
10. stmmac_pltfr_probe()          /* الـ main stmmac probe */
11. pm_runtime_get_sync()         /* ضمن MAC مشتغل */
12. if internal PHY:
      get_ephy_nodes()
      sun8i_dwmac_register_mdio_mux()
    else:
      sun8i_dwmac_reset()
13. pm_runtime_put()
```

**Error paths:**
```
dwmac_mux:    reset_control_put + clk_put
dwmac_remove: pm_runtime_put_noidle + stmmac_pltfr_remove
dwmac_syscon: sun8i_dwmac_unset_syscon
```

**Key details:**
- `plat_dat->rx_coe = STMMAC_RX_COE_TYPE2` — بيعلم الـ stmmac إن الـ EMAC بيدعم RX checksum offload type 2
- `plat_dat->tx_fifo_size = 4096`, `rx_fifo_size = 16384` — قيم ثابتة للـ Allwinner EMAC
- `STMMAC_FLAG_HAS_SUN8I` — flag خاص بيعدل سلوك الـ stmmac core لمناسبة الـ sun8i hardware
- الـ `pm_runtime_get_sync()` ضروري لأن `stmmac_pltfr_probe()` بتعمل runtime suspend في النهاية، والـ reset بيحتاج الـ MAC يكون صاحي

---

#### `sun8i_dwmac_init`

```c
static int sun8i_dwmac_init(struct device *dev, void *priv)
```

الـ `plat_dat->init` callback — بيتكلمها الـ stmmac core في `stmmac_dvr_probe()` وعند كل resume.

**Flow:**
```
1. if gmac->regulator: regulator_enable()
2. if gmac->use_internal_phy: sun8i_dwmac_power_internal_phy()
```

**Error path:** لو الـ internal PHY فشل، `regulator_disable()` وreturn error.

---

#### `sun8i_dwmac_exit`

```c
static void sun8i_dwmac_exit(struct device *dev, void *priv)
```

عكس `sun8i_dwmac_init`. بتوقف الـ internal PHY (لو موجود) وبتعطل الـ regulator.

**Callers:** `sun8i_dwmac_shutdown()` وعند كل suspend عبر الـ stmmac platform PM ops.

---

#### `sun8i_dwmac_remove`

```c
static void sun8i_dwmac_remove(struct platform_device *pdev)
```

**Flow:**
```
if internal PHY:
    mdio_mux_uninit(gmac->mux_handle)
    sun8i_dwmac_unpower_internal_phy()
    reset_control_put(gmac->rst_ephy)
    clk_put(gmac->ephy_clk)

stmmac_pltfr_remove(pdev)
sun8i_dwmac_unset_syscon(gmac)
```

**Key detail:** الترتيب مهم — الـ MDIO mux لازم يتدجستر قبل الـ internal PHY يتوقف، والـ stmmac يتـ remove قبل الـ SYSCON يتـ reset.

---

### Group 5: Internal PHY & MDIO Mux

الـ H3 وV3s SoCs عندهم internal EPHY (Ethernet PHY) مدمجة في نفس الـ die. الـ MDIO bus محتاج mux بين الـ internal PHY والـ external PHY. المجموعة دي بتدير هذا الـ mux والـ PHY power.

---

#### `get_ephy_nodes`

```c
static int get_ephy_nodes(struct stmmac_priv *priv)
```

بتتصفح الـ Device Tree عشان تلاقي الـ internal PHY node وتجيب منه الـ clock والـ reset control.

**DT structure المتوقعة:**
```
emac {
    mdio-mux {
        mdio@1 {  /* allwinner,sun8i-h3-mdio-internal */
            ephy: ethernet-phy@1 {
                clocks = <&ccu CLK_BUS_EPHY>;
                resets = <&ccu RST_BUS_EPHY>;
            };
        };
    };
};
```

**Flow:**
```
mdio_mux = of_get_child_by_name(of_node, "mdio-mux")
mdio_internal = of_get_compatible_child(mdio_mux, "allwinner,sun8i-h3-mdio-internal")

for_each_child_of_node_scoped(mdio_internal, iphynode):
    ephy_clk = of_clk_get(iphynode, 0)
    if failed: continue
    rst_ephy = of_reset_control_get_exclusive(iphynode, NULL)
    if -EPROBE_DEFER: return -EPROBE_DEFER  /* critical! */
    if other error: continue
    return 0  /* found! */

return -ENODEV
```

**Key detail:** الـ `-EPROBE_DEFER` handling مهم جداً — لو الـ reset controller ما اتـ probe لسه، لازم نرجع defer عشان الـ probe يتعمل retry لاحقاً.

---

#### `sun8i_dwmac_power_internal_phy`

```c
static int sun8i_dwmac_power_internal_phy(struct stmmac_priv *priv)
```

بتشغل الـ internal EPHY عن طريق تفعيل الـ clock وعمل hardware reset.

**Flow:**
```
if internal_phy_powered: warn and return 0

clk_prepare_enable(gmac->ephy_clk)
reset_control_reset(gmac->rst_ephy)  /* assert then deassert */
gmac->internal_phy_powered = true
```

**Key detail للـ reset:** الـ `reset_control_reset()` بيعمل assert ثم deassert في خطوة واحدة. ده ضروري لأن الـ U-Boot ممكن يسيب الـ EPHY في حالة deasserted (مش في reset)، وفي الحالة دي الـ EMAC hardware reset ممكن يفشل لو الـ EPHY مش في حالة reset نظيفة.

---

#### `sun8i_dwmac_unpower_internal_phy`

```c
static void sun8i_dwmac_unpower_internal_phy(struct sunxi_priv_data *gmac)
```

عكس `power_internal_phy`. بتعمل `clk_disable_unprepare()` وتعمل assert للـ reset.

**Guard:** لو `internal_phy_powered == false`، بترجع فوراً. ده بيمنع double-unpowering.

---

#### `mdio_mux_syscon_switch_fn`

```c
static int mdio_mux_syscon_switch_fn(int current_child, int desired_child,
                                     void *data)
```

الـ callback اللي بيتكلمه الـ `mdio-mux` subsystem لما يحتاج يعمل switch بين الـ internal وexternal MDIO bus.

**Parameters:**
- `current_child` — الـ ID الحالي للـ mux (`-1` في أول call)
- `desired_child` — الـ `reg` value من الـ target mdio node في الـ DT
- `data` — pointer على الـ `stmmac_priv`

**IDs المستخدمة:**
- `DWMAC_SUN8I_MDIO_MUX_INTERNAL_ID = 1` — internal PHY
- `DWMAC_SUN8I_MDIO_MUX_EXTERNAL_ID = 2` — external PHY

**Flow:**
```
if current_child == desired_child: return 0  /* no switch needed */

regmap_field_read(regmap_field, &reg)
switch desired_child:
    case INTERNAL:
        val = (reg & ~H3_EPHY_MUX_MASK) | H3_EPHY_SELECT
        use_internal_phy = true
    case EXTERNAL:
        val = (reg & ~H3_EPHY_MUX_MASK) | H3_EPHY_SHUTDOWN
        use_internal_phy = false

regmap_field_write(regmap_field, val)

if use_internal_phy:
    sun8i_dwmac_power_internal_phy()
else:
    sun8i_dwmac_unpower_internal_phy()

sun8i_dwmac_reset()  /* MAC reset mandatory after syscon change */
```

**Key detail:** الـ XOR (`current_child ^ desired_child`) في الـ if condition بيعمل optimized comparison — لو الاتنين متساويين، الـ XOR بيدي صفر وما بيعملش switch.

---

#### `sun8i_dwmac_register_mdio_mux`

```c
static int sun8i_dwmac_register_mdio_mux(struct stmmac_priv *priv)
```

بتسجل الـ MDIO mux مع الـ `mdio-mux` kernel subsystem.

**Flow:**
```
mdio_mux = of_get_child_by_name(of_node, "mdio-mux")
ret = mdio_mux_init(dev, mdio_mux, mdio_mux_syscon_switch_fn,
                    &gmac->mux_handle, priv, priv->mii)
of_node_put(mdio_mux)
return ret
```

**Key detail:** `priv->mii` هو الـ parent MDIO bus — الـ `mdio-mux` subsystem بيخلي الـ child buses تظهر كـ virtual buses فوقه. الـ `mux_handle` بيتحفظ في `gmac->mux_handle` لاستخدامه عند الـ `mdio_mux_uninit()` في `remove()`.

---

### الـ ops Tables

#### `sun8i_dwmac_dma_ops` — المربوط بـ `mac->dma`

```c
static const struct stmmac_dma_ops sun8i_dwmac_dma_ops = {
    .reset               = sun8i_dwmac_dma_reset,
    .init                = sun8i_dwmac_dma_init,
    .init_rx_chan        = sun8i_dwmac_dma_init_rx,
    .init_tx_chan        = sun8i_dwmac_dma_init_tx,
    .dump_regs           = sun8i_dwmac_dump_regs,
    .dma_rx_mode         = sun8i_dwmac_dma_operation_mode_rx,
    .dma_tx_mode         = sun8i_dwmac_dma_operation_mode_tx,
    .enable_dma_transmission = sun8i_dwmac_enable_dma_transmission,
    .enable_dma_irq      = sun8i_dwmac_enable_dma_irq,
    .disable_dma_irq     = sun8i_dwmac_disable_dma_irq,
    .start_tx            = sun8i_dwmac_dma_start_tx,
    .stop_tx             = sun8i_dwmac_dma_stop_tx,
    .start_rx            = sun8i_dwmac_dma_start_rx,
    .stop_rx             = sun8i_dwmac_dma_stop_rx,
    .dma_interrupt       = sun8i_dwmac_dma_interrupt,
};
```

#### `sun8i_dwmac_ops` — المربوط بـ `mac->mac`

```c
static const struct stmmac_ops sun8i_dwmac_ops = {
    .core_init      = sun8i_dwmac_core_init,
    .set_mac        = sun8i_dwmac_set_mac,
    .dump_regs      = sun8i_dwmac_dump_mac_regs,
    .rx_ipc         = sun8i_dwmac_rx_ipc_enable,
    .set_filter     = sun8i_dwmac_set_filter,
    .flow_ctrl      = sun8i_dwmac_flow_ctrl,
    .set_umac_addr  = sun8i_dwmac_set_umac_addr,
    .get_umac_addr  = sun8i_dwmac_get_umac_addr,
    .set_mac_loopback = sun8i_dwmac_set_mac_loopback,
};
```

---

### ملاحظات عامة على الـ Locking

كما هو موضح في الـ comment في أول الـ file:

> `Locking: no locking is necessary in this file because all necessary locking is done in the "stmmac files"`

كل الـ functions دي بتتكلم من contexts محددة بيتضمنها الـ stmmac core:
- الـ DMA ops: بتتكلم من interrupt context أو من process context مع الـ spinlock اللي بيمسكه الـ stmmac
- الـ MAC ops: process context دايماً
- الـ platform/probe functions: process context بس

ما فيش لازمة لـ additional locking في الـ glue layer نفسها.
## Phase 5: دليل الـ Debugging الشامل

الـ driver المقصود هو `dwmac-sun8i` — الـ glue layer الخاص بـ Allwinner EMAC (sun8i/sun50i SoCs زي H3, A64, H6, R40, V3s, A83T). بيشتغل فوق الـ `stmmac` subsystem وبيتحكم في الـ PHY (internal EPHY أو external) عبر `mdio-mux` + `syscon` regmap.

---

### Software Level

#### 1. Debugfs Entries

الـ `stmmac` subsystem بيعمل entries في `/sys/kernel/debug/` لما بيكون `CONFIG_DEBUG_FS=y`:

```bash
# اعرف اسم الـ interface الأول
IFACE=$(ip link | grep -oP 'eth\d+' | head -1)

# الـ stmmac بيعمل directory باسم الـ device
ls /sys/kernel/debug/stmmac/
# مثال: /sys/kernel/debug/stmmac/5020000.ethernet/
```

| Entry | المحتوى | كيف تقرأها |
|-------|---------|------------|
| `descriptors` | محتوى الـ TX/RX descriptor rings | `cat /sys/kernel/debug/stmmac/5020000.ethernet/descriptors` |
| `dma_cap` | الـ capabilities اللي اكتشفها الـ driver | `cat /sys/kernel/debug/stmmac/5020000.ethernet/dma_cap` |
| `rx_ring` | حالة الـ RX ring كل descriptor | `cat /sys/kernel/debug/stmmac/5020000.ethernet/rx_ring` |
| `tx_ring` | حالة الـ TX ring | `cat /sys/kernel/debug/stmmac/5020000.ethernet/tx_ring` |

```bash
# قراءة شاملة لكل حاجة في debugfs الخاصة بالـ device
find /sys/kernel/debug/stmmac/ -type f | xargs -I{} sh -c 'echo "=== {} ==="; cat {}'
```

#### 2. Sysfs Entries

```bash
# حالة الـ net device
cat /sys/class/net/eth0/operstate        # up / down / unknown
cat /sys/class/net/eth0/carrier          # 1 = link up
cat /sys/class/net/eth0/carrier_changes  # عدد مرات تغيير الـ link
cat /sys/class/net/eth0/speed            # السرعة بالـ Mbps
cat /sys/class/net/eth0/duplex           # full / half

# الـ statistics الأساسية
cat /sys/class/net/eth0/statistics/rx_errors
cat /sys/class/net/eth0/statistics/tx_errors
cat /sys/class/net/eth0/statistics/rx_dropped
cat /sys/class/net/eth0/statistics/tx_dropped

# الـ regulator الخاص بالـ PHY (لو موجود)
ls /sys/class/regulator/
cat /sys/class/regulator/regulator.X/name   # دور على "phy"
cat /sys/class/regulator/regulator.X/state  # enabled / disabled

# الـ clock الخاص بالـ EPHY (H3/V3s)
ls /sys/kernel/debug/clk/
cat /sys/kernel/debug/clk/ephy/clk_enable_count
cat /sys/kernel/debug/clk/emac/clk_rate

# reset control (عبر reset framework)
ls /sys/kernel/debug/reset/
```

#### 3. Ftrace — Tracepoints والـ Events

```bash
# فعّل الـ tracing
mount -t tracefs nodev /sys/kernel/tracing   # لو مش متمونت

cd /sys/kernel/tracing

# --- network layer events ---
echo 1 > events/net/net_dev_xmit/enable
echo 1 > events/net/netif_receive_skb/enable
echo 1 > events/net/napi_poll/enable

# --- stmmac-specific tracepoints (لو موجودة في kernel version) ---
echo 1 > events/stmmac/enable   # كل events الـ subsystem

# --- تتبع الـ mdio transactions ---
echo 1 > events/mdio/mdio_access/enable

# --- تتبع الـ regmap (للـ syscon) ---
echo 1 > events/regmap/regmap_reg_read/enable
echo 1 > events/regmap/regmap_reg_write/enable

# شغّل الـ tracing
echo 1 > tracing_on
echo 0 > tracing_on   # وقّف بعد الحدث

# اقرأ النتيجة
cat trace | grep -E "(emac|stmmac|eth0|mdio)"
```

**تتبع دالة بعينها بـ function tracer:**

```bash
cd /sys/kernel/tracing
echo function > current_tracer
echo 'sun8i_dwmac_*' > set_ftrace_filter
echo 1 > tracing_on
# ... استنى الحدث ...
cat trace
echo nop > current_tracer   # وقّف
```

#### 4. Printk و Dynamic Debug

الـ driver بيستخدم `dev_err`, `dev_warn`, `dev_info`, `dev_dbg` — الـ `dev_dbg` بيكون صامت بدون تفعيل dynamic debug.

```bash
# فعّل الـ debug messages للـ driver ده بالاسم
echo 'module dwmac_sun8i +p' > /sys/kernel/debug/dynamic_debug/control

# أو بالـ file name
echo 'file dwmac-sun8i.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل stmmac debug messages
echo 'module stmmac +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep dwmac

# لتفعيل وقت الـ boot (في kernel cmdline)
# dyndbg="file dwmac-sun8i.c +p; module stmmac +p"
```

**مستويات الـ printk الحالية:**

```bash
cat /proc/sys/kernel/printk   # current default minimum boot-time
# مثال: 4  4  1  7
# الأول = console loglevel

# عشان تشوف كل messages
dmesg -w --level=debug,info,notice,warn,err | grep -iE "(emac|stmmac|dwmac|ephy|eth)"
```

#### 5. Kernel Config Options للـ Debugging

| Option | الوظيفة | ملاحظة |
|--------|---------|--------|
| `CONFIG_STMMAC_ETH=m/y` | الـ driver الأساسي | لازم |
| `CONFIG_DWMAC_SUN8I=m/y` | الـ glue layer نفسه | لازم |
| `CONFIG_DEBUG_FS` | يفعّل debugfs (descriptors, dma_cap) | مهم جداً |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg` dynamically | مهم جداً |
| `CONFIG_NET_DEV_REFCNT_TRACKER` | يتبع الـ device references | للتحقيق في leaks |
| `CONFIG_NETDEV_NOTIFIER_ERROR_INJECT` | inject errors في الـ notifier | اختبار |
| `CONFIG_PHYLIB` | الـ PHY library | لازم |
| `CONFIG_DEBUG_REGMAP` | debug على كل regmap access | مفيد للـ syscon |
| `CONFIG_REGMAP_DEBUGFS` | regmap في debugfs | بيظهر قيم الـ registers |
| `CONFIG_RESET_CONTROLLER` | الـ reset framework للـ EPHY | لازم للـ H3/V3s |
| `CONFIG_CLKDEV_LOOKUP` | يساعد في debug الـ clocks | |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف spinlock misuse | عام |
| `CONFIG_KASAN` | اكتشاف memory corruption | للتحقيق العميق |
| `CONFIG_NET_RX_BUSY_POLL` | تحسين الـ RX latency + debug | اختياري |
| `CONFIG_MDIO_BUS` | الـ MDIO bus framework | لازم |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "(STMMAC|DWMAC|DEBUG_FS|DYNAMIC_DEBUG|REGMAP)"
```

#### 6. Devlink وأدوات الـ Subsystem

الـ `dwmac-sun8i` مش بيدعم الـ `devlink` API مباشرة. الأدوات المخصصة:

```bash
# --- ethtool: الأهم ---
ethtool eth0                        # link speed, duplex, autoneg
ethtool -S eth0                     # statistics كاملة (من stmmac_extra_stats)
ethtool -d eth0                     # register dump (بيستدعي sun8i_dwmac_dump_regs)
ethtool -k eth0                     # offload features
ethtool -i eth0                     # driver info (version, bus-info)
ethtool -a eth0                     # pause parameters
ethtool -c eth0                     # coalescing settings
ethtool --set-priv-flags eth0 ...   # private flags

# --- ip/iproute2 ---
ip -s link show eth0                # TX/RX stats مع errors
ip link set eth0 down && ip link set eth0 up   # إعادة تشغيل الـ interface

# --- mii-tool (قديم لكن مفيد) ---
mii-tool -v eth0                    # حالة الـ PHY عبر MDIO

# --- phytool ---
phytool read eth0/1/0               # اقرأ register 0 من PHY address 1
phytool write eth0/1/0 0x1200       # اكتب قيمة للـ PHY register

# --- mdio-tool (للـ MDIO bus) ---
# تحقق من الـ PHY address المستخدمة في DT
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|------------|--------|------|
| `EMAC reset timeout` | الـ EMAC ما رجعش لـ reset بعد 100ms (bit 0 في BASIC_CTL1) | تحقق من الـ clock (stmmac هل شغّال؟)، قد يكون الـ cable مش موصول |
| `Cannot get mdio-mux node` | مفيش `mdio-mux` في الـ DT تحت الـ EMAC node | راجع الـ DT: لازم يكون في `mdio-mux` child |
| `Cannot get internal_mdio node` | مفيش child بـ compatible `allwinner,sun8i-h3-mdio-internal` | راجع الـ DT structure |
| `Cannot enable internal PHY` | `clk_prepare_enable(ephy_clk)` فشل | تحقق من الـ clock أو `ephy_clk` في DT |
| `Cannot reset internal PHY` | `reset_control_reset(rst_ephy)` فشل | الـ EPHY reset غير متاح أو محتاج regulator |
| `Could not parse MDIO addr` | `of_mdio_parse_addr` فشل | الـ PHY node ما عندوش `reg` property |
| `Invalid TX clock delay` | `allwinner,tx-delay-ps` أكبر من `tx_delay_max` | H3: max 700ps، R40: لا يدعم tx delay |
| `Invalid RX clock delay` | `allwinner,rx-delay-ps` أكبر من `rx_delay_max` | H3/A64: max 3100ps، R40: max 700ps |
| `Unable to map syscon` | الـ `syscon` phandle في DT مش موجود أو الـ device ما probe هوش | لازم الـ syscon driver يكون probe قبل الـ EMAC |
| `Missing dwmac-sun8i variant` | الـ `compatible` في DT مش في `sun8i_dwmac_match` | تحقق من الـ compatible string |
| `Fail to enable regulator` | الـ PHY regulator ما اتفعّلش | تحقق من الـ power supply / PMIC |
| `Too many address, switching to promiscuous` | عدد الـ MAC addresses أكتر من 8 (unicast_filter_entries) | طبيعي في بعض الحالات — مش error حقيقي |
| `Failed to register mux` | `mdio_mux_init()` فشل | تحقق من الـ mdio-mux DT nodes كاملة |
| `Invalid child ID` في mdio mux | قيمة `reg` في الـ mdio child مش 1 أو 2 | الـ internal MDIO لازم `reg = <1>` والـ external `reg = <2>` |
| `tx-delay must be a multiple of 100` | القيمة مش مضروبة في 100 | الوحدة picoseconds، استخدم 100, 200, 300... |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في sun8i_dwmac_power_internal_phy() — بعد clk_prepare_enable فشل */
ret = clk_prepare_enable(gmac->ephy_clk);
if (ret) {
    WARN_ON(1); /* أضف هنا عشان تعرف الـ call stack */
    dev_err(priv->device, "Cannot enable internal PHY\n");
    return ret;
}

/* في mdio_mux_syscon_switch_fn() — للتحقق من state صح */
WARN_ON(current_child == desired_child); /* المفروض ما يتكلمش لو نفسهم */

/* في sun8i_dwmac_reset() — timeout */
if (err) {
    dump_stack(); /* مين استدعاها ووقت الـ timeout؟ */
    dev_err(priv->device, "EMAC reset timeout\n");
}

/* في sun8i_dwmac_dma_interrupt() — overflow */
if (v & EMAC_RX_OVERFLOW_INT) {
    WARN_ONCE(1, "RX overflow detected on %s\n", priv->dev->name);
    ret |= tx_hard_error;
}

/* في sun8i_dwmac_set_syscon() — للـ phy_interface غير المدعوم */
default:
    WARN_ON_ONCE(1); /* إضافة قبل dev_err */
    dev_err(dev, "Unsupported interface mode: %s", ...);
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# --- تحقق من حالة الـ link في الـ kernel vs الـ PHY ---
ethtool eth0
# بيرجع: Speed: 100Mb/s  Duplex: Full  Link detected: yes

# قارن مع الـ PHY register مباشرة (PHY basic status reg = 0x01)
phytool read eth0/1/1   # الـ 1 ده PHY address من DT، والأخير register index
# bit 2 = link status

# --- تحقق من الـ syscon register الخاص بالـ EMAC ---
# H3: syscon @ base + 0x30  (reg_field sun8i_syscon_reg_field)
# R40: CCU @ base + 0x164   (reg_field sun8i_ccu_reg_field)
# الـ base address من DT property "syscon" phandle

# مثال H3 (system control base = 0x01C00000):
devmem2 0x01C00030 w   # اقرأ EMAC_CLK register
# bit 15: H3_EPHY_SELECT (1=internal, 0=external)
# bit 16: H3_EPHY_SHUTDOWN (1=off, 0=on)
# bits [14:10]: TX delay
# bits [9:5]: RX delay
# bit 2: SYSCON_EPIT (1=RGMII, 0=MII)
# bits [1:0]: SYSCON_ETCS

# تحقق من الـ EMAC registers مباشرة
# الـ base address من DT property "reg" للـ EMAC node (مثلاً 0x01C30000 على H3)
devmem2 0x01C30000 w   # EMAC_BASIC_CTL0
devmem2 0x01C30004 w   # EMAC_BASIC_CTL1
devmem2 0x01C30008 w   # EMAC_INT_STA
devmem2 0x01C3000C w   # EMAC_INT_EN
devmem2 0x01C300B0 w   # EMAC_TX_DMA_STA
devmem2 0x01C300C0 w   # EMAC_RX_DMA_STA
```

#### 2. Register Dump Techniques

```bash
# --- devmem2: اقرأ كل الـ EMAC register space (0x00 → 0xC8) ---
BASE=0x01C30000   # H3 EMAC base — تأكد من DT
for offset in $(seq 0 4 200); do
    addr=$((BASE + offset))
    printf "0x%03X: " $offset
    devmem2 $(printf "0x%08X" $addr) w 2>/dev/null | grep -oP '0x[0-9A-Fa-f]+'
done

# --- ethtool register dump (الأسهل) ---
ethtool -d eth0 | head -60
# بيستدعي sun8i_dwmac_dump_regs() التي تقرأ 0x00→0xC8

# --- io utility (من package iotools) ---
io -4 -r 0x01C30000   # قراءة 32-bit
io -4 -r 0x01C30008   # INT_STA

# --- /dev/mem مع python ---
python3 - <<'EOF'
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x100, offset=0x01C30000)
    for i in range(0, 0xCC, 4):
        if i in (0x32, 0x3C): continue
        val = struct.unpack('<I', m[i:i+4])[0]
        print(f"0x{i:03X}: 0x{val:08X}")
    m.close()
EOF
```

**تفسير أهم الـ registers:**

```
EMAC_BASIC_CTL0 (0x00):
  bit[0]  = DUPLEX_FULL    → 1 يعني full duplex
  bit[1]  = LOOPBACK       → 1 يعني loopback mode فعّال (مش طبيعي)
  bits[3:2] = SPEED        → 00=1G, 10=10M, 11=100M

EMAC_INT_STA (0x08):
  bit[0]  = TX_INT         → TX frame transmitted
  bit[1]  = TX_DMA_STOP    → TX DMA وقف (مشكلة!)
  bit[2]  = TX_BUF_UA      → TX buffer unavailable
  bit[3]  = TX_TIMEOUT     → TX timeout (مشكلة!)
  bit[4]  = TX_UNDERFLOW   → TX underflow (مشكلة!)
  bit[8]  = RX_INT         → RX frame received
  bit[9]  = RX_BUF_UA      → RX buffer unavailable
  bit[10] = RX_DMA_STOP    → RX DMA وقف (مشكلة!)
  bit[12] = RX_OVERFLOW    → RX overflow (مشكلة!)
  bit[16] = RGMII_STA_INT  → RGMII status change

EMAC_TX_DMA_STA (0xB0):
  بيديك عنوان الـ descriptor اللي الـ DMA بيشتغل عليه حالياً

EMAC_RX_DMA_STA (0xC0):
  نفس فكرة TX لكن للـ RX
```

#### 3. Logic Analyzer وـ Oscilloscope

**للـ RGMII debugging:**

```
نقاط القياس:
- TX_CLK (125MHz لـ 1Gbps، 25MHz لـ 100Mbps، 2.5MHz لـ 10Mbps)
- RX_CLK (من الـ PHY للـ MAC)
- TX_DATA[3:0] + TX_EN + TX_CTL
- RX_DATA[3:0] + RX_DV + RX_CTL
- MDC (max 2.5MHz) + MDIO

أهم ما تشوفه:
1. الـ TX/RX delays صح → بيعوضوا الـ setup/hold time
   H3 مثلاً: allwinner,tx-delay-ps = <700> يعني delay = 700ps
2. الـ RGMII_STA_INT بيظهر في الـ kernel لما الـ link يتغير
3. لو الـ link instability: شوف TX_CLK jitter

للـ RMII:
- REF_CLK (50MHz لازم من external oscillator أو PHY)
- RXD[1:0] + CRS_DV
- TXD[1:0] + TX_EN
- SYSCON_RMII_EN=1 و SYSCON_ETCS_EXT_GMII لازم يكونوا set

للـ MII:
- TX_CLK من الـ PHY + RX_CLK من الـ PHY
- 4-bit data bus للجهتين
```

**إعداد Logic Analyzer:**

```
Sample rate: >= 10x الـ clock frequency
  RGMII: >= 1.25 Gsps للـ 1G
  RMII:  >= 500 Msps
  MDIO:  >= 25 Msps

Triggers:
- على TX_ERR أو RX_ERR
- على MDC falling edge لـ MDIO transactions
- على CRS_DV toggle لـ RMII
```

#### 4. مشاكل Hardware شائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في الـ dmesg | السبب |
|---------|---------------------|-------|
| RGMII TX/RX delay غلط | كثرة `ethtool -S eth0` tx/rx errors، أو link يتكسر عند load | `allwinner,tx-delay-ps` أو `rx-delay-ps` غلط — جرّب قيم مختلفة بخطوة 100ps |
| EPHY reset ما اشتغلش | `Cannot reset internal PHY`، الـ interface بييجي لكن ما بيشتغلش | U-Boot خلّى الـ EPHY بدون reset — الـ driver بيعمل `reset_control_reset()` إجبارياً |
| Regulator مش شغّال | `Fail to enable regulator`، ثم reboot | الـ PHY power supply مش موصول أو PMIC driver مش loaded |
| syscon ما اتـprobe هوش | `Unable to map syscon: -517 (EPROBE_DEFER)` | الـ syscon driver بييجي بعد الـ EMAC — طبيعي، بيحل نفسه بالـ deferred probe |
| RMII بدون 50MHz clock | الـ interface بييجي لكن link ما بييجيش | الـ REF_CLK مش موجود أو غلط — `SYSCON_ETCS_EXT_GMII` لازم مع external clock |
| Internal PHY address غلط | PHY لا يُكتشف | `reg` property في PHY node في DT لازم يطابق الـ EPHY address المبرمجة في `H3_EPHY_ADDR_SHIFT` |

#### 5. Device Tree Debugging

```bash
# --- تحقق من الـ DT المحمّل ---
# الـ DT الفعلي اللي الـ kernel شايله
ls /proc/device-tree/soc/ethernet@*/
cat /proc/device-tree/soc/ethernet@*/compatible     # لازم allwinner,sun8i-h3-emac مثلاً
cat /proc/device-tree/soc/ethernet@*/status         # لازم "okay"

# --- الـ syscon phandle ---
# لازم يشاور على system-control node صح
cat /proc/device-tree/soc/ethernet@*/syscon         # binary phandle

# --- الـ PHY node ---
ls /proc/device-tree/soc/ethernet@*/mdio-mux/
ls /proc/device-tree/soc/ethernet@*/mdio-mux/mdio@1/  # internal mdio
cat /proc/device-tree/soc/ethernet@*/mdio-mux/mdio@1/ethernet-phy@*/reg  # PHY addr

# --- TX/RX delays ---
# أقرأ القيمة المضبوطة
xxd /proc/device-tree/soc/ethernet@*/allwinner,tx-delay-ps
xxd /proc/device-tree/soc/ethernet@*/allwinner,rx-delay-ps

# --- تحقق من الـ EPHY clock ---
cat /proc/device-tree/soc/ethernet@*/mdio-mux/mdio@1/ethernet-phy@*/clocks

# --- compare مع الـ DTS file الأصلي ---
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 30 "ethernet@"
```

**DT صح للـ H3 (مقتطف نموذجي):**

```dts
emac: ethernet@1c30000 {
    compatible = "allwinner,sun8i-h3-emac";
    syscon = <&syscon>;
    reg = <0x01c30000 0x10000>;
    phy-handle = <&int_mii_phy>;
    phy-mode = "mii";
    allwinner,leds-active-low;   /* لو LEDs active low */
    /* لـ RGMII: */
    /* phy-mode = "rgmii-id"; */
    /* allwinner,tx-delay-ps = <700>; */
    /* allwinner,rx-delay-ps = <100>; */

    mdio-mux {
        compatible = "allwinner,sun8i-h3-mdio-mux";
        mdio@1 {
            compatible = "allwinner,sun8i-h3-mdio-internal";
            reg = <1>;
            int_mii_phy: ethernet-phy@1 {
                compatible = "ethernet-phy-ieee802.3-c22";
                reg = <1>;
                clocks = <&ccu CLK_BUS_EPHY>;
                resets = <&ccu RST_BUS_EPHY>;
            };
        };
        mdio@2 {
            reg = <2>;
            #address-cells = <1>;
            #size-cells = <0>;
        };
    };
};
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
#!/bin/bash
# === dwmac-sun8i Complete Debug Script ===

IFACE=${1:-eth0}
EMAC_BASE=${2:-0x01C30000}   # H3 default — غيّر حسب الـ SoC
SYSCON_BASE=${3:-0x01C00000}  # H3 syscon

echo "=== 1. Driver & Link Info ==="
ethtool $IFACE
ethtool -i $IFACE

echo -e "\n=== 2. Extended Statistics ==="
ethtool -S $IFACE

echo -e "\n=== 3. Register Dump (via ethtool) ==="
ethtool -d $IFACE

echo -e "\n=== 4. Kernel Messages (last 50) ==="
dmesg | grep -iE "(emac|stmmac|dwmac|ephy|$IFACE)" | tail -50

echo -e "\n=== 5. EMAC Core Registers ==="
for reg_name_offset in \
    "BASIC_CTL0:0x00" "BASIC_CTL1:0x04" \
    "INT_STA:0x08"   "INT_EN:0x0C" \
    "TX_CTL0:0x10"   "TX_CTL1:0x14" \
    "RX_CTL0:0x24"   "RX_CTL1:0x28" \
    "TX_DMA_STA:0xB0" "TX_CUR_DESC:0xB4" \
    "RX_DMA_STA:0xC0" "RX_CUR_DESC:0xC4"; do
    name=$(echo $reg_name_offset | cut -d: -f1)
    offset=$(echo $reg_name_offset | cut -d: -f2)
    addr=$(printf "0x%08X" $((EMAC_BASE + offset)))
    val=$(devmem2 $addr w 2>/dev/null | grep -oP '0x[0-9A-Fa-f]+' | tail -1)
    printf "  %-15s @ %s = %s\n" $name $addr "$val"
done

echo -e "\n=== 6. SYSCON EMAC_CLK Register (@ syscon+0x30) ==="
EMAC_CLK_ADDR=$(printf "0x%08X" $((SYSCON_BASE + 0x30)))
val=$(devmem2 $EMAC_CLK_ADDR w 2>/dev/null | grep -oP '0x[0-9A-Fa-f]+' | tail -1)
echo "  EMAC_CLK @ $EMAC_CLK_ADDR = $val"
val_dec=$((val))
echo "  EPHY_SELECT (bit15) = $(( (val_dec >> 15) & 1 ))  [1=internal]"
echo "  EPHY_SHUTDOWN (bit16) = $(( (val_dec >> 16) & 1 )) [1=shutdown]"
echo "  EPIT (bit2) = $(( (val_dec >> 2) & 1 ))           [1=RGMII]"
echo "  ETCS (bits1:0) = $(( val_dec & 3 ))               [0=MII,1=ext_GMII,2=int_GMII]"
echo "  TX_delay (bits14:10) = $(( (val_dec >> 10) & 0x1F )) raw → $(( ((val_dec >> 10) & 0x1F) * 100 ))ps"
echo "  RX_delay (bits9:5) = $(( (val_dec >> 5) & 0x1F )) raw → $(( ((val_dec >> 5) & 0x1F) * 100 ))ps"

echo -e "\n=== 7. IP Statistics ==="
ip -s link show $IFACE

echo -e "\n=== 8. MDIO/PHY Check ==="
mii-tool -v $IFACE 2>/dev/null || echo "mii-tool not available"

echo -e "\n=== 9. Carrier & Speed ==="
echo "carrier: $(cat /sys/class/net/$IFACE/carrier 2>/dev/null)"
echo "speed:   $(cat /sys/class/net/$IFACE/speed 2>/dev/null)"
echo "duplex:  $(cat /sys/class/net/$IFACE/duplex 2>/dev/null)"
echo "operstate: $(cat /sys/class/net/$IFACE/operstate 2>/dev/null)"

echo -e "\n=== 10. Debugfs ==="
find /sys/kernel/debug/stmmac/ -maxdepth 2 -type f 2>/dev/null | \
    while read f; do echo "--- $f ---"; cat "$f" 2>/dev/null; done
```

#### تفسير مثال لـ `ethtool -S eth0`

```
# مثال output مع تفسير:
NIC statistics:
     tx_normal_irq_n: 15234       # عدد TX interrupts طبيعية ← كلما أكبر = traffic أكتر
     rx_normal_irq_n: 87451       # عدد RX interrupts طبيعية
     tx_process_stopped_irq: 0    # لو > 0 ← TX DMA وقف غير طبيعي (TX_DMA_STOP أو TX_BUF_UA)
     tx_undeflow_irq: 0           # لو > 0 ← TX underflow، FIFO فرغ قبل ما الـ frame يخلص
     tx_early_irq: 0              # TX early interrupt (معلوماتي)
     rx_buf_unav_irq: 3           # ← مشكلة! الـ RX buffers نفدت، الـ driver بطيء
     rx_process_stopped_irq: 0    # لو > 0 ← RX DMA وقف
     rx_overflow_irq: 0           # لو > 0 ← الـ RX FIFO overflow ← traffic كتير جداً
     irq_rgmii_n: 12              # RGMII status change interrupts (link events)
```

```bash
# --- تفعيل dynamic debug وقت run ---
echo 'file dwmac-sun8i.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module, t = thread id
dmesg -w | grep dwmac_sun8i   # شوف الـ debug output

# --- اختبار الـ loopback داخل الـ MAC ---
# sun8i_dwmac_set_mac_loopback() بتـset bit[1] في BASIC_CTL0
ethtool -K eth0 loopback on   # لو الـ driver بيدعمه
# أو مباشرة:
# devmem2 0x01C30000 w   # اقرأ BASIC_CTL0
# devmem2 0x01C30000 w 0x00000002  # set EMAC_LOOPBACK bit

# --- مراقبة الـ interrupts ---
watch -n1 'cat /proc/interrupts | grep eth'
# لو الـ interrupt counter ما بيتحركش = مشكلة في الـ IRQ routing

# --- تتبع الـ MDIO transactions بـ ftrace ---
echo 1 > /sys/kernel/tracing/events/mdio/mdio_access/enable
echo 1 > /sys/kernel/tracing/tracing_on
phytool read eth0/1/0
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
# output مثال:
# eth0-7342 [001] .... mdio_access: bus=eth0, addr=1, regnum=0, read=1, val=0x1200

# --- تحقق من الـ clock tree ---
cat /sys/kernel/debug/clk/clk_summary | grep -E "(emac|ephy|gmac)"
# لازم تشوف الـ emac و ephy clocks enabled وبالـ rate الصح
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: OrangePi One (Allwinner H3) — الـ EMAC reset بيـ timeout

#### العنوان
**EMAC reset timeout على بورد OrangePi One بدون كابل Ethernet**

#### السياق
بتشتغل على bring-up لـ IoT gateway مبني على OrangePi One (H3 SoC). البورد بتبوت تمام لحد ما بتشغّل الـ network interface — فجأة الكرنل بيـ panic أو بيـ hang عند تحميل الـ driver.

#### المشكلة
الـ kernel log بيظهر:

```
sun8i-emac 1c30000.ethernet: EMAC reset timeout
```

الـ driver بيتعلق في `sun8i_dwmac_reset()` ومش بيكمل الـ probe.

#### التحليل
في `sun8i_dwmac_reset()` (السطر 741–760):

```c
static int sun8i_dwmac_reset(struct stmmac_priv *priv)
{
    u32 v;
    int err;

    v = readl(priv->ioaddr + EMAC_BASIC_CTL1);
    writel(v | 0x01, priv->ioaddr + EMAC_BASIC_CTL1);

    /* The timeout was previously set to 10ms, but some board (OrangePI0)
     * need more if no cable plugged. 100ms seems OK
     */
    err = readl_poll_timeout(priv->ioaddr + EMAC_BASIC_CTL1, v,
                             !(v & 0x01), 100, 100000);
```

الـ hardware بيحتاج وقت أطول للـ reset لما مفيش كابل مربوط — الـ timeout اتبقى 100ms بعد ما كان 10ms بالظبط لأجل OrangePi. لو كانت نسخة kernel قديمة أو patch قديم الـ timeout هيكون أقل من اللازم.

ثانيًا: لو الـ internal EPHY مش اتـ power-up صح قبل الـ reset، الـ MAC بيـ stall. في `sun8i_dwmac_probe()` (السطر 1214–1227):

```c
if (gmac->variant->soc_has_internal_phy) {
    ret = get_ephy_nodes(priv);       // يجيب clk + reset للـ EPHY
    if (ret)
        goto dwmac_remove;
    ret = sun8i_dwmac_register_mdio_mux(priv);
    ...
} else {
    ret = sun8i_dwmac_reset(priv);    // reset بس لو external PHY
```

الـ H3 بيمشي مسار الـ internal PHY — لو `get_ephy_nodes()` فشل (مثلاً DT مش صح)، الـ reset مش بيحصل خالص.

#### الحل

**1. تحقق من الـ DT:**

```bash
# شوف mdio-mux موجود صح
dtc -I fs /proc/device-tree/soc/ethernet@1c30000/ | grep -A 20 mdio-mux
```

**2. تأكد من وجود `allwinner,sun8i-h3-mdio-internal` node في DTS:**

```dts
mdio-mux {
    compatible = "mdio-mux-mmioreg";
    ...
    mdio_int: mdio@1 {
        compatible = "allwinner,sun8i-h3-mdio-internal";
        reg = <1>;
        #address-cells = <1>;
        #size-cells = <0>;
        int_mii_phy: ethernet-phy@1 {
            reg = <1>;
            clocks = <&ccu CLK_BUS_EPHY>;
            resets = <&ccu RST_BUS_EPHY>;
        };
    };
};
```

**3. لو المشكلة في الـ timeout فقط، تأكد إن الـ kernel نسخته >= 4.19 اللي فيها الـ fix:**

```bash
grep -n "100000" drivers/net/ethernet/stmicro/stmmac/dwmac-sun8i.c
# يظهر: 100000 = 100ms وده الصح
```

#### الدرس المستفاد
الـ reset timeout على sun8i EMAC بيتأثر بوجود الكابل وبحالة الـ internal EPHY. الـ DT لازم يحتوي على الـ mdio-mux node كامل مع الـ clocks والـ resets — لو واحدة ناقصة الـ probe بيفشل صامت وبييجي reset timeout مش EPHY error.

---

### السيناريو الثاني: Allwinner H616 (Android TV Box) — الـ Ethernet مش بيـ link-up على RGMII

#### العنوان
**Allwinner H616 TV Box: الـ Ethernet موجود في الكرنل بس link دايمًا down**

#### السياق
بتعمل Android TV box مبني على Allwinner H616. الـ kernel بيـ load الـ driver بدون errors، الـ interface `eth0` موجود، بس `ip link show eth0` دايمًا بيظهر `NO-CARRIER`. الـ PHY خارجي RTL8211F مربوط بـ RGMII.

#### المشكلة
الـ link مش بيجي رغم إن الكابل مربوط وصح. الـ PHY من ناحيته بيـ advertise الـ link بس الـ MAC مش بيقراه صح.

#### التحليل
الـ H616 بيستخدم `emac_variant_h6` (السطر 133–144):

```c
static const struct emac_variant emac_variant_h6 = {
    .syscon_field = &sun8i_syscon_reg_field,
    .soc_has_internal_phy = false,   // مفيش internal PHY
    .support_mii = true,
    .support_rmii = true,
    .support_rgmii = true,
    .rx_delay_max = 31,
    .tx_delay_max = 7,
};
```

في `sun8i_dwmac_set_syscon()` (السطر 977–994) الـ RGMII mode:

```c
case PHY_INTERFACE_MODE_RGMII:
case PHY_INTERFACE_MODE_RGMII_ID:
case PHY_INTERFACE_MODE_RGMII_RXID:
case PHY_INTERFACE_MODE_RGMII_TXID:
    reg |= SYSCON_EPIT | SYSCON_ETCS_INT_GMII;
    break;
```

الـ `SYSCON_EPIT` (BIT(2)) = select RGMII، و`SYSCON_ETCS_INT_GMII` (0x2) = use internal clock. لو الـ tx-delay أو rx-delay القيم غلط في الـ DT، الـ RGMII timing بيبوظ وبيطلع link فاشل رغم إن الـ MAC اتضبط.

الـ syscon register بيتكتب هنا:

```c
regmap_field_write(gmac->regmap_field, reg);
```

لو الـ `allwinner,tx-delay-ps` أو `allwinner,rx-delay-ps` مش موجودين في DT، القيم بتبقى صفر — وده ممكن يكون غلط لـ RTL8211F.

#### الحل

**1. اقرأ الـ syscon register الحالي:**

```bash
# syscon base للـ H616 عادةً 0x03000000
devmem2 0x03000030 w
# أو
io -4 0x03000030
```

**2. احسب القيم الصح للـ RTL8211F على RGMII:**

الـ RTL8211F عادةً بيحتاج tx-delay=700ps (= 7 × 100ps)، rx-delay=1400ps (= 14 × 100ps):

```dts
&emac {
    compatible = "allwinner,sun50i-h6-emac";
    syscon = <&syscon>;
    allwinner,tx-delay-ps = <700>;
    allwinner,rx-delay-ps = <1400>;
    phy-handle = <&ext_rgmii_phy>;
    phy-mode = "rgmii-id";
};
```

**3. تحقق من الـ register value المكتوب:**

```c
// SYSCON_EPIT = BIT(2) = 0x4
// SYSCON_ETCS_INT_GMII = 0x2
// tx_delay=7 << SYSCON_ETXDC_SHIFT(10) = 0x1C00
// rx_delay=14 << SYSCON_ERXDC_SHIFT(5) = 0x01C0
// reg = 0x4 | 0x2 | 0x1C00 | 0x01C0 = 0x1DC6
```

```bash
# تحقق
devmem2 0x03000030 w
# المتوقع: 0x....1DC6
```

#### الدرس المستفاد
الـ RGMII timing delays على sun8i/sun50i بتفرق بين PHY وتاني. الـ driver بيكتب الـ syscon register مرة واحدة وقت الـ probe — لو القيم غلط في DT مش هيكون فيه link. دايمًا قارن الـ syscon register الفعلي بالقيمة المتوقعة من الحسبة.

---

### السيناريو الثالث: Allwinner A64 (Industrial Gateway) — الـ Multicast filtering بيـ flood الـ switch

#### العنوان
**A64 gateway: كل الـ multicast traffic بيعدي رغم وجود IGMP snooping على الـ switch**

#### السياق
Industrial gateway مبني على Allwinner A64، مربوط بـ managed switch. الـ gateway مفروض تستقبل multicast groups محددة بس، بس الـ CPU load عالي بسبب معالجة كل الـ multicast frames.

#### المشكلة
الـ gateway بتستقبل كل الـ multicast حتى اللي مش مشتركة فيه. الـ switch بيقول إن الـ port بيطلب كل الـ multicast.

#### التحليل
في `sun8i_dwmac_set_filter()` (السطر 678–717):

```c
static void sun8i_dwmac_set_filter(struct mac_device_info *hw,
                                   struct net_device *dev)
{
    void __iomem *ioaddr = hw->pcsr;
    u32 v;
    int i = 1;
    struct netdev_hw_addr *ha;
    int macaddrs = netdev_uc_count(dev) + netdev_mc_count(dev) + 1;

    v = EMAC_FRM_FLT_CTL;

    if (dev->flags & IFF_PROMISC) {
        v = EMAC_FRM_FLT_RXALL;
    } else if (dev->flags & IFF_ALLMULTI) {
        v |= EMAC_FRM_FLT_MULTICAST;      // accept all multicast
    } else if (macaddrs <= hw->unicast_filter_entries) {
        // نضيف كل العناوين في الـ hardware filter
        ...
    } else {
        // عدد العناوين أكبر من 8 (unicast_filter_entries)
        // فبنعمل promiscuous
        v = EMAC_FRM_FLT_RXALL;
    }
    writel(v, ioaddr + EMAC_RX_FRM_FLT);
}
```

الـ MAC عنده 8 filter slots فقط (`mac->unicast_filter_entries = 8`). لو الـ application اشترك في أكتر من 7 multicast groups، الـ `macaddrs` بيتجاوز 8 والكود بيطلع من الـ else branch ويعمل `EMAC_FRM_FLT_RXALL` = promiscuous — وده بيخلي الـ MAC يقبل كل الـ multicast.

مش فيه hardware multicast hash filter في هذا الـ variant — الـ driver مش بيدعم `NETIF_F_HW_L2FW_DOFFLOAD` أو multicast hash.

#### الحل

**1. اعرف عدد الـ multicast groups:**

```bash
ip maddress show dev eth0
# أو
cat /proc/net/dev_mcast
```

**2. قلل عدد الـ multicast subscriptions:**

```bash
# في الـ application، استخدم IP_MULTICAST_FILTER بحذر
# وقلل عدد الـ groups لأقل من 7
```

**3. حل بديل: استخدم user-space multicast filtering:**

```bash
# شغّل الـ interface بدون IFF_ALLMULTI manually
ip link set eth0 allmulticast off
# وخلي الـ application يعمل filtering في user-space
```

**4. على مستوى الـ DT — مفيش حل في الـ hardware لأن الـ sun8i EMAC مش بيدعم multicast hash:**
المشكلة architectural في الـ hardware وأحسن solution هو تقليل عدد الـ groups.

```bash
# راقب الحالة
ethtool -S eth0 | grep -i multi
# أو
ip -s link show eth0
```

#### الدرس المستفاد
الـ sun8i EMAC عنده 8 hardware address filter slots بس، وبيفضل يعمل promiscuous لو العدد اتجاوز. الـ applications على industrial gateways لازم تكون واعية بهذا القيد وتحدد عدد الـ multicast subscriptions. الـ kernel بيطبع تحذير `"Too many address, switching to promiscuous"` — راقبه في dmesg.

---

### السيناريو الرابع: Allwinner H3 (Custom Board) — الـ Internal PHY مش بيـ power-up بعد suspend/resume

#### العنوان
**H3 custom board: الـ Ethernet بيموت بعد `suspend` وما بيرجعش بعد `resume`**

#### السياق
بتشتغل على custom embedded board مبني على H3 لتطبيق IoT sensor. البورد بتدخل في system suspend كل ساعة لتوفير الطاقة. المشكلة إن بعد الـ resume، الـ `eth0` بيكون موجود بس ما بيقدرش يعمل link.

#### المشكلة
```
sun8i-emac: Internal PHY already powered
# أو
dwmac-sun8i 1c30000.ethernet: EMAC reset timeout
```

الـ Ethernet بيفشل بعد أول suspend/resume cycle.

#### التحليل
الـ driver بيستخدم `pm_runtime` وبيـ delegate الـ suspend/resume لـ `stmmac_pltfr_pm_ops`. عند الـ exit (suspend):

```c
static void sun8i_dwmac_exit(struct device *dev, void *priv)
{
    struct sunxi_priv_data *gmac = priv;

    if (gmac->variant->soc_has_internal_phy)
        sun8i_dwmac_unpower_internal_phy(gmac);  // بتعمل assert reset + disable clk

    if (gmac->regulator)
        regulator_disable(gmac->regulator);
}
```

وفي `sun8i_dwmac_unpower_internal_phy()` (السطر 841–849):

```c
static void sun8i_dwmac_unpower_internal_phy(struct sunxi_priv_data *gmac)
{
    if (!gmac->internal_phy_powered)
        return;

    clk_disable_unprepare(gmac->ephy_clk);
    reset_control_assert(gmac->rst_ephy);
    gmac->internal_phy_powered = false;   // reset الـ flag
}
```

عند الـ resume، الـ init بيتم عبر `sun8i_dwmac_init()`:

```c
static int sun8i_dwmac_init(struct device *dev, void *priv)
{
    ...
    if (gmac->use_internal_phy) {
        ret = sun8i_dwmac_power_internal_phy(netdev_priv(ndev));
        ...
    }
```

وفي `sun8i_dwmac_power_internal_phy()` (السطر 807–839):

```c
static int sun8i_dwmac_power_internal_phy(struct stmmac_priv *priv)
{
    struct sunxi_priv_data *gmac = priv->plat->bsp_priv;
    int ret;

    if (gmac->internal_phy_powered) {
        dev_warn(priv->device, "Internal PHY already powered\n");
        return 0;
    }
    ...
    ret = reset_control_reset(gmac->rst_ephy);  // pulse reset
```

لكن `reset_control_reset()` بتعمل assert ثم deassert. لو الـ U-Boot أو bootloader بيـ leave الـ EPHY في حالة غريبة بعد الـ resume، الـ reset_control_reset قد ما يكفيش.

**السبب الأعمق:** لو `gmac->use_internal_phy` بقى `false` بسبب mdio-mux state corruption، الـ power-internal-phy مش بيتستدعى خالص.

#### الحل

**1. راقب الـ flag state:**

```bash
# أضف dynamic debug
echo "module dwmac_sun8i +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep -i "internal\|mux\|ephy"
```

**2. تحقق من الـ syscon register بعد الـ resume:**

```bash
# قبل suspend
devmem2 0x01c00030 w > /tmp/before_suspend.txt
# بعد resume
devmem2 0x01c00030 w > /tmp/after_resume.txt
diff /tmp/before_suspend.txt /tmp/after_resume.txt
```

**3. الحل الحقيقي:** في الـ DT، تأكد إن الـ EPHY reset مـ control صح:

```dts
int_mii_phy: ethernet-phy@1 {
    reg = <1>;
    clocks = <&ccu CLK_BUS_EPHY>;
    resets = <&ccu RST_BUS_EPHY>;
};
```

**4. لو المشكلة في الـ U-Boot state:**

```bash
# في U-Boot قبل الـ boot، reset الـ EPHY يدويًا
# أو في الـ kernel، أضف quirk في sun8i_dwmac_power_internal_phy
# لعمل explicit assert قبل reset_control_reset
```

#### الدرس المستفاد
الـ `internal_phy_powered` flag هو الـ gatekeeper لعمليات الـ EPHY. لو الـ suspend/resume sequence اتعطل في المنتصف (مثلاً power failure أثناء suspend)، الـ flag ممكن يكون inconsistent مع الـ hardware state. دايمًا اختبر suspend/resume دورة كاملة على الـ hardware الفعلي — مش بس في QEMU.

---

### السيناريو الخامس: Allwinner R40 (Custom NAS Board) — الـ RGMII TX corruption بسبب غلط في الـ clock delay

#### العنوان
**R40 NAS board: الـ Ethernet بيـ link بس الـ TCP throughput منخفض جداً وفيه packet corruption**

#### السياق
بتشتغل على custom NAS مبني على Allwinner R40 (sun8i-r40). الـ Ethernet بيـ link على Gigabit بس الـ iperf3 بيدي 50–100 Mbps بس مع retransmissions كتير. الـ ping بيمشي بس بيحصل flakey.

#### المشكلة
الـ packets بتوصل corrupt — الـ RX checksum errors وال retransmissions بيدلوا على timing problem في الـ RGMII interface.

#### التحليل
الـ R40 بيستخدم `emac_variant_r40`:

```c
static const struct emac_variant emac_variant_r40 = {
    .syscon_field = &sun8i_ccu_reg_field,   // CCU مش syscon عادي
    .support_mii = true,
    .support_rgmii = true,
    .rx_delay_max = 7,                       // أقصر من H3 (31)
    // tx_delay_max = 0 = مش بيدعم TX delay!
};
```

لاحظ إن `tx_delay_max = 0` في `emac_variant_r40` — ده معناه إن الـ TX delay chain مش موجود في الـ hardware.

في `sun8i_dwmac_set_syscon()` (السطر 945–958):

```c
if (!of_property_read_u32(node, "allwinner,tx-delay-ps", &val)) {
    if (val % 100) {
        dev_err(dev, "tx-delay must be a multiple of 100\n");
        return -EINVAL;
    }
    val /= 100;
    dev_dbg(dev, "set tx-delay to %x\n", val);
    if (val <= gmac->variant->tx_delay_max) {    // R40: tx_delay_max = 0
        reg |= (val << SYSCON_ETXDC_SHIFT);
    } else {
        dev_err(dev, "Invalid TX clock delay: %d\n", val);
        return -EINVAL;
    }
}
```

لو حد حط `allwinner,tx-delay-ps = <200>` في DT للـ R40 يعتقد إن ده هيساعد، الـ driver هيـ return `-EINVAL` لأن `200/100 = 2 > tx_delay_max(0)` — وده بيمنع الـ probe كله.

**أما السبب الفعلي للـ corruption:**
الـ R40 بيستخدم `sun8i_ccu_reg_field` (register 0x164 في CCU) مش `sun8i_syscon_reg_field` (0x30 في syscon). لو الـ DT حدد الـ syscon phandle للـ CCU node، الـ probe بيمشي، بس لو حدد الـ syscon العادي، الـ regmap_field بيكتب في register غلط وبيعمل corruption في الـ CCU.

```c
/* sun8i_dwmac_probe() */
regmap = sun8i_dwmac_get_syscon_from_dev(pdev->dev.of_node);
if (IS_ERR(regmap))
    regmap = syscon_regmap_lookup_by_phandle(pdev->dev.of_node, "syscon");
```

لو الـ fallback لـ syscon حصل بدل CCU، الـ `gmac->regmap_field` بيكتب في register غلط تماماً.

#### الحل

**1. تحقق من الـ DT للـ R40:**

```dts
// صح للـ R40
&emac {
    compatible = "allwinner,sun8i-r40-gmac";
    syscon = <&ccu>;    // لازم يشاور على الـ CCU مش الـ syscon العادي
    // لا تحط allwinner,tx-delay-ps للـ R40 لأن tx_delay_max = 0
    allwinner,rx-delay-ps = <300>;  // max = 7 × 100 = 700ps
    phy-mode = "rgmii";
};
```

**2. اقرأ الـ CCU register الصح:**

```bash
# CCU base للـ R40 عادةً 0x01C20000
# reg field = 0x164
devmem2 0x01C20164 w
```

**3. تحقق من الـ corruption pattern:**

```bash
# شغّل iperf3 وراقب الـ stats
ethtool -S eth0 | grep -E "rx_overflow|rx_error|tx_under"

# وراقب الـ kernel errors
dmesg | grep -E "sun8i|dwmac|emac" | tail -20
```

**4. لو محتاج TX delay، استخدم PHY-side delay بدل MAC-side:**

```dts
// على RTL8211F مثلاً
ethernet-phy@1 {
    reg = <1>;
    // فعّل internal TX delay في الـ PHY نفسه
    // عبر phy-mode = "rgmii-txid" على مستوى الـ MAC node
};

&emac {
    phy-mode = "rgmii-txid";  // PHY بيعمل TX delay
};
```

#### الدرس المستفاد
كل variant من sun8i EMAC عنده capabilities مختلفة — الـ R40 مش بيدعم TX delay على مستوى MAC. دايمًا ارجع لـ `emac_variant_*` struct قبل ما تضيف أي delay values في DT. كمان الـ R40 بيستخدم CCU مش syscon — لو الـ phandle غلط، الكتابة في regmap بتروح في register غلط وبتعمل corruption غير مباشر.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتتكلم عن الـ stmmac وعائلة الـ dwmac glue drivers:

| المقال | الرابط |
|--------|--------|
| إضافة dwmac-sun8i لـ stmmac — الـ patch series الأساسي | [lwn.net/Articles/724248](https://lwn.net/Articles/724248/) |
| إضافة glue layer لـ Allwinner A20 GMAC (السابق لـ sun8i) | [lwn.net/Articles/580138](https://lwn.net/Articles/580138/) |
| إضافة OXNAS Glue Driver لـ stmmac | [lwn.net/Articles/704258](https://lwn.net/Articles/704258/) |
| إضافة STi GMAC Ethernet glue layer | [lwn.net/Articles/585359](https://lwn.net/Articles/585359/) |
| إضافة Toshiba Visconti SoCs glue driver | [lwn.net/Articles/846017](https://lwn.net/Articles/846017/) |
| stmmac: إضافة DW MAC GPIOs و Baikal-T1 GMAC | [lwn.net/Articles/845381](https://lwn.net/Articles/845381/) |
| أول دعم لـ STMicroelectronics Ethernet في الـ kernel | [lwn.net/Articles/355663](https://lwn.net/Articles/355663/) |

**الـ** article الأهم هنا هو [724248](https://lwn.net/Articles/724248/) لأنه بيوثق بالظبط الـ patch series اللي أضاف `dwmac-sun8i.c` للـ kernel.

---

### الـ Documentation الرسمية داخل الـ kernel

```
Documentation/networking/device_drivers/ethernet/stmicro/stmmac.rst
Documentation/devicetree/bindings/net/allwinner,sun8i-a83t-emac.yaml
Documentation/devicetree/bindings/net/dwmac-sun8i.txt   (قديم، استُبدل بالـ YAML)
Documentation/arch/arm/sunxi.rst
```

**الـ** stmmac documentation الرسمية على kernel.org:
- [docs.kernel.org — stmmac driver](https://docs.kernel.org/networking/device_drivers/ethernet/stmicro/stmmac.html)
- [kernel.org — ARM Allwinner SoCs](https://www.kernel.org/doc/html/v5.6/arm/sunxi.html)
- [docs.kernel.org — ARM Allwinner SoCs](https://docs.kernel.org/arch/arm/sunxi.html)
- [kernel.org — dwmac-sun8i DT bindings (النص القديم)](https://www.kernel.org/doc/Documentation/devicetree/bindings/net/dwmac-sun8i.txt)

---

### Kernel Commits المهمة

الـ commits دي هي اللي أضافت وطورت الـ driver:

| الحدث | المرجع |
|-------|--------|
| الـ patch series الأصلي (v6) اللي أضاف `dwmac-sun8i.c` | [patchwork — v6/05/21](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20170501124520.3769-6-clabbe.montjoie@gmail.com/) |
| إضافة دعم mdio-mux للـ internal/external MDIO | [patchwork — v5/10](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20170911190850.GA2291@Red/) |
| الـ DTS: إضافة sun8i EMAC لـ H3/H5 | [patchwork — v3/07/20](https://patchwork.ozlabs.org/project/netdev/patch/20170403091444.29876-8-clabbe.montjoie@gmail.com/) |
| إضافة دعم V3s EMAC | [mail-archive](https://www.mail-archive.com/netdev@vger.kernel.org/msg215215.html) |
| الـ dwmac-sun8i على GitHub (torvalds/linux) | [github.com/torvalds/linux — dwmac-sun8i.c](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/stmicro/stmmac/dwmac-sun8i.c) |
| DT bindings YAML (allwinner,sun8i-a83t-emac) | [github.com/torvalds/linux — bindings YAML](https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/net/allwinner,sun8i-a83t-emac.yaml) |

---

### نقاشات Mailing List

الـ mailing list discussions دي مهمة لفهم قرارات التصميم:

- **الـ patch series الأصلي (v2)** — إضافة الـ dwmac-sun8i driver:
  [lists.infradead.org — PATCH v2 00/20](http://lists.infradead.org/pipermail/linux-arm-kernel/2017-March/493667.html)

- **الـ patch series النهائي (v6)**:
  [lists.infradead.org — PATCH v6 05/21](http://lists.infradead.org/pipermail/linux-arm-kernel/2017-June/516159.html)

- **نقاش MDIO configuration** على netdev:
  [spinics.net — netdev Re: PATCH v6](https://www.spinics.net/lists/netdev/msg442409.html)

- **نقاش EPHY reset timeout bug**:
  [mail-archive.com — EPHY reset](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1412840.html)

- **نقاش دعم Pine64 (A64 EMAC)**:
  [lkml.kernel.org — sun50i-a64 pine64](https://lkml.kernel.org/netdev/20170314141856.24560-17-clabbe.montjoie@gmail.com/)

- **الـ stable backport fix** (kernel 5.17):
  [lore.kernel.org — PATCH 5.17](https://lore.kernel.org/all/20220510130744.069999230@linuxfoundation.org/)

---

### kernelnewbies.org — التغييرات عبر الـ kernel versions

الـ kernelnewbies بيوثق التغييرات المهمة في stmmac عبر الإصدارات:

| الإصدار | التغيير | الرابط |
|---------|---------|--------|
| Linux 5.6 | إضافة TSN support باستخدام TAPRIO API و ETF لـ stmmac | [kernelnewbies.org/Linux_5.6](https://kernelnewbies.org/Linux_5.6) |
| Linux 6.1 | تحسينات متعددة في stmmac | [kernelnewbies.org/Linux_6.1](https://kernelnewbies.org/Linux_6.1) |
| Linux 6.5 | تحديثات على stmmac | [kernelnewbies.org/Linux_6.5](https://kernelnewbies.org/Linux_6.5) |
| Linux 6.8 | stmmac EST implementation + HW-accelerated VLAN stripping | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| Linux 6.12 | إضافة Loongson platform لـ stmmac | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) |
| Linux 6.15 | Tx metadata launch time + FSD EQoS لـ stmmac | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) |
| Linux 6.17 | تحويل stmmac "pcs" إلى phylink | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) |

---

### elinux.org — موارد Allwinner

| الصفحة | الرابط |
|--------|--------|
| Allwinner A1X overview | [elinux.org/A1x](https://elinux.org/A1x) |
| Supporting a new ARM platform: the Allwinner example (PDF) | [elinux.org — Allwinner ARM Platform](https://elinux.org/images/5/51/Ripard--supporting_a_new_arm_platform_the_allwinner_example.pdf) |

---

### linux-sunxi.org — مرجع مجتمع Allwinner

**الـ** linux-sunxi.org هو المرجع التاريخي لجهود mainlining الـ Allwinner SoCs:

| الصفحة | الرابط |
|--------|--------|
| Sun8i EMAC — وصف تقني كامل | [linux-sunxi.org/Sun8i_emac](https://linux-sunxi.org/Sun8i_emac) |
| Ethernet — نظرة عامة على دعم الـ ethernet في Allwinner | [linux-sunxi.org/Ethernet](https://linux-sunxi.org/Ethernet) |
| Linux mainlining effort | [linux-sunxi.org/Linux_mainlining_effort](https://linux-sunxi.org/Linux_mainlining_effort) |
| Linux mainlining history | [linux-sunxi.org/Linux_mainlining_history](https://linux-sunxi.org/Linux_mainlining_history) |

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 17**: Network Drivers — شرح أساسي لبنية الـ `net_device` و `ndo_*` callbacks
- **الفصل 9**: Communicating with Hardware — I/O registers و `ioremap`
- **الفصل 14**: The Linux Device Model — `platform_device` و Device Tree
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 17**: Devices and Modules — فهم `platform_driver` و module init
- **الفصل 13**: The Virtual Filesystem — مفيد لفهم sysfs والـ device model
- يُكمّل LDD3 من ناحية architecture الـ kernel الداخلية

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 15**: Embedded Networking — يتكلم عن Ethernet في الأنظمة المدمجة
- **الفصل 16**: Device Drivers — كيفية كتابة drivers للـ SoC
- مناسب جدًا لفهم سياق `dwmac-sun8i` كـ embedded platform driver

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل 15**: Network Interface Cards — تفاصيل network driver internals
- يغطي NAPI، DMA، وتكامل الـ PHY

---

### ملفات الـ kernel المرتبطة مباشرة

```
drivers/net/ethernet/stmicro/stmmac/dwmac-sun8i.c      ← الملف الرئيسي
drivers/net/ethernet/stmicro/stmmac/stmmac_platform.c  ← platform glue core
drivers/net/ethernet/stmicro/stmmac/stmmac_main.c      ← driver core
drivers/net/ethernet/stmicro/stmmac/stmmac.h            ← shared headers
drivers/net/ethernet/stmicro/stmmac/dwmac1000_core.c   ← GMAC 1000 core ops
include/linux/stmmac.h                                   ← public platform data
drivers/net/mdio-mux.c                                   ← mdio-mux framework
Documentation/devicetree/bindings/net/allwinner,sun8i-a83t-emac.yaml
```

---

### Search Terms للبحث عن معلومات إضافية

لو عايز تبحث أكتر، استخدم الـ search terms دي:

```
dwmac-sun8i Allwinner EMAC stmmac glue layer
sun8i_emac internal PHY EPHY H3 H5 A64
stmmac platform_data of_data callbacks
regmap syscon EMAC clock register Allwinner
mdio-mux internal external PHY Linux kernel
RGMII RMII MII delay chain Allwinner H3
phylink phy_interface_t stmmac sun8i
pm_runtime suspend resume stmmac Allwinner
Corentin Labbe dwmac-sun8i netdev patch
linux-sunxi EMAC mainlining Orange Pi H3
```

---

### Patchwork — تتبع تطور الـ driver

**الـ** Patchwork بيسمح لك تتابع كل revision من الـ patch series:

- [patchwork.kernel.org — linux-arm-kernel stmmac sun8i](https://patchwork.kernel.org/project/linux-arm-kernel/list/?q=dwmac-sun8i)
- [patchwork.ozlabs.org — netdev sun8i](https://patchwork.ozlabs.org/project/netdev/list/?q=dwmac-sun8i)
## Phase 8: Writing simple module

### الفكرة: Kprobe على `sun8i_dwmac_dma_reset`

**الـ function المختارة** هي `sun8i_dwmac_dma_reset` — دي أول function اتحطت في `stmmac_dma_ops` لـ sun8i، وبتعمل reset كامل للـ EMAC DMA عن طريق كتابة صفر في كل registers التحكم وتمسح كل interrupt bits. اخترناها لأنها:
- بتتنادى كل ما الـ network interface بييجي up أو بييعمل recovery
- بتكشف إيمتى الـ driver بيعيد initialize الـ hardware من الأول
- آمنة تماماً للـ hook عليها بـ kprobe بدون أي side effects

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_sun8i_reset.c
 *
 * Hooks sun8i_dwmac_dma_reset() to log every time the Allwinner
 * EMAC DMA engine is reset (e.g. on interface up or error recovery).
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit     */
#include <linux/kprobes.h>     /* kprobe API                            */
#include <linux/kernel.h>      /* pr_info()                             */
#include <linux/io.h>          /* void __iomem, readl()                 */

/* ------------------------------------------------------------------ */
/*  EMAC register offsets (mirrored from dwmac-sun8i.c)               */
/* ------------------------------------------------------------------ */
#define EMAC_RX_CTL1      0x28
#define EMAC_TX_CTL1      0x14
#define EMAC_RX_FRM_FLT   0x38
#define EMAC_INT_EN       0x0C
#define EMAC_INT_STA      0x08

/* ------------------------------------------------------------------ */
/*  pre-handler: runs just BEFORE sun8i_dwmac_dma_reset() executes    */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument (void __iomem *ioaddr) is in RDI.
     * On ARM64 it is in x0.
     * We use regs_get_kernel_argument() which is arch-agnostic since
     * kernel 5.11 and always gives argument N (0-based).
     */
    void __iomem *ioaddr = (void __iomem *)regs_get_kernel_argument(regs, 0);

    /* Read the current interrupt-enable and interrupt-status registers
     * BEFORE the reset wipes them — that tells us what was active.     */
    u32 int_en  = readl(ioaddr + EMAC_INT_EN);
    u32 int_sta = readl(ioaddr + EMAC_INT_STA);
    u32 rx_ctl1 = readl(ioaddr + EMAC_RX_CTL1);
    u32 tx_ctl1 = readl(ioaddr + EMAC_TX_CTL1);

    pr_info("sun8i_dwmac: DMA RESET triggered | "
            "ioaddr=%px INT_EN=0x%08x INT_STA=0x%08x "
            "RX_CTL1=0x%08x TX_CTL1=0x%08x\n",
            ioaddr, int_en, int_sta, rx_ctl1, tx_ctl1);

    return 0; /* 0 = continue executing the probed function normally */
}

/* ------------------------------------------------------------------ */
/*  post-handler: runs just AFTER sun8i_dwmac_dma_reset() returns     */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* After reset, INT_EN should be 0 and INT_STA should be 0x1FFFFFF */
    pr_info("sun8i_dwmac: DMA RESET completed (return value in ax/x0)\n");
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "sun8i_dwmac_dma_reset", /* kernel symbol to probe  */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/*  module init                                                         */
/* ------------------------------------------------------------------ */
static int __init sun8i_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("sun8i_kprobe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("sun8i_kprobe: planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit sun8i_kprobe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("sun8i_kprobe: removed from %s\n", kp.symbol_name);
}

module_init(sun8i_kprobe_init);
module_exit(sun8i_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("Kprobe on sun8i_dwmac_dma_reset to trace EMAC DMA resets");
```

---

### Makefile

```makefile
obj-m += kprobe_sun8i_reset.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# build & load
make
sudo insmod kprobe_sun8i_reset.ko

# trigger a reset by toggling the interface (on H3 board):
sudo ip link set eth0 down && sudo ip link set eth0 up

# watch output
sudo dmesg | grep sun8i

# unload
sudo rmmod kprobe_sun8i_reset
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيوفر `struct kprobe`، `register_kprobe()`، و`regs_get_kernel_argument()` |
| `linux/io.h` | عشان نستخدم `readl()` لقراءة registers الـ EMAC قبل ما الـ reset يمسحها |
| `linux/module.h` | الـ macros الأساسية لأي kernel module: `MODULE_LICENSE`، `module_init`، إلخ |

**الـ `linux/kernel.h`** بيجيب `pr_info()` اللي بنطبع بيها الـ log بدون ما نحتاج user-space.

---

#### الـ pre-handler وحججه

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ `struct kprobe *p`** هو pointer للـ kprobe نفسه — مفيد لو عندك أكتر من probe وعايز تعرف أي واحد اتضرب.
**الـ `struct pt_regs *regs`** بيحمل state الـ CPU registers لحظة الـ trap؛ منه بنسحب الـ argument الأول (ioaddr) بـ `regs_get_kernel_argument(regs, 0)` اللي بيشتغل على x86-64 و ARM64 بدون #ifdef.

بنقرأ `INT_EN`, `INT_STA`, `RX_CTL1`, `TX_CTL1` **قبل** ما الـ reset يكتب صفر فيهم — ده بيديك snapshot مهم لحالة الـ EMAC اللحظة اللي حصل فيها المشكلة أو الـ initialization.

الـ return `0` معناها "كمل تنفيذ الـ function الأصلية"؛ لو رجعنا قيمة تانية الـ kprobe هيتخطى الـ function وده مش مطلوب هنا.

---

#### الـ post-handler

```c
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
```

**الـ `flags`** بيحمل processor flags وقت الـ return؛ مش محتاجينه هنا بس lazem نعلنه عشان الـ signature صح.
بنطبع رسالة بسيطة تأكيد إن الـ reset خلص — لو عايز تتعمق ممكن تقرأ `RX_CTL1` تاني وتتأكد إنه بقى صفر.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "sun8i_dwmac_dma_reset",
    ...
};
```

**الـ `symbol_name`** بيخلي الـ kernel يحل العنوان وقت `register_kprobe()` تلقائياً — مش محتاج تحسب offset يدوي. الـ function لازم تكون exported أو على الأقل موجودة في `kallsyms` (وهي موجودة لأن الـ driver بيشتغل كـ module أو built-in).

---

#### الـ `module_init` و `module_exit`

`register_kprobe()` في الـ init بتزرع الـ breakpoint في الذاكرة وتربط الـ handlers.
`unregister_kprobe()` في الـ exit **ضروري** لأن لو نسيتها وعمل الـ module unload، الـ breakpoint هيفضل في الذاكرة وهيعمل kernel panic أول ما الـ CPU يوصله.
