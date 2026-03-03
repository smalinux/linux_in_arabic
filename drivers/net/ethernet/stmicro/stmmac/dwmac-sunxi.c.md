## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الملف ده جزء من subsystem اسمه **STMMAC (STMicroelectronics MAC Driver)**، وهو الـ Ethernet driver الرئيسي في Linux لكل الـ chips اللي بتستخدم الـ **DesignWare MAC (DWMAC)** IP core من شركة Synopsys — وده IP بتشتريه شركات كتير زي Allwinner وRockchip وMediatek وحتى Intel، وبيحطوه جوه chips بتاعتهم.

الملف بالتحديد هو الـ **glue layer** الخاص بشركة **Allwinner** على الـ SoC اسمه **sun7i (A20)**.

---

### القصة من الأول: إيه المشكلة اللي بيحلها الملف ده؟

#### تخيل إنك بتشتري كيت Ethernet جاهز

شركة **Synopsys** بتبيع الـ IP core بتاع الـ DWMAC كـ "كيت جاهز" للشركات اللي بتصنع الـ SoCs. الكيت ده بيشتغل وبيعمل networking — بس هو مش كامل لوحده. كل شركة بتاخده وبتحطه جوه الـ chip بتاعتها، وبتوصله بباقي أجزاء الـ chip بالطريقة بتاعتها هي.

هنا بتظهر المشكلة:

- **الـ DWMAC core** نفسه منظم وموحد → فيه driver رئيسي واحد بيشتغل معاه: `stmmac_main.c`
- **بس كل شركة** بتوصل الـ clock وبتشغل الـ PHY وبتدي الـ voltage بطريقة مختلفة خالص

يعني الـ DWMAC في Allwinner A20 محتاج:
1. إنك تضبط الـ **TX clock** على frequency معينة حسب نوع الـ interface (MII أو RGMII).
2. إنك تشغل الـ **voltage regulator** اللي بيدي كهرباء لـ chip الـ PHY.
3. إنك تعمل ده في اللحظة الصح (عند الـ init والـ exit).

الـ `stmmac_main.c` (الـ driver الرئيسي) مش عارف يعمل ده لوحده — لأن كل chipset مختلف. فالحل؟

> كل vendor بيكتب **glue layer** صغير يوصل بين الـ generic driver وخصائص الـ hardware بتاعه. ده بالظبط دور `dwmac-sunxi.c`.

---

### الـ Big Picture: إيه اللي بيعمله الملف ده؟

```
                   ┌─────────────────────────────────┐
                   │        Linux Kernel              │
                   │                                  │
                   │  ┌─────────────────────────┐    │
                   │  │   stmmac_main.c          │    │
                   │  │  (Generic DWMAC Driver)  │    │
                   │  └────────────┬────────────┘    │
                   │               │ callbacks         │
                   │               ▼                  │
                   │  ┌─────────────────────────┐    │
                   │  │   dwmac-sunxi.c          │    │
                   │  │  (Allwinner Glue Layer)  │    │
                   │  └────┬──────────┬──────────┘    │
                   │       │          │               │
                   │       ▼          ▼               │
                   │  TX Clock    Regulator           │
                   │  (125 MHz    (3.3V for PHY)      │
                   │   or 25MHz)                      │
                   └─────────────────────────────────┘
                                  │
                                  ▼
                         Allwinner A20 SoC
                         (sun7i-a20-gmac HW)
```

الملف بيعمل **3 حاجات بس**:

| الحاجة | الوظيفة |
|--------|---------|
| `sun7i_gmac_init()` | بيشغل الـ regulator لو موجود، وبيضبط الـ TX clock على الـ rate الصح |
| `sun7i_gmac_exit()` | بيقفل الـ clock والـ regulator لما الـ interface بيتوقف |
| `sun7i_set_clk_tx_rate()` | بيغير سرعة الـ TX clock لو speed تغير (مثلاً من 1G لـ 100M) |

---

### إيه دلالة الـ Clock Rates دي؟

الـ Ethernet TX clock لازم يتطابق مع سرعة الـ interface:

```
RGMII / GMII @ 1Gbps  →  TX Clock = 125 MHz  (SUN7I_GMAC_GMII_RGMII_RATE)
MII / GMII @ 100Mbps  →  TX Clock =  25 MHz  (SUN7I_GMAC_MII_RATE)
```

الـ clock driver في Allwinner بيدعم **auto-reparenting** — يعني لما تعمل `clk_set_rate()` بـ 125 MHz، الـ clock تلقائياً بتروح على source مناسب (PLL) من غير ما تحدد ده manually.

---

### الفرق بين RGMII و MII والـ clock enable

في حالة **RGMII**: الـ clock لازم يتشغل (`clk_prepare_enable`) لأن الـ Ethernet controller محتاجه طول الوقت.

في حالة **MII**: الـ clock بس بيتعمله `clk_prepare` من غير enable — لأن الـ PHY هو اللي بيجيب الـ clock من براه في حالة MII.

---

### قصة الـ Probe: إزاي الـ Driver بيتشغل؟

```
boot → Device Tree → Kernel finds "allwinner,sun7i-a20-gmac"
     → sun7i_gmac_probe() يتنفذ
         ├── stmmac_get_platform_resources()   [يجيب IRQ + base address]
         ├── devm_stmmac_probe_config_dt()     [يقرأ DT ويملا plat_dat]
         ├── devm_clk_get("allwinner_gmac_tx") [يجيب TX clock]
         ├── devm_regulator_get_optional("phy")[يجيب voltage regulator]
         ├── يملا callbacks (init/exit/set_clk_tx_rate)
         └── devm_stmmac_pltfr_probe()         [يسلم للـ generic driver]
```

الـ `plat_dat` هو الجسر — struct بيتملا هنا بكل المعلومات والـ callbacks اللي الـ generic driver محتاجها.

---

### الـ Device Tree Node المقابل

في ملف `sun7i-a20.dtsi`:

```dts
gmac: ethernet@1c50000 {
    compatible = "allwinner,sun7i-a20-gmac";
    reg = <0x01c50000 0x10000>;
    clocks = <&ccu CLK_AHB_GMAC>, <&gmac_tx_clk>;
    clock-names = "stmmaceth", "allwinner_gmac_tx";
    snps,pbl = <2>;
    snps,fixed-burst;
};
```

وفي ملف Board مثل `sun7i-a20-bananapi.dts`:

```dts
&gmac {
    pinctrl-0 = <&gmac_rgmii_pins>;
    phy-mode = "rgmii-id";
    phy-supply = <&reg_gmac_3v3>;  /* الـ regulator اللي بيتشغل في sun7i_gmac_init */
};
```

---

### الملفات المكوّنة للـ Subsystem

#### الـ Core (Generic DWMAC Driver)

| الملف | الدور |
|-------|-------|
| `drivers/net/ethernet/stmicro/stmmac/stmmac_main.c` | قلب الـ driver: TX/RX, interrupts, napi |
| `drivers/net/ethernet/stmicro/stmmac/stmmac_platform.c` | كود الـ platform probe المشترك |
| `drivers/net/ethernet/stmicro/stmmac/stmmac_mdio.c` | إدارة الـ MDIO bus والـ PHY |
| `drivers/net/ethernet/stmicro/stmmac/stmmac_ethtool.c` | ethtool interface |
| `drivers/net/ethernet/stmicro/stmmac/stmmac_ptp.c` | PTP / hardware timestamping |

#### الـ HW Abstraction Layer

| الملف | الدور |
|-------|-------|
| `drivers/net/ethernet/stmicro/stmmac/dwmac1000_core.c` | GMAC 1000 (DWMAC v3.x) core ops |
| `drivers/net/ethernet/stmicro/stmmac/dwmac4_core.c` | DWMAC v4.x core ops |
| `drivers/net/ethernet/stmicro/stmmac/hwif.c` | اختيار الـ HW interface الصح |
| `drivers/net/ethernet/stmicro/stmmac/dwmac_lib.c` | مكتبة مشتركة للـ DMA |

#### الـ Headers الأساسية

| الملف | الدور |
|-------|-------|
| `include/linux/stmmac.h` | الـ `plat_stmmacenet_data` struct والـ platform data |
| `drivers/net/ethernet/stmicro/stmmac/stmmac.h` | الـ internal structs للـ driver |
| `drivers/net/ethernet/stmicro/stmmac/stmmac_platform.h` | API بين الـ glue layers والـ platform core |

#### الـ Glue Layers لـ Vendors تانية (للمقارنة)

| الملف | الـ Vendor / SoC |
|-------|------------------|
| `dwmac-sunxi.c` | Allwinner sun7i (A20) — **الملف الحالي** |
| `dwmac-sun8i.c` | Allwinner sun8i (H3, H5...) |
| `dwmac-sun55i.c` | Allwinner sun55i (A523...) |
| `dwmac-rk.c` | Rockchip |
| `dwmac-stm32.c` | STMicroelectronics STM32 |
| `dwmac-intel.c` | Intel PCIe Ethernet |
| `dwmac-mediatek.c` | MediaTek |

#### الـ Device Tree Files

| الملف | الدور |
|-------|-------|
| `arch/arm/boot/dts/allwinner/sun7i-a20.dtsi` | تعريف الـ GMAC node الأساسي |
| `arch/arm/boot/dts/allwinner/sun7i-a20-bananapi.dts` | مثال board بيستخدم RGMII مع regulator |
| `arch/arm/boot/dts/allwinner/sun7i-a20-cubieboard2.dts` | board تاني يستخدم نفس الـ GMAC |

---

### ملخص جملة واحدة

**الـ** `dwmac-sunxi.c` هو الطبقة الرفيعة اللي بتربط بين الـ generic DWMAC driver وبين الـ clock وalـ regulator الخاص بـ Allwinner A20 — مش أكتر من كده.
## Phase 2: شرح الـ STMMAC / DWMAC Glue Layer Framework

---

### المشكلة اللي بيحلها الـ Framework ده

الـ **DesignWare MAC (DWMAC)** هو IP core من Synopsys — STMicroelectronics بتشتريه وبتدمجه في SoCs زي STM32. Allwinner كمان بتاخد نفس الـ IP وبتحطه جوا chips زي A20 (sun7i). Rockchip، Amlogic، Apple — كلهم بيعملوا نفس الحاجة.

المشكلة: الـ DWMAC core نفسه (رجسترات الـ MAC، الـ DMA، الـ MTL) **واحد في كل الـ SoCs**. بس كل vendor عنده:

1. **Clock wiring مختلف** — الـ TX clock ممكن تيجي من PLL مختلف أو تتوصل بطريقة تانية.
2. **Regulator للـ PHY** — بعض الـ SoCs عندها power rail منفصل للـ PHY محتاج يتفعّل قبل الـ link.
3. **PHY interface selection** — الـ SoC بيحدد MII/RGMII/GMII عبر hardware straps أو clock rate.
4. **FIFO sizes وـ features مختلفة** بحسب الـ silicon revision.

لو كل vendor كتب driver منفصل من الصفر → code duplication هائل. الـ framework بيحل ده بـ **separation of concerns**: كود الـ MAC العام في مكان واحد، وكل SoC بتكتب فقط الـ **glue layer** اللي بيتعامل مع الفروق.

---

### الحل — فكرة الـ Framework

الـ kernel بيعمل طبقتين:

| الطبقة | الملف | المسؤولية |
|--------|-------|-----------|
| **Core driver** | `stmmac_main.c`, `stmmac_dma.c`, ... | الـ MAC/DMA registers، TX/RX rings، NAPI، phylink |
| **Platform glue** | `dwmac-sunxi.c`، `dwmac-rk.c`، ... | Clocks، regulators، PHY interface mode، SoC-specific quirks |

الربط بينهم عبر struct واحدة هي **`plat_stmmacenet_data`** — دي الـ "contract" بين الـ glue وبين الـ core.

---

### التشبيه الحقيقي — المطار والطيران

تخيل الـ DWMAC core هو **شركة طيران** — عارفة إزاي تطير الطائرات، تدير الـ check-in، تتعامل مع الـ passengers. بس هي محتاجة في كل مطار جديد حد **يجهّز الأرضية**:

- يشغّل الوقود (= regulator للـ PHY)
- يضبط مسار التوصيل للـ runway (= clock rate للـ TX)
- يحدد نوع الـ gate (MII vs RGMII = interface type)
- يعمل cleanup لما الطائرة تغادر (= `exit` callback)

الـ `sun7i_gmac_probe` هو **مدير المطار في A20** — بيجهز كل ده ويسلّمه للـ core driver عبر الـ `plat_stmmacenet_data`. الـ core driver مش محتاج يعرف إنه في Allwinner A20 أو Rockchip RK3399 — هو بس بيسأل: "فين الـ `init`؟ فين الـ `exit`؟".

التفصيل في التشبيه:

| المطار | الـ Kernel |
|--------|-----------|
| وقود الطائرة | `regulator_enable(gmac->regulator)` |
| سرعة الـ runway للإقلاع | `clk_set_rate(tx_clk, 125MHz)` لـ RGMII |
| نوع الـ gate (domestic/international) | `phy_interface_t interface` |
| مدير المطار | `sun7i_gmac_probe` |
| شركة الطيران | `stmmac_main.c` (core driver) |
| عقد التشغيل | `plat_stmmacenet_data` |
| إغلاق المطار | `sun7i_gmac_exit` |

---

### الـ Big Picture — أين يقع الـ glue في الـ kernel؟

```
+---------------------------+
|     Userspace / Apps      |
+---------------------------+
           |  socket()
+---------------------------+
|      Network Stack        |
|   (TCP/IP, skb, netdev)   |
+---------------------------+
           |
+---------------------------+
|   stmmac Core Driver      |  ← stmmac_main.c, stmmac_dma.c
|   (MAC regs, DMA rings,   |     هنا بيعيش الكود العام
|    NAPI, phylink glue)    |
+---------------------------+
     |            |
     |            +---------------------------+
     |            |  plat_stmmacenet_data     |  ← الـ contract
     |            |  (callbacks + config)     |
     |            +---------------------------+
     |                        |
+----+----+        +----------+-----------+
|  PHY    |        |  Platform Glue Layer  |  ← dwmac-sunxi.c (نحن هنا)
| (phylib)|        |  dwmac-sunxi.c        |
+---------+        |  - sun7i_gmac_probe   |
                   |  - sun7i_gmac_init    |
                   |  - sun7i_gmac_exit    |
                   |  - sun7i_set_clk_tx   |
                   +----------+-----------+
                              |
              +---------------+---------------+
              |                               |
     +--------+--------+           +---------+--------+
     |   Clock Driver  |           |  Regulator Driver |
     | (CCU / clk-sun) |           |  (AXP209 PMIC)    |
     +-----------------+           +------------------+
              |                               |
     +--------+--------+           +---------+--------+
     |  Allwinner A20  |           |  PHY Power Rail   |
     |  GMAC Hardware  |           |  (3.3V or 1.8V)   |
     +-----------------+           +------------------+
```

---

### الـ Core Abstraction — `plat_stmmacenet_data`

ده الـ struct المركزي اللي الـ framework بياخد فكرته منه. هو بيحتوي على:

#### 1. Static configuration (بتتملا في الـ probe)

```c
plat_dat->tx_coe        = 1;           /* MAC بيعمل TX checksum offload */
plat_dat->core_type     = DWMAC_CORE_GMAC; /* نوع الـ IP core */
plat_dat->tx_fifo_size  = 4096;        /* size الـ TX FIFO في bytes */
plat_dat->rx_fifo_size  = 16384;       /* size الـ RX FIFO */
```

#### 2. Callbacks (hooks بتتعبى بـ function pointers)

```c
plat_dat->init             = sun7i_gmac_init;      /* عند بدء الـ driver */
plat_dat->exit             = sun7i_gmac_exit;      /* عند إيقاف الـ driver */
plat_dat->set_clk_tx_rate  = sun7i_set_clk_tx_rate; /* عند تغيير السرعة */
```

الـ core driver بيشيل مسؤوليته بعيدة عن التفاصيل — بس بينادي:
```c
if (priv->plat->init)
    priv->plat->init(dev, priv->plat->bsp_priv);
```

#### 3. Private data pointer

```c
plat_dat->bsp_priv = gmac;  /* pointer لـ sunxi_priv_data */
```

الـ core driver بيمرر الـ `bsp_priv` لكل callback — فالـ glue بيقدر يوصل لـ state خاصة بيه زي `tx_clk` و`regulator`.

---

### الـ Structs والعلاقات بينها

```
platform_device (pdev)
       |
       | stmmac_get_platform_resources()
       v
stmmac_resources
  .addr    → MMIO base address (MAC registers)
  .irq     → interrupt number
  .mac[6]  → MAC address bytes
       |
       | devm_stmmac_probe_config_dt()
       v
plat_stmmacenet_data              sunxi_priv_data
  .phy_interface  ─────────────→  .interface
  .bsp_priv ───────────────────→  .tx_clk   → struct clk*
  .init     → sun7i_gmac_init     .regulator → struct regulator*
  .exit     → sun7i_gmac_exit     .clk_enabled (flag)
  .set_clk_tx_rate → ...
  .tx_coe = 1
  .tx_fifo_size = 4096
  .rx_fifo_size = 16384
  .core_type = DWMAC_CORE_GMAC
       |
       | devm_stmmac_pltfr_probe()
       v
stmmac_priv  (الـ core driver state)
  .plat → plat_stmmacenet_data
  .dev  → net_device
  .dma_conf → TX/RX rings
```

---

### الـ Clock Architecture في sun7i

ده من أذكى أجزاء الـ glue — الـ GMAC hardware بيدعم MII (10/100 Mbps) وـ RGMII (1000 Mbps). اختيار الـ mode مش بيتم عبر register — بيتم عبر **تغيير rate الـ TX clock**:

```
PHY Mode       TX Clock Rate     Action
-----------    ---------------   -------------------------
RGMII          125 MHz           clk_prepare_enable()  ← clock runs
MII            25 MHz            clk_prepare() only    ← clock stopped
GMII@1000      125 MHz           clk_prepare_enable()
GMII@100       25 MHz            clk_prepare() only
```

**ليه الفرق في `enable`؟**

الـ RGMII بيحتاج الـ TX clock تتدفع للـ PHY لأن الـ PHY بيستخدمها كـ reference. الـ MII الـ PHY هو اللي بيولّد الـ clock — فالـ MAC بس بيتزامن معاها.

```c
if (phy_interface_mode_is_rgmii(gmac->interface)) {
    clk_set_rate(gmac->tx_clk, SUN7I_GMAC_GMII_RGMII_RATE); /* 125 MHz */
    clk_prepare_enable(gmac->tx_clk); /* enable: MAC drives the clock */
    gmac->clk_enabled = 1;
} else {
    clk_set_rate(gmac->tx_clk, SUN7I_GMAC_MII_RATE); /* 25 MHz */
    clk_prepare(gmac->tx_clk); /* prepare only: PHY drives it */
}
```

الـ **Clock framework** (مش جزء من STMMAC) هو اللي بيدير الـ `clk_set_rate`، الـ parent switching، والـ power gating. هنا الـ glue layer بس بيطلب منه.

الـ **Regulator framework** (كمان مش جزء من STMMAC) بيدير الـ `regulator_enable`/`disable` للـ PHY power rail — غالباً من PMIC زي AXP209.

---

### الـ Probe Flow خطوة خطوة

```
sun7i_gmac_probe(pdev)
    │
    ├─1─ stmmac_get_platform_resources(pdev, &stmmac_res)
    │       └─ بياخد الـ MMIO address والـ IRQ من الـ device tree
    │
    ├─2─ devm_stmmac_probe_config_dt(pdev, stmmac_res.mac)
    │       └─ بيقرأ الـ DT وبيملا plat_dat بالـ phy-mode، mdio، clocks، ...
    │
    ├─3─ devm_kzalloc() → sunxi_priv_data
    │       └─ بيخصص الـ private state للـ glue
    │
    ├─4─ devm_clk_get(dev, "allwinner_gmac_tx")
    │       └─ بياخد handle للـ TX clock من الـ CCU driver
    │
    ├─5─ devm_regulator_get_optional(dev, "phy")
    │       └─ بياخد handle للـ PHY regulator (optional)
    │       └─ لو -EPROBE_DEFER → defer (الـ PMIC لسه مش ready)
    │
    ├─6─ تعبئة plat_dat بالـ callbacks والـ config
    │       └─ init, exit, set_clk_tx_rate, tx_coe, fifo sizes, ...
    │
    └─7─ devm_stmmac_pltfr_probe(pdev, plat_dat, &stmmac_res)
            └─ الـ core driver بياخد الكنترول ويبدأ الـ full initialization
```

---

### ما الذي يملكه الـ Framework vs ما يفوّضه للـ Glue

| المسؤولية | مين بيعملها؟ |
|-----------|-------------|
| TX/RX DMA ring setup | Core (`stmmac_dma.c`) |
| MAC register configuration | Core (`dwmac_lib.c`) |
| NAPI poll / packet processing | Core (`stmmac_main.c`) |
| phylink integration | Core |
| Clock enable/disable | **Glue** (`sun7i_gmac_init/exit`) |
| Regulator enable/disable | **Glue** |
| Clock rate selection per speed | **Glue** (`sun7i_set_clk_tx_rate`) |
| PHY interface mode mapping | **Glue** (بيقرأ الـ DT ويخزنها) |
| FIFO size declaration | **Glue** (بيعرف الـ silicon) |
| TX checksum offload flag | **Glue** (بيعرف الـ hardware capability) |
| Suspend/Resume PM | Core يتصل بـ `stmmac_pltfr_pm_ops` |

---

### نقطة مهمة — `devm_` وـ lifetime management

كل الـ allocations في الـ probe بتستخدم `devm_*`:

- `devm_kzalloc` → الـ memory بتتحرر لما الـ device يتـ remove
- `devm_clk_get` → الـ clock handle بيتـ release تلقائياً
- `devm_regulator_get_optional` → الـ regulator بيتـ release تلقائياً
- `devm_stmmac_pltfr_probe` → الـ core driver بيتـ cleanup تلقائياً

يعني الـ glue **مش محتاج يكتب `remove` function** — الـ devres framework بيعمل كل ده.

---

### الـ `of_device_id` و Device Tree Matching

```c
static const struct of_device_id sun7i_dwmac_match[] = {
    { .compatible = "allwinner,sun7i-a20-gmac" },
    { }
};
```

الـ kernel بيقارن الـ `compatible` string في الـ DT node بالـ table ده. لما بيلاقي match، بينادي `sun7i_gmac_probe`. الـ `plat_dat` بيتـ populate من الـ DT node نفسه عبر `devm_stmmac_probe_config_dt`.

الـ DT node المقابل بيبدو كده:

```dts
gmac: ethernet@1c50000 {
    compatible = "allwinner,sun7i-a20-gmac";
    reg = <0x01c50000 0x10000>;
    interrupts = <GIC_SPI 85 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&ccu CLK_AHB_GMAC>, <&ccu CLK_GMAC_TX>;
    clock-names = "stmmaceth", "allwinner_gmac_tx";
    phy-mode = "mii";
    phy-supply = <&reg_vmmc>;
};
```

`phy-mode` → بيتقرأ في `devm_stmmac_probe_config_dt` ويتحط في `plat_dat->phy_interface`، اللي الـ glue بياخده في `gmac->interface`.

`allwinner_gmac_tx` clock name → بيتطابق مع `devm_clk_get(dev, "allwinner_gmac_tx")`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Macros — Cheatsheet

#### الـ Clock Rate Macros

| Macro | القيمة | الاستخدام |
|---|---|---|
| `SUN7I_GMAC_GMII_RGMII_RATE` | 125,000,000 Hz | TX clock rate لـ RGMII/GMII @ 1Gbps |
| `SUN7I_GMAC_MII_RATE` | 25,000,000 Hz | TX clock rate لـ MII @ 100Mbps |

#### الـ `dwmac_core_type` Enum

| القيمة | المعنى |
|---|---|
| `DWMAC_CORE_MAC100` | MAC قديم 100Mbps |
| `DWMAC_CORE_GMAC` | Gigabit MAC — **ده اللي بيستخدمه sunxi** |
| `DWMAC_CORE_GMAC4` | GMAC الإصدار الرابع |
| `DWMAC_CORE_XGMAC` | 10 Gigabit MAC |

#### الـ `STMMAC_FLAG_*` Bitmask Flags (في `plat_stmmacenet_data.flags`)

| Flag | Bit | المعنى |
|---|---|---|
| `STMMAC_FLAG_SPH_DISABLE` | BIT(1) | تعطيل Split-Header |
| `STMMAC_FLAG_USE_PHY_WOL` | BIT(2) | Wake-on-LAN عبر PHY |
| `STMMAC_FLAG_HAS_SUN8I` | BIT(3) | SoC من نوع sun8i |
| `STMMAC_FLAG_TSO_EN` | BIT(4) | TCP Segmentation Offload |
| `STMMAC_FLAG_RX_CLK_RUNS_IN_LPI` | BIT(10) | RX clock شغال في LPI |

> **ملاحظة:** الـ dwmac-sunxi **مش بيضبط أي flag** صراحةً — بيعتمد على الـ defaults.

#### الـ CSR Clock Range Macros (MDC dividers)

| Macro | القيمة | نطاق الـ CSR Clock |
|---|---|---|
| `STMMAC_CSR_60_100M` | 0x0 | 60–100 MHz |
| `STMMAC_CSR_100_150M` | 0x1 | 100–150 MHz |
| `STMMAC_CSR_20_35M` | 0x2 | 20–35 MHz |
| `STMMAC_CSR_35_60M` | 0x3 | 35–60 MHz |
| `STMMAC_CSR_150_250M` | 0x4 | 150–250 MHz |
| `STMMAC_CSR_250_300M` | 0x5 | 250–300 MHz |

#### الـ MTL Queue Algorithm Constants

| Macro | القيمة | المعنى |
|---|---|---|
| `MTL_TX_ALGORITHM_WRR` | 0x0 | Weighted Round Robin |
| `MTL_TX_ALGORITHM_SP` | 0x3 | Strict Priority |
| `MTL_RX_ALGORITHM_SP` | 0x4 | Strict Priority للـ RX |
| `MTL_QUEUE_AVB` | 0x0 | Audio Video Bridging queue |
| `MTL_QUEUE_DCB` | 0x1 | Data Center Bridging queue |

#### الـ RX COE (Checksum Offload Engine) Constants

| Macro | القيمة | المعنى |
|---|---|---|
| `STMMAC_RX_COE_NONE` | 0 | لا يوجد checksum offload |
| `STMMAC_RX_COE_TYPE1` | 1 | Type 1 — بيكشف الـ checksum بس |
| `STMMAC_RX_COE_TYPE2` | 2 | Type 2 — بيكشف ويتجاهل الـ bad checksum |

---

### 1. الـ Structs المهمة

#### `struct sunxi_priv_data`

**الغرض:** ده الـ private state الخاص بـ Allwinner sunxi GMAC glue layer. بيتخزن كـ `bsp_priv` جوه `plat_stmmacenet_data`.

```c
struct sunxi_priv_data {
    phy_interface_t interface;   // MII / RGMII / GMII
    int clk_enabled;             // 1 لو TX clock مفعّل حالياً
    struct clk *tx_clk;          // مؤشر لـ TX clock من CCU
    struct regulator *regulator; // optional — لتشغيل PHY power supply
};
```

| الحقل | النوع | الغرض |
|---|---|---|
| `interface` | `phy_interface_t` | نوع الـ interface (RGMII/MII/GMII) — بيتقرأ من Device Tree |
| `clk_enabled` | `int` | flag بيتتبع حالة الـ TX clock (enabled/prepared only) |
| `tx_clk` | `struct clk *` | الـ TX clock من Allwinner CCU — بتتغير rate-ه حسب السرعة |
| `regulator` | `struct regulator *` | طاقة الـ PHY — optional، لو مش موجود `NULL` |

**الارتباط بالـ structs التانية:**
- **الـ** `plat_stmmacenet_data.bsp_priv` بيشاور عليه كـ `void *`
- بيتحوّل لـ `struct sunxi_priv_data *` في كل callback

---

#### `struct plat_stmmacenet_data`

**الغرض:** الـ platform data الرئيسية للـ STMMAC core. بتحتوي على كل الـ hardware capabilities والـ callbacks. الـ glue layer (sunxi) بيملا الحقول المهمة فيها.

**الحقول اللي بيضبطها الـ sunxi driver:**

| الحقل | القيمة المضبوطة | السبب |
|---|---|---|
| `tx_coe` | 1 | الـ hardware بيدعم TX checksum offload |
| `core_type` | `DWMAC_CORE_GMAC` | SoC بيستخدم GMAC core |
| `bsp_priv` | `gmac` (sunxi_priv_data) | pointer للـ private data |
| `init` | `sun7i_gmac_init` | callback التهيئة |
| `exit` | `sun7i_gmac_exit` | callback التنظيف |
| `set_clk_tx_rate` | `sun7i_set_clk_tx_rate` | callback تغيير سرعة الـ clock |
| `tx_fifo_size` | 4096 bytes | حجم TX FIFO |
| `rx_fifo_size` | 16384 bytes | حجم RX FIFO |
| `phy_interface` | من DT | نوع الـ PHY interface |

**الحقول الرئيسية في الـ struct كله:**

| الحقل | الغرض |
|---|---|
| `core_type` | نوع الـ DWMAC core |
| `phy_interface` | interface بين MAC والـ PHY |
| `dma_cfg` | إعدادات الـ DMA burst |
| `axi` | إعدادات AXI bus |
| `rx_queues_cfg[]` | إعدادات RX queues (لغاية 8) |
| `tx_queues_cfg[]` | إعدادات TX queues (لغاية 8) |
| `bsp_priv` | pointer للـ BSP private data |
| `init` / `exit` | lifecycle callbacks |
| `set_clk_tx_rate` | callback لضبط TX clock عند تغيير السرعة |
| `stmmac_clk` | الـ main STMMAC clock |
| `tx_fifo_size` / `rx_fifo_size` | أحجام الـ FIFOs |
| `flags` | bitmask للـ feature flags |

---

#### `struct stmmac_resources`

**الغرض:** بيحتوي على الـ hardware resources (memory-mapped registers، interrupts، MAC address) اللي بيتجمعها `stmmac_get_platform_resources()` من الـ platform device.

| الحقل | الغرض |
|---|---|
| `addr` | base address للـ GMAC registers (MMIO) |
| `mac` | MAC address (6 bytes) |
| `wol_irq` | Wake-on-LAN interrupt |
| `lpi_irq` | Low Power Idle interrupt |
| `irq` | الـ main interrupt |

---

#### `struct stmmac_dma_cfg`

**الغرض:** إعدادات الـ DMA engine — بيتحدد منها burst length وغيره.

| الحقل | المعنى |
|---|---|
| `pbl` | Programmable Burst Length |
| `txpbl` / `rxpbl` | burst length مستقل لكل TX/RX |
| `pblx8` | ضرب الـ pbl في 8 |
| `fixed_burst` | fixed burst mode |
| `aal` | Address-Aligned Beats |
| `eame` | Enhanced Address Mode Enable |

---

#### `struct stmmac_axi`

**الغرض:** إعدادات AXI bus interface — للـ SoCs اللي بتستخدم AXI interconnect.

| الحقل | المعنى |
|---|---|
| `axi_lpi_en` | تفعيل LPI على AXI |
| `axi_wr_osr_lmt` | حد outstanding write requests |
| `axi_rd_osr_lmt` | حد outstanding read requests |
| `axi_blen_regval` | burst lengths المسموح بيها |

---

### 2. مخطط علاقات الـ Structs

```
                    platform_device (pdev)
                          │
                          │ stmmac_get_platform_resources()
                          ▼
                  stmmac_resources
                  ┌──────────────┐
                  │ addr (MMIO)  │
                  │ mac[6]       │
                  │ irq          │
                  └──────┬───────┘
                         │
                         │ devm_stmmac_probe_config_dt()
                         ▼
              plat_stmmacenet_data
       ┌──────────────────────────────────────┐
       │ core_type = DWMAC_CORE_GMAC          │
       │ phy_interface                        │
       │ tx_coe = 1                           │
       │ tx_fifo_size = 4096                  │
       │ rx_fifo_size = 16384                 │
       │                                      │
       │ init ──────────► sun7i_gmac_init()   │
       │ exit ──────────► sun7i_gmac_exit()   │
       │ set_clk_tx_rate► sun7i_set_clk_tx_rate() │
       │                                      │
       │ bsp_priv ─────────────────┐          │
       │ dma_cfg ──► stmmac_dma_cfg│          │
       │ axi ──────► stmmac_axi    │          │
       │ mdio_bus_data             │          │
       └───────────────────────────┼──────────┘
                                   │
                                   ▼
                       sunxi_priv_data
                  ┌────────────────────────┐
                  │ interface              │
                  │ clk_enabled            │
                  │ tx_clk ──► clk (CCU)  │
                  │ regulator ──► PHY VCC  │
                  └────────────────────────┘
```

---

### 3. مخطط الـ Lifecycle

```
                        ┌─────────────────────┐
                        │   module_init()      │
                        │ (module_platform_    │
                        │  driver macro)       │
                        └──────────┬──────────┘
                                   │ register platform driver
                                   ▼
                        ┌─────────────────────┐
                        │  kernel matches DT   │
                        │  "allwinner,         │
                        │   sun7i-a20-gmac"    │
                        └──────────┬──────────┘
                                   │
                                   ▼
                  ┌────────────────────────────────┐
                  │      sun7i_gmac_probe()         │
                  │                                 │
                  │  1. get platform resources      │
                  │  2. parse DT → plat_dat         │
                  │  3. alloc sunxi_priv_data       │
                  │  4. get tx_clk                  │
                  │  5. get optional regulator       │
                  │  6. fill plat_dat fields        │
                  │  7. devm_stmmac_pltfr_probe()  │
                  └───────────────┬────────────────┘
                                  │
                                  ▼
                  ┌────────────────────────────────┐
                  │    stmmac core calls            │
                  │    plat_dat->init()             │
                  │    = sun7i_gmac_init()          │
                  │                                 │
                  │  • regulator_enable()           │
                  │  • clk_set_rate(tx_clk, ...)   │
                  │  • clk_prepare_enable() or      │
                  │    clk_prepare()                │
                  └───────────────┬────────────────┘
                                  │
                                  ▼
                  ┌────────────────────────────────┐
                  │         DEVICE RUNNING          │
                  │                                 │
                  │  speed change event:            │
                  │  set_clk_tx_rate() called       │
                  │  → clk_disable/unprepare        │
                  │  → clk_set_rate (new rate)      │
                  │  → clk_prepare_enable           │
                  └───────────────┬────────────────┘
                                  │
                                  ▼
                  ┌────────────────────────────────┐
                  │    stmmac core calls            │
                  │    plat_dat->exit()             │
                  │    = sun7i_gmac_exit()          │
                  │                                 │
                  │  • clk_disable() if enabled     │
                  │  • clk_unprepare()              │
                  │  • regulator_disable()          │
                  └───────────────┬────────────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   module_exit()      │
                        │ unregister driver    │
                        └─────────────────────┘
```

---

### 4. مخطط الـ Call Flow

#### 4.1 الـ Probe Flow

```
kernel: device match "allwinner,sun7i-a20-gmac"
  └─► sun7i_gmac_probe(pdev)
        ├─► stmmac_get_platform_resources(pdev, &stmmac_res)
        │     └─► reads MMIO base, IRQ, MAC addr from DT/platform
        │
        ├─► devm_stmmac_probe_config_dt(pdev, stmmac_res.mac)
        │     └─► parses DT: phy-mode, clocks, mdio, etc.
        │         returns allocated plat_stmmacenet_data
        │
        ├─► devm_kzalloc(dev, sizeof(sunxi_priv_data))
        │     └─► allocates BSP private struct
        │
        ├─► devm_clk_get(dev, "allwinner_gmac_tx")
        │     └─► gets handle to TX clock from CCU driver
        │
        ├─► devm_regulator_get_optional(dev, "phy")
        │     └─► optional PHY regulator (e.g., AXP209 LDO)
        │         NULL if not present in DT
        │
        ├─► fills plat_dat:
        │     plat_dat->tx_coe = 1
        │     plat_dat->core_type = DWMAC_CORE_GMAC
        │     plat_dat->bsp_priv = gmac
        │     plat_dat->init = sun7i_gmac_init
        │     plat_dat->exit = sun7i_gmac_exit
        │     plat_dat->set_clk_tx_rate = sun7i_set_clk_tx_rate
        │     plat_dat->tx_fifo_size = 4096
        │     plat_dat->rx_fifo_size = 16384
        │
        └─► devm_stmmac_pltfr_probe(pdev, plat_dat, &stmmac_res)
              ├─► calls plat_dat->init()  ← sun7i_gmac_init()
              └─► registers net_device, sets up DMA, IRQ, etc.
```

#### 4.2 الـ Init Flow (clock/regulator setup)

```
stmmac core calls plat_dat->init(dev, bsp_priv)
  └─► sun7i_gmac_init(dev, priv)
        ├─► [if regulator] regulator_enable(gmac->regulator)
        │     └─► powers on PHY chip (e.g., via PMIC)
        │
        ├─► [if RGMII/GMII interface]
        │     ├─► clk_set_rate(tx_clk, 125_000_000)
        │     │     └─► CCU auto-reparents to PLL_PERIPH / 5
        │     └─► clk_prepare_enable(tx_clk)
        │           └─► gmac->clk_enabled = 1
        │
        └─► [if MII interface]
              ├─► clk_set_rate(tx_clk, 25_000_000)
              └─► clk_prepare(tx_clk)   ← NOT enabled yet
                    (MII TX clock comes from PHY, not SoC)
```

#### 4.3 الـ Speed Change Flow

```
phylink detects speed change
  └─► stmmac core calls plat_dat->set_clk_tx_rate(bsp_priv, clk_tx_i, iface, speed)
        └─► sun7i_set_clk_tx_rate(bsp_priv, clk_tx_i, interface, speed)
              │
              └─► [only if PHY_INTERFACE_MODE_GMII]
                    ├─► [if clk_enabled]
                    │     ├─► clk_disable(tx_clk)
                    │     └─► gmac->clk_enabled = 0
                    │
                    ├─► clk_unprepare(tx_clk)
                    │
                    ├─► [if speed == 1000]
                    │     ├─► clk_set_rate(tx_clk, 125_000_000)
                    │     ├─► clk_prepare_enable(tx_clk)
                    │     └─► gmac->clk_enabled = 1
                    │
                    └─► [if speed != 1000]
                          ├─► clk_set_rate(tx_clk, 25_000_000)
                          └─► clk_prepare(tx_clk)
```

#### 4.4 الـ Exit Flow

```
driver removal or interface shutdown
  └─► stmmac core calls plat_dat->exit(dev, bsp_priv)
        └─► sun7i_gmac_exit(dev, priv)
              ├─► [if clk_enabled]
              │     ├─► clk_disable(tx_clk)
              │     └─► gmac->clk_enabled = 0
              │
              ├─► clk_unprepare(tx_clk)
              │     └─► يخلي الـ clock جاهز للـ free
              │
              └─► [if regulator]
                    └─► regulator_disable(gmac->regulator)
                          └─► بيوقف طاقة الـ PHY
```

---

### 5. لا يوجد Explicit Locking في هذا الملف

الـ `dwmac-sunxi.c` **مش بيستخدم أي locks صراحةً** — وده تصميم مقصود:

| السبب | التفاصيل |
|---|---|
| **Locking في الـ stmmac core** | الـ callbacks (`init`, `exit`, `set_clk_tx_rate`) بيتنادى من الـ stmmac core اللي هو بيتحكم في الـ serialization |
| **الـ `clk` API thread-safe** | الـ clock framework في الـ kernel بيعمل internal locking (`clk_prepare_lock` و `enable_lock`) |
| **الـ `regulator` API thread-safe** | الـ regulator framework بيعمل internal mutex |
| **الـ `clk_enabled` flag** | مش محتاج lock لأن الـ callbacks بتتنادى في سياق واحد (الـ rtnl_lock في الغالب) |

```
stmmac core (rtnl_lock held)
  └─► sun7i_gmac_init/exit/set_clk_tx_rate
        └─► clk API (internal: prepare_lock + enable_lock)
              └─► regulator API (internal: mutex)
                    └─► hardware registers
```

**ترتيب الـ Locks الضمني:**
```
rtnl_lock (kernel networking)
  └─► clk_prepare_lock (clock framework, sleeping)
        └─► clk_enable_lock (clock framework, spinlock)
```

> الـ glue layer نفسه نظيف من الـ locking — بيعتمد على ضمانات الـ kernel subsystems.

---

### 6. خلاصة العلاقات الكاملة

```
  Device Tree
  ──────────
  "allwinner,sun7i-a20-gmac"
  phy-mode = "rgmii"
  clocks = <&gmac_tx>
  phy-supply = <&reg_gmac_3v3>
       │
       │ kernel match
       ▼
  platform_device
       │
       │ sun7i_gmac_probe()
       ▼
  ┌─────────────────────────────────────────────────────┐
  │                 sunxi glue layer                     │
  │                                                     │
  │  sunxi_priv_data ◄──────────────────────────────┐  │
  │  ┌──────────────┐                               │  │
  │  │ interface    │                               │  │
  │  │ clk_enabled  │                               │  │
  │  │ tx_clk ──────┼──► Allwinner CCU              │  │
  │  │ regulator ───┼──► AXP209 PMIC (optional)     │  │
  │  └──────────────┘                               │  │
  │                                                 │  │
  │  plat_stmmacenet_data ──────────────────────────┘  │
  │  ┌──────────────────────────────────────────────┐   │
  │  │ core_type = DWMAC_CORE_GMAC                  │   │
  │  │ tx_coe = 1, tx_fifo=4K, rx_fifo=16K         │   │
  │  │ init/exit/set_clk_tx_rate callbacks          │   │
  │  │ bsp_priv ──► sunxi_priv_data                 │   │
  │  └──────────────────┬───────────────────────────┘   │
  └─────────────────────┼───────────────────────────────┘
                        │ devm_stmmac_pltfr_probe()
                        ▼
              stmmac core driver
              ┌──────────────────────────┐
              │  net_device              │
              │  DMA engine              │
              │  IRQ handler             │
              │  phylink / MDIO          │
              │  ethtool ops             │
              └──────────────────────────┘
                        │
                        ▼
              DWMAC/GMAC hardware IP
              (Synopsys DesignWare MAC)
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs — Cheatsheet

| Function / API | النوع | الغرض |
|---|---|---|
| `sun7i_gmac_init` | Callback (init) | تشغيل الـ regulator والـ TX clock عند بدء الـ interface |
| `sun7i_gmac_exit` | Callback (exit) | إيقاف الـ TX clock والـ regulator عند إغلاق الـ interface |
| `sun7i_set_clk_tx_rate` | Callback (set_clk_tx_rate) | ضبط معدل الـ TX clock حسب الـ speed في GMII mode |
| `sun7i_gmac_probe` | Platform Driver probe | تهيئة الـ driver كله وربطه بالـ stmmac core |
| `stmmac_get_platform_resources` | stmmac API | استخراج الـ resources (IRQ, MMIO) من الـ platform device |
| `devm_stmmac_probe_config_dt` | stmmac API | قراءة الـ device tree وبناء `plat_stmmacenet_data` |
| `devm_stmmac_pltfr_probe` | stmmac API | تسجيل الـ driver مع الـ stmmac core كاملاً |
| `regulator_enable` / `regulator_disable` | Regulator API | تشغيل/إيقاف الـ PHY power supply |
| `clk_set_rate` | Clock API | ضبط تردد الـ TX clock لاختيار الـ interface mode |
| `clk_prepare_enable` / `clk_disable` / `clk_unprepare` | Clock API | إدارة دورة حياة الـ clock |
| `devm_clk_get` | Clock API | الحصول على handle للـ TX clock من الـ device tree |
| `devm_regulator_get_optional` | Regulator API | الحصول على handle الـ PHY regulator (اختياري) |
| `devm_kzalloc` | Memory API | تخصيص الـ private data بشكل مُدار |
| `phy_interface_mode_is_rgmii` | PHY API | التحقق هل الـ interface هو RGMII |
| `module_platform_driver` | Module macro | تسجيل/إلغاء تسجيل الـ platform driver تلقائياً |

---

### التصنيف المنطقي للـ Functions

```
┌─────────────────────────────────────────────────┐
│              dwmac-sunxi.c Functions             │
├──────────────────┬──────────────────────────────┤
│  Probe / Setup   │  sun7i_gmac_probe             │
├──────────────────┼──────────────────────────────┤
│  Runtime Init    │  sun7i_gmac_init              │
├──────────────────┼──────────────────────────────┤
│  Runtime Speed   │  sun7i_set_clk_tx_rate        │
├──────────────────┼──────────────────────────────┤
│  Cleanup / Exit  │  sun7i_gmac_exit              │
└──────────────────┴──────────────────────────────┘
```

---

### Group 1: Probe / Setup

هذه الـ group مسؤولة عن تهيئة الـ driver من الصفر عند اكتشاف الـ device في الـ device tree. بتمثل نقطة الدخول الوحيدة للـ kernel لتحميل الـ glue layer الخاص بـ Allwinner sun7i GMAC.

---

#### `sun7i_gmac_probe`

```c
static int sun7i_gmac_probe(struct platform_device *pdev)
```

**ما بتعمله:**
الـ probe function الرئيسية للـ platform driver. بتقرأ الـ resources من الـ device tree، وبتخصص الـ private data، وبتجمع كل الـ callbacks وبتسلمهم للـ stmmac core عشان يكمّل تسجيل الـ network device.

**الـ Parameters:**
- `pdev` — الـ `platform_device` اللي وفّره الـ kernel بعد ما لقى compatible string `allwinner,sun7i-a20-gmac` في الـ device tree.

**الـ Return Value:**
- `0` في حالة النجاح.
- قيمة سالبة (error code) في حالة الفشل — بترجع مباشرة من أي خطوة فشلت.

**Key Details والـ Flow:**

```
sun7i_gmac_probe()
  │
  ├─ stmmac_get_platform_resources()
  │    └─ يجيب MMIO base + IRQ + mac address من الـ DT
  │
  ├─ devm_stmmac_probe_config_dt()
  │    └─ يبني plat_stmmacenet_data من الـ DT
  │         (phy-mode, clocks, mdio, etc.)
  │
  ├─ devm_kzalloc()  →  struct sunxi_priv_data *gmac
  │
  ├─ gmac->interface = plat_dat->phy_interface
  │    └─ محتاجينه في init و set_clk_tx_rate
  │
  ├─ devm_clk_get(dev, "allwinner_gmac_tx")
  │    └─ الـ TX clock — مطلوب، لو مش موجود → ENODEV
  │
  ├─ devm_regulator_get_optional(dev, "phy")
  │    ├─ لو -EPROBE_DEFER → يرجع defer (الـ regulator مش جاهز لسه)
  │    └─ لو أي error تاني → يتجاهله (regulator اختياري)
  │
  ├─ ضبط plat_dat:
  │    tx_coe = 1          (TX checksum offload مدعوم)
  │    core_type = DWMAC_CORE_GMAC
  │    bsp_priv = gmac
  │    init = sun7i_gmac_init
  │    exit = sun7i_gmac_exit
  │    set_clk_tx_rate = sun7i_set_clk_tx_rate
  │    tx_fifo_size = 4096
  │    rx_fifo_size = 16384
  │
  └─ devm_stmmac_pltfr_probe()
       └─ يسجّل الـ net_device مع الـ stmmac subsystem
```

**Caller Context:**
بيتستدعى من الـ kernel عبر الـ platform bus matching — process context، لا يحتاج locks.

**Side Effects:**
كل الـ resources مُدارة بـ `devm_*` — لو حصل فشل في أي خطوة، الـ kernel بيـunwind تلقائياً.

---

### Group 2: Runtime Init / Exit Callbacks

الـ stmmac core بيستدعي الـ `init` و `exit` callbacks في كل مرة بيفتح أو بيقفل الـ interface. مش بس عند الـ probe — يعني ممكن يتستدعوا أكتر من مرة في دورة حياة الـ device (مثلاً بعد suspend/resume).

---

#### `sun7i_gmac_init`

```c
static int sun7i_gmac_init(struct device *dev, void *priv)
```

**ما بتعمله:**
بتشغّل الـ PHY regulator لو موجود، وبعدين بتضبط الـ TX clock بالتردد المناسب حسب الـ interface mode (RGMII أو MII)، وبتـenable الـ clock لو RGMII.

**الـ Parameters:**
- `dev` — الـ `struct device` للـ platform device (بيتستخدم بشكل inderct هنا).
- `priv` — pointer للـ `struct sunxi_priv_data` — بيتـcast منه `gmac`.

**الـ Return Value:**
- `0` في حالة النجاح.
- قيمة سالبة من `regulator_enable` أو `clk_prepare` في حالة الفشل.

**الـ Clock Rate المختارة:**

| Interface | Rate | السبب |
|---|---|---|
| RGMII | 125 MHz (`SUN7I_GMAC_GMII_RGMII_RATE`) | 1 Gbps يحتاج 125 MHz TX clock |
| MII | 25 MHz (`SUN7I_GMAC_MII_RATE`) | 100 Mbps يحتاج 25 MHz TX clock |

**Key Details:**
- في حالة RGMII: `clk_prepare_enable` يُشغّل الـ clock فعلاً ويضبط `clk_enabled = 1`.
- في حالة MII: `clk_prepare` بس بدون enable — الـ clock مش شغال لأن الـ MAC بيولّد الـ clock خودو.
- لو `clk_prepare` فشل في MII mode وفي regulator، بيعمل `regulator_disable` قبل ما يرجع الـ error (cleanup صحيح).

**Caller Context:**
بيتستدعى من `stmmac_open()` — process context.

---

#### `sun7i_gmac_exit`

```c
static void sun7i_gmac_exit(struct device *dev, void *priv)
```

**ما بتعمله:**
بتعمل cleanup عكسي للـ `sun7i_gmac_init` — بتوقف وبتـunprepare الـ TX clock، وبتـdisable الـ regulator لو موجود.

**الـ Parameters:**
- `dev` — الـ `struct device` (مش مستخدم مباشرة).
- `priv` — pointer للـ `struct sunxi_priv_data`.

**الـ Return Value:**
`void` — مفيش return value.

**Key Details:**
```c
// يتعامل مع الحالتين (RGMII كان clock enabled، MII مش enabled)
if (gmac->clk_enabled) {
    clk_disable(gmac->tx_clk);   // يوقف الـ clock signal
    gmac->clk_enabled = 0;
}
clk_unprepare(gmac->tx_clk);     // يفك الـ prepare دايماً

if (gmac->regulator)
    regulator_disable(gmac->regulator);
```

- الـ flag `clk_enabled` بيحمي من double-disable للـ clock.
- `clk_disable` لازم يتستدعى قبل `clk_unprepare` — ده الـ sequence الصحيح في الـ clock framework.

**Caller Context:**
بيتستدعى من `stmmac_release()` — process context.

---

### Group 3: Runtime Speed Adjustment Callback

---

#### `sun7i_set_clk_tx_rate`

```c
static int sun7i_set_clk_tx_rate(void *bsp_priv, struct clk *clk_tx_i,
                                  phy_interface_t interface, int speed)
```

**ما بتعمله:**
بتغيّر معدل الـ TX clock ديناميكياً عند تغيير الـ speed في GMII mode. بتـdisable وبتـunprepare الـ clock الأول، وبعدين بتضبط الـ rate الجديد وبتـre-enable.

**الـ Parameters:**
- `bsp_priv` — pointer للـ `struct sunxi_priv_data` — بيتـcast منه `gmac`.
- `clk_tx_i` — handle للـ TX clock من الـ stmmac core (مش مستخدم هنا — الـ driver بيستخدم `gmac->tx_clk` المحفوظ).
- `interface` — الـ PHY interface mode الحالي.
- `speed` — الـ speed بالـ Mbps (100 أو 1000).

**الـ Return Value:**
- `0` دايماً — حتى لو الـ interface مش GMII، بترجع 0 بدون أي عمل.

**Key Details:**
- الـ function دي بتشتغل **فقط لو** `interface == PHY_INTERFACE_MODE_GMII`.
- في GMII mode، الـ MAC ممكن يشتغل بـ MII speed (100 Mbps) أو GMII speed (1000 Mbps)، والتمييز بيحصل هنا.
- الـ clock auto-reparenting في Allwinner CCU بيتم عبر `clk_set_rate` — الـ clock framework بيختار الـ parent المناسب تلقائياً.

**Flow:**

```
sun7i_set_clk_tx_rate()
  │
  ├─ interface != GMII → return 0 (no-op)
  │
  └─ interface == GMII:
       ├─ clk_disable()       (لو كان enabled)
       ├─ clk_unprepare()
       │
       ├─ speed == 1000:
       │    clk_set_rate(125 MHz)
       │    clk_prepare_enable()
       │    clk_enabled = 1
       │
       └─ speed != 1000 (100 Mbps):
            clk_set_rate(25 MHz)
            clk_prepare()
            (clock NOT enabled — MAC generates its own)
```

**Caller Context:**
بيتستدعى من `stmmac_adjust_link()` أو `phylink` callbacks عند تغيير الـ link speed — process context أو softirq context حسب الـ phylink implementation.

---

### Group 4: Module / Driver Registration

---

#### `module_platform_driver(sun7i_dwmac_driver)`

ده مش function عادية — macro بيتمدد لـ `module_init` و `module_exit` اللي بتسجّل وبتلغي تسجيل الـ platform driver تلقائياً.

```c
static struct platform_driver sun7i_dwmac_driver = {
    .probe  = sun7i_gmac_probe,
    .driver = {
        .name           = "sun7i-dwmac",
        .pm             = &stmmac_pltfr_pm_ops,  /* suspend/resume */
        .of_match_table = sun7i_dwmac_match,
    },
};
```

**الـ `of_match_table`:**

```c
static const struct of_device_id sun7i_dwmac_match[] = {
    { .compatible = "allwinner,sun7i-a20-gmac" },
    { }  /* sentinel */
};
```

- الـ kernel بيمطابق الـ compatible string دي مع الـ device tree nodes.
- `MODULE_DEVICE_TABLE(of, ...)` بيضيف الـ match table في الـ module metadata عشان `modprobe` يقدر يلودها تلقائياً.

**الـ `stmmac_pltfr_pm_ops`:**
بيوفر `suspend` و `resume` جاهزين من الـ stmmac platform layer — الـ glue layer مش محتاجة تكتبهم.

---

### الـ `struct sunxi_priv_data` — Private State

```c
struct sunxi_priv_data {
    phy_interface_t interface;   /* PHY interface: MII, RGMII, GMII */
    int clk_enabled;             /* flag: هل TX clock شغال حالياً */
    struct clk *tx_clk;          /* TX clock handle */
    struct regulator *regulator; /* PHY power supply (optional) */
};
```

الـ struct ده هو الـ state الوحيد الخاص بالـ glue layer. بيتخزن في `plat_dat->bsp_priv` ويتمرر لكل الـ callbacks كـ `void *priv`.

---

### تدفق الـ Clock في الـ Allwinner CCU

```
                ┌─────────────────────┐
                │  Allwinner CCU      │
                │                     │
PLL_PERIPH ─────┤─► MII parent        │
                │       (25 MHz)      │
PLL_VIDEO  ─────┤─► RGMII parent      │
                │       (125 MHz)     │
                │         ↓           │
                │   allwinner_gmac_tx │──► GMAC TX Clock Line
                └─────────────────────┘
```

الـ `clk_set_rate()` بيغيّر الـ rate، وبالـ auto-reparenting feature في الـ CCU driver الـ clock بيختار الـ parent المناسب تلقائياً — ده هو سبب بساطة الكود.

---

### جدول الـ Error Paths

| الخطوة | الـ Error | الـ Action |
|---|---|---|
| `stmmac_get_platform_resources` | أي error | return مباشرة |
| `devm_stmmac_probe_config_dt` | `IS_ERR` | return `PTR_ERR` |
| `devm_kzalloc` | `NULL` | return `-ENOMEM` |
| `devm_clk_get` | `IS_ERR` | return `PTR_ERR` + log error |
| `devm_regulator_get_optional` | `-EPROBE_DEFER` | return defer |
| `devm_regulator_get_optional` | أي error آخر | يتجاهله (optional) |
| `regulator_enable` في init | error | return error |
| `clk_prepare` في init MII | error | `regulator_disable` + return error |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs Entries

**الـ stmmac** بيعمل entries جوه `/sys/kernel/debug/stmmac/` لما يكون `CONFIG_DEBUG_FS=y`.

```bash
# اعرف الـ entries المتاحة
ls /sys/kernel/debug/stmmac/

# الـ entry الرئيسي للـ GMAC على A20
# الاسم بيبقى اسم الـ net device (eth0 مثلاً)
cat /sys/kernel/debug/stmmac/eth0/descriptors_status
cat /sys/kernel/debug/stmmac/eth0/dma_cap
cat /sys/kernel/debug/stmmac/eth0/mac_registers
```

**الـ `descriptors_status`** بيوري حالة الـ TX/RX descriptors، لو الـ DMA واقف أو فيه descriptor مش بيتحرر هتلاقيه هنا.

**الـ `dma_cap`** بيوري الـ hardware capabilities اللي الـ GMAC core بيدعمها (Checksum Offload, Jumbo, etc.) — متأكد إن `tx_coe` اتشال صح.

```bash
# مثال لـ output من dma_cap
cat /sys/kernel/debug/stmmac/eth0/dma_cap
# HW Feature0: 0x0001d0f7
# tx_coe: 1   (TX checksum offload)
# rx_coe: 2   (RX checksum offload type 2)
```

---

#### 2. sysfs Entries

```bash
# تحقق من حالة الـ interface
cat /sys/class/net/eth0/operstate         # up / down
cat /sys/class/net/eth0/carrier           # 1 = link up, 0 = no link
cat /sys/class/net/eth0/speed             # 10 / 100 / 1000
cat /sys/class/net/eth0/duplex            # half / full
cat /sys/class/net/eth0/tx_queue_len
cat /sys/class/net/eth0/statistics/tx_errors
cat /sys/class/net/eth0/statistics/rx_errors
cat /sys/class/net/eth0/statistics/rx_dropped

# الـ regulator اللي بيشغل الـ PHY (phy-supply في DT)
ls /sys/class/regulator/
# دور على الـ regulator المرتبط بـ phy-supply
cat /sys/class/regulator/regulator.X/state    # enabled / disabled
cat /sys/class/regulator/regulator.X/voltage

# الـ clock الخاص بالـ TX
ls /sys/kernel/debug/clk/
cat /sys/kernel/debug/clk/gmac_tx/clk_rate          # لازم 125000000 مع RGMII
cat /sys/kernel/debug/clk/gmac_tx/clk_enable_count  # لازم 1 لو enabled
```

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# mount tracefs لو مش موجود
mount -t tracefs nodev /sys/kernel/tracing

cd /sys/kernel/tracing

# تفعيل الـ net events
echo 1 > events/net/net_dev_xmit/enable
echo 1 > events/net/netif_receive_skb/enable
echo 1 > events/net/napi_poll/enable
echo 1 > events/skb/kfree_skb/enable    # drop tracking

# تفعيل الـ regulator events
echo 1 > events/regulator/regulator_enable/enable
echo 1 > events/regulator/regulator_disable/enable

# تفعيل الـ clk events
echo 1 > events/clk/clk_set_rate/enable
echo 1 > events/clk/clk_enable/enable
echo 1 > events/clk/clk_disable/enable
echo 1 > events/clk/clk_prepare/enable

# شغّل الـ tracer
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
cat trace
```

**تتبع الـ probe بالتفصيل باستخدام function_graph:**

```bash
echo function_graph > current_tracer
echo 'sun7i_gmac_probe' > set_graph_function
echo 'sun7i_gmac_init' >> set_graph_function
echo 1 > tracing_on
# افعل rmmod/modprobe للـ driver
modprobe dwmac-sunxi
echo 0 > tracing_on
cat trace | head -100
```

---

#### 4. printk / Dynamic Debug

```bash
# تفعيل الـ dynamic debug لكل الـ stmmac subsystem
echo 'module stmmac +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module dwmac_sunxi +p' > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل لملف معين فقط
echo 'file dwmac-sunxi.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file stmmac_main.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p=printk, f=function name, l=line number, m=module name, t=thread id

# تحقق من الـ messages في dmesg
dmesg -w | grep -E 'stmmac|gmac|sun7i|dwmac'

# رفع مستوى الـ loglevel لو محتاج
echo 8 > /proc/sys/kernel/printk

# مثال على output متوقع عند probe ناجح
# [    3.142] sun7i-dwmac 1c50000.ethernet: no regulator found
# [    3.145] stmmac_dvr_probe: eth0 - (dev. name: 1c50000.ethernet)
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_STMMAC_DEBUG` | يفعّل verbose logging جوه الـ stmmac core |
| `CONFIG_DEBUG_FS` | لازم عشان debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بـ `echo 'module stmmac +p'` |
| `CONFIG_NET_DROP_MONITOR` | يتتبع الـ dropped packets |
| `CONFIG_DEBUG_SPINLOCK` | يكشف spinlock issues |
| `CONFIG_KALLSYMS` | بيوري أسماء الـ functions في stack traces |
| `CONFIG_CLK_DEBUG` | يفعّل debugfs entries للـ clock framework |
| `CONFIG_REGULATOR_DEBUG` | verbose logging للـ regulator |
| `CONFIG_NETDEV_NOTIFIER_ERROR_INJECT` | testing network notifiers |
| `CONFIG_NET_POLL_CONTROLLER` | لازم لـ netconsole debugging |
| `CONFIG_DEBUG_SHIRQ` | يتحقق من الـ shared IRQ handlers |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks (lockdep) |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'STMMAC|CLK_DEBUG|REGULATOR_DEBUG'
```

---

#### 6. ethtool وأدوات الـ Subsystem

```bash
# إحصائيات مفصّلة من الـ driver
ethtool -S eth0

# معلومات الـ driver والـ firmware
ethtool -i eth0
# driver: stmmac-pci  أو  stmmac
# version: ...

# تحقق من الـ link settings
ethtool eth0
# Speed: 1000Mb/s
# Duplex: Full
# Port: MII  أو  FIBRE

# الـ PHY registers
ethtool -d eth0              # register dump للـ MAC
ethtool --phy-statistics eth0

# اختبار loopback (لو الـ hardware يدعم)
ethtool -t eth0 offline

# الـ MDIO/PHY
# اقرأ PHY registers مباشرة
mii-tool -v eth0
mdio-tool eth0 0 0   # اقرأ register 0 من PHY address 0

# تحقق من الـ offload capabilities
ethtool -k eth0
# tx-checksumming: on   ← لازم on لأن tx_coe=1
# rx-checksumming: on

# الـ pause frames
ethtool -a eth0

# ip link لرؤية الـ flags والـ counters
ip -s link show eth0
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الإصلاح |
|---|---|---|
| `could not get tx clock` | الـ clock `allwinner_gmac_tx` مش موجود في الـ DT أو مش registered | تأكد من `clocks = <&ccu CLK_BUS_EMAC>` و`clock-names = "allwinner_gmac_tx"` في الـ DT |
| `no regulator found` | مفيش `phy-supply` في الـ DT — مش error، just info | لو الـ PHY محتاج power: أضف `phy-supply = <&reg_gmac_3v3>` |
| `EPROBE_DEFER` على الـ regulator | الـ regulator driver مش probe اتعمله لسه | تأكد إن الـ regulator driver موجود وبيـload قبل الـ GMAC |
| `stmmac_dvr_probe: MAC address error` | الـ MAC address مش اتعمله set من DT ولا OTP | أضف `local-mac-address` في DT أو اشتغل بـ random MAC |
| `DMA engine initialization failed` | مشكلة في الـ MMIO mapping أو الـ clock | تأكد إن الـ `reg` property في DT صح و bus clock enabled |
| `failed to reset the dma` | الـ GMAC core مش بيرد — احتمال الـ AHB clock واقف | Enable `CONFIG_CLK_SUNXI` وتأكد من `ahb_gmac` clock |
| `carrier lost` / `Link is Down` | الـ PHY مش شايل الـ link | تحقق من الـ cable، الـ PHY power (regulator)، والـ clock rate |
| `TX timeout` | الـ TX DMA واقف، probaly الـ TX clock بالـ rate الغلط | تحقق من `clk_enabled` وإن الـ rate 125MHz مع RGMII |
| `NAPI schedule error` | الـ NAPI registration فشلت | kernel bug أو memory issue — check `dmesg` للـ OOM |
| `clk_set_rate() failed` | الـ clock driver رفض الـ rate المطلوب | تحقق من الـ clock tree والـ PLL settings للـ A20 |

---

#### 8. نقاط استراتيجية لـ dump_stack() وـ WARN_ON()

في الـ driver نفسه، فيه نقاط منطقية للـ debugging:

```c
/* في sun7i_gmac_init() — تحقق من الـ regulator */
ret = regulator_enable(gmac->regulator);
if (ret) {
    WARN_ON(ret);   /* يطبع stack trace + يكمل */
    return ret;
}

/* في sun7i_set_clk_tx_rate() — تحقق من الـ rate */
int actual = clk_get_rate(gmac->tx_clk);
WARN_ON(actual != SUN7I_GMAC_GMII_RGMII_RATE && speed == 1000);

/* في sun7i_gmac_exit() — تحقق من الـ state */
WARN_ON(gmac->clk_enabled && !gmac->tx_clk);
```

**استخدام الـ `dump_stack()` خارج الـ driver** لتتبع من بيستدعي الـ `init`/`exit`:

```bash
# بدون تعديل الكود — استخدم kprobe من bpftrace
bpftrace -e 'kprobe:sun7i_gmac_init { printf("%s\n", kstack); }'
bpftrace -e 'kprobe:sun7i_gmac_exit { printf("%s\n", kstack); }'
```

---

### Hardware Level

---

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel State

```bash
# الـ Kernel يعتقد إن الـ clock enabled — نتحقق من الـ hardware
cat /sys/kernel/debug/clk/gmac_tx/clk_enable_count
# لو 0 والـ clk_enabled=1 في الـ struct: مشكلة

# الـ interface المستخدم
ethtool eth0 | grep 'Port\|Speed\|Duplex'
# مقارنة بـ DT:
# phy-mode = "rgmii" → لازم Speed: 1000Mb/s

# PHY ID تحقق
cat /sys/class/net/eth0/phydev/phy_id    # مثال: 0x001cc915 (RTL8211E)
cat /sys/class/net/eth0/phydev/interface # rgmii

# تحقق من الـ MII status
mii-tool eth0 -v
# eth0: negotiated 1000baseT-FD flow-control, link ok
```

---

#### 2. Register Dump Techniques

**الـ A20 GMAC base address: `0x01C50000`**

```bash
# تأكد إن /dev/mem متاح (CONFIG_STRICT_DEVMEM=n أو devmem2)
# تثبيت devmem2 لو مش موجود
apt install devmem2   # على Armbian/Ubuntu

# اقرأ الـ GMAC Configuration Register (offset 0x0000)
devmem2 0x01C50000 w
# Value at address 0x01C50000 (0xf1c50000): 0x0080A00C
# Bit 11 = FES (Fast Ethernet Speed)
# Bit 14 = PS (Port Select): 1=MII, 0=GMII/RGMII

# MAC Frame Filter Register (offset 0x0004)
devmem2 0x01C50004 w

# GMAC RGMII Status Register (A20-specific, offset 0x00D8)
devmem2 0x01C500D8 w
# Bit 0 = Link Status
# Bit 1-2 = Speed: 00=2.5MHz, 01=25MHz, 10=125MHz

# DMA Bus Mode Register (offset 0x1000)
devmem2 0x01C51000 w

# DMA Status Register (offset 0x1014) — مهم جداً
devmem2 0x01C51014 w
# Bit 0 = TI (Transmit Interrupt)
# Bit 1 = TPS (Transmit Process Stopped) ← PROBLEM
# Bit 4 = OVF (Receive Overflow)
# Bit 7 = RI (Receive Interrupt)

# قراءة batch لأهم الـ registers
for offset in 0x0000 0x0004 0x0008 0x1000 0x1004 0x1008 0x1010 0x1014 0x1018; do
    addr=$(printf "0x%08x" $((0x01C50000 + offset)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value" | awk '{print $NF}')
    printf "0x01C5%04x = %s\n" $((offset)) "$val"
done
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس على A20:**

```
                 A20 SoC
    ┌────────────────────────────────┐
    │  GMAC Core                     │
    │  ┌──────────┐                  │
    │  │ TX_CLK ──┼──────────────────┼──► TP1 (125MHz RGMII / 25MHz MII)
    │  │ RXD[3:0] ┼──────────────────┼──► TP2-TP5
    │  │ TXD[3:0] ┼──────────────────┼──► TP6-TP9
    │  │ TX_EN ───┼──────────────────┼──► TP10
    │  │ RX_DV ───┼──────────────────┼──► TP11
    │  └──────────┘                  │
    └────────────────────────────────┘
```

**إعدادات الـ Logic Analyzer:**
- **Sample Rate**: 500MHz+ للـ RGMII (125MHz clock)، 100MHz كافي للـ MII
- **Trigger**: على rising edge الـ `TX_EN`
- **Protocol Decoder**: RGMII أو MII (متوفر في Sigrok/PulseView)

```bash
# استخدام sigrok-cli مع logic analyzer
sigrok-cli -d fx2lafw --config samplerate=500m \
  --channels D0=TXD0,D1=TXD1,D2=TXD2,D3=TXD3,D4=TX_EN,D5=TX_CLK \
  --triggers D4=r \
  --samples 1000000 \
  -P rgmii:txd0=TXD0:txd1=TXD1:txd2=TXD2:txd3=TXD3:tx_en=TX_EN:tx_clk=TX_CLK \
  -o capture.sr
```

**Oscilloscope:**
- تحقق من الـ TX_CLK: لازم 125MHz مع RGMII و25MHz مع MII
- القياس على الـ `allwinner_gmac_tx` clock output pin
- لو الـ clock مش ظاهر: مشكلة في `clk_set_rate` أو الـ PLL

---

#### 4. Hardware Issues الشائعة وـ Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | التشخيص |
|---|---|---|
| الـ PHY مش متشغّل كهربائياً | `Link is Down` بعد kup | الـ regulator مش enabled — `dmesg | grep regulator` |
| الـ TX_CLK بالـ rate الغلط | `TX timeout` + packets مش بتوصل | `cat /sys/kernel/debug/clk/gmac_tx/clk_rate` |
| RGMII delay settings غلط | link up بس packets corrupt/dropped | محتاج `phy-handle` مع `rxc-skew-ps` في DT |
| MDC/MDIO مش شغال | `PHY: Could not attach` | تحقق من الـ pull-ups على MDIO lines |
| الـ crystal/oscillator للـ PHY غلط | `auto-negotiation failed` | قيس الـ 25MHz clock على الـ PHY |
| كابل أو connector مشكلة | carrier يجي ويروح | `ethtool eth0 | grep Link` + oscilloscope |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT المحمّل فعلاً
ls /proc/device-tree/soc@01c00000/ethernet@1c50000/

# الـ compatible string
cat /proc/device-tree/soc@01c00000/ethernet@1c50000/compatible
# allwinner,sun7i-a20-gmac

# الـ phy-mode
cat /proc/device-tree/soc@01c00000/ethernet@1c50000/phy-mode
# rgmii  أو  mii  أو  gmii

# الـ clocks
xxd /proc/device-tree/soc@01c00000/ethernet@1c50000/clocks
# هتلاقي phandle للـ AHB clock والـ TX clock

# تحقق من الـ clock names
cat /proc/device-tree/soc@01c00000/ethernet@1c50000/clock-names
# lازم تشوف: allwinner_gmac_tx
# لأن الـ driver بيعمل: devm_clk_get(dev, "allwinner_gmac_tx")

# الـ regulator (اختياري)
cat /proc/device-tree/soc@01c00000/ethernet@1c50000/phy-supply 2>/dev/null
# لو مفيش output: مفيش regulator → gmac->regulator = NULL

# dump كامل للـ DT node
dtc -I fs /proc/device-tree 2>/dev/null | \
  grep -A 30 'ethernet@1c50000'
```

**DT صح مقابل DT غلط:**

```dts
/* DT صح لـ RGMII */
ethernet@1c50000 {
    compatible = "allwinner,sun7i-a20-gmac";
    reg = <0x01c50000 0x10000>;
    interrupts = <GIC_SPI 85 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-names = "macirq";
    clocks = <&ccu CLK_BUS_EMAC>, <&ccu CLK_GMAC_TX>;
    clock-names = "stmmaceth", "allwinner_gmac_tx";  /* ← اسم لازم يطابق الكود */
    phy-mode = "rgmii";
    phy-supply = <&reg_gmac_3v3>;  /* اختياري */
    phy-handle = <&ext_rgmii_phy>;
    status = "okay";
};
```

```bash
# تحقق من إن الـ driver اتعرف على الـ DT
dmesg | grep 'sun7i-dwmac\|1c50000.ethernet'
# sun7i-dwmac 1c50000.ethernet: no regulator found
# stmmac_dvr_probe: Driver Info for 1c50000.ethernet
```

---

### Practical Commands

---

#### مجموعة شاملة جاهزة للنسخ

```bash
#!/bin/bash
# ===== sun7i GMAC Debug Script =====

GMAC_BASE=0x01C50000
IFACE=eth0

echo "=== 1. Link Status ==="
ip link show $IFACE
ethtool $IFACE

echo "=== 2. Clock Status ==="
cat /sys/kernel/debug/clk/gmac_tx/clk_rate 2>/dev/null || \
  find /sys/kernel/debug/clk -name '*gmac*' -exec cat {}/clk_rate \;

echo "=== 3. PHY Info ==="
mii-tool -v $IFACE 2>/dev/null
cat /sys/class/net/$IFACE/phydev/phy_id 2>/dev/null

echo "=== 4. Error Counters ==="
ethtool -S $IFACE 2>/dev/null | grep -E 'err|drop|miss|over'
ip -s link show $IFACE

echo "=== 5. DMA Status Register ==="
devmem2 $(printf "0x%x" $((GMAC_BASE + 0x1014))) w 2>/dev/null

echo "=== 6. RGMII Status Register ==="
devmem2 $(printf "0x%x" $((GMAC_BASE + 0x00D8))) w 2>/dev/null

echo "=== 7. dmesg Recent ==="
dmesg | tail -30 | grep -E 'stmmac|gmac|sun7i|eth0|phy'

echo "=== 8. Offload Caps ==="
ethtool -k $IFACE | grep -E 'checksum|scatter|tso'

echo "=== 9. regulator state ==="
for d in /sys/class/regulator/regulator.*/; do
  name=$(cat $d/name 2>/dev/null)
  state=$(cat $d/state 2>/dev/null)
  volt=$(cat $d/voltage 2>/dev/null)
  [[ "$name" == *gmac* || "$name" == *phy* ]] && \
    echo "$name: $state @ ${volt}uV"
done

echo "=== 10. TX Queue Stats ==="
cat /sys/kernel/debug/stmmac/$IFACE/descriptors_status 2>/dev/null
```

#### تفعيل الـ Dynamic Debug وقراءة الـ Output

```bash
# تفعيل
echo 'file dwmac-sunxi.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module stmmac +p' > /sys/kernel/debug/dynamic_debug/control

# قراءة real-time
dmesg -w | grep -E 'stmmac|dwmac|gmac' &

# إعادة تحميل الـ driver لرؤية الـ probe messages
ip link set eth0 down
rmmod dwmac_sunxi stmmac stmmac_platform
modprobe dwmac_sunxi
dmesg | tail -50
```

#### مثال على Output وكيفية تفسيره

```
# dmesg عند probe ناجح:
[    2.914] sun7i-dwmac 1c50000.ethernet: no regulator found
# ← gmac->regulator = NULL، طبيعي لو مفيش phy-supply في DT

[    2.920] stmmac_dvr_probe: Driver Info for 1c50000.ethernet
[    2.921]   GMAC/MAC4 variants
[    2.921]   Ring mode enabled
[    2.922]   DMA HW capability register supported

[    3.100] 1c50000.ethernet eth0: PHY [1c50000.ethernet:01] driver [RTL8211E]
# ← PHY اتعرف، address 1

[    3.110] 1c50000.ethernet eth0: No Safety Features support found
[    3.115] 1c50000.ethernet eth0: registered PTP clock
[    3.120] 1c50000.ethernet eth0: Link is Up - 1Gbps/Full - flow control rx/tx
# ← RGMII بيشتغل بـ 1000Mbps، الـ TX clock = 125MHz

# dmesg عند مشكلة في الـ clock:
[    2.900] sun7i-dwmac 1c50000.ethernet: could not get tx clock
# ← clock-names في DT مش "allwinner_gmac_tx"

# dmesg عند TX timeout:
[   45.123] 1c50000.ethernet eth0: WATCHDOG: BUG! Ehtype=0x0 Ecookie=0x0
[   45.124] 1c50000.ethernet eth0: TX DMA map failed
# ← TX clock rate غلط أو DMA issue
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Orange Pi بيجيب GMAC بس مش شغال على RGMII

#### العنوان
**Allwinner A20 GMAC لا يرفع link مع PHY على RGMII في custom carrier board**

#### السياق
شركة بتعمل **industrial gateway** باستخدام Orange Pi (Allwinner A20 = sun7i) وبتحط عليه PHY خارجي من نوع RTL8211E مربوط بـ RGMII. البورد custom والفريق عامل DT node للـ GMAC.

#### المشكلة
بعد boot الـ kernel، الـ `eth0` موجود لكن:
```bash
$ ip link show eth0
eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> ...
$ dmesg | grep stmmac
stmmac-sunxi 1c50000.ethernet: no carrier
```
الـ PHY مش بيجاوب على MDIO وmييجي link أبداً.

#### التحليل
الـ driver بيعمل التالي في `sun7i_gmac_probe`:

```c
gmac->interface = plat_dat->phy_interface;  // بياخد interface من DT
```

ثم في `sun7i_gmac_init`:

```c
if (phy_interface_mode_is_rgmii(gmac->interface)) {
    clk_set_rate(gmac->tx_clk, SUN7I_GMAC_GMII_RGMII_RATE); // 125 MHz
    clk_prepare_enable(gmac->tx_clk);
    gmac->clk_enabled = 1;
}
```

الـ `allwinner_gmac_tx` clock لازم يتفعّل وrate بتاعه 125 MHz عشان RGMII يشتغل. لو الـ DT فيه `phy-mode = "mii"` بدل `"rgmii"`, هيعمل:

```c
clk_set_rate(gmac->tx_clk, SUN7I_GMAC_MII_RATE); // 25 MHz فقط
clk_prepare(gmac->tx_clk);  // prepare بس - مش enable!
```

وده معناه إن الـ clock مش enabled، والـ GMAC مش هيبعت TX على الإطلاق.

#### الحل

**افتح DT file:**
```bash
$ grep -r "sun7i-a20-gmac" arch/arm/boot/dts/
arch/arm/boot/dts/sun7i-a20-orangepi.dts
```

**صحح الـ phy-mode:**
```dts
&gmac {
    pinctrl-names = "default";
    pinctrl-0 = <&gmac_rgmii_pins>;
    phy-handle = <&phy1>;
    phy-mode = "rgmii";          /* كان "mii" — ده سبب المشكلة */
    status = "okay";
};
```

**تحقق من الـ clock بعد التغيير:**
```bash
$ cat /sys/kernel/debug/clk/gmac_tx/clk_rate
125000000
$ cat /sys/kernel/debug/clk/gmac_tx/clk_enable_count
1
```

#### الدرس المستفاد
**الـ `phy-mode` في DT هو اللي بيحدد كل حاجة** — الـ clock rate والـ enable state. لو غلط، مش هيطلع error واضح، بس الـ link مش هيطلع أبداً لأن TX clock مش متفعّل. دايما اتحقق من `clk_enable_count` في debugfs قبل ما تشك في الـ PHY نفسه.

---

### السيناريو الثاني: Allwinner H616 Android TV Box مش بيعدي إنترنت بعد suspend/resume

#### العنوان
**الـ Ethernet بيوقف بعد suspend في Android TV Box مبني على H616**

#### السياق
منتج **Android TV Box** بيستخدم Allwinner H616 (sun50i). المستخدم بيعمل standby والبوكس بيرجع، لكن الـ Ethernet مبقاش شغال وبيحتاج reboot كامل.

#### المشكلة
```bash
# بعد resume
$ ping 192.168.1.1
ping: connect: Network is unreachable
$ ethtool eth0
Link detected: no
```

`dmesg` بعد resume:
```
stmmac-sunxi: sun7i_gmac_init: regulator_enable failed: -ENODEV
```

#### التحليل
الـ `stmmac_pltfr_pm_ops` — اللي بيتسجل في:
```c
.pm = &stmmac_pltfr_pm_ops,
```

بيعمل resume عن طريق استدعاء `plat_dat->init` اللي هو `sun7i_gmac_init`. الكود ده:

```c
static int sun7i_gmac_init(struct device *dev, void *priv)
{
    struct sunxi_priv_data *gmac = priv;
    int ret = 0;

    if (gmac->regulator) {
        ret = regulator_enable(gmac->regulator);  // هنا بتفشل
        if (ret)
            return ret;  // بيرجع error وmييكملش
    }
    ...
}
```

**السبب:** الـ regulator اتعمله `disable` في `sun7i_gmac_exit` أثناء suspend:
```c
static void sun7i_gmac_exit(struct device *dev, void *priv)
{
    ...
    if (gmac->regulator)
        regulator_disable(gmac->regulator);  // اتوقف
}
```

لكن في بعض platforms، الـ regulator `always-on` ومش مسموح يتعمله `enable` مرة تانية من غير `disable` صريح، أو الـ regulator driver نفسه محتاج وقت أطول بعد resume.

#### الحل

**اتحقق من الـ regulator:**
```bash
$ cat /sys/kernel/debug/regulator/regulator_summary | grep phy
phy-supply    1   1  3300000 mV
```

**في DT، لو الـ regulator always-on:**
```dts
phy_vcc: regulator-phy {
    compatible = "regulator-fixed";
    regulator-name = "phy-vcc";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    regulator-always-on;     /* أضف ده */
    regulator-boot-on;
};
```

**أو اعمل delay في resume بزيادة:**
```bash
# اتحقق إيه الـ regulator enable time
$ cat /sys/kernel/debug/regulator/phy/enable_count
```

#### الدرس المستفاد
الـ `sun7i_gmac_exit` دايماً بيعمل `regulator_disable`، والـ `sun7i_gmac_init` دايماً بيعمل `regulator_enable`. لو الـ regulator مش designed يتعمل له cycle زي ده، هيفشل على resume. **الحل الأسلم** هو تعريف الـ regulator كـ `always-on` في DT لو هو power rail ثابت.

---

### السيناريو الثالث: Custom Board بـ A20 — الـ GMAC شغال بس الـ Speed Negotiation غلط

#### العنوان
**الـ GMAC بيتفاوض على 100Mbps بدل 1000Mbps مع RGMII PHY على A20**

#### السياق
فريق بيعمل **custom embedded board** بـ Allwinner A20 لاستخدام في IoT gateway. مربطوا RTL8211F PHY بـ RGMII وشغّلوا الـ driver. الـ link طلع لكن throughput كان حوالي 90 Mbps بدل 900+ Mbps.

#### المشكلة
```bash
$ ethtool eth0 | grep Speed
Speed: 100Mb/s    ← المفروض 1000Mb/s
$ iperf3 -c 192.168.1.100
[ ID] Interval   Transfer   Bandwidth
...   0-10 sec   112 MBytes  94 Mbits/sec
```

#### التحليل
الـ `sun7i_set_clk_tx_rate` بيتحكم في clock لما الـ speed بيتغير:

```c
static int sun7i_set_clk_tx_rate(void *bsp_priv, struct clk *clk_tx_i,
                                 phy_interface_t interface, int speed)
{
    struct sunxi_priv_data *gmac = bsp_priv;

    if (interface == PHY_INTERFACE_MODE_GMII) {  // شرط GMII فقط
        ...
        if (speed == 1000) {
            clk_set_rate(gmac->tx_clk, SUN7I_GMAC_GMII_RGMII_RATE); // 125MHz
            clk_prepare_enable(gmac->tx_clk);
            gmac->clk_enabled = 1;
        } else {
            clk_set_rate(gmac->tx_clk, SUN7I_GMAC_MII_RATE); // 25MHz
            clk_prepare(gmac->tx_clk);
        }
    }
    return 0;
}
```

**المشكلة:** الـ condition `interface == PHY_INTERFACE_MODE_GMII` — لو الـ DT محدد `phy-mode = "rgmii"` فالـ interface هيكون `PHY_INTERFACE_MODE_RGMII` مش `GMII`، فالـ function مش هتعمل حاجة خالص لو speed اتغير!

يعني أول init كان شغال صح (125MHz في `sun7i_gmac_init`)، بس لما phylink بيطلب `set_clk_tx_rate` أثناء negotiation وبيعدي RGMII interface، الـ function بترجع 0 من غير ما تغير الـ clock.

#### الحل

**افهم التصميم:** الـ function de designed أساساً للـ GMII mode اللي فيه الـ clock بيتغير مع الـ speed. في RGMII الـ clock ثابت على 125MHz.

**لو محتاج support كامل:**
```bash
# اتحقق إن الـ init ضبط الـ clock صح
$ cat /sys/kernel/debug/clk/gmac_tx/clk_rate
125000000  # لازم تكون كده مع RGMII
```

**المشكلة الحقيقية:** الـ autoneg بتاع الـ PHY — اتحقق:
```bash
$ ethtool -s eth0 speed 1000 duplex full autoneg off
$ ethtool eth0 | grep -E "Speed|Duplex"
Speed: 1000Mb/s
Duplex: Full
```

لو اشتغل، المشكلة في PHY autoneg أو cable/switch، مش في الـ driver.

#### الدرس المستفاد
الـ `sun7i_set_clk_tx_rate` **بتشتغل بس مع GMII**. لـ RGMII، الـ clock بيتضبط مرة واحدة في `sun7i_gmac_init` وبيفضل ثابت. لو الـ speed مش 1G، اشك في الـ PHY configuration والـ cable quality قبل ما تشك في الـ clock management.

---

### السيناريو الرابع: Board Bring-Up — الـ GMAC probe بيفشل بـ EPROBE_DEFER على كل reboot

#### العنوان
**`sun7i_gmac_probe` بيرجع `-EPROBE_DEFER` على loop بسبب regulator dependency**

#### السياق
فريق بيعمل **board bring-up** لـ custom SBC بـ Allwinner A20. البورد عنده PMIC خارجي (AXP209) بيوفر power لـ PHY. الفريق حاط regulator node في DT.

#### المشكلة
```bash
$ dmesg | grep gmac
sun7i-dwmac 1c50000.ethernet: probe with driver sun7i-dwmac failed with error -517
sun7i-dwmac 1c50000.ethernet: probe with driver sun7i-dwmac failed with error -517
# بيتكرر كل شوية
```

`-517 = -EPROBE_DEFER` — الـ driver بيطلب إعادة probe لاحقاً.

#### التحليل
في `sun7i_gmac_probe`:

```c
gmac->regulator = devm_regulator_get_optional(dev, "phy");
if (IS_ERR(gmac->regulator)) {
    if (PTR_ERR(gmac->regulator) == -EPROBE_DEFER)
        return -EPROBE_DEFER;  // بيرجع defer لو الـ regulator لسه مجاش
    dev_info(dev, "no regulator found\n");
    gmac->regulator = NULL;
}
```

الـ `devm_regulator_get_optional` بترجع `-EPROBE_DEFER` لو الـ regulator provider (الـ PMIC driver) لسه ما probe-يش. ده **behavior صح** — kernel بيحاول تاني لما الـ PMIC يبقى جاهز.

**لكن لو بيتكرر للأبد:**
- الـ PMIC driver مش بيشتغل أساساً
- أو اسم الـ regulator في DT غلط

#### الحل

**خطوة 1: اتحقق من الـ PMIC driver:**
```bash
$ dmesg | grep axp209
axp20x-regulator axp20x-regulator: AXP20x regulators driver loaded
# لو مش ظاهر، الـ PMIC مش شغال
```

**خطوة 2: اتحقق من الـ DT regulator name:**
```dts
/* في GMAC node */
gmac: ethernet@1c50000 {
    compatible = "allwinner,sun7i-a20-gmac";
    phy-supply = <&reg_dcdc3>;  /* اسم الـ regulator property */
    ...
};

/* في PMIC node */
reg_dcdc3: dcdc3 {
    regulator-name = "vcc-io";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
};
```

الـ driver بيطلب الـ regulator بالاسم `"phy"` عشان:
```c
devm_regulator_get_optional(dev, "phy");
// ده بيدور على property اسمها "phy-supply" في DT
```

**خطوة 3: لو الـ PHY مش محتاج regulator خارجي:**
```dts
/* شيل سطر phy-supply من DT */
gmac: ethernet@1c50000 {
    compatible = "allwinner,sun7i-a20-gmac";
    /* بدون phy-supply — الـ driver هيعمل gmac->regulator = NULL */
};
```

```bash
$ dmesg | grep gmac
sun7i-dwmac 1c50000.ethernet: no regulator found  # ده الـ dev_info اللي في الكود
```

#### الدرس المستفاد
الـ `-EPROBE_DEFER` مش error حقيقي — ده kernel بيقولك "استنى". الكود في الملف ده بيتعامل معاه صح. **المشكلة الحقيقية دايماً** في DT أو في الـ PMIC driver نفسه. اتحقق من الـ PMIC قبل ما تشك في الـ GMAC driver.

---

### السيناريو الخامس: IoT Gateway بـ A20 — الـ Network بيوقف بعد ساعات تشغيل

#### العنوان
**الـ GMAC link بيسقط بشكل عشوائي بعد ساعات من التشغيل في بيئة حرارة عالية**

#### السياق
**IoT gateway** صناعي بيشتغل في بيئة حرارة عالية (٥٠ درجة). بعد ٤-٦ ساعات من التشغيل، الـ Ethernet بيوقف ومبيرجعش من غير reboot.

#### المشكلة
```bash
# بعد crash الـ network
$ ip link show eth0
eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> ...
$ dmesg | tail -20
stmmac-sunxi: tx timeout
stmmac-sunxi: tx timeout
stmmac-sunxi: reset mac...
stmmac-sunxi: sun7i_gmac_init failed: -5  # -EIO
```

#### التحليل
لما الـ MAC يعمل reset، بيستدعي `sun7i_gmac_exit` ثم `sun7i_gmac_init` من جديد.

**في `sun7i_gmac_exit`:**
```c
static void sun7i_gmac_exit(struct device *dev, void *priv)
{
    struct sunxi_priv_data *gmac = priv;

    if (gmac->clk_enabled) {
        clk_disable(gmac->tx_clk);
        gmac->clk_enabled = 0;
    }
    clk_unprepare(gmac->tx_clk);

    if (gmac->regulator)
        regulator_disable(gmac->regulator);  // بيوقف الـ PHY power
}
```

**ثم في `sun7i_gmac_init`:**
```c
if (gmac->regulator) {
    ret = regulator_enable(gmac->regulator);  // بيحاول يرجع Power
    if (ret)
        return ret;  // لو فشل بيرجع error
}
```

**التسلسل عند درجة حرارة عالية:**
1. الـ TX timeout بيحصل بسبب thermal throttling أو PHY instability
2. الـ stmmac framework بيعمل reset
3. `sun7i_gmac_exit` بيوقف الـ regulator → PHY بيتقفل
4. `sun7i_gmac_init` بيحاول يرجعه بس الـ regulator محتاج وقت أكتر عند درجة حرارة عالية
5. `regulator_enable` بترجع `-EIO` → الـ init بيفشل

#### الحل

**خطوة 1: راقب الحرارة:**
```bash
$ cat /sys/class/thermal/thermal_zone*/temp
# لو > 85000 (85°C) ده مشكلة thermal
```

**خطوة 2: اتحقق من الـ PHY reset time في datasheet وقارنه بالـ regulator:**
```bash
# اتحقق من enable_time للـ regulator
$ cat /sys/kernel/debug/regulator/phy/type
VOLTAGE
```

**خطوة 3: في DT، أضف `regulator-enable-ramp-delay`:**
```dts
phy_vcc: regulator-phy {
    compatible = "regulator-fixed";
    regulator-name = "phy-vcc";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    enable-active-high;
    regulator-enable-ramp-delay = <10000>; /* 10ms بعد enable */
};
```

**خطوة 4: اتحقق من الـ TX FIFO وRX FIFO sizes اللي اتضبطت في probe:**
```c
plat_dat->tx_fifo_size = 4096;   // 4KB TX
plat_dat->rx_fifo_size = 16384;  // 16KB RX
```

لو الـ traffic كتير، الـ TX FIFO الصغير (4KB) ممكن يسبب timeouts. فكّر في تقليل الـ TX load أو تشغيل hardware TX checksum offload:
```c
plat_dat->tx_coe = 1;  // موجود بالفعل في الكود
```

**خطوة 5: اعمل monitoring script:**
```bash
#!/bin/bash
# monitor_eth.sh
while true; do
    if ! ip link show eth0 | grep -q "LOWER_UP"; then
        echo "$(date): Link down — attempting recovery"
        ip link set eth0 down
        sleep 2
        ip link set eth0 up
        dhclient eth0
    fi
    sleep 30
done
```

#### الدرس المستفاد
الـ `sun7i_gmac_exit/init` cycle عند reset مش بيعمل **PHY reset delay**. في بيئات حرارة عالية، الـ regulator والـ PHY بيحتاجوا وقت أكتر للاستقرار. الحل الأمثل هو إما تحسين التبريد، أو إضافة `ramp-delay` للـ regulator في DT، أو التأكد من إن الـ system load مش بيسبب TX timeouts أصلاً. **دايماً اختبر الـ embedded network hardware في درجة الحرارة القصوى** المحددة في spec قبل الإنتاج.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتوثق تطور الـ driver وسياقه في kernel:

| المقال | الأهمية |
|--------|---------|
| [net: stmmac: Add Allwinner A20 GMAC ethernet controller glue layer](https://lwn.net/Articles/580138/) | الـ patch الأصلي اللي أضاف `dwmac-sunxi.c` — فيه شرح glue layer والـ clock mux |
| [Add Allwinner A20 GMAC ethernet support](https://lwn.net/Articles/583965/) | الـ DT patches للـ A20 boards — اتمرجت في 3.15 |
| [net-next: stmmac: add dwmac-sun8i ethernet driver](https://lwn.net/Articles/724248/) | الجيل التاني من Allwinner EMAC — مقارنة مفيدة مع `dwmac-sunxi` |
| [net: stmmac: improve driver statistics](https://lwn.net/Articles/938398/) | تطوير stmmac core اللي بيأثر على كل الـ glue layers |
| [net: stmmac: Add DW MAC GPIOs and Baikal-T1 GMAC support](https://lwn.net/Articles/845381/) | مثال على glue layer تاني — pattern مشابه لـ sunxi |

---

### التوثيق الرسمي في kernel

**الـ Documentation paths** الموجودة في الـ source tree:

```
Documentation/networking/device_drivers/ethernet/stmicro/stmmac.rst
    → التوثيق الرئيسي لـ stmmac driver (features، parameters، ethtool)

Documentation/networking/devlink/stmmac.rst
    → devlink interface للـ stmmac

Documentation/devicetree/bindings/net/allwinner,sun7i-a20-gmac.yaml
    → DT binding الرسمي للـ sun7i GMAC — compatible، clocks، regulators

Documentation/devicetree/bindings/clock/allwinner,sun7i-a20-gmac-clk.yaml
    → binding الـ clock الخاص بالـ GMAC TX clock

Documentation/devicetree/bindings/net/stmmac.txt
    → DT binding للـ stmmac core (shared properties)

Documentation/arch/arm/sunxi.rst
    → نظرة عامة على Allwinner sunxi SoCs في kernel
```

**الـ Kconfig entry:**

```
drivers/net/ethernet/stmicro/stmmac/Kconfig
    → CONFIG_DWMAC_SUNXI — Allwinner A20/A31 GMAC support
```

---

### Kernel Commits المهمة

**الـ commits اللي أسست الـ driver:**

| الـ Commit | الوصف |
|-----------|-------|
| `[3.14]` Platform glue for sunxi | أول merge لـ `dwmac-sunxi.c` في mainline |
| `[3.15]` DT patches for A20 boards | إضافة DT nodes للـ boards المختلفة |

**للبحث في git log:**

```bash
git log --oneline -- drivers/net/ethernet/stmicro/stmmac/dwmac-sunxi.c
git log --oneline -- Documentation/devicetree/bindings/net/allwinner,sun7i-a20-gmac.yaml
```

**الـ author الأصلي:** Chen-Yu Tsai `<wens@csie.org>` — بيساهم في Allwinner mainlining.

---

### نقاشات Mailing List

- **[LWN.net — Allwinner A20 GMAC glue layer patch thread](https://lwn.net/Articles/580138/)** — النقاش الأصلي على `netdev` و`linux-arm-kernel` mailing lists
- **[Patchwork — stmmac: add a generic dwmac driver](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1431598266-25736-5-git-send-email-manabian@gmail.com/)** — مثال على كيفية إضافة glue layer generically
- **[linux-wireless: PATCH v3 01/11 — sun8i H6 EMAC support](https://www.spinics.net/lists/linux-wireless/msg184967.html)** — تطوير الجيل التاني (sun8i)

**للبحث في الـ mailing list مباشرة:**

```
https://lore.kernel.org/netdev/?q=dwmac-sunxi
https://lore.kernel.org/linux-arm-kernel/?q=allwinner+gmac
```

---

### Kernelnewbies.org — تتبع التغييرات عبر الإصدارات

الـ stmmac بيظهر في changelog كل إصدار تقريبًا — الأهم:

| الصفحة | ما يخص stmmac/sunxi |
|--------|---------------------|
| [Linux_5.6](https://kernelnewbies.org/Linux_5.6) | TSN support بـ TAPRIO API |
| [Linux_6.1](https://kernelnewbies.org/Linux_6.1) | تحسينات EEE و PCS |
| [Linux_6.5](https://kernelnewbies.org/Linux_6.5) | تحديثات stmmac متعددة |
| [Linux_6.8](https://kernelnewbies.org/Linux_6.8) | EST implementation، HW VLAN stripping |
| [Linux_6.12](https://kernelnewbies.org/Linux_6.12) | Loongson platform support |
| [Linux_6.15](https://kernelnewbies.org/Linux_6.15) | Tx metadata launch time، EEE work |
| [Linux_6.17](https://kernelnewbies.org/Linux_6.17) | PCS conversion to phylink |

---

### eLinux.org

- **[Hack A10 devices](https://elinux.org/Hack_A10_devices)** — معلومات عن Allwinner A10 hardware، مرجع للفهم التاريخي
- **[linux-sunxi.org/Ethernet](https://linux-sunxi.org/Ethernet)** — الـ wiki الأهم عمليًا لـ Allwinner Ethernet — فيه معلومات عن clock mux، PHY wiring، وتاريخ الـ mainlining

---

### مراجع إضافية على الويب

- **[Linux Kernel Driver DataBase — CONFIG_DWMAC_SUNXI](https://cateee.net/lkddb/web-lkddb/DWMAC_SUNXI.html)** — الـ Kconfig option، versions متاحة فيها الـ driver
- **[allwinner,sun7i-a20-gmac.txt — kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/net/allwinner,sun7i-a20-gmac.txt)** — النسخة القديمة (txt) من الـ DT binding
- **[ARM Allwinner SoCs — Linux Kernel Documentation](https://docs.kernel.org/5.10/arm/sunxi.html)** — نظرة عامة على sunxi ecosystem في kernel
- **[linux-sunxi/linux-sunxi — GitHub](https://github.com/linux-sunxi/linux-sunxi)** — الـ out-of-tree kernel للـ Allwinner SoCs (تاريخي)
- **[dwmac-sun8i.c — torvalds/linux GitHub](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/stmicro/stmmac/dwmac-sun8i.c)** — الجيل التاني من الـ driver للمقارنة

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)
- **Chapter 12:** PCI Drivers — فهم `platform_device` و `probe/remove`
- **Chapter 14:** The Linux Device Model — `of_match_table`، `devm_*` APIs
- متاح مجانًا: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **Chapter 5:** System Calls — فهم kernel/userspace boundary
- **Chapter 17:** Devices and Modules — `module_platform_driver`، `MODULE_DEVICE_TABLE`

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **Chapter 15:** Networking — Ethernet drivers في embedded context
- **Chapter 16:** Kernel Debugging — مفيد لـ debugging network drivers

#### Understanding Linux Network Internals — Christian Benvenuti
- أعمق مرجع لـ Linux networking stack
- **Chapter 8:** Device Registration — `net_device` lifecycle
- **Chapter 11:** Frame Reception — من الـ hardware interrupt لـ socket buffer

---

### Search Terms للبحث عن معلومات أكثر

```
# للبحث في kernel mailing lists
site:lore.kernel.org dwmac-sunxi
site:lore.kernel.org "allwinner" "gmac"
site:lwn.net stmmac glue layer

# للبحث في kernel git
git log --all --grep="dwmac-sunxi" --oneline
git log --all --grep="sun7i.*gmac" --oneline
git log --all --grep="allwinner.*stmmac" --oneline

# Google queries مفيدة
"dwmac-sunxi" site:github.com
"stmmac_platform" kernel documentation
"plat_stmmacenet_data" init callback
"phy_interface_mode_is_rgmii" kernel
allwinner A20 GMAC "clock mux" linux

# للبحث في kernel source مباشرة
grep -r "dwmac-sunxi" Documentation/
grep -r "allwinner,sun7i-a20-gmac" arch/arm/
```

---

### مسارات الـ Source Code ذات الصلة

```
# الـ glue layer نفسه
drivers/net/ethernet/stmicro/stmmac/dwmac-sunxi.c

# الجيل التاني (sun8i) — للمقارنة
drivers/net/ethernet/stmicro/stmmac/dwmac-sun8i.c

# الـ stmmac core
drivers/net/ethernet/stmicro/stmmac/stmmac_main.c
drivers/net/ethernet/stmicro/stmmac/stmmac_platform.c

# الـ platform data structures
include/linux/stmmac.h

# الـ DT bindings
Documentation/devicetree/bindings/net/allwinner,sun7i-a20-gmac.yaml
Documentation/devicetree/bindings/clock/allwinner,sun7i-a20-gmac-clk.yaml

# الـ Kconfig
drivers/net/ethernet/stmicro/stmmac/Kconfig
    → CONFIG_DWMAC_SUNXI
```
## Phase 8: Writing simple module

### الفكرة

**`sun7i_gmac_init`** هي الدالة اللي بتتنفذ كل مرة الـ GMAC driver بيتشغل على أجهزة Allwinner A20 — بتفعّل الـ regulator وبتضبط الـ TX clock rate حسب الـ PHY interface. هنستخدم **kprobe** عليها عشان نشوف بالظبط امتى الـ GMAC بيتهيأ وإيه الـ interface اللي بيشتغل بيها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_sunxi_gmac.c
 *
 * Hook sun7i_gmac_init() to log GMAC init events on Allwinner A20.
 * Uses kprobe so no driver source modification needed.
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit       */
#include <linux/kprobes.h>      /* kprobe struct and registration API       */
#include <linux/device.h>       /* struct device, dev_name()                */
#include <linux/phy.h>          /* phy_interface_t, PHY_INTERFACE_MODE_*    */
#include <linux/printk.h>       /* pr_info()                                */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe على sun7i_gmac_init لمراقبة تهيئة GMAC على Allwinner A20");

/* ------------------------------------------------------------------ */
/*  pre-handler: بيتنفذ قبل ما sun7i_gmac_init نفسها تشتغل            */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * sun7i_gmac_init(struct device *dev, void *priv)
     *
     * On x86-64:  rdi = dev,  rsi = priv
     * On ARM64:   x0  = dev,  x1  = priv
     * On ARM32:   r0  = dev,  r1  = priv
     *
     * We read 'dev' only — safe to dereference since kprobe fires
     * with interrupts still enabled at function entry.
     */
#if defined(CONFIG_X86_64)
    struct device *dev = (struct device *)regs->di;
#elif defined(CONFIG_ARM64)
    struct device *dev = (struct device *)regs->regs[0];
#elif defined(CONFIG_ARM)
    struct device *dev = (struct device *)regs->ARM_r0;
#else
    struct device *dev = NULL;
#endif

    if (dev)
        pr_info("[sunxi-gmac-probe] sun7i_gmac_init called — device: %s\n",
                dev_name(dev));
    else
        pr_info("[sunxi-gmac-probe] sun7i_gmac_init called — dev ptr unavailable on this arch\n");

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/*  post-handler: بيتنفذ بعد ما sun7i_gmac_init ترجع بنجاح            */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * On x86-64 the return value lives in rax after the function returns.
     * We log it to know whether init succeeded or failed.
     */
#if defined(CONFIG_X86_64)
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#elif defined(CONFIG_ARM)
    long ret = (long)regs->ARM_r0;
#else
    long ret = 0;
#endif

    pr_info("[sunxi-gmac-probe] sun7i_gmac_init returned: %ld (%s)\n",
            ret, ret == 0 ? "OK" : "ERROR");
}

/* ------------------------------------------------------------------ */
/*  kprobe struct — بيربط الـ handlers بالدالة المستهدفة               */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "sun7i_gmac_init", /* اسم الـ symbol زي ما هو في /proc/kallsyms */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init sunxi_gmac_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[sunxi-gmac-probe] register_kprobe failed: %d\n", ret);
        pr_err("[sunxi-gmac-probe] تأكد إن CONFIG_KPROBES=y وإن dwmac-sunxi.ko موجود\n");
        return ret;
    }

    pr_info("[sunxi-gmac-probe] kprobe registered on sun7i_gmac_init @ %p\n",
            kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit sunxi_gmac_kprobe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[sunxi-gmac-probe] kprobe unregistered\n");
}

module_init(sunxi_gmac_kprobe_init);
module_exit(sunxi_gmac_kprobe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module: `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | يعرّف `struct kprobe` و`register_kprobe` / `unregister_kprobe` |
| `linux/device.h` | يعرّف `struct device` و`dev_name()` عشان نطبع اسم الجهاز |
| `linux/phy.h` | يعرّف `phy_interface_t` اللي بيستخدمها الـ driver لتحديد نوع الـ interface |
| `linux/printk.h` | `pr_info()` / `pr_err()` للـ logging |

---

#### الـ pre_handler

الـ pre-handler بيتنفذ **قبل** أول instruction في `sun7i_gmac_init`، فالـ arguments لسه موجودة في الـ registers. بنقرأ `dev` (أول argument) عشان نطبع اسم الجهاز اللي بيتهيأ، وده بيساعدنا نعرف أي platform device الـ GMAC اتعمله init.

الكود ببيستخدم `#if defined(CONFIG_X86_64)` إلخ عشان قراءة الـ registers مختلفة على كل معمارية — ARM32 بيحط الـ arguments في `r0..r3`، ARM64 في `x0..x7`، وx86-64 في `rdi, rsi, rdx, rcx`.

---

#### الـ post_handler

بيتنفذ **بعد** ما الدالة ترجع ولقرأها الـ return value من الـ register المناسب. لو الـ init فشل (قيمة سالبة) هيظهر `ERROR` في الـ log، وده مفيد جداً في الـ debugging لأن `sun7i_gmac_init` بترجع أخطاء الـ regulator وأخطاء الـ clock prepare.

---

#### الـ kprobe struct

`symbol_name` بيحدد الدالة بالاسم بدل العنوان المطلق — الـ kernel بيبحث عنها في `/proc/kallsyms` وقت الـ register. الشرط الوحيد إن الـ symbol يكون exported أو على الأقل موجود في الـ kallsyms (وده صح لأن `sun7i_gmac_init` جزء من module محمّل).

---

#### الـ module_init و module_exit

`register_kprobe` بتزرع breakpoint على الدالة المستهدفة في الـ kernel memory. في الـ exit، **لازم** نعمل `unregister_kprobe` قبل ما الـ module يتفك من الـ memory — لو متعملش، الـ kernel هيحاول يستدعي handler pointer على عنوان مش موجود وهيحصل kernel panic.

---

### Makefile وطريقة التشغيل

```makefile
# Makefile
obj-m += kprobe_sunxi_gmac.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# تحميل الـ module
sudo insmod kprobe_sunxi_gmac.ko

# مراقبة الـ log (المفروض يظهر لما يتعمل probe على جهاز Allwinner A20)
sudo dmesg -w | grep sunxi-gmac-probe

# تفريغ الـ module
sudo rmmod kprobe_sunxi_gmac
```

---

### مثال على الـ output المتوقع

```
[sunxi-gmac-probe] kprobe registered on sun7i_gmac_init @ ffffffffc0a12340
[sunxi-gmac-probe] sun7i_gmac_init called — device: 1c50000.ethernet
[sunxi-gmac-probe] sun7i_gmac_init returned: 0 (OK)
```

**`1c50000.ethernet`** هو اسم الـ platform device على الـ A20 SoC — الرقم ده هو الـ base address للـ GMAC peripheral في الـ device tree.
