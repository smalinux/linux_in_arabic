## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**الـ** `sun4i-emac.c` هو driver الـ Ethernet لشرائح Allwinner A10 (كود اسمها sun4i) — وهي شرائح ARM رخيصة استخدمتها لوحات مثل Cubieboard وOlinuXino في حوالي 2012-2013. الـ EMAC اختصار لـ **Ethernet Media Access Controller**، يعني المتحكم اللي بيربط الـ CPU بكابل الشبكة.

---

### القصة من البداية — تخيل المشكلة

تخيل عندك لوحة ARM رخيصة (زي Raspberry Pi بس أرخص). اللوحة دي فيها chip اسمها Allwinner A10. جوا الـ chip دي في hardware block اسمه EMAC — ده زي "بوابة الشبكة" — ومهمته إنه يبعت ويستقبل packets على شبكة الإيثرنت.

المشكلة: الـ Linux kernel ما يعرفش يتكلم مع EMAC ده تلقائيًا. لازم حد يكتب driver — يعني برنامج بيقول للـ kernel "لو عايز تبعت packet، اكتب في الـ register ده، ولو جالك interrupt معناه وصل packet جديد، اقراه من الـ FIFO ده."

الـ `sun4i-emac.c` هو الـ driver ده بالظبط.

---

### الصورة الكبيرة — إيه اللي بيحصل؟

```
[ مكدس الشبكة في الـ kernel ]
         ↕
[ sun4i-emac driver ]   ← الملف ده
         ↕
[ EMAC hardware registers ]  ← mapped في الـ memory
         ↕
[ PHY chip ]  ← عبر MDIO bus
         ↕
[ كابل الإيثرنت ]
```

الـ driver بيعمل حاجات تلاتة أساسية:

1. **الإعداد (Probe/Init):** لما الـ kernel يلاقي الـ hardware في الـ Device Tree، بيستدعي `emac_probe()` — دي بتحجز الـ memory، بتفعّل الـ clock، بتطالب بالـ SRAM، وبتسجّل نفسها كـ network device.

2. **الإرسال (TX):** لما برنامج عندك يعمل `send()` على socket، الـ kernel بيوصّل الـ packet لـ `emac_start_xmit()` — دي بتكتب البيانات في الـ TX FIFO الموجود جوا الـ chip مباشرة عبر `writel`.

3. **الاستقبال (RX):** لما الـ chip تستقبل packet من الكابل، بترفع interrupt. الـ handler `emac_interrupt()` بيستدعي `emac_rx()` اللي بتقرا البيانات من الـ RX FIFO وبتدّيها للـ kernel عبر `netif_rx()`.

---

### تفاصيل مهمة تميّز الـ Driver ده

#### الـ SRAM المُشارَك
الـ A10 chip عندها **SRAM** صغيرة مُشتركة بين أجهزة متعددة. الـ EMAC محتاج جزء منها كـ RX buffer. عشان كده الـ driver لازم يطلب الـ SRAM أول ما يشتغل عبر `sunxi_sram_claim()` ويحررها لما يوقف عبر `sunxi_sram_release()`. لو ما عملش كده، الاستقبال مش هيشتغل.

#### الـ TX FIFOs الاتنين
الـ EMAC عنده **FIFO للإرسال اتنين (channel 0 و channel 1)**. الـ driver بيستخدمهم بشكل pipeline: يبعت أول packet في channel 0، والتانية في channel 1. لو الاتنين ممليين، بيوقف الـ queue عبر `netif_stop_queue()` ويستنى interrupt إن الـ hardware خلّص إرسال.

#### الـ DMA للاستقبال الكبير
لما الـ packet كبيرة (أكبر من الـ MTU)، الـ driver بيحاول يستخدم **DMA** بدل إنه يقرا الـ bytes واحدة واحدة من الـ CPU. الـ DMA بيعمل transfer من الـ RX FIFO في الـ EMAC مباشرة للـ RAM، وبعدين `emac_dma_done_callback()` بتتنادى لما يخلص.

#### الـ PHY و MDIO
الـ EMAC نفسه بيتعامل مع الـ MAC layer بس. أما الـ PHY (اللي بيتحكم في الإشارة الفيزيائية على الكابل) فده chip منفصل. الـ driver بيتواصل معاه عبر **MDIO bus** (بروتوكول تحكم بسيط). الـ `emac_mdio_probe()` بتربط الـ MAC بالـ PHY، و`emac_handle_link_change()` بتتنادى أوتوماتيكيًا لما يتغير الـ link (مثلًا حد وصّل كابل أو فصله).

#### الـ EMAC_UNDOCUMENTED_MAGIC
في `emac_rx()` في check غريب:
```c
if (reg_val != EMAC_UNDOCUMENTED_MAGIC)  /* 0x0143414d */
```
ده magic number مش موثّق في أي datasheet رسمي. الـ hardware بيحط الـ value دي في أول الـ RX FIFO عشان يدل إن في packet صالحة. لو ما لقاش الـ magic ده، يعني الـ FIFO فيه data فسدانة فبيعمل flush وبيرجع.

---

### ليه الـ Driver ده مهم؟

- بدونه، اللوحات المبنية على A10 (زي Cubieboard) ما تقدرش تتصل بالشبكة.
- الـ driver بيخلّي الـ Linux networking stack الكاملة (TCP/IP, sockets, etc.) تشتغل على hardware زهيد الثمن.
- بيديك نموذج واضح لـ **platform driver** بسيط نسبيًا بدون complexity زيادة.

---

### الملفات المكوّنة للـ Subsystem

| الملف | الدور |
|-------|-------|
| `drivers/net/ethernet/allwinner/sun4i-emac.c` | الـ driver الأساسي — الملف ده |
| `drivers/net/ethernet/allwinner/sun4i-emac.h` | تعريفات الـ registers وقيم الـ bits |
| `drivers/net/ethernet/allwinner/Kconfig` | إعدادات الـ build (CONFIG_SUN4I_EMAC) |
| `drivers/net/ethernet/allwinner/Makefile` | قواعد الـ compilation |
| `drivers/net/mdio/mdio-sun4i.c` | driver الـ MDIO bus الخاص بـ Allwinner (للتواصل مع الـ PHY) |
| `drivers/soc/sunxi/sunxi_sram.c` | إدارة الـ SRAM المشتركة في شرائح Allwinner |
| `include/linux/soc/sunxi/sunxi_sram.h` | header الـ SRAM API |

---

### ملفات تانية مهمة تعرفها

- **`include/linux/netdevice.h`** — تعريف `struct net_device` وكل الـ ops الأساسية للـ network driver.
- **`include/linux/phy.h`** — الـ PHY abstraction layer، بيعرّف `struct phy_device` وـ `phy_connect`.
- **`include/linux/dmaengine.h`** — واجهة الـ DMA engine اللي الـ driver بيستخدمها لاستقبال الـ packets الكبيرة.

---

### الـ Subsystem اللي ينتمي ليه

الملف ده جزء من **Network Device Drivers subsystem** في الـ Linux kernel، تحت تصنيف **Ethernet drivers** لـ vendor اسمه **Allwinner Technology**. الـ SoC المستهدف هو **Allwinner A10 (sun4i)** المستخدم في لوحات ARM منخفضة التكلفة.
## Phase 2: شرح الـ Linux Network Device (netdev) Framework

---

### المشكلة اللي الـ Framework ده بيحلها

تخيل إن عندك عشرات الـ Ethernet controllers مختلفة — Allwinner EMAC، Intel e1000، Broadcom BCM54xx، Realtek r8169. كل واحد فيهم ليه registers مختلفة، FIFOs مختلفة، وطريقة إرسال واستقبال مختلفة. فوق ده، الـ TCP/IP stack فوق (الـ socket layer، الـ routing table، الـ qdisc scheduler) محتاج يتكلم مع "network interface" واحدة موحدة بدون ما يعرف أي hardware بيشتغل تحته.

**المشكلة الجوهرية:** إزاي تعزل الـ hardware-specific code عن الـ generic networking stack بطريقة نظيفة وموحدة؟

---

### الحل — الـ netdev Framework

الـ kernel بيحل المشكلة دي عن طريق **abstraction layer** محوره struct **`net_device`** — ده هو "العقد" اللي بين الـ driver والـ kernel networking stack.

الـ driver بيعمل الآتي:
1. يخصص `net_device` عن طريق `alloc_etherdev()`
2. يملي `net_device_ops` — جدول function pointers (زي vtable في C++)
3. يسجل الـ device عند الـ kernel بـ `register_netdev()`
4. الـ kernel بيتعامل مع الباقي — scheduling، queueing، stats، sysfs، إلخ.

---

### الـ Big Picture Architecture

```
  User Space
  ──────────────────────────────────────────────────
     socket()  send()  recv()  ioctl()  ethtool
         │         │       │       │        │
  ──────────────────────────────────────────────────
  Kernel: VFS / Socket Layer (AF_INET, SOCK_STREAM)
         │
         ▼
  ┌─────────────────────────────────────┐
  │       TCP / UDP / IP Stack          │
  │  (routing, fragmentation, checksum) │
  └──────────────┬──────────────────────┘
                 │  sk_buff (skb) ▼ up / ▲ down
  ┌──────────────▼──────────────────────┐
  │       Traffic Control (qdisc)       │  ← pfifo_fast, tbf, etc.
  │   dev_queue_xmit() / netif_rx()     │
  └──────────────┬──────────────────────┘
                 │
  ══════════════════════════════════════════  ← netdev framework boundary
                 │
  ┌──────────────▼──────────────────────────────────────┐
  │             struct net_device                        │
  │  .netdev_ops → { .ndo_start_xmit, .ndo_open, ... }  │
  │  .ethtool_ops → { .get_link_ksettings, ... }         │
  │  .stats  →  rx_packets, tx_bytes, rx_errors, ...     │
  └──────────────┬──────────────────────────────────────┘
                 │   (driver owns this layer)
  ┌──────────────▼──────────────────────────────────────┐
  │         sun4i-emac driver                            │
  │  emac_open / emac_stop / emac_start_xmit / emac_rx  │
  │  emac_interrupt / emac_timeout                       │
  │  ┌────────────────┐   ┌──────────────────────────┐   │
  │  │  PHY Subsystem │   │  DMA Engine (dmaengine)  │   │
  │  │  (phylib)      │   │  sun4i DMA controller    │   │
  │  │  phy_connect() │   │  dma_request_chan()       │   │
  │  └────────────────┘   └──────────────────────────┘   │
  └──────────────┬──────────────────────────────────────┘
                 │
  ┌──────────────▼──────────────────────────────────────┐
  │         Allwinner A10 EMAC Hardware                  │
  │  TX FIFO (ch0 + ch1) │ RX FIFO │ MAC │ MII/RMII bus │
  └─────────────────────────────────────────────────────┘
```

---

### التشبيه الواقعي — مطعم بيdelivery

تخيل مطعم بيشتغل بـ delivery:

| عنصر في المطعم | المقابل في الـ kernel |
|---|---|
| **نظام الطلبات الموحد** (app أو تليفون) | Socket API — نفس الـ interface لأي network device |
| **الـ dispatcher** اللي بيوزع الطلبات | الـ qdisc scheduler (pfifo، tbf، etc.) |
| **الـ "order ticket"** اللي بييجي للمطبخ | الـ `sk_buff` (packet) |
| **قائمة الخدمات** اللي المطبخ لازم يقدمها | `net_device_ops` (vtable) |
| **المطبخ نفسه** (بيطبخ بطريقته الخاصة) | الـ driver (sun4i-emac) |
| **موردين المواد الخام** | الـ hardware (EMAC FIFO registers) |
| **سايق الـ delivery** اللي بيتكلم مع الطريق | الـ PHY chip (بيتكلم مع الكابل الفعلي) |
| **المدير اللي بيشوف إيه الطريق متاح** | الـ phylib — بيراقب link state |
| **الـ conveyor belt** في المطبخ | الـ DMA engine — ينقل البيانات بدون CPU |

**التعمق في التشبيه:**

- لما الـ app تعمل `send()` — ده زي الزبون بيطلب. الـ kernel بيعمل `sk_buff`، بيمرره للـ qdisc، واللي بيتصل بـ `ndo_start_xmit()` — زي ما الـ dispatcher بيبعت الطلب للمطبخ.
- الـ `ndo_start_xmit` في sun4i هو `emac_start_xmit()` — بيحط الـ packet في TX FIFO الـ hardware. ده زي الطباخ بيحط الأكل في الكيس.
- الـ PHY chip هو مين بيقرر "هل فيه زبون على الطريق؟" (link up/down). الـ phylib هو الـ framework اللي بيتكلم مع الـ PHY عبر MDIO bus، ولما الـ link يتغير بيعمل callback لـ `emac_handle_link_change()` — زي ما السايق بييتصل بالمطبخ يقول "الطريق فيه زحمة، غيّر السرعة."
- الـ DMA engine هو الـ conveyor belt — لما الـ packet كبير (>= MTU)، الـ driver مش بيـ copy الداتا يدويًا، بيخلي الـ DMA hardware ينقل من FIFO لـ RAM بشكل مستقل.

---

### الـ Core Abstraction — `struct net_device`

ده المحور الأساسي. كل network interface في الـ kernel هو `net_device`. الـ driver بيخصصه كالتالي:

```c
/* alloc_etherdev بيخصص net_device + private data بعده مباشرة في الميموري */
ndev = alloc_etherdev(sizeof(struct emac_board_info));

/* db = private data بتاع الـ driver، مخزنة مباشرة بعد net_device في الميموري */
db = netdev_priv(ndev);
```

**العلاقة بين الـ structs:**

```
  Memory Layout بعد alloc_etherdev():
  ┌────────────────────────────────┐  ◄─ ndev
  │         struct net_device      │
  │   .name = "eth0"               │
  │   .irq  = 42                   │
  │   .netdev_ops = &emac_ops      │
  │   .ethtool_ops = &emac_ethtool │
  │   .phydev → phy_device         │
  │   .stats  → net_device_stats   │
  ├────────────────────────────────┤  ◄─ db = netdev_priv(ndev)
  │      struct emac_board_info    │
  │   .membase → MMIO registers    │
  │   .clk     → clock handle      │
  │   .lock    → spinlock          │
  │   .tx_fifo_stat                │
  │   .rx_chan → DMA channel       │
  │   .phy_node → DT node          │
  │   .link / .speed / .duplex     │
  └────────────────────────────────┘
```

الـ `netdev_priv()` ببساطة بيرجع pointer بعد الـ `net_device` struct مباشرة. ده design مقصود — كل حاجة تخص الـ hardware الـ driver بيحطها هناك.

---

### الـ `net_device_ops` — الـ vtable

```c
static const struct net_device_ops emac_netdev_ops = {
    .ndo_open        = emac_open,       /* ifconfig eth0 up */
    .ndo_stop        = emac_stop,       /* ifconfig eth0 down */
    .ndo_start_xmit  = emac_start_xmit, /* send packet */
    .ndo_tx_timeout  = emac_timeout,    /* watchdog expired */
    .ndo_set_rx_mode = emac_set_rx_mode,/* promisc / multicast */
    .ndo_eth_ioctl   = phy_do_ioctl_running, /* ethtool/mii-tool */
    .ndo_set_mac_address = emac_set_mac_address,
};
```

الـ kernel بيستدعي الـ ops دي بشكل uniform بغض النظر عن الـ hardware. الـ `emac_start_xmit` مثلًا:

```c
static netdev_tx_t emac_start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct emac_board_info *db = netdev_priv(dev);
    /* بيختار TX channel (0 أو 1) — الـ EMAC عنده FIFO مزدوج */
    channel = db->tx_fifo_stat & 3;
    if (channel == 3)
        return NETDEV_TX_BUSY; /* كل الـ channels مشغولة — الـ kernel هيعيد المحاولة */

    /* كتابة الـ packet في TX FIFO مباشرة عبر MMIO */
    emac_outblk_32bit(db->membase + EMAC_TX_IO_DATA_REG, skb->data, skb->len);

    /* لو الاتنين channels امتلوا — وقف الـ queue */
    if ((db->tx_fifo_stat & 3) == 3)
        netif_stop_queue(dev);

    dev_consume_skb_any(skb); /* free الـ skb — الـ driver أخد ownership */
    return NETDEV_TX_OK;
}
```

---

### الـ PHY Subsystem (phylib) — subsystem تاني مهم

**الـ phylib** هو الـ framework اللي بيتحكم في الـ Physical Layer chip (الـ PHY) — الشريحة اللي بتتكلم مع الكابل فعليًا.

> **الـ MDIO bus:** serial bus بسيط (2 wires: MDC + MDIO) بيستخدمه الـ MAC للتحدث مع الـ PHY — اقرأ registers الـ PHY (speed، link status، autoneg).

**الـ flow في sun4i-emac:**

```
  emac_open()
      │
      ▼
  emac_mdio_probe()
      │  of_phy_connect(ndev, phy_node, emac_handle_link_change, ...)
      │  ← بيربط الـ MAC بالـ PHY من الـ Device Tree
      ▼
  phy_start(dev->phydev)
      │  phylib بيبدأ polling/interrupt على الـ PHY
      │  كل ما الـ link يتغير → callback
      ▼
  emac_handle_link_change()
      │  بيقرأ phydev->speed, phydev->duplex
      │  بيكتب في MAC registers
      ▼
  emac_update_speed()   →  EMAC_MAC_SUPP_REG  (100M bit)
  emac_update_duplex()  →  EMAC_MAC_CTL1_REG  (full/half duplex)
```

الـ driver هنا **ما بيعملش polling على PHY بنفسه** — الـ phylib بيتكفل بده. الـ driver بس بيقول "لما يحصل تغيير، نادّيلي."

---

### الـ DMA Engine Framework — subsystem تالت

**الـ DMAengine** هو الـ kernel framework الموحد للتحدث مع DMA controllers. بدل ما كل driver يكتب DMA code خاص بيه، بيستخدم API موحد.

> **الـ DMA هنا:** الـ sun4i EMAC عنده RX FIFO في الـ MMIO space. لما packet كبيرة تيجي، بدل ما الـ CPU يقرأ كل word بـ `readsl()`، بيطلب من الـ DMA controller ينقل الداتا من الـ FIFO لـ RAM تلقائيًا.

**الـ flow في sun4i-emac:**

```c
static int emac_dma_inblk_32bit(struct emac_board_info *db,
                                struct sk_buff *skb, void *rdptr, int count)
{
    /* 1. map الـ buffer في RAM للـ DMA */
    rxbuf = dma_map_single(db->dev, rdptr, count, DMA_FROM_DEVICE);

    /* 2. بعد ما الـ DMAengine يتكلم مع السخ DMA controller
     *    بيجيب descriptor يصف العملية: من فين؟ لفين؟ كام؟ */
    desc = dmaengine_prep_slave_single(db->rx_chan, rxbuf, count,
                                       DMA_DEV_TO_MEM,
                                       DMA_PREP_INTERRUPT | DMA_CTRL_ACK);

    /* 3. register callback — هيتشغل لما الـ DMA يخلص */
    desc->callback = emac_dma_done_callback;
    desc->callback_param = req;

    /* 4. submit وابدأ */
    cookie = dmaengine_submit(desc);
    dma_async_issue_pending(db->rx_chan);
}
```

لما الـ DMA يخلص، بيتنادى `emac_dma_done_callback()`:

```c
static void emac_dma_done_callback(void *arg)
{
    /* unmap الـ buffer — CPU يقدر يقرأ الداتا دلوقتي */
    dma_unmap_single(db->dev, req->rxbuf, rxlen, DMA_FROM_DEVICE);

    /* بعد كده تحديد الـ protocol (IPv4؟ ARP؟) وإرسال للـ stack */
    skb->protocol = eth_type_trans(skb, dev);
    netif_rx(skb);  /* ← بيدفع الـ packet للـ IP stack */
}
```

**الـ DMA channel configuration:**

```c
/* في emac_configure_dma() */
conf.direction = DMA_DEV_TO_MEM;          /* من hardware FIFO لـ RAM */
conf.src_addr  = db->emac_rx_fifo;        /* عنوان FIFO الـ فيزيائي */
conf.src_addr_width = DMA_SLAVE_BUSWIDTH_4_BYTES; /* 32-bit transfers */
conf.src_maxburst = 4;                    /* burst of 4 words */
dmaengine_slave_config(db->rx_chan, &conf);
```

---

### الـ `sk_buff` — الـ Packet Container

> **prerequisite:** الـ `sk_buff` هو الـ struct اللي الـ kernel بيستخدمه لتمثيل كل packet في كل طبقات الـ networking stack. بيحمل الداتا + metadata (protocol، timestamps، checksum flags، etc.)

في sun4i-emac، الـ RX path:

```
  Hardware RX FIFO
        │
        │  emac_rx() بتقرأ الـ header الأول
        │  EMAC_UNDOCUMENTED_MAGIC ← magic number لازم يكون موجود
        │
        ▼
  netdev_alloc_skb(dev, rxlen + 4)  ← خصص skb جديد
        │
        │  skb_reserve(skb, 2)     ← align الداتا على 4 bytes
        │  rdptr = skb_put(skb, rxlen - 4)  ← reserve مكان للداتا
        │
        ▼
  emac_inblk_32bit(EMAC_RX_IO_DATA_REG, rdptr, rxlen)
  /* أو عبر DMA لو الـ packet كبيرة */
        │
        ▼
  skb->protocol = eth_type_trans(skb, dev)
  /* بيشيل الـ Ethernet header وبيحدد الـ protocol (0x0800=IPv4, 0x0806=ARP) */
        │
        ▼
  netif_rx(skb)
  /* بيدفع الـ packet لـ IP stack عبر software interrupt (softirq) */
```

---

### الـ Interrupt Handler — قلب الـ Driver

```c
static irqreturn_t emac_interrupt(int irq, void *dev_id)
{
    spin_lock(&db->lock);  /* حماية من re-entrancy */

    /* Disable all interrupts أثناء المعالجة — pattern شائع */
    writel(0, db->membase + EMAC_INT_CTL_REG);

    /* اقرأ وامسح الـ interrupt status register */
    int_status = readl(db->membase + EMAC_INT_STA_REG);
    writel(int_status, db->membase + EMAC_INT_STA_REG);

    /* RX interrupt */
    if ((int_status & 0x100) && (db->emacrx_completed_flag == 1)) {
        db->emacrx_completed_flag = 0;
        emac_rx(dev);  /* معالجة الـ packets المستلمة */
    }

    /* TX complete interrupt */
    if (int_status & EMAC_INT_STA_TX_COMPLETE)
        emac_tx_done(dev, db, int_status);

    spin_unlock(&db->lock);
    return IRQ_HANDLED;
}
```

**ملاحظة الـ `emacrx_completed_flag`:** الـ EMAC hardware عنده bug محتمل — لو الـ RX interrupt جه وفيه DMA لسه شغال، محتاج flag يمنع الـ re-entry. لو الـ DMA لسه شغال (`emacrx_completed_flag == 0`)، مش بيعيد enable الـ RX interrupt — الـ DMA callback هو اللي هيعمل ده لما يخلص.

---

### ما الـ Framework بيملكه vs ما بيفوضه للـ Driver

| المسؤولية | مين بيتكفل بيها |
|---|---|
| تخصيص `net_device` struct | **kernel** — `alloc_etherdev()` |
| تسجيل الـ interface في `/sys/class/net/` | **kernel** — `register_netdev()` |
| الـ qdisc scheduling (traffic shaping) | **kernel** |
| الـ ARP، IP routing، TCP/UDP | **kernel** |
| الـ ethtool stats aggregation | **kernel** (باستخدام ops الـ driver) |
| watchdog timer للـ TX timeout | **kernel** — بينادي `ndo_tx_timeout` |
| init الـ hardware registers | **driver** — `emac_powerup()`, `emac_setup()` |
| كتابة/قراءة الـ TX/RX FIFOs | **driver** — `emac_outblk_32bit()`, `emac_inblk_32bit()` |
| interrupt handling الـ hardware-specific | **driver** — `emac_interrupt()` |
| التحدث مع الـ PHY عبر MDIO | **driver** (باستخدام phylib API) |
| إدارة الـ clock (`clk_prepare_enable`) | **driver** |
| حجز الـ SRAM (`sunxi_sram_claim`) | **driver** |
| إعداد الـ DMA channel | **driver** (باستخدام dmaengine API) |
| تحديد الـ MAC address في Hardware | **driver** — `emac_set_mac_address()` |

---

### الـ Lifecycle — من probe لحد remove

```
platform_driver_register(emac_driver)
        │
        ▼  (kernel اكتشف compatible = "allwinner,sun4i-a10-emac" في DT)
emac_probe()
    ├── alloc_etherdev()          ← خصص net_device + emac_board_info
    ├── of_iomap()                ← map الـ MMIO registers
    ├── emac_configure_dma()      ← احجز DMA channel
    ├── devm_clk_get() + clk_prepare_enable()  ← فعّل الـ clock
    ├── sunxi_sram_claim()        ← احجز الـ SRAM (الـ EMAC محتاج SRAM خاص)
    ├── emac_powerup() + emac_reset()
    ├── register_netdev()         ← "eth0" ظهر في النظام
        │
        ▼  (المستخدم يعمل: ip link set eth0 up)
emac_open()
    ├── request_irq()
    ├── emac_reset() + emac_init_device()
    ├── emac_mdio_probe()         ← ربط MAC بالـ PHY
    ├── phy_start()               ← phylib بدأ مراقبة الـ link
    └── netif_start_queue()       ← الـ TX queue جاهزة

        [normal operation: TX via emac_start_xmit, RX via emac_interrupt → emac_rx]

        │  (المستخدم يعمل: ip link set eth0 down)
emac_stop()
    ├── netif_stop_queue()
    ├── phy_stop() + emac_mdio_remove()
    ├── emac_shutdown()
    └── free_irq()

emac_remove()
    ├── dmaengine_terminate_all() + dma_release_channel()
    ├── unregister_netdev()
    ├── sunxi_sram_release()
    ├── clk_disable_unprepare()
    └── free_netdev()
```

---

### لماذا الـ SRAM؟ — تفصيلة مهمة في Allwinner

الـ Allwinner A10 (sun4i) EMAC مش بيستخدم الـ DDR RAM مباشرة للـ RX/TX FIFOs — بيستخدم جزء من الـ on-chip SRAM. الـ SRAM ده مشترك بين أكتر من peripheral، فالـ driver لازم يطلب تخصيصه بـ `sunxi_sram_claim()` قبل ما يشغل الـ hardware.

ده سبب وجود `sunxi_sram.h` في الـ includes — مش موجود في drivers تانية.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags، Enums، وخيارات الـ Config — Cheatsheet

#### EMAC_CTL_REG (0x00) — Global Control

| Bit | Macro | المعنى |
|-----|-------|--------|
| 0 | `EMAC_CTL_RESET` | Reset الـ EMAC |
| 1 | `EMAC_CTL_TX_EN` | تفعيل Transmit |
| 2 | `EMAC_CTL_RX_EN` | تفعيل Receive |

#### EMAC_TX_MODE_REG (0x04) — TX Mode

| Bit | Macro | المعنى |
|-----|-------|--------|
| 0 | `EMAC_TX_MODE_ABORTED_FRAME_EN` | إرسال الـ frames المرفوضة |
| 1 | `EMAC_TX_MODE_DMA_EN` | تفعيل DMA على TX (غير مستخدم فعلياً) |

#### EMAC_RX_CTL_REG (0x3c) — RX Control

| Bit | Macro | المعنى |
|-----|-------|--------|
| 1 | `EMAC_RX_CTL_AUTO_DRQ_EN` | Auto data request |
| 2 | `EMAC_RX_CTL_DMA_EN` | تفعيل DMA للاستقبال |
| 3 | `EMAC_RX_CTL_FLUSH_FIFO` | مسح الـ RX FIFO |
| 4 | `EMAC_RX_CTL_PASS_ALL_EN` | Promiscuous mode |
| 5 | `EMAC_RX_CTL_PASS_CTL_EN` | قبول Control frames |
| 6 | `EMAC_RX_CTL_PASS_CRC_ERR_EN` | قبول frames بـ CRC خاطئ |
| 7 | `EMAC_RX_CTL_PASS_LEN_ERR_EN` | قبول frames بخطأ طول |
| 8 | `EMAC_RX_CTL_PASS_LEN_OOR_EN` | قبول frames خارج نطاق الطول |
| 16 | `EMAC_RX_CTL_ACCEPT_UNICAST_EN` | قبول Unicast |
| 17 | `EMAC_RX_CTL_DA_FILTER_EN` | فلترة الـ Destination Address |
| 20 | `EMAC_RX_CTL_ACCEPT_MULTICAST_EN` | قبول Multicast |
| 21 | `EMAC_RX_CTL_HASH_FILTER_EN` | Hash filter للـ multicast |
| 22 | `EMAC_RX_CTL_ACCEPT_BROADCAST_EN` | قبول Broadcast |
| 24 | `EMAC_RX_CTL_SA_FILTER_EN` | فلترة الـ Source Address |
| 25 | `EMAC_RX_CTL_SA_FILTER_INVERT_EN` | عكس فلتر الـ SA |

#### EMAC_INT_CTL_REG (0x54) — Interrupt Control

| Bit | Macro | المعنى |
|-----|-------|--------|
| 0 | `EMAC_INT_CTL_TX0_EN` | Interrupt لـ TX channel 0 |
| 1 | `EMAC_INT_CTL_TX1_EN` | Interrupt لـ TX channel 1 |
| 2 | `EMAC_INT_CTL_TX0_ABRT_EN` | Interrupt لـ TX0 abort |
| 3 | `EMAC_INT_CTL_TX1_ABRT_EN` | Interrupt لـ TX1 abort |
| 8 | `EMAC_INT_CTL_RX_EN` | Interrupt للاستقبال |

#### EMAC_INT_STA_REG (0x58) — Interrupt Status

| Bit | Macro | المعنى |
|-----|-------|--------|
| 0 | `EMAC_INT_STA_TX0_COMPLETE` | اكتمل إرسال TX0 |
| 1 | `EMAC_INT_STA_TX1_COMPLETE` | اكتمل إرسال TX1 |
| 2 | `EMAC_INT_STA_TX0_ABRT` | انقطع TX0 |
| 3 | `EMAC_INT_STA_TX1_ABRT` | انقطع TX1 |
| 8 | `EMAC_INT_STA_RX_COMPLETE` | استقبال مكتمل |

#### EMAC_MAC_CTL0_REG (0x5c) — MAC Control 0

| Bit | Macro | المعنى |
|-----|-------|--------|
| 2 | `EMAC_MAC_CTL0_RX_FLOW_CTL_EN` | Flow control على الاستقبال |
| 3 | `EMAC_MAC_CTL0_TX_FLOW_CTL_EN` | Flow control على الإرسال |
| 15 | `EMAC_MAC_CTL0_SOFT_RESET` | Soft reset للـ MAC |

#### EMAC_MAC_CTL1_REG (0x60) — MAC Control 1

| Bit | Macro | المعنى |
|-----|-------|--------|
| 0 | `EMAC_MAC_CTL1_DUPLEX_EN` | Full duplex |
| 1 | `EMAC_MAC_CTL1_LEN_CHECK_EN` | فحص طول الـ frame |
| 2 | `EMAC_MAC_CTL1_HUGE_FRAME_EN` | قبول frames كبيرة جداً |
| 3 | `EMAC_MAC_CTL1_DELAYED_CRC_EN` | CRC متأخر |
| 4 | `EMAC_MAC_CTL1_CRC_EN` | إضافة CRC تلقائياً |
| 5 | `EMAC_MAC_CTL1_PAD_EN` | Padding للـ frames الصغيرة |
| 6 | `EMAC_MAC_CTL1_PAD_CRC_EN` | Padding مع CRC |
| 7 | `EMAC_MAC_CTL1_AD_SHORT_FRAME_EN` | قبول short frames |
| 12 | `EMAC_MAC_CTL1_BACKOFF_DIS` | إلغاء Backoff |

#### RX Header Descriptor Macros

| Macro | المعنى |
|-------|--------|
| `EMAC_RX_IO_DATA_LEN(x)` | استخراج طول الـ frame (bits 15:0) |
| `EMAC_RX_IO_DATA_STATUS(x)` | استخراج status (bits 31:16) |
| `EMAC_RX_IO_DATA_STATUS_CRC_ERR` | bit 4 — خطأ CRC |
| `EMAC_RX_IO_DATA_STATUS_LEN_ERR` | bits 6:5 — خطأ طول |
| `EMAC_RX_IO_DATA_STATUS_OK` | bit 7 — الـ frame صحيح |

#### Magic Constants

| Macro | القيمة | المعنى |
|-------|--------|--------|
| `EMAC_UNDOCUMENTED_MAGIC` | `0x0143414d` | الـ header الأول المتوقع في RX FIFO |
| `EMAC_EEPROM_MAGIC` | `0x444d394b` | للـ EEPROM (غير مستخدم في هذا الكود) |

#### Module Parameters

| Parameter | Default | الوصف |
|-----------|---------|-------|
| `debug` | `-1` → `EMAC_DEFAULT_MSG_ENABLE=0` | أقنعة debug messages |
| `watchdog` | `5000ms` | TX timeout بالمللي ثانية |

---

### الـ Structs المهمة

#### `struct emac_board_info`

**الغرض:** البنية الرئيسية الخاصة بالـ driver — تُخزَّن كـ private data داخل الـ `net_device` عبر `netdev_priv()`. تحتوي على كل حالة الـ driver.

```c
struct emac_board_info {
    struct clk          *clk;           /* clock للـ EMAC من CCU */
    struct device       *dev;           /* الـ device الأصلي (للـ logging والـ DMA) */
    struct platform_device *pdev;       /* الـ platform device (للـ DMA resources) */
    spinlock_t          lock;           /* يحمي الـ register access وحالة الـ TX */
    void __iomem        *membase;       /* عنوان الـ MMIO بعد الـ iomap */
    u32                 msg_enable;     /* أقنعة debug (netif_msg_*) */
    struct net_device   *ndev;          /* back-pointer للـ net_device */
    u16                 tx_fifo_stat;   /* bitmap: bit0=channel0 busy, bit1=channel1 busy */

    int                 emacrx_completed_flag; /* 1=جاهز لاستقبال، 0=DMA شغال */

    struct device_node  *phy_node;      /* OF node للـ PHY */
    unsigned int        link;           /* حالة الـ link (1=up, 0=down) */
    unsigned int        speed;          /* السرعة الحالية (10 أو 100 Mbps) */
    unsigned int        duplex;         /* 0=half, 1=full */

    phy_interface_t     phy_interface;  /* نوع الواجهة: MII/RMII */
    struct dma_chan     *rx_chan;        /* قناة DMA للاستقبال (اختيارية) */
    phys_addr_t         emac_rx_fifo;   /* العنوان الفيزيائي لـ EMAC_RX_IO_DATA_REG */
};
```

**الحقل `tx_fifo_stat`:**
- الـ EMAC يملك **TX FIFO مزدوج** (channel 0 و channel 1)
- `bit 0 = 1` → channel 0 مشغول
- `bit 1 = 1` → channel 1 مشغول
- `tx_fifo_stat == 3` → كلاهما مشغول → `NETDEV_TX_BUSY`

**الحقل `emacrx_completed_flag`:**
- يُستخدم كـ mutex بسيط بين الـ ISR والـ DMA callback
- `1` = جاهز لاستقبال interrupt جديد
- `0` = DMA transfer قيد التشغيل، الـ RX interrupt معطّل مؤقتاً

---

#### `struct emac_dma_req`

**الغرض:** حاوية مؤقتة تُنشأ لكل عملية DMA RX، تحمل كل السياق اللازم لـ callback بعد اكتمال الـ DMA.

```c
struct emac_dma_req {
    struct emac_board_info          *db;    /* back-pointer للـ driver state */
    struct dma_async_tx_descriptor  *desc;  /* الـ DMA descriptor المُعدّ */
    struct sk_buff                  *skb;   /* الـ buffer المخصص للبيانات */
    dma_addr_t                      rxbuf;  /* العنوان الفيزيائي للـ skb data */
    int                             count;  /* عدد البايتات المراد نقلها */
};
```

**دورة حياتها:** تُنشأ في `emac_dma_inblk_32bit()` وتُحذف في `emac_dma_done_callback()`.

---

### علاقات الـ Structs

```
platform_device (pdev)
│
├─► device (pdev->dev) ◄────────────────────────────────┐
│                                                         │
└─► net_device (ndev)                                     │
     │  (private data: sizeof(emac_board_info))           │
     └─► emac_board_info (db) ──────────────────────────►┘
          │
          ├─► clk *clk              (CCU clock)
          ├─► void __iomem *membase (MMIO registers)
          ├─► net_device *ndev      (back-pointer)
          ├─► platform_device *pdev (back-pointer)
          ├─► device_node *phy_node (OF PHY node)
          ├─► dma_chan *rx_chan      (DMA engine channel)
          │    └─► [أثناء RX] emac_dma_req
          │              ├─► emac_board_info *db
          │              ├─► dma_async_tx_descriptor *desc
          │              └─► sk_buff *skb
          │
          └─► spinlock_t lock

net_device (ndev)
     ├─► phydev (phy_device) ─── يربطها of_phy_connect()
     │      ├─► link, speed, duplex
     │      └─► adjust_link → emac_handle_link_change()
     ├─► netdev_ops → emac_netdev_ops
     └─► ethtool_ops → emac_ethtool_ops
```

---

### رسم ASCII لعلاقة الـ Structs

```
┌──────────────────────────────────────────────────────────┐
│                   platform_device (pdev)                 │
│  .dev ──────────────────────────────────────────────┐    │
│  .driver_data ──► net_device (ndev)                 │    │
└──────────────────────────────────────────────────────┘    │
                          │                                 │
          alloc_etherdev() │                                 │
          sizeof(emac_board_info) as private                │
                          ▼                                 │
┌──────────────────────────────────────────────────────────┐│
│                  net_device (ndev)                       ││
│  .priv ──────► emac_board_info (db)                     ││
│  .phydev ────► phy_device                               ││
│  .netdev_ops ► emac_netdev_ops                          ││
│  .ethtool_ops► emac_ethtool_ops                         ││
└──────────────────────────────────────────────────────────┘│
                          │                                 │
               netdev_priv()                               │
                          ▼                                 │
┌──────────────────────────────────────────────────────────┐│
│               emac_board_info (db)                       ││
│  .dev ───────────────────────────────────────────────────┘│
│  .pdev ──────────────────────────────────────────────────┘
│  .ndev ──────────────────────────────────────────────────►(ndev above)
│  .clk ───────► clk (CCU)
│  .membase ───► MMIO region (0x01c0b000 typical)
│  .phy_node ──► device_node (OF)
│  .rx_chan ───► dma_chan
│  .lock ──────► spinlock_t
└──────────────────────────────────────────────────────────┘
         │ (أثناء DMA RX فقط)
         ▼
┌──────────────────────────────┐
│      emac_dma_req            │
│  .db ────► emac_board_info   │
│  .desc ──► dma_async_tx_desc │
│  .skb ───► sk_buff           │
│  .rxbuf ─► dma_addr_t        │
│  .count ─► int               │
└──────────────────────────────┘
```

---

### دورة حياة الـ Driver — Lifecycle Diagram

```
الـ Kernel يكتشف الـ Device في الـ DT
            │
            ▼
    platform_driver_register()
    module_platform_driver(emac_driver)
            │
            ▼
┌───────────────────────────┐
│       emac_probe()        │
│  1. alloc_etherdev()      │  ← يخصص net_device + emac_board_info
│  2. of_iomap()            │  ← يعمل map للـ MMIO
│  3. irq_of_parse_and_map()│  ← يحجز رقم الـ IRQ
│  4. emac_configure_dma()  │  ← يطلب DMA channel (اختياري)
│  5. devm_clk_get()        │  ← يحصل على الـ clock
│  6. clk_prepare_enable()  │  ← يشغّل الـ clock
│  7. sunxi_sram_claim()    │  ← يحجز الـ SRAM للـ Ethernet
│  8. of_parse_phandle()    │  ← يحصل على PHY node
│  9. emac_powerup()        │  ← يهيئ الـ MAC registers
│  10. emac_reset()         │  ← يعمل hardware reset
│  11. register_netdev()    │  ← يسجّل الـ interface
└───────────────────────────┘
            │
            ▼
   [interface down - carrier off]
            │
     ifconfig eth0 up
            │
            ▼
┌───────────────────────────┐
│       emac_open()         │
│  1. request_irq()         │  ← يسجّل الـ ISR
│  2. emac_reset()          │  ← reset ثاني
│  3. emac_init_device()    │  ← يفعّل TX/RX + interrupts
│  4. emac_mdio_probe()     │  ← of_phy_connect() → يربط PHY
│  5. phy_start()           │  ← يبدأ autoneg
│  6. netif_start_queue()   │  ← يسمح بإرسال packets
└───────────────────────────┘
            │
            ▼
   [interface up - traffic]
            │
     ifconfig eth0 down
            │
            ▼
┌───────────────────────────┐
│       emac_stop()         │
│  1. netif_stop_queue()    │
│  2. netif_carrier_off()   │
│  3. phy_stop()            │
│  4. emac_mdio_remove()    │  ← phy_disconnect()
│  5. emac_shutdown()       │  ← يعطّل interrupts/TX/RX
│  6. free_irq()            │
└───────────────────────────┘
            │
     rmmod / device removed
            │
            ▼
┌───────────────────────────┐
│       emac_remove()       │
│  1. dmaengine_terminate() │  ← يوقف أي DMA transfer
│  2. dma_release_channel() │
│  3. unregister_netdev()   │
│  4. sunxi_sram_release()  │
│  5. clk_disable_unprepare()│
│  6. irq_dispose_mapping() │
│  7. iounmap()             │
│  8. free_netdev()         │
└───────────────────────────┘
```

---

### دورة حياة الـ DMA Request

```
emac_rx() تكتشف packet كبيرة (rxlen >= mtu) ويوجد rx_chan
            │
            ▼
emac_dma_inblk_32bit(db, skb, rdptr, count)
  │
  ├─► dma_map_single()          ← يحول rdptr إلى dma_addr_t (rxbuf)
  ├─► dmaengine_prep_slave_single() ← يعدّ DMA descriptor
  ├─► emac_alloc_dma_req()      ← kzalloc للـ emac_dma_req
  ├─► desc->callback = emac_dma_done_callback
  ├─► desc->callback_param = req
  ├─► dmaengine_submit()        ← يضيف descriptor للـ queue
  └─► dma_async_issue_pending() ← يطلق الـ DMA transfer
            │
            │  [DMA يعمل في الخلفية]
            │
            ▼
emac_dma_done_callback(req)     ← تُستدعى من DMA engine context
  │
  ├─► dma_unmap_single()        ← يحرر الـ DMA mapping
  ├─► eth_type_trans()          ← يحدد protocol الـ packet
  ├─► netif_rx()                ← يرفع الـ packet للـ network stack
  ├─► dev->stats.rx_bytes/packets++ ← إحصائيات
  ├─► EMAC_RX_CTL_DMA_EN ← clear  ← يعيد CPU receive mode
  ├─► EMAC_INT_CTL_RX_EN ← set    ← يعيد تفعيل RX interrupt
  ├─► db->emacrx_completed_flag = 1
  └─► emac_free_dma_req()       ← kfree
```

---

### Call Flow Diagrams

#### TX Path — إرسال packet

```
kernel network stack
  │  skb جاهز للإرسال
  ▼
emac_start_xmit(skb, dev)
  │
  ├─ [check] tx_fifo_stat & 3 == 3?  → NETDEV_TX_BUSY (رجوع فوراً)
  │
  ├─ spin_lock_irqsave(&db->lock)
  │
  ├─ writel(channel, EMAC_TX_INS_REG)   ← اختيار channel 0 أو 1
  │
  ├─ emac_outblk_32bit(EMAC_TX_IO_DATA_REG, skb->data, skb->len)
  │    └─ writesl() ← يكتب البيانات مباشرة للـ FIFO بـ 32-bit words
  │
  ├─ db->tx_fifo_stat |= 1 << channel  ← يعلّم الـ channel كـ busy
  │
  ├─ writel(skb->len, EMAC_TX_PL0/1_REG) ← طول الـ packet
  │
  ├─ writel(CTL | 1, EMAC_TX_CTL0/1_REG) ← يطلق الإرسال
  │
  ├─ [if both channels busy] netif_stop_queue()
  │
  ├─ spin_unlock_irqrestore(&db->lock)
  │
  └─ dev_consume_skb_any(skb) → NETDEV_TX_OK
```

#### TX Completion — ISR → tx_done

```
Hardware ينتهي من الإرسال → IRQ
  │
  ▼
emac_interrupt(irq, dev)
  │
  ├─ spin_lock(&db->lock)         ← ليس irqsave لأننا في ISR
  ├─ writel(0, EMAC_INT_CTL_REG)  ← تعطيل كل الـ interrupts
  ├─ int_status = readl(EMAC_INT_STA_REG)
  ├─ writel(int_status, EMAC_INT_STA_REG) ← clear by write
  │
  ├─ [TX complete?]
  │    └─► emac_tx_done(dev, db, int_status)
  │              ├─ db->tx_fifo_stat &= ~(tx_status & 3) ← يحرر الـ channel
  │              ├─ dev->stats.tx_packets++
  │              └─ netif_wake_queue()   ← يسمح بإرسال مجدداً
  │
  ├─ [re-enable interrupts]
  │    └─ writel(TX_EN|TX_ABRT_EN|RX_EN, EMAC_INT_CTL_REG)
  │
  └─ spin_unlock(&db->lock)
```

#### RX Path — استقبال packet (CPU mode)

```
Hardware يستقبل packet → RX IRQ
  │
  ▼
emac_interrupt()
  ├─ int_status & 0x100 (RX bit)  AND emacrx_completed_flag == 1
  │
  ├─ emacrx_completed_flag = 0   ← يمنع تداخل استقبال جديد
  │
  └─► emac_rx(dev)
        │
        └─ [loop while rxcount > 0]
             │
             ├─ readl(EMAC_RX_FBC_REG)        ← عدد الـ frames المنتظرة
             ├─ readl(EMAC_RX_IO_DATA_REG)    ← Magic check (0x0143414d)
             │    [لو مش magic صح → flush FIFO وعودة]
             │
             ├─ readl(EMAC_RX_IO_DATA_REG)    ← rxhdr (len + status)
             ├─ rxlen = EMAC_RX_IO_DATA_LEN(rxhdr)
             ├─ rxstatus = EMAC_RX_IO_DATA_STATUS(rxhdr)
             │
             ├─ [status check: CRC err, len err, runt < 0x40]
             │
             ├─ netdev_alloc_skb(dev, rxlen + 4)
             │
             ├─ [rxlen >= mtu AND rx_chan exists?]
             │    YES → emac_dma_inblk_32bit() → break (DMA يكمل لوحده)
             │    NO  → emac_inblk_32bit(EMAC_RX_IO_DATA_REG, rdptr, rxlen)
             │              └─ readsl() ← يقرأ بـ 32-bit words
             │
             ├─ eth_type_trans(skb, dev)
             ├─ netif_rx(skb)             ← يرفع للـ stack
             └─ stats.rx_bytes/packets++
```

#### PHY Link Change Flow

```
phylib يكتشف تغيير في الـ link (polling أو interrupt)
  │
  ▼
emac_handle_link_change(dev)
  │
  ├─ [speed changed?]
  │    ├─ spin_lock_irqsave(&db->lock)
  │    ├─ db->speed = phydev->speed
  │    ├─ emac_update_speed(dev)
  │    │    └─ readl/writel(EMAC_MAC_SUPP_REG) ← set/clear EMAC_MAC_SUPP_100M
  │    └─ spin_unlock_irqrestore(&db->lock)
  │
  ├─ [duplex changed?]
  │    ├─ spin_lock_irqsave(&db->lock)
  │    ├─ db->duplex = phydev->duplex
  │    ├─ emac_update_duplex(dev)
  │    │    └─ readl/writel(EMAC_MAC_CTL1_REG) ← set/clear EMAC_MAC_CTL1_DUPLEX_EN
  │    └─ spin_unlock_irqrestore(&db->lock)
  │
  └─ [link state changed?]
       ├─ db->link = phydev->link
       └─ phy_print_status()
```

#### Suspend/Resume Flow

```
System Suspend
  │
  ▼
emac_suspend(dev, state)
  ├─ netif_carrier_off()
  ├─ netif_device_detach()
  └─ emac_shutdown()
       ├─ writel(0, EMAC_INT_CTL_REG)  ← تعطيل interrupts
       ├─ clear EMAC_INT_STA_REG
       └─ EMAC_CTL_REG &= ~(TX_EN|RX_EN|RESET)

System Resume
  │
  ▼
emac_resume(dev)
  ├─ emac_reset(db)
  │    ├─ writel(0, EMAC_CTL_REG); udelay(200)
  │    └─ writel(EMAC_CTL_RESET, EMAC_CTL_REG); udelay(200)
  ├─ emac_init_device(ndev)
  │    ├─ emac_update_speed()
  │    ├─ emac_update_duplex()
  │    └─ enable TX/RX + interrupts
  └─ netif_device_attach()
```

---

### استراتيجية الـ Locking

#### الـ `db->lock` (spinlock)

هو الـ lock الوحيد في الـ driver، ويحمي:

| ما يحميه | السبب |
|----------|-------|
| كتابة/قراءة الـ MMIO registers | الـ EMAC يعتمد على address register ضمني — أي تداخل يفسد القراءة/الكتابة |
| `db->tx_fifo_stat` | يُقرأ ويُعدَّل من `start_xmit` و`emac_tx_done` (في ISR) |
| `db->speed` و`db->duplex` | تُحدَّث من PHY callback وتُقرأ في `init_device` |
| تسلسل reset + init | منع تنفيذ `emac_init_device` و`emac_reset` بشكل متزامن |

#### جدول استخدام الـ Lock

| الدالة | نوع الـ Lock | السبب |
|--------|-------------|-------|
| `emac_interrupt()` | `spin_lock()` (بدون irqsave) | نحن **في** الـ ISR، الـ IRQs معطّلة أصلاً |
| `emac_start_xmit()` | `spin_lock_irqsave()` | ممكن يتقاطع مع الـ ISR |
| `emac_timeout()` | `spin_lock_irqsave()` | يعمل في softirq context |
| `emac_handle_link_change()` | `spin_lock_irqsave()` | يُستدعى من PHY workqueue |
| `emac_init_device()` | `spin_lock_irqsave()` | يُستدعى من `emac_open` و`emac_resume` |

#### ترتيب الـ Locks (Lock Ordering)

الـ driver لا يأخذ أكثر من lock واحد في نفس الوقت (`db->lock` فقط)، فمفيش مشكلة deadlock من ناحية ترتيب.

#### نقطة مهمة: `emacrx_completed_flag`

مش lock حقيقي لكنه بيعمل وظيفة **serialization** بين الـ ISR والـ DMA callback:

```
ISR:
  if (int_status & RX_BIT) && (emacrx_completed_flag == 1):
    emacrx_completed_flag = 0   ← يمنع ISR التالي من الدخول
    emac_rx()

DMA callback:
  emacrx_completed_flag = 1    ← يسمح للـ ISR بالدخول مرة ثانية
  re-enable RX interrupt
```

المشكلة: الـ flag ده مش `atomic_t` ولا `volatile` بشكل صريح — لكنه محمي ضمنياً لأن الـ ISR يأخذ `db->lock` وهو `spinlock` اللي بيعمل الـ memory barrier اللازم.

---

### خلاصة معمارية

```
┌─────────────────────────────────────────────────────────────────┐
│                    Network Stack (上层)                          │
└──────────────┬───────────────────────────────┬──────────────────┘
               │ netif_rx() [RX]               │ start_xmit() [TX]
               ▼                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      emac driver                                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  emac_rx()   │    │emac_interrupt│    │emac_start_xmit() │  │
│  │  CPU/DMA RX  │◄───│()  ISR       │    │  Direct FIFO TX  │  │
│  └──────┬───────┘    └──────┬───────┘    └────────┬─────────┘  │
│         │                  │ spin_lock             │ spin_lock  │
│         │ DMA mode         └───────────────────────┘            │
│  ┌──────▼───────┐                                               │
│  │emac_dma_req  │                                               │
│  │+ DMA engine  │                                               │
│  └──────────────┘                                               │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ MMIO (readl/writel)
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│               Allwinner A10 EMAC Hardware                       │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ TX FIFO │  │ RX FIFO │  │ MAC Core │  │ MII/PHY Interface│  │
│  │(2 chan) │  │         │  │          │  │                  │  │
│  └─────────┘  └─────────┘  └──────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                                    │
                                                    │ MII bus
                                                    ▼
                                            ┌──────────────┐
                                            │  PHY device  │
                                            │ (phy_device) │
                                            └──────────────┘
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### Registration & Lifecycle

| Function | النوع | الغرض |
|---|---|---|
| `emac_probe` | `platform_driver.probe` | تهيئة الـ device الكاملة وتسجيله في الـ network stack |
| `emac_remove` | `platform_driver.remove` | تحرير كل الـ resources وإلغاء التسجيل |
| `emac_open` | `ndo_open` | فتح الـ interface، تسجيل الـ IRQ، بدء الـ PHY |
| `emac_stop` | `ndo_stop` | إيقاف الـ interface، تحرير الـ IRQ |
| `emac_suspend` | `platform_driver.suspend` | إيقاف الـ device للـ PM |
| `emac_resume` | `platform_driver.resume` | إعادة تشغيل الـ device بعد الـ suspend |

#### Hardware Initialization

| Function | الغرض |
|---|---|
| `emac_reset` | hardware reset بالـ EMAC_CTL_REG |
| `emac_powerup` | تهيئة الـ MAC: flush FIFO، MII clock، MAC address |
| `emac_setup` | ضبط TX/MAC parameters: flow control، CRC، padding، frame limits |
| `emac_init_device` | تفعيل RX/TX والـ interrupts بعد الـ reset |
| `emac_configure_dma` | طلب الـ DMA channel وتهيئة الـ slave config للـ RX |

#### PHY / MDIO

| Function | الغرض |
|---|---|
| `emac_mdio_probe` | ربط الـ MAC بالـ PHY عبر `of_phy_connect` |
| `emac_mdio_remove` | فصل الـ PHY |
| `emac_handle_link_change` | callback من الـ phylib عند تغيّر الـ link state |
| `emac_update_speed` | تحديث الـ MAC speed register |
| `emac_update_duplex` | تحديث الـ MAC duplex register |

#### TX Path

| Function | الغرض |
|---|---|
| `emac_start_xmit` | إرسال الـ SKB عبر أحد الـ TX FIFOs |
| `emac_tx_done` | معالجة TX complete interrupt، تحديث الـ stats |
| `emac_timeout` | معالجة watchdog timeout، إعادة تشغيل الـ device |
| `emac_outblk_32bit` | كتابة block بيانات للـ TX FIFO بـ 32-bit words |

#### RX Path

| Function | الغرض |
|---|---|
| `emac_rx` | استقبال الـ packets من الـ RX FIFO |
| `emac_inblk_32bit` | قراءة block بيانات من الـ RX FIFO بـ 32-bit words (CPU) |
| `emac_dma_inblk_32bit` | قراءة block بيانات بالـ DMA engine |
| `emac_dma_done_callback` | DMA completion callback — يرفع الـ SKB للـ network stack |
| `emac_alloc_dma_req` | تخصيص `emac_dma_req` context |
| `emac_free_dma_req` | تحرير الـ `emac_dma_req` |

#### Interrupt & Poll

| Function | الغرض |
|---|---|
| `emac_interrupt` | الـ IRQ handler الرئيسي |
| `emac_poll_controller` | netpoll/netconsole support |

#### Ethtool & Config

| Function | الغرض |
|---|---|
| `emac_get_drvinfo` | اسم الـ driver والـ bus info |
| `emac_get_msglevel` | قراءة الـ msg_enable |
| `emac_set_msglevel` | كتابة الـ msg_enable |
| `emac_set_rx_mode` | ضبط الـ RX filtering (promiscuous, multicast, broadcast) |
| `emac_set_mac_address` | تغيير الـ MAC address وكتابته للـ hardware |
| `emac_shutdown` | إيقاف الـ interrupts والـ RX/TX |

---

### Group 1: Hardware Initialization & Reset

المجموعة دي بتضمن كل الـ functions اللي بتعمل low-level hardware bring-up. ترتيبها في الـ probe: `emac_reset` → `emac_powerup` → `emac_setup` → `emac_init_device`. كل function بتتولى مستوى مختلف من التهيئة.

---

#### `emac_reset`

```c
static void emac_reset(struct emac_board_info *db)
```

بتعمل hardware reset للـ EMAC عن طريق كتابة `0` في الـ `EMAC_CTL_REG` لمدة 200µs، بعدين كتابة `EMAC_CTL_RESET` لمدة 200µs تانية. ده soft reset بيصفر حالة الـ hardware كاملة.

**Parameters:**
- `db` — الـ `emac_board_info` اللي فيها `membase` لعناوين الـ registers

**Return:** void

**Key details:**
- بتستخدم `udelay(200)` — يعني في الـ atomic context مش مشكلة
- لازم تتعمل قبل أي initialization تانية
- بتتستدعى من: `emac_probe`، `emac_open`، `emac_timeout`، `emac_resume`

---

#### `emac_powerup`

```c
static unsigned int emac_powerup(struct net_device *ndev)
```

بتهيئ الـ MAC على 4 مراحل: (1) flush الـ RX FIFO، (2) clear الـ MAC soft-reset bit، (3) ضبط الـ MII clock divisor على 72 (المناسب لـ AHB ≤ 150 MHz)، (4) كتابة الـ MAC address للـ registers `EMAC_MAC_A0_REG` و`EMAC_MAC_A1_REG`. في الآخر بتستدعي `emac_setup` لإكمال الـ MAC config.

**Parameters:**
- `ndev` — الـ `net_device` — منه بتاخد `dev_addr` و`netdev_priv`

**Return:** دايمًا `0` (لا يوجد error path حقيقي)

**Key details:**
- الـ MII clock divisor اللي بيتحط (`EMAC_MAC_MCFG_MII_CLKD_72` = `0x0d << 2`) بيعطي تقسيم بـ 72 للـ AHB clock للوصول لـ MDC ≤ 2.5 MHz
- بتتستدعى من `emac_probe` فقط

---

#### `emac_setup`

```c
static unsigned int emac_setup(struct net_device *ndev)
```

بتضبط الـ MAC control registers:
- `EMAC_TX_MODE_REG`: تفعيل إعادة إرسال الـ aborted frames
- `EMAC_MAC_CTL0_REG`: تفعيل RX/TX flow control
- `EMAC_MAC_CTL1_REG`: تفعيل length check، CRC generation، automatic padding
- `EMAC_MAC_IPGT_REG`: ضبط الـ inter-packet gap لـ full duplex (0x15)
- `EMAC_MAC_IPGR_REG`: ضبط الـ IPG1/IPG2
- `EMAC_MAC_CLRT_REG`: ضبط نافذة الـ collision وعدد retries
- `EMAC_MAC_MAXF_REG`: أقصى حجم frame = `0x0600` (1536 bytes)

**Parameters:**
- `ndev` — الـ `net_device`

**Return:** دايمًا `0`

**Key details:**
- لا توجد حماية بـ spinlock هنا — يُفترض الاستدعاء من سياق بدون race
- بتتستدعى من `emac_powerup`

---

#### `emac_init_device`

```c
static void emac_init_device(struct net_device *dev)
```

بتفعل الـ RX/TX وبتضبط الـ interrupt mask. بتستدعي `emac_update_speed` و`emac_update_duplex` لكتابة الـ current PHY state للـ MAC registers، بعدين بتكتب في `EMAC_CTL_REG` لتفعيل `EMAC_CTL_TX_EN | EMAC_CTL_RX_EN | EMAC_CTL_RESET`، وبعدين بتفعل interrupts TX/TX_ABRT/RX في `EMAC_INT_CTL_REG`.

**Parameters:**
- `dev` — الـ `net_device`

**Return:** void

**Key details:**
- محمية بـ `spin_lock_irqsave` — آمنة للاستدعاء من أي context
- بتتستدعى من: `emac_open`، `emac_timeout`، `emac_resume`

---

#### `emac_configure_dma`

```c
static int emac_configure_dma(struct emac_board_info *db)
```

بتطلب الـ DMA channel المسمى `"rx"` وبتضبطه للـ slave mode (DEV_TO_MEM)، اتجاه البيانات من الـ EMAC RX FIFO (`EMAC_RX_IO_DATA_REG`) للـ memory بـ 4-byte bus width وـ burst size = 4. بتحفظ عنوان الـ FIFO الفيزيائي في `db->emac_rx_fifo`.

**Parameters:**
- `db` — الـ board info

**Return:** `0` في حالة النجاح، error code سالب في حالة الفشل. لو فشلت، الـ driver بيكمل بدون DMA (CPU copy fallback).

**Key details:**
- الـ DMA channel بيتطلب `dma-names = "rx"` في الـ DT node
- لو `dma_request_chan` فشل، `db->rx_chan` بيتضبط على `NULL` والـ driver بيستخدم `emac_inblk_32bit` بدلًا منها
- error path بيحرر الـ channel لو الـ slave config فشل

**Pseudocode:**
```
emac_configure_dma():
  regs = platform_get_resource(MEM, 0)
  db->emac_rx_fifo = regs->start + EMAC_RX_IO_DATA_REG
  db->rx_chan = dma_request_chan("rx")
  if IS_ERR(rx_chan): goto out_clear_chan
  conf = {DEV_TO_MEM, 4-byte width, src=emac_rx_fifo, burst=4}
  dmaengine_slave_config(rx_chan, conf)
  if err: dma_release_channel(); goto out_clear_chan
  return 0
```

---

### Group 2: PHY & Link Management

الـ PHY في الـ sun4i-emac مرتبط عبر الـ phylib. الـ driver مش بيدي الـ PHY أي interrupt ويعتمد على الـ polling اللي بتعمله الـ phylib.

---

#### `emac_mdio_probe`

```c
static int emac_mdio_probe(struct net_device *dev)
```

بتربط الـ MAC بالـ PHY عبر `of_phy_connect` باستخدام الـ `phy_node` اللي اتجابت من الـ DT في الـ probe. بتسجل `emac_handle_link_change` كـ callback. بتحدد الـ max speed بـ 100 Mbps (الـ hardware لا يدعم Gigabit). بتصفر `link`، `speed`، `duplex` في الـ `db`.

**Parameters:**
- `dev` — الـ `net_device`

**Return:** `0` أو `-ENODEV` لو الـ PHY مش موجود

**Key details:**
- الـ PHY interrupts غير مدعومة — الـ `irq` parameter في `of_phy_connect` = `0`
- بتتستدعى من `emac_open` فقط بعد `emac_init_device`

---

#### `emac_mdio_remove`

```c
static void emac_mdio_remove(struct net_device *dev)
```

بتستدعي `phy_disconnect` لفصل الـ PHY من الـ MAC وإيقاف الـ phylib polling. بتتستدعى من `emac_stop`.

---

#### `emac_handle_link_change`

```c
static void emac_handle_link_change(struct net_device *dev)
```

الـ callback اللي الـ phylib بيستدعيه لما حالة الـ link تتغير. بيقارن الـ `phydev->speed` و`phydev->duplex` و`phydev->link` بالـ cached values في `db`. لو في تغيير: بيحدث الـ cache، بيستدعي `emac_update_speed`/`emac_update_duplex` محميًا بـ `spin_lock_irqsave`، ولو الـ link وقع بيضبط `speed=0` و`duplex=-1`. في الآخر بيستدعي `phy_print_status` لو في تغيير.

**Parameters:**
- `dev` — الـ `net_device`

**Return:** void

**Key details:**
- بيتاستدعى من سياق الـ phylib workqueue (sleepable context)
- الـ register writes محمية بـ spinlock لأن الـ ISR ممكن يشتغل في نفس الوقت

---

#### `emac_update_speed`

```c
static void emac_update_speed(struct net_device *dev)
```

بتقرا `EMAC_MAC_SUPP_REG`، بتضبط أو بتمسح `EMAC_MAC_SUPP_100M` (bit 8) حسب `db->speed == SPEED_100`.

**Key details:** لازم تتاستدعى مع `spin_lock_irqsave` محيطة بيها.

---

#### `emac_update_duplex`

```c
static void emac_update_duplex(struct net_device *dev)
```

بتقرا `EMAC_MAC_CTL1_REG`، بتضبط أو بتمسح `EMAC_MAC_CTL1_DUPLEX_EN` (bit 0) حسب `db->duplex`.

**Key details:** نفس الـ locking requirement كـ `emac_update_speed`.

---

### Group 3: TX Path

الـ TX path بيعتمد على **double-buffered hardware FIFOs** — الـ EMAC بيوفر FIFO 0 وFIFO 1. الـ driver بيتتبع أي الـ FIFOs محجوز عبر `db->tx_fifo_stat` (bitmask من 2 bits).

---

#### `emac_start_xmit`

```c
static netdev_tx_t emac_start_xmit(struct sk_buff *skb, struct net_device *dev)
```

الـ hot path للإرسال. بتحدد الـ channel المتاح (0 أو 1)، بتكتب بيانات الـ SKB للـ TX FIFO عبر `emac_outblk_32bit`، بتكتب الطول في الـ `EMAC_TX_PL0/1_REG`، وبتبدأ الإرسال بكتابة bit 0 في `EMAC_TX_CTL0/1_REG`. لو الاتنين FIFOs ممتلئين بتستدعي `netif_stop_queue`.

**Parameters:**
- `skb` — الـ socket buffer للإرسال
- `dev` — الـ `net_device`

**Return:** `NETDEV_TX_OK` أو `NETDEV_TX_BUSY` لو الاتنين FIFOs ممتلئين

**Key details:**
- محمية بـ `spin_lock_irqsave`
- الـ SKB بيتحرر بـ `dev_consume_skb_any` مباشرة — الـ hardware بينقل البيانات synchronously للـ FIFO
- `db->tx_fifo_stat` bitmask: bit 0 = channel 0 مشغول، bit 1 = channel 1 مشغول
- لو الاتنين bits = 1 (قيمة 3): `return NETDEV_TX_BUSY`

**Pseudocode:**
```
emac_start_xmit(skb, dev):
  channel = db->tx_fifo_stat & 3
  if channel == 3: return NETDEV_TX_BUSY
  channel = (channel == 1) ? 1 : 0
  spin_lock_irqsave()
  writel(channel, EMAC_TX_INS_REG)          // select FIFO
  emac_outblk_32bit(TX_IO_DATA_REG, skb->data, skb->len)
  db->tx_fifo_stat |= (1 << channel)
  if channel == 0:
    writel(skb->len, EMAC_TX_PL0_REG)
    writel(CTL0 | 1, EMAC_TX_CTL0_REG)     // start TX
  else:
    writel(skb->len, EMAC_TX_PL1_REG)
    writel(CTL1 | 1, EMAC_TX_CTL1_REG)
  if (tx_fifo_stat & 3) == 3:
    netif_stop_queue()
  spin_unlock_irqrestore()
  dev_consume_skb_any(skb)
  return NETDEV_TX_OK
```

---

#### `emac_tx_done`

```c
static void emac_tx_done(struct net_device *dev, struct emac_board_info *db,
                          unsigned int tx_status)
```

بتعالج الـ TX complete interrupt. بتمسح الـ bits في `db->tx_fifo_stat` المناظرة للـ FIFOs اللي خلصت. لو الاتنين bits محددين في `tx_status` (قيمة 3): بتزود `tx_packets` بـ 2، غير كده بـ 1. بتستدعي `netif_wake_queue` لإعادة تشغيل الـ TX queue.

**Parameters:**
- `dev` — الـ `net_device`
- `db` — الـ board info
- `tx_status` — الـ interrupt status register value (lower 2 bits = channel done flags)

**Return:** void

**Key details:**
- بتتاستدعى من داخل `emac_interrupt` وهو شايل الـ `db->lock` spinlock — يعني لا يوجد locking إضافي هنا

---

#### `emac_timeout`

```c
static void emac_timeout(struct net_device *dev, unsigned int txqueue)
```

الـ watchdog handler لما الـ TX تتوقف أكتر من `watchdog` milliseconds (default: 5 ثواني). بتوقف الـ queue، بتعمل reset كامل للـ device، بتستدعي `emac_init_device`، وبتعيد تشغيل الـ queue.

**Parameters:**
- `dev` — الـ `net_device`
- `txqueue` — رقم الـ TX queue (غير مستخدم)

**Return:** void

**Key details:**
- بتتاستدعى من الـ network stack في سياق softirq/process
- محمية بـ `spin_lock_irqsave` — بتعمل reset وهي شايلة الـ lock وده ممكن يسبب stall قصير

---

#### `emac_outblk_32bit`

```c
static void emac_outblk_32bit(void __iomem *reg, void *data, int count)
```

بتكتب `count` bytes للـ MMIO register باستخدام `writesl` (repeated 32-bit store). بتعمل `round_up(count, 4) / 4` لتحديد عدد الـ 32-bit words.

**Parameters:**
- `reg` — عنوان الـ MMIO (`EMAC_TX_IO_DATA_REG`)
- `data` — مؤشر للبيانات
- `count` — عدد الـ bytes

**Key details:** الـ `writesl` بتضمن little-endian byte order على كل الـ ARM platforms.

---

### Group 4: RX Path

الـ RX path بيدعم وضعين: **CPU copy** عبر `emac_inblk_32bit` و**DMA transfer** عبر `emac_dma_inblk_32bit`. الـ DMA بيتستخدم فقط لو الـ frame size ≥ `dev->mtu` و`db->rx_chan != NULL`.

---

#### `emac_rx`

```c
static void emac_rx(struct net_device *dev)
```

الـ main RX processing function. بتعمل polling loop على الـ RX FIFO حتى ما يبقاش في packets. لكل packet:
1. بتقرا الـ magic word (لازم يكون `EMAC_UNDOCUMENTED_MAGIC` = `0x0143414d`) — لو مختلف بتعمل flush للـ FIFO وترجع
2. بتقرا الـ header word: lower 16 bits = length، upper 16 bits = status
3. بتتحقق من الـ minimum frame size (64 bytes) ومن status bits (CRC error، length error)
4. لو الـ packet صح: بتعمل `netdev_alloc_skb`، بتاخد pointer من `skb_put`، وبتقرا البيانات إما بـ DMA أو CPU
5. بتضبط الـ `skb->protocol` وبتستدعي `netif_rx`

**Parameters:**
- `dev` — الـ `net_device`

**Return:** void

**Key details:**
- بتتاستدعى من داخل `emac_interrupt` وهو شايل الـ `db->lock` spinlock
- لو `netdev_alloc_skb` فشلت: الـ `continue` بيعمل skip للـ packet لكن **من غير** إنه يقرا بياناتها من الـ FIFO — ده bug محتمل يخلي الـ FIFO في state غلط
- الـ CRC الأخيرة (4 bytes) بتتقطع بـ `skb_put(skb, rxlen - 4)` لكن `emac_inblk_32bit` بتقرا `rxlen` كامل — الـ CRC بتكتب في buffer بس ما بتتحدش للـ skb

**Pseudocode:**
```
emac_rx(dev):
  while True:
    rxcount = readl(EMAC_RX_FBC_REG)
    if !rxcount:
      re-enable interrupts
      rxcount = readl(EMAC_RX_FBC_REG)  // double-check for race
      if !rxcount: return
    magic = readl(EMAC_RX_IO_DATA_REG)
    if magic != EMAC_UNDOCUMENTED_MAGIC:
      flush RX FIFO; re-enable RX & interrupts; return
    rxhdr = readl(EMAC_RX_IO_DATA_REG)
    rxlen = rxhdr & 0xffff
    rxstatus = (rxhdr >> 16) & 0xffff
    if rxlen < 64: good_packet = false
    if !(rxstatus & STATUS_OK): good_packet = false; update stats
    if good_packet:
      skb = netdev_alloc_skb(rxlen + 4)
      rdptr = skb_put(skb, rxlen - 4)
      if rxlen >= mtu && rx_chan:
        enable DMA mode
        if emac_dma_inblk_32bit() == 0: break  // DMA takes over
        disable DMA mode
      emac_inblk_32bit(RX_IO_DATA_REG, rdptr, rxlen)  // CPU copy fallback
      netif_rx(skb)
```

---

#### `emac_inblk_32bit`

```c
static void emac_inblk_32bit(void __iomem *reg, void *data, int count)
```

بتقرا `count` bytes من الـ MMIO register باستخدام `readsl` (repeated 32-bit load). مثيلة `emac_outblk_32bit` في الاتجاه العكسي.

---

#### `emac_dma_inblk_32bit`

```c
static int emac_dma_inblk_32bit(struct emac_board_info *db,
        struct sk_buff *skb, void *rdptr, int count)
```

بتبدأ DMA transfer من الـ EMAC RX FIFO للـ SKB buffer:
1. `dma_map_single` — بتعمل map للـ destination buffer للـ DMA
2. `dmaengine_prep_slave_single` — بتحضر الـ DMA descriptor
3. `emac_alloc_dma_req` — بتخصص الـ context object
4. بتضبط الـ callback على `emac_dma_done_callback`
5. `dmaengine_submit` + `dma_async_issue_pending` — بتبدأ التنفيذ الفعلي

**Parameters:**
- `db` — الـ board info
- `skb` — الـ socket buffer الهدف
- `rdptr` — مؤشر داخل الـ SKB data buffer
- `count` — عدد الـ bytes

**Return:** `0` لو الـ DMA بدأ بنجاح، error code سالب في حالة فشل أي خطوة

**Key details:**
- الـ error path بيضمن cleanup صحيح: `emac_free_dma_req` ثم `dmaengine_desc_free` ثم `dma_unmap_single`
- لو رجعت `0`: الـ ISR بيبقى disabled للـ RX interrupt حتى الـ DMA callback يكمل
- بتتاستدعى من داخل `emac_rx` وهو داخل الـ ISR

---

#### `emac_dma_done_callback`

```c
static void emac_dma_done_callback(void *arg)
```

الـ DMA completion callback. بتتاستدعى من الـ DMA engine tasklet/thread. بتعمل:
1. `dma_unmap_single` — تحرير الـ DMA mapping
2. `eth_type_trans` + `netif_rx` — تسليم الـ SKB للـ network stack
3. تحديث `rx_bytes` و`rx_packets`
4. إلغاء تفعيل الـ DMA mode في `EMAC_RX_CTL_REG`
5. إعادة تفعيل الـ RX interrupt في `EMAC_INT_CTL_REG`
6. ضبط `db->emacrx_completed_flag = 1`
7. `emac_free_dma_req`

**Parameters:**
- `arg` — مؤشر للـ `emac_dma_req` context

**Return:** void

**Key details:**
- بتتاستدعى من الـ DMA engine context — مش بتشيل الـ `db->lock` — الـ register writes هنا ممكن تحصل بدون lock مع الـ ISR (race محتمل في الـ INT_CTL_REG)
- بعد ما بتخلص: الـ interrupt handler يقدر يستقبل RX interrupts جديدة تاني

---

#### `emac_alloc_dma_req` / `emac_free_dma_req`

```c
static struct emac_dma_req *
emac_alloc_dma_req(struct emac_board_info *db,
                   struct dma_async_tx_descriptor *desc, struct sk_buff *skb,
                   dma_addr_t rxbuf, int count)

static void emac_free_dma_req(struct emac_dma_req *req)
```

**الـ `emac_alloc_dma_req`:** بتخصص `emac_dma_req` بـ `kzalloc(GFP_ATOMIC)` وبتملاها بالـ context المطلوب للـ DMA callback. الـ `GFP_ATOMIC` لأنها بتتاستدعى من داخل الـ ISR.

**الـ `emac_free_dma_req`:** wrapper على `kfree` فقط.

---

### Group 5: Interrupt Handler

---

#### `emac_interrupt`

```c
static irqreturn_t emac_interrupt(int irq, void *dev_id)
```

الـ top-half ISR. بتشتغل بـ `spin_lock` (مش `irqsave` — الـ IRQ نفسه disabled automatically). البنية:

1. `writel(0, EMAC_INT_CTL_REG)` — تعطيل كل الـ interrupts أول حاجة
2. قراءة وكتابة (clear) الـ `EMAC_INT_STA_REG`
3. لو `int_status & 0x100` (RX complete) والـ `emacrx_completed_flag == 1`: ضبط الـ flag على 0 واستدعاء `emac_rx`
4. لو `int_status & EMAC_INT_STA_TX_COMPLETE`: استدعاء `emac_tx_done`
5. لو `int_status & EMAC_INT_STA_TX_ABRT`: طباعة debug message
6. إعادة تفعيل الـ interrupts — لو `emacrx_completed_flag == 1`: تفعيل TX+RX، غير كده: تفعيل TX فقط (الـ RX interrupt يظل disabled حتى الـ DMA ينهي)

**Parameters:**
- `irq` — رقم الـ IRQ
- `dev_id` — مؤشر للـ `net_device`

**Return:** دايمًا `IRQ_HANDLED`

**Key details:**
- `emacrx_completed_flag` هو الـ synchronization mechanism بين الـ ISR والـ DMA callback — مش thread-safe بشكل كامل (no memory barriers explicit)
- الـ mask `0x100` بدل `EMAC_INT_STA_RX_COMPLETE` ده inconsistency — لكنهم متساويين (bit 8)
- بتستخدم `spin_lock` مش `spin_lock_irqsave` لأن الـ hardware تلقائيًا بيعطل الـ IRQ الحالي لما يدخل الـ handler

**Pseudocode:**
```
emac_interrupt(irq, dev_id):
  spin_lock(&db->lock)
  writel(0, INT_CTL_REG)                    // mask all
  int_status = readl(INT_STA_REG)
  writel(int_status, INT_STA_REG)           // clear
  if (int_status & RX_COMPLETE) && emacrx_completed_flag:
    emacrx_completed_flag = 0
    emac_rx(dev)
  if int_status & TX_COMPLETE:
    emac_tx_done(dev, db, int_status)
  // re-enable based on DMA state
  if emacrx_completed_flag:
    re-enable TX | TX_ABRT | RX
  else:
    re-enable TX | TX_ABRT only
  spin_unlock(&db->lock)
  return IRQ_HANDLED
```

---

#### `emac_poll_controller`

```c
static void emac_poll_controller(struct net_device *dev)
```

بتتاستدعى من netconsole عشان تعمل poll للـ device بدون waiting للـ interrupt. بتعطل الـ IRQ، بتستدعي `emac_interrupt` manually، بتعيد تشغيل الـ IRQ.

**Key details:** متوفرة فقط لو `CONFIG_NET_POLL_CONTROLLER=y`.

---

### Group 6: Open / Stop / Lifecycle

---

#### `emac_open`

```c
static int emac_open(struct net_device *dev)
```

بتتاستدعى لما الـ interface يتفعل (`ip link set up` أو `ifconfig up`):
1. `request_irq` — تسجيل `emac_interrupt`
2. `emac_reset` + `emac_init_device` — تهيئة الـ hardware
3. `emac_mdio_probe` — ربط الـ PHY
4. `phy_start` — بدء الـ phylib state machine
5. `netif_start_queue` — السماح للـ TX queue بالعمل

**Return:** `0` أو `-EAGAIN` (IRQ) أو error من `emac_mdio_probe`

**Error path:** لو `emac_mdio_probe` فشل: `free_irq` قبل الرجوع بالـ error.

---

#### `emac_stop`

```c
static int emac_stop(struct net_device *ndev)
```

عكس `emac_open` بالترتيب:
1. `netif_stop_queue` + `netif_carrier_off`
2. `phy_stop` — إيقاف الـ phylib
3. `emac_mdio_remove` — فصل الـ PHY
4. `emac_shutdown` — إيقاف الـ hardware interrupts والـ RX/TX
5. `free_irq`

**Return:** دايمًا `0`

---

#### `emac_shutdown`

```c
static void emac_shutdown(struct net_device *dev)
```

بتوقف الـ hardware بالكامل: تعطيل كل الـ interrupts، مسح الـ interrupt status، تعطيل الـ RX/TX والـ reset bit. بتتاستدعى من `emac_stop` و`emac_suspend`.

---

#### `emac_suspend` / `emac_resume`

```c
static int emac_suspend(struct platform_device *dev, pm_message_t state)
static int emac_resume(struct platform_device *dev)
```

**`emac_suspend`:** بتعمل `netif_carrier_off` + `netif_device_detach` + `emac_shutdown`. لا تحرر الـ IRQ أو الـ DMA.

**`emac_resume`:** بتعمل `emac_reset` + `emac_init_device` + `netif_device_attach`. لا تعيد تشغيل الـ PHY — الـ phylib بتعمل ده تلقائيًا.

**Key details:** الاتنين بيرجعوا `0` دايمًا. الـ DMA channel والـ IRQ بيفضلوا allocated طول الـ suspend.

---

### Group 7: Registration (Probe / Remove)

---

#### `emac_probe`

```c
static int emac_probe(struct platform_device *pdev)
```

الـ function الأهم في الـ driver — بتعمل full initialization وتسجيل الـ device. التسلسل:

```
alloc_etherdev()                    → allocate net_device + emac_board_info
SET_NETDEV_DEV()                    → link net_device للـ platform device
of_iomap(np, 0)                     → map MMIO registers
irq_of_parse_and_map(np, 0)         → get IRQ number
emac_configure_dma()               → setup DMA (optional, non-fatal)
devm_clk_get() + clk_prepare_enable() → enable EMAC clock
sunxi_sram_claim()                 → claim SRAM region for EMAC use
of_parse_phandle("phy-handle")     → get PHY DT node
of_get_ethdev_address()            → read MAC from DT (or eth_hw_addr_random)
emac_powerup() + emac_reset()      → initial hardware setup
register_netdev()                  → register with network stack
```

**Parameters:**
- `pdev` — الـ `platform_device` من الـ OF match

**Return:** `0` أو error code سالب

**Error paths (unwind ladder):**
```
out:                → free_netdev
out_iounmap:        → iounmap
out_dispose_mapping: → irq_dispose_mapping + dma_release_channel
out_clk_disable_unprepare: → clk_disable_unprepare
out_release_sram:   → sunxi_sram_release
```

**Key details:**
- الـ `sunxi_sram_claim` ضروري لأن الـ EMAC بيستخدم الـ internal SRAM كـ packet buffer — بدونها الـ hardware ما يشتغلش
- لو `emac_configure_dma` فشل: الـ driver بيكمل بدون DMA (CPU copy only) ومش بيفشل الـ probe
- بعد `register_netdev` الـ device يبقى visible للـ userspace لكن الـ interface لسه down

---

#### `emac_remove`

```c
static void emac_remove(struct platform_device *pdev)
```

عكس الـ probe بالترتيب العكسي:
1. إنهاء وتحرير الـ DMA channel لو موجود (`dmaengine_terminate_all` + `dma_release_channel`)
2. `unregister_netdev`
3. `sunxi_sram_release`
4. `clk_disable_unprepare`
5. `irq_dispose_mapping`
6. `iounmap`
7. `free_netdev`

---

### Group 8: Ethtool & Configuration

---

#### `emac_get_drvinfo`

```c
static void emac_get_drvinfo(struct net_device *dev, struct ethtool_drvinfo *info)
```

بتملا `info->driver` بـ `"sun4i-emac"` و`info->bus_info` باسم الـ device من `dev_name`. بتستخدم `strscpy` لضمان null-termination.

---

#### `emac_get_msglevel` / `emac_set_msglevel`

```c
static u32 emac_get_msglevel(struct net_device *dev)
static void emac_set_msglevel(struct net_device *dev, u32 value)
```

قراءة وكتابة `db->msg_enable` — الـ bitmask اللي بيتحكم في مستوى الـ debug messages (`NETIF_MSG_*`). بيتستخدم مع `netif_msg_*()` macros في الكود.

---

#### `emac_set_rx_mode`

```c
static void emac_set_rx_mode(struct net_device *ndev)
```

بتضبط الـ EMAC RX filtering في `EMAC_RX_CTL_REG`:
- لو `IFF_PROMISC`: تفعيل `EMAC_RX_CTL_PASS_ALL_EN`
- دايمًا: قبول unicast، multicast، broadcast، وتفعيل الـ DA filter
- لا يدعم الـ multicast hash filtering رغم وجود الـ registers (`EMAC_RX_HASH0/1_REG`)

**Key details:** بتتاستدعى من الـ network stack — مش محمية بـ spinlock (الـ MMIO write مش atomic مع الـ ISR).

---

#### `emac_set_mac_address`

```c
static int emac_set_mac_address(struct net_device *dev, void *p)
```

بتغير الـ MAC address أثناء التشغيل:
1. لو الـ interface running: ترجع `-EBUSY`
2. `eth_hw_addr_set` — تحديث الـ `dev->dev_addr`
3. كتابة الـ 6 bytes للـ `EMAC_MAC_A0_REG` و`EMAC_MAC_A1_REG`

**Parameters:**
- `dev` — الـ `net_device`
- `p` — مؤشر لـ `struct sockaddr` — `sa_data` فيها الـ MAC address الجديد

**Return:** `0` أو `-EBUSY`

---

### جدول الـ `emac_netdev_ops` و`emac_ethtool_ops`

| Hook | Implementation |
|---|---|
| `ndo_open` | `emac_open` |
| `ndo_stop` | `emac_stop` |
| `ndo_start_xmit` | `emac_start_xmit` |
| `ndo_tx_timeout` | `emac_timeout` |
| `ndo_set_rx_mode` | `emac_set_rx_mode` |
| `ndo_eth_ioctl` | `phy_do_ioctl_running` |
| `ndo_validate_addr` | `eth_validate_addr` |
| `ndo_set_mac_address` | `emac_set_mac_address` |
| `ndo_poll_controller` | `emac_poll_controller` (optional) |
| `get_drvinfo` | `emac_get_drvinfo` |
| `get_link` | `ethtool_op_get_link` |
| `get_link_ksettings` | `phy_ethtool_get_link_ksettings` |
| `set_link_ksettings` | `phy_ethtool_set_link_ksettings` |
| `get_msglevel` | `emac_get_msglevel` |
| `set_msglevel` | `emac_set_msglevel` |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs

**الـ** `sun4i-emac` مش بيسجّل entries في debugfs بشكل مباشر، لكن الـ PHY subsystem والـ netdev بيوفّروا entries مفيدة:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل entries الـ net
ls /sys/kernel/debug/net/

# الـ PHY state من خلال mdio bus
ls /sys/kernel/debug/mdio/
cat /sys/kernel/debug/mdio/<bus-id>/<phy-addr>/
```

**الـ** DMA engine debugging (لأن الـ driver بيستخدم `dma_request_chan` للـ RX):

```bash
# DMA engine state
cat /sys/kernel/debug/dmaengine/summary

# الـ DMA channel الخاص بـ EMAC RX
ls /sys/kernel/debug/dmaengine/
```

---

#### 2. sysfs

كل entries الـ netdev وإحصائياتها متاحة عبر sysfs:

```bash
# إحصائيات الـ interface
cat /sys/class/net/eth0/statistics/rx_packets
cat /sys/class/net/eth0/statistics/rx_bytes
cat /sys/class/net/eth0/statistics/rx_errors
cat /sys/class/net/eth0/statistics/rx_crc_errors
cat /sys/class/net/eth0/statistics/rx_length_errors
cat /sys/class/net/eth0/statistics/tx_packets
cat /sys/class/net/eth0/statistics/tx_bytes
cat /sys/class/net/eth0/statistics/tx_aborted_errors

# الـ link state والـ speed والـ duplex
cat /sys/class/net/eth0/operstate
cat /sys/class/net/eth0/speed
cat /sys/class/net/eth0/duplex
cat /sys/class/net/eth0/carrier

# الـ MAC address المبرمجة في الـ chip
cat /sys/class/net/eth0/address

# الـ PHY device
ls /sys/class/net/eth0/phydev/
cat /sys/class/net/eth0/phydev/phy_id
cat /sys/class/net/eth0/phydev/phy_interface

# الـ DMA channel الخاص بـ RX
ls /sys/bus/platform/devices/1c0b000.ethernet/dma-requests
```

---

#### 3. ftrace

**الـ tracepoints المفيدة:**

```bash
# enable net tracepoints
echo 1 > /sys/kernel/debug/tracing/events/net/enable

# trace RX packet path
echo 1 > /sys/kernel/debug/tracing/events/net/netif_rx/enable
echo 1 > /sys/kernel/debug/tracing/events/net/net_dev_xmit/enable
echo 1 > /sys/kernel/debug/tracing/events/net/napi_poll/enable

# trace DMA events
echo 1 > /sys/kernel/debug/tracing/events/dma/enable

# filter على الـ interface بس
echo 'name == "eth0"' > /sys/kernel/debug/tracing/events/net/netif_rx/filter

# قرا الـ trace
cat /sys/kernel/debug/tracing/trace_pipe
```

**الـ function tracer للـ driver:**

```bash
# trace functions الخاصة بـ sun4i_emac
echo 'emac_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

**الـ IRQ tracing لتشخيص مشاكل الـ interrupt:**

```bash
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
cat /sys/kernel/debug/tracing/trace_pipe | grep sun4i
```

---

#### 4. printk / Dynamic Debug

**الـ** driver بيستخدم `netif_msg_*` macros مع `db->msg_enable` للتحكم في الـ messages. يتحكم فيها إما عبر module param أو ethtool:

```bash
# تفعيل كل debug messages عبر module param عند التحميل
modprobe sun4i-emac debug=0xFFFF

# أو عبر ethtool بعد التحميل
ethtool -s eth0 msglvl 0xFFFF

# القيم المتاحة لـ msg_enable (netif_msg bits):
# 0x0001 = NETIF_MSG_DRV       (driver init)
# 0x0002 = NETIF_MSG_PROBE     (probe)
# 0x0004 = NETIF_MSG_LINK      (link state)
# 0x0008 = NETIF_MSG_TIMER     (tx timeout)
# 0x0020 = NETIF_MSG_INTR      (interrupt handler)
# 0x0040 = NETIF_MSG_TX_ERR    (tx errors)
# 0x0080 = NETIF_MSG_RX_ERR    (rx errors)
# 0x0100 = NETIF_MSG_TX_DONE   (tx completion)
# 0x0200 = NETIF_MSG_RX_STATUS (rx header/count)
# 0x0400 = NETIF_MSG_PKTDATA   (packet data)
# 0x1000 = NETIF_MSG_IFDOWN    (interface down)
# 0x2000 = NETIF_MSG_IFUP      (interface up)

# dynamic debug للـ dev_dbg calls
echo 'module sun4i_emac +p' > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل كل الـ subsystem
echo 'file sun4i-emac.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# التحقق من الـ dynamic debug rules
cat /sys/kernel/debug/dynamic_debug/control | grep sun4i
```

**الـ flags المتاحة في dynamic debug:**
- `p` = طباعة الـ message
- `f` = إظهار اسم الـ function
- `l` = إظهار رقم السطر
- `m` = إظهار اسم الـ module
- `t` = إظهار الـ thread ID

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوصف |
|---|---|
| `CONFIG_NET_EMAC_SUNXI` | تفعيل الـ driver أصلاً (=m للـ module) |
| `CONFIG_NETDEVICES_DEBUG` | debug عام للـ network devices |
| `CONFIG_NET_POLL_CONTROLLER` | يفعّل `emac_poll_controller` للـ netconsole |
| `CONFIG_PHYLIB` | PHY library – لازم لأن الـ driver يعتمد عليها |
| `CONFIG_DEBUG_FS` | يفعّل debugfs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل dynamic debug |
| `CONFIG_DMA_ENGINE` | DMA engine للـ RX path |
| `CONFIG_DMA_DEBUG` | يكشف mismatch في dma_map/unmap |
| `CONFIG_DMA_API_DEBUG` | أعمق debug للـ DMA API |
| `CONFIG_SLUB_DEBUG` | يكشف corruption في الـ kzalloc (emac_dma_req) |
| `CONFIG_DEBUG_SPINLOCK` | يكشف deadlocks في `db->lock` |
| `CONFIG_LOCKDEP` | lock dependency checker – مفيد لـ spinlock الـ driver |
| `CONFIG_KASAN` | kernel address sanitizer لكشف memory bugs |
| `CONFIG_SUNXI_SRAM` | لازم لـ `sunxi_sram_claim` في الـ probe |

---

#### 6. أدوات خاصة بالـ Subsystem

**ethtool:**

```bash
# Driver info
ethtool -i eth0

# Link settings
ethtool eth0

# إحصائيات موسّعة
ethtool -S eth0

# PHY registers dump
ethtool -d eth0

# تغيير الـ msg level
ethtool -s eth0 msglvl 0xFFFF

# Link negotiation يدوي للاختبار
ethtool -s eth0 speed 100 duplex full autoneg off
```

**ip / ss tools:**

```bash
# عرض إحصائيات الـ interface
ip -s link show eth0

# عرض errors بالتفصيل
ip -s -s link show eth0
```

**مراقبة الـ DMA:**

```bash
# حالة الـ DMA channels
cat /sys/kernel/debug/dmaengine/summary

# الـ pending transactions
dma_stat=$(cat /sys/kernel/debug/dmaengine/summary | grep sun4i)
echo "$dma_stat"
```

---

#### 7. رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `could not find the PHY` | `of_phy_connect` فشلت، الـ PHY مش موجود أو الـ phy-handle في DT غلط | تحقق من `phy-handle` في الـ DT، وتأكد أن الـ MDIO bus probe نجح |
| `cannot probe MDIO bus` | فشل `emac_mdio_probe` — الـ phy-handle مش موجود أو الـ driver مش loaded | تأكد أن `CONFIG_PHYLIB` مفعّل وأن الـ DT صح |
| `dma mapping error` | `dma_map_single` فشلت — مشكلة في الـ IOMMU أو الـ memory | تحقق من `CONFIG_DMA_API_DEBUG`، وتأكد أن الـ DMA mask صح |
| `prepare slave single failed` | `dmaengine_prep_slave_single` رجعت NULL — الـ DMA channel مش جاهز | تحقق من الـ DMA controller driver، وتأكد أن الـ channel `rx` موجود في DT |
| `alloc emac dma req error` | `kzalloc` فشلت — memory ضغط عالي | راجع `/proc/meminfo`، ابحث عن memory leaks |
| `dma submit error` | `dma_submit_error` — الـ cookie سالب | الـ DMA engine في حالة خطأ، restart الـ channel |
| `failed to request dma channel. dma is disabled` | `dma_request_chan` فشلت، الـ driver هيكمّل بدون DMA | تحقق من DT أن `dmas` و`dma-names = "rx"` موجودين |
| `Error couldn't enable clock` | `clk_prepare_enable` فشلت | تحقق من الـ clock tree في DT والـ CCU driver |
| `Error couldn't map SRAM to device` | `sunxi_sram_claim` فشلت — الـ SRAM محجوز لجهاز تاني | تحقق من الـ sram DT node وأن `CONFIG_SUNXI_SRAM` مفعّل |
| `No irq resource` | `irq_of_parse_and_map` فشلت — الـ interrupts property غايبة من DT | أضف `interrupts` property في الـ DT node |
| `failed to remap registers` | `of_iomap` فشلت — الـ reg property غلط أو memory مش متاحة | تحقق من `reg` property في DT |
| `tx time out.` | الـ TX stuck — الـ hardware مش بيكمّل الإرسال | الـ driver بيعمل reset تلقائي؛ راجع الـ PHY link وإشارة الـ cable |
| `ab : %x` | TX abort interrupt — packet اتبعتت وفشلت بسبب collision أو error | مشكلة في الـ link أو الـ cable، راجع الـ PHY status |
| `receive header: %x` — والقيمة ≠ `0x0143414d` | الـ EMAC_UNDOCUMENTED_MAGIC مش متطابق — الـ RX FIFO corrupted | الـ driver بيعمل flush للـ FIFO تلقائياً، لكن لو متكرر راجع الـ clock والـ SRAM |
| `crc error` | الـ rxstatus فيه `EMAC_RX_IO_DATA_STATUS_CRC_ERR` | مشكلة في الـ PHY أو الـ cable، أو noise على الـ line |
| `length error` | الـ rxstatus فيه `EMAC_RX_IO_DATA_STATUS_LEN_ERR` | الـ packet size مش متطابق مع الـ header |
| `Registering netdev failed!` | `register_netdev` فشلت | تحقق من الـ MAC address ومن الـ netdev resources |
| `RX: Bad Packet (runt)` | الـ packet أقل من 64 byte | مشكلة في الطرف التاني أو الـ PHY |

---

#### 8. نقاط استراتيجية لـ dump_stack() / WARN_ON()

```c
/* في emac_interrupt() — للتحقق من أن الـ interrupt مش بييجي من context غلط */
WARN_ON(!in_interrupt());

/* في emac_dma_done_callback() — للتحقق من أن الـ skb ملوش مشكلة */
WARN_ON(!skb);
WARN_ON(skb->len == 0);

/* في emac_start_xmit() — للتحقق من الـ spinlock */
WARN_ON(!spin_is_locked(&db->lock));

/* في emac_rx() — لو الـ magic value جه غلط كتير */
if (reg_val != EMAC_UNDOCUMENTED_MAGIC) {
    dev_warn(db->dev, "bad RX magic: 0x%08x\n", reg_val);
    dump_stack();  /* لو بدنا full trace */
}

/* في emac_dma_inblk_32bit() — بعد dma_mapping_error */
if (ret) {
    WARN_ONCE(1, "DMA mapping failed: %d\n", ret);
    dump_stack();
}

/* في emac_probe() — بعد sunxi_sram_claim فشل */
WARN_ON(ret);  /* بدل الـ goto العادي أثناء التطوير */
```

---

### Hardware Level

---

#### 1. التحقق من حالة الـ Hardware مقارنةً بحالة الـ Kernel

```bash
# قارن MAC address اللي في الـ kernel مع اللي في الـ registers
# الـ kernel يقول:
cat /sys/class/net/eth0/address

# الـ hardware يقول (بعد معرفة base address من DT):
# EMAC_MAC_A1_REG = base + 0x9c  → bytes [0:2]
# EMAC_MAC_A0_REG = base + 0x98  → bytes [3:5]
devmem2 0x1c0b09c w   # byte 0,1,2 بالـ MSB
devmem2 0x1c0b098 w   # byte 3,4,5

# قارن الـ link state:
# kernel:
cat /sys/class/net/eth0/operstate
# hardware (PHY register MR1 - status):
phytool read eth0/0 1   # يقرا PHY register 1

# قارن الـ speed setting:
# kernel: db->speed
ethtool eth0 | grep Speed
# hardware: EMAC_MAC_SUPP_REG bit 8
devmem2 0x1c0b074 w   # EMAC_MAC_SUPP_REG
```

---

#### 2. Register Dump Techniques

**الـ EMAC base address للـ A10:** `0x01C0B000`

**جدول الـ registers المهمة:**

| Register | Offset | الوظيفة |
|---|---|---|
| `EMAC_CTL_REG` | `0x00` | التحكم العام (RX/TX enable, reset) |
| `EMAC_TX_MODE_REG` | `0x04` | TX mode |
| `EMAC_TX_CTL0_REG` | `0x0c` | TX channel 0 control |
| `EMAC_TX_CTL1_REG` | `0x10` | TX channel 1 control |
| `EMAC_TX_STA_REG` | `0x20` | TX status |
| `EMAC_RX_CTL_REG` | `0x3c` | RX control |
| `EMAC_RX_STA_REG` | `0x48` | RX status |
| `EMAC_RX_FBC_REG` | `0x50` | عدد الـ packets في RX FIFO |
| `EMAC_INT_CTL_REG` | `0x54` | Interrupt control |
| `EMAC_INT_STA_REG` | `0x58` | Interrupt status |
| `EMAC_MAC_CTL0_REG` | `0x5c` | MAC control 0 |
| `EMAC_MAC_CTL1_REG` | `0x60` | MAC control 1 (duplex, CRC, pad) |
| `EMAC_MAC_MCFG_REG` | `0x7c` | MII clock config |
| `EMAC_MAC_A0_REG` | `0x98` | MAC address bytes 3-5 |
| `EMAC_MAC_A1_REG` | `0x9c` | MAC address bytes 0-2 |

```bash
# dump كامل للـ EMAC registers (devmem2)
BASE=0x01C0B000
for offset in 0x00 0x04 0x08 0x0c 0x10 0x14 0x18 0x1c \
              0x20 0x24 0x3c 0x40 0x44 0x48 0x4c 0x50 \
              0x54 0x58 0x5c 0x60 0x64 0x68 0x6c 0x70 \
              0x74 0x7c 0x98 0x9c; do
    addr=$((BASE + offset))
    printf "0x%08x (offset 0x%02x): " $addr $offset
    devmem2 $addr w 2>/dev/null | grep "Read at"
done

# أو باستخدام io utility (من package 'ioutil' أو 'ioport')
io -4 -r 0x01C0B000 40

# أو باستخدام /dev/mem مع Python
python3 -c "
import mmap, os, struct
EMAC_BASE = 0x01C0B000
PAGE_SIZE = 4096
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), PAGE_SIZE, offset=EMAC_BASE)
    regs = ['CTL','TX_MODE','TX_FLOW','TX_CTL0','TX_CTL1','TX_INS',
            'TX_PL0','TX_PL1','TX_STA','TX_IO']
    for i, name in enumerate(regs):
        val = struct.unpack('<I', m[i*4:(i+1)*4])[0]
        print(f'EMAC_{name}_REG: 0x{val:08x}')
    m.close()
"
```

**قراءة الـ EMAC_INT_STA_REG لتشخيص الـ interrupts:**

```bash
# اقرا interrupt status register
devmem2 0x01C0B058 w
# bit 0: TX0 complete
# bit 1: TX1 complete
# bit 2: TX0 abort
# bit 3: TX1 abort
# bit 8: RX complete
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس على الـ MII/RMII bus:**

```
MII Interface pins (RMII on A10):
┌─────────────────────────────────────────┐
│  Signal      │ Direction │ الوصف        │
├─────────────────────────────────────────┤
│  RMII_TXD0   │ MAC→PHY   │ TX data bit0 │
│  RMII_TXD1   │ MAC→PHY   │ TX data bit1 │
│  RMII_TXEN   │ MAC→PHY   │ TX enable    │
│  RMII_RXD0   │ PHY→MAC   │ RX data bit0 │
│  RMII_RXD1   │ PHY→MAC   │ RX data bit1 │
│  RMII_CRSDV  │ PHY→MAC   │ Carrier/DV   │
│  RMII_RXERR  │ PHY→MAC   │ RX error     │
│  RMII_REFCLK │ External  │ 50MHz ref    │
│  MDC         │ MAC→PHY   │ MDIO clock   │
│  MDIO        │ Bidir     │ MDIO data    │
└─────────────────────────────────────────┘
```

**نصائح عملية:**
- قِس الـ `RMII_REFCLK` — لازم تكون 50 MHz ثابتة. لو مفيش clock، مفيش اتصال.
- لو الـ `RMII_CRSDV` مش بييجي high وقت الإرسال، مشكلة في الـ PHY أو الـ cable.
- راقب الـ `MDC` / `MDIO` للتأكد أن الـ MDIO transactions بتتم صح — الـ driver بيكتب الـ MII clock divider في `EMAC_MAC_MCFG_REG` (قيمة `0x0d << 2` = divide by 72).
- لو شايف glitches على الـ RMII_REFCLK، راجع مصدر الـ clock في `EMAC_CLK_REG` في الـ CCU (Clock Control Unit).

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ Log | التشخيص |
|---|---|---|
| الـ PHY مش موجود أو مش متصل | `could not find the PHY` | تحقق أن الـ MDIO bus يرى الـ PHY: `cat /sys/bus/mdio_bus/devices/*/phy_id` |
| الـ RMII clock مش شغال | `Error couldn't enable clock` ثم لا يوجد link | تحقق من الـ CCU registers وأن `clk_prepare_enable` نجحت |
| الـ SRAM محجوز | `Error couldn't map SRAM to device` | الـ SRAM اللي EMAC بيحتاجه محجوز لـ CPU أو جهاز تاني، راجع الـ DT |
| الـ cable مقطوع / PHY مش شغال | لا يوجد link-up بعد `phy_start` | `ethtool eth0` يقول `Link detected: no`، راجع الـ cable والـ PHY power |
| CRC errors متكررة | `crc error` / `rx_crc_errors` يزيد | noise أو cable رديء أو duplex mismatch |
| RX FIFO corruption | `receive header: 0xXXXXXXXX` (magic غلط) | الـ SRAM timing أو الـ clock غلط، راجع الـ voltage |
| TX stuck / timeout | `tx time out.` بشكل متكرر | الـ PHY في حالة خطأ أو الـ TX FIFO ممتلئ ومش بيتفرّغ |
| DMA مش شغال | `failed to request dma channel` | الـ DMA controller driver مش loaded أو الـ DT ناقص `dmas` property |

---

#### 5. Device Tree Debugging

**الـ DT المتوقع للـ sun4i-emac:**

```dts
/* مثال DT صح للـ A10 EMAC */
emac: ethernet@1c0b000 {
    compatible = "allwinner,sun4i-a10-emac";
    reg = <0x01c0b000 0x1000>;            /* base + size */
    interrupts = <55>;                      /* IRQ number */
    clocks = <&ahb_gates 17>;             /* AHB clock gate */
    phy-handle = <&phy0>;
    /* أو القديم: phy = <&phy0>; */

    /* DMA للـ RX path */
    dmas = <&dma SUN4I_DMA_DEDICATED 27>;
    dma-names = "rx";

    /* SRAM mapping */
    allwinner,sram = <&emac_sram 1>;
};

mdio: mdio@1c0b000 {
    phy0: ethernet-phy@0 {
        reg = <0>;
    };
};
```

**التحقق من الـ DT المحمّل:**

```bash
# شوف الـ DT اللي اتحمّل فعلاً
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | grep -A 20 "1c0b000"

# تحقق من أن الـ compatible string صح
cat /sys/firmware/devicetree/base/soc/ethernet@1c0b000/compatible
# المتوقع: allwinner,sun4i-a10-emac

# تحقق من الـ reg property
xxd /sys/firmware/devicetree/base/soc/ethernet@1c0b000/reg
# المتوقع: 01 c0 b0 00 00 00 10 00

# تحقق من الـ IRQ
cat /sys/firmware/devicetree/base/soc/ethernet@1c0b000/interrupts | xxd

# تحقق من وجود الـ phy-handle
ls /sys/firmware/devicetree/base/soc/ethernet@1c0b000/ | grep phy

# تحقق من الـ DMA names
cat /sys/firmware/devicetree/base/soc/ethernet@1c0b000/dma-names
# المتوقع: rx

# تحقق من الـ SRAM claim
cat /proc/iomem | grep -i sram
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

**1. تشخيص سريع شامل:**

```bash
#!/bin/bash
# sun4i-emac quick diagnosis script
IFACE=eth0
BASE=0x01C0B000

echo "=== Interface Status ==="
ip -s -s link show $IFACE

echo ""
echo "=== ethtool Info ==="
ethtool $IFACE
ethtool -i $IFACE

echo ""
echo "=== PHY Device ==="
ls /sys/class/net/$IFACE/phydev/ 2>/dev/null
cat /sys/class/net/$IFACE/phydev/phy_id 2>/dev/null

echo ""
echo "=== RX/TX Stats ==="
cat /sys/class/net/$IFACE/statistics/rx_packets
cat /sys/class/net/$IFACE/statistics/rx_errors
cat /sys/class/net/$IFACE/statistics/rx_crc_errors
cat /sys/class/net/$IFACE/statistics/tx_packets
cat /sys/class/net/$IFACE/statistics/tx_errors
```

**2. تفعيل الـ debug messages بالكامل:**

```bash
# عبر ethtool
ethtool -s eth0 msglvl 0xFFFF

# عبر dynamic debug
echo 'module sun4i_emac +pflmt' > /sys/kernel/debug/dynamic_debug/control

# تابع الـ dmesg
dmesg -w | grep -E "(sun4i|emac|eth0)"
```

**3. dump جميع الـ EMAC registers:**

```bash
BASE=0x01C0B000
declare -A REGS=(
    [0x00]="CTL"        [0x04]="TX_MODE"   [0x0c]="TX_CTL0"
    [0x10]="TX_CTL1"    [0x14]="TX_INS"    [0x18]="TX_PL0"
    [0x1c]="TX_PL1"     [0x20]="TX_STA"    [0x3c]="RX_CTL"
    [0x48]="RX_STA"     [0x50]="RX_FBC"    [0x54]="INT_CTL"
    [0x58]="INT_STA"    [0x5c]="MAC_CTL0"  [0x60]="MAC_CTL1"
    [0x64]="MAC_IPGT"   [0x68]="MAC_IPGR"  [0x6c]="MAC_CLRT"
    [0x70]="MAC_MAXF"   [0x74]="MAC_SUPP"  [0x7c]="MAC_MCFG"
    [0x98]="MAC_A0"     [0x9c]="MAC_A1"
)
for offset in "${!REGS[@]}"; do
    addr=$(printf "0x%08x" $((BASE + offset)))
    name=${REGS[$offset]}
    val=$(devmem2 $addr w 2>/dev/null | awk '/Read at/{print $NF}')
    printf "EMAC_%-12s [%s] = %s\n" "$name" "$addr" "$val"
done | sort -k3
```

**4. مراقبة الـ interrupts في real-time:**

```bash
# شوف الـ IRQ number أول
cat /proc/interrupts | grep sun4i

# راقب الـ interrupt count
watch -n 1 "cat /proc/interrupts | grep -E '(CPU|sun4i|emac)'"
```

**5. اختبار الـ DMA channel:**

```bash
# تحقق أن الـ DMA channel اتطلب
dmesg | grep -i "dma"
cat /sys/kernel/debug/dmaengine/summary

# لو DMA مش شغال، الـ driver هيكمّل بـ PIO (emac_inblk_32bit)
# تحقق من log:
dmesg | grep "configure dma failed"
```

**6. تشخيص مشكلة RX magic value:**

```bash
# فعّل debug messages للـ RX
ethtool -s eth0 msglvl 0x0280   # RX_STATUS + RX_ERR

# تابع الـ dmesg
dmesg -w | grep -E "(receive header|rxhdr|RXCount|RxLen)"

# الـ output المتوقع لو كل شيء تمام:
# [ 12.345] sun4i-emac eth0: RXCount: 1
# [ 12.346] sun4i-emac eth0: receive header: 143414d  ← الـ magic
# [ 12.347] sun4i-emac eth0: rxhdr: XXXX
# [ 12.348] sun4i-emac eth0: RX: status 80, length 005c
```

**7. تشخيص TX timeout:**

```bash
# راقب الـ watchdog timer
ethtool -s eth0 msglvl 0x0008   # TIMER messages

# تابع:
dmesg -w | grep "tx time out"

# لو بيحصل كتير، ارفع الـ watchdog timeout
# عن طريق module param (لازم reload):
rmmod sun4i_emac
modprobe sun4i_emac watchdog=10000   # 10 ثواني بدل 5
```

**8. تشخيص مشاكل الـ PHY:**

```bash
# dump PHY registers
ethtool -d eth0

# قرا PHY registers يدوي
phytool read eth0/0 0   # Control register
phytool read eth0/0 1   # Status register  (bit 2 = link up)
phytool read eth0/0 4   # ANAR (autoneg advertisement)
phytool read eth0/0 5   # ANLPAR (link partner ability)

# restart autoneg
phytool write eth0/0 0 0x1200

# أو عبر ethtool
ethtool -r eth0   # restart autoneg
```

**9. مثال output وتفسيره:**

```
# ethtool eth0
Settings for eth0:
        Supported ports: [ TP MII ]
        Supported link modes: 10baseT/Half 10baseT/Full
                              100baseT/Half 100baseT/Full   ← الـ EMAC يدعم max 100M
        Speed: 100Mb/s                                      ← الـ link اشتغل بـ 100M
        Duplex: Full                                        ← Full duplex
        Auto-negotiation: on
        Port: MII
        PHYAD: 0
        Transceiver: internal
        Link detected: yes                                  ← الـ link شغال

# ip -s -s link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP
    RX: bytes  packets  errors  dropped overrun mcast
        1234567 10000    0       0       0       0        ← صفر errors = ممتاز
    TX: bytes  packets  errors  dropped carrier collsns
        987654  8000     0       0       0       0

# لو شفت rx_crc_errors بيزيد:
#   → مشكلة cable أو duplex mismatch أو PHY voltage
# لو شفت tx_aborted_errors بيزيد:
#   → collision أو link unstable أو FIFO overflow
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Allwinner H616 — IoT Gateway — مشكلة الـ SRAM Claim

#### العنوان
الـ EMAC مش بيشتغل على board جديدة بـ H616 — فشل في `sunxi_sram_claim`

#### السياق
Engineer بيعمل bring-up لـ industrial IoT gateway بيشتغل على Allwinner H616 SoC. الـ board مخصصة لتجميع بيانات من حساسات عبر Ethernet وإرسالها لـ cloud. بعد تحميل الـ kernel، الـ `eth0` مش بتظهر خالص.

#### المشكلة
الـ `dmesg` بيطلع:

```
sun4i-emac 1c0b000.ethernet: Error couldn't map SRAM to device
```

الـ interface مش بتتسجل، والمشروع واقف.

#### التحليل
في `emac_probe()` السطر 1020:

```c
ret = sunxi_sram_claim(&pdev->dev);
if (ret) {
    dev_err(&pdev->dev, "Error couldn't map SRAM to device\n");
    goto out_clk_disable_unprepare;
}
```

الـ EMAC في Allwinner بيحتاج SRAM داخلي خاص به كـ TX/RX buffer. الـ `sunxi_sram_claim()` بتطلب إن الـ DT يحتوي على `allwinner,sram` phandle صحيح يشاور على الـ SRAM region المخصصة للـ EMAC. لو الـ DTS مكتوبة غلط أو مش بيوصل للـ SRAM controller، الـ probe بيفشل.

Flow الفشل:
```
emac_probe()
  └─> sunxi_sram_claim()        ← بتتصل بـ sunxi SRAM driver
        └─> of_parse_phandle()  ← بتدور على "allwinner,sram" في DT
              └─> ENODEV أو EINVAL لو الـ phandle مش موجود
```

#### الحل
تحقق من الـ DTS:

```bash
# شوف الـ SRAM node موجود صح
dtc -I dtb -O dts /boot/dtbs/allwinner/sun50i-h616-board.dtb | grep -A5 sram
```

الـ DTS المطلوبة:

```dts
/* SRAM controller node */
sram-controller@3000000 {
    compatible = "allwinner,sun50i-h616-system-control";
    reg = <0x03000000 0x1000>;
    #address-cells = <1>;
    #size-cells = <1>;

    sram_c: sram@28000 {
        compatible = "mmio-sram";
        reg = <0x00028000 0x30000>;
        #address-cells = <1>;
        #size-cells = <1>;
        ranges = <0 0x00028000 0x30000>;

        emac_sram: sram-section@0 {
            compatible = "allwinner,sun50i-h616-sram-c",
                         "allwinner,sun4i-a10-sram-c1";
            reg = <0x0 0x4000>;
        };
    };
};

/* EMAC node */
emac: ethernet@5020000 {
    compatible = "allwinner,sun4i-a10-emac";
    reg = <0x05020000 0x10000>;
    interrupts = <GIC_SPI 82 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&ccu CLK_BUS_EMAC>;
    resets = <&ccu RST_BUS_EMAC>;
    allwinner,sram = <&emac_sram 1>;  /* ← هنا المشكلة لو مفقودة */
    phy-handle = <&ext_rgmii_phy>;
    status = "okay";
};
```

#### الدرس المستفاد
الـ `sun4i-emac` driver مش بيشتغل بدون الـ SRAM السحري. الـ SRAM بيُستخدم كـ packet buffer داخلي للـ EMAC hardware. أي board جديدة لازم الـ DTS يكون فيها `allwinner,sram` phandle صريح وصحيح — مش optional.

---

### السيناريو 2: Allwinner A10 — Android TV Box — TX Timeout متكرر

#### العنوان
الـ `eth0` بيوقف كل بضع دقايق وبيرجع — TX watchdog timeout

#### السياق
Android TV box قديمة على Allwinner A10 (sun4i) بتستخدم `sun4i-emac`. المستخدمين بيشتكوا إن الـ streaming بيتقطع كل 3–5 دقايق ثم بيرجع لوحده. الـ logcat بيظهر network drops.

#### المشكلة
الـ `dmesg` بيطلع:

```
sun4i-emac eth0: tx time out.
```

وبعدها الـ driver بيعمل reset وبيرجع يشتغل، لكن المشكلة بترجع.

#### التحليل
الـ `emac_timeout()` السطر 516:

```c
static void emac_timeout(struct net_device *dev, unsigned int txqueue)
{
    struct emac_board_info *db = netdev_priv(dev);
    unsigned long flags;

    if (netif_msg_timer(db))
        dev_err(db->dev, "tx time out.\n");

    spin_lock_irqsave(&db->lock, flags);
    netif_stop_queue(dev);
    emac_reset(db);        /* hard reset للـ EMAC */
    emac_init_device(dev); /* إعادة إعداد الـ MAC وتفعيل interrupts */
    netif_trans_update(dev);
    netif_wake_queue(dev);
    spin_unlock_irqrestore(&db->lock, flags);
}
```

السبب الحقيقي غالبًا في `emac_start_xmit()` السطر 547:

```c
channel = db->tx_fifo_stat & 3;
if (channel == 3)
    return NETDEV_TX_BUSY;  /* كلا الـ TX channels ممتلئين */
```

لما `tx_fifo_stat == 3` الـ queue بتوقف. لو جاء interrupt تأخر أو اتسقط بسبب shared IRQ، الـ `emac_tx_done()` مش بتتنادى، وبالتالي `tx_fifo_stat` مش بيتصفى، والـ queue فاضلة واقفة لـ `watchdog` ثانية (الـ default 5000ms).

فحص الـ interrupt:
```bash
cat /proc/interrupts | grep emac
# لو العداد واقف — الـ interrupt مش وصال
```

#### الحل

```bash
# زيادة الـ watchdog timeout مؤقتًا لتأكيد السبب
modprobe sun4i_emac watchdog=10000

# تفعيل debug messages
ethtool -s eth0 msglvl 0xffff
# ثم شوف dmesg
dmesg | grep -i emac
```

لو المشكلة إن الـ IRQ line مشترك وبيتأخر:

```dts
ethernet@1c0b000 {
    interrupts = <76 IRQ_TYPE_LEVEL_HIGH>;  /* تأكد من الـ trigger type */
};
```

تأكيد الإصلاح:
```bash
# شوف TX stats
ethtool -S eth0 | grep tx
ip -s link show eth0
```

#### الدرس المستفاد
الـ `tx_fifo_stat` هو semaphore يدوي بيتحكم في الـ dual TX FIFO. أي تأخير في IRQ delivery بيسبب deadlock مؤقت ينتهي فقط بالـ watchdog. على platforms القديمة لازم تتحقق من الـ IRQ trigger type في الـ DT وتتأكد مش في IRQ storm من peripheral تاني.

---

### السيناريو 3: Allwinner A10 — Custom Board Bring-up — EMAC_UNDOCUMENTED_MAGIC panic

#### العنوان
الـ RX packets دايمًا بتتسقط — FIFO flush loop ما بتخلص

#### السياق
Engineer بيعمل bring-up لـ custom industrial controller board بـ A10. الـ hardware مشابه لـ reference design لكن فيه تعديل في الـ PHY connection. الـ `eth0` بتظهر وبتاخد link، لكن `ping` مش بيشتغل — الـ packets بتتبعت لكن الـ RX صفر.

#### المشكلة
الـ `tcpdump` على board تانية بيشوف الـ packets وصلت. لكن الـ board نفسها مش بتستقبل. الـ `ip -s link show eth0` بيطلع:

```
RX: bytes  packets  errors  dropped
    0      0        0       0
```

صفر خالص، كأن الـ interrupt بيجي بس الـ RX processing مش بتحصل.

#### التحليل
في `emac_rx()` السطر 651:

```c
reg_val = readl(db->membase + EMAC_RX_IO_DATA_REG);
if (reg_val != EMAC_UNDOCUMENTED_MAGIC) {
    /* disable RX */
    reg_val = readl(db->membase + EMAC_CTL_REG);
    writel(reg_val & ~EMAC_CTL_RX_EN, db->membase + EMAC_CTL_REG);

    /* Flush RX FIFO */
    reg_val = readl(db->membase + EMAC_RX_CTL_REG);
    writel(reg_val | (1 << 3), db->membase + EMAC_RX_CTL_REG);

    do {
        reg_val = readl(db->membase + EMAC_RX_CTL_REG);
    } while (reg_val & (1 << 3));  /* ← busy-wait loop */

    /* enable RX */
    ...
    return;
}
```

الـ `EMAC_UNDOCUMENTED_MAGIC` هو `0x0143414D` ("MAC\x01" بـ little-endian). ده الـ magic word اللي الـ hardware بيحطه في أول الـ RX FIFO قبل كل packet. لو الـ hardware غلط أو الـ SRAM مش متوصل صح، القيمة دي مش بتيجي صح، وبالتالي كل packet بتتعمل لها FIFO flush وبتترمى.

**السبب الجذري**: الـ MII clock divider مش صح، فالـ PHY بيرجع link لكن الـ MAC header بيتكرب:

```c
/* في emac_powerup() */
reg_val |= EMAC_MAC_MCFG_MII_CLKD_72;  /* clock divider = 72 */
```

لو الـ AHB clock الفعلي مختلف عن المفروض، الـ MDC/MDIO communication بتتعطل.

#### الحل

```bash
# راقب الـ RX errors عبر ethtool
ethtool -S eth0

# فعل debug كامل
echo 'module sun4i_emac +p' > /sys/kernel/debug/dynamic_debug/control
# أو
modprobe sun4i_emac debug=0xffff
dmesg | grep "receive header"
# لو بتشوف values غير 0x0143414D باستمرار → المشكلة في SRAM/clock
```

تحقق من الـ AHB clock:

```bash
cat /sys/kernel/debug/clk/ahb/clk_rate
# لو AHB > 200MHz → divider 72 مش كافي للـ MDC
```

الإصلاح في الـ DT أو إضافة clock constraint:

```dts
ethernet@1c0b000 {
    clocks = <&ahb_gates 9>;
    clock-names = "ahb";
    /* تأكد الـ AHB clock ضمن الـ range المدعوم */
};
```

#### الدرس المستفاد
الـ `EMAC_UNDOCUMENTED_MAGIC` (`0x0143414D`) ليه اسم يدل على إن الـ Allwinner ما وثقوهوش رسميًا. هو في الحقيقة packet preamble marker. أي مشكلة في الـ clock أو SRAM بتخلي الـ FIFO يرجع garbage بدل الـ magic — وكل الـ packets بتترمى صامتة.

---

### السيناريو 4: Allwinner A10 — Embedded Linux — مشكلة DMA لـ Jumbo Frames

#### العنوان
الـ packets الكبيرة بتتسقط — DMA channel مش بيتعمل configure صح

#### السياق
Engineer بيطور نظام تحكم صناعي بـ Allwinner A10 وبيستخدم Modbus/TCP على Ethernet. الـ small packets (Modbus ~256 byte) شغالة تمام، لكن لما بيحاول ينقل firmware updates عبر TFTP (packets ~1468 byte) السرعة بتتراجع بشكل كبير وفيه packet loss.

#### المشكلة
```bash
tftp -g -r firmware.bin 192.168.1.100
# بيخلص بعد ساعة بدل دقيقة، وفيه retransmissions كتير
```

#### التحليل
في `emac_rx()` السطر 734:

```c
if (rxlen >= dev->mtu && db->rx_chan) {
    reg_val = readl(db->membase + EMAC_RX_CTL_REG);
    reg_val |= EMAC_RX_CTL_DMA_EN;
    writel(reg_val, db->membase + EMAC_RX_CTL_REG);
    if (!emac_dma_inblk_32bit(db, skb, rdptr, rxlen))
        break;  /* نجح DMA → اخرج من الـ while loop */

    /* لو DMA فشل → fallback لـ CPU copy */
    reg_val = readl(db->membase + EMAC_RX_CTL_REG);
    reg_val &= ~EMAC_RX_CTL_DMA_EN;
    writel(reg_val, db->membase + EMAC_RX_CTL_REG);
}
emac_inblk_32bit(...)  /* CPU copy */
```

الـ `emac_configure_dma()` بتحتاج `dmas = <&dma ...>` في الـ DT. لو الـ DMA channel مش متحدد، الـ `db->rx_chan` بيبقى `NULL` وكل الـ packets الكبيرة بتتنسخ بـ CPU — بطيء جدًا وبيرهق الـ system أثناء الـ interrupt.

فحص الـ DMA:
```bash
# شوف لو الـ DMA channel اتطلب
dmesg | grep -i dma
# لو فيه: "configure dma failed. disable dma." → الـ DT ناقصة
dmesg | grep sun4i-emac
```

في `emac_probe()` السطر 1005:
```c
if (emac_configure_dma(db))
    netdev_info(ndev, "configure dma failed. disable dma.\n");
/* لاحظ: مش fatal — الـ probe بيكمل بدون DMA */
```

#### الحل

أضف الـ DMA في الـ DTS:

```dts
dma: dma-controller@1c02000 {
    compatible = "allwinner,sun4i-a10-dma";
    reg = <0x01c02000 0x1000>;
    interrupts = <27>;
    clocks = <&ahb_gates 6>;
    #dma-cells = <2>;
};

ethernet@1c0b000 {
    compatible = "allwinner,sun4i-a10-emac";
    reg = <0x01c0b000 0x1000>;
    interrupts = <55>;
    clocks = <&ahb_gates 9>;
    dmas = <&dma 1 28>;      /* ← DMA channel للـ RX */
    dma-names = "rx";
    phy-handle = <&phy0>;
    status = "okay";
};
```

التحقق بعد الإصلاح:

```bash
# شوف CPU usage أثناء TFTP
top -d 1
# قبل الإصلاح: CPU 80%+ أثناء النقل
# بعد الإصلاح: CPU < 20%

# تحقق من الـ DMA stats
cat /sys/kernel/debug/dmaengine/summary
```

#### الدرس المستفاد
الـ DMA RX في `sun4i-emac` اختياري — الـ driver بيشتغل بدونه لكن بأداء سيء للـ packets الكبيرة (أكبر من أو يساوي الـ MTU). هذا الفشل الصامت مقصود في الكود (`netdev_info` مش `netdev_err`) لكنه بيسبب performance degradation مش واضح السبب. دايمًا فعل الـ DMA channel في الـ DTS على الـ boards اللي بتنقل بيانات كبيرة.

---

### السيناريو 5: Allwinner A10 — Smart Meter — MAC Address عشوائي بعد كل reboot

#### العنوان
الـ MAC address بيتغير مع كل boot — مشكلة في الـ DT أو الـ EEPROM

#### السياق
شركة بتصنع smart electricity meters بـ Allwinner A10 وبتستخدم `sun4i-emac` للاتصال بـ DLMS/COSEM server. بعد firmware update، الـ meters بدأت تاخد MAC addresses عشوائية مع كل reboot — وده بيكسر الـ DHCP reservations والـ network management system.

#### المشكلة
```bash
ip link show eth0
# أول boot:  link/ether aa:bb:cc:dd:ee:ff
# تاني boot: link/ether 12:34:56:78:9a:bc  (مختلف!)
dmesg | grep MAC
# sun4i-emac eth0: using random MAC address aa:bb:cc:dd:ee:ff
```

#### التحليل
في `emac_probe()` السطر 1036:

```c
/* Read MAC-address from DT */
ret = of_get_ethdev_address(np, ndev);
if (ret) {
    /* if the MAC address is invalid get a random one */
    eth_hw_addr_random(ndev);
    dev_warn(&pdev->dev, "using random MAC address %pM\n",
             ndev->dev_addr);
}
```

الـ `of_get_ethdev_address()` بتدور على `local-mac-address` أو `mac-address` property في الـ DT node. لو مش موجودين أو الـ format غلط، بيعمل random MAC.

بعد كده في `emac_powerup()` السطر 462:

```c
/* set mac_address to chip */
writel(ndev->dev_addr[0] << 16 | ndev->dev_addr[1] << 8 | ndev->dev_addr[2],
       db->membase + EMAC_MAC_A1_REG);
writel(ndev->dev_addr[3] << 16 | ndev->dev_addr[4] << 8 | ndev->dev_addr[5],
       db->membase + EMAC_MAC_A0_REG);
```

الـ MAC بيتكتب في الـ registers لكن مش في EEPROM أو persistent storage — فمع كل reboot بيرجع يولد random.

فحص الـ DT:
```bash
dtc -I dtb -O dts /boot/dtbs/*.dtb | grep -A3 ethernet
# لو مفيش local-mac-address → المشكلة هنا
```

#### الحل

**الحل 1**: ثبت الـ MAC في الـ DT (مناسب للـ production):

```dts
ethernet@1c0b000 {
    compatible = "allwinner,sun4i-a10-emac";
    local-mac-address = [00 11 22 33 44 55];  /* MAC ثابت لكل device */
    phy-handle = <&phy0>;
    status = "okay";
};
```

**الحل 2**: اقرأ الـ MAC من EEPROM وحطه في kernel cmdline:

```bash
# في U-Boot
mac=$(i2c read 0x50 0x0 6 -s)
setenv bootargs "${bootargs} ethaddr=${mac}"
```

**الحل 3**: استخدم `nvmem` لقراءة MAC من SID (Secure ID) فقرات الـ SoC:

```dts
ethernet@1c0b000 {
    nvmem-cells = <&mac_address>;
    nvmem-cell-names = "mac-address";
};

sid: efuse@1c23800 {
    compatible = "allwinner,sun4i-a10-sid";
    reg = <0x01c23800 0x10>;
    #address-cells = <1>;
    #size-cells = <1>;

    mac_address: mac-address@0 {
        reg = <0x0 0x6>;
    };
};
```

التحقق:

```bash
# بعد الإصلاح
dmesg | grep -i mac
# يجب إن الـ warning "using random MAC" ما يظهرش

# تأكيد الثبات بعد عدة boots
for i in 1 2 3; do reboot; sleep 30; ip link show eth0 | grep ether; done
```

#### الدرس المستفاد
الـ `sun4i-emac` مش عنده EEPROM support فعال في الكود الحالي — الـ `EMAC_EEPROM_MAGIC` موجود في الـ header بس مش مستخدم. في أي product يحتاج MAC ثابت، لازم تحدده صريح في الـ DT أو تقرأه من persistent storage في early boot. الاعتماد على الـ random MAC في production بيكسر الـ network management وبيعمل مشاكل في الـ DHCP leases.

---
## Phase 7: مصادر ومراجع

### مقدمة

الـ `sun4i-emac` driver كُتب بواسطة Stefan Roese وMaxime Ripard، وبيغطي Ethernet controller الموجود في SoC الـ Allwinner A10 (sun4i). المصادر دي بتغطي كل الجوانب من hardware registers لـ kernel networking internals.

---

### مقالات LWN.net

| العنوان | الأهمية |
|---------|---------|
| [ARM: sunxi: Add support for A10 Ethernet controller](https://lwn.net/Articles/551763/) | الـ patch series الأصلية اللي أضافت الـ driver للـ mainline kernel — لازم تقراها |
| [net-next: ethernet: add sun8i-emac driver](https://lwn.net/Articles/694927/) | الجيل الجديد من Allwinner EMAC — مقارنة مفيدة بـ sun4i-emac |
| [net: stmmac: Add Allwinner A20 GMAC ethernet controller](https://lwn.net/Articles/580138/) | Gigabit GMAC على A20 — نفس البيئة، hardware مختلف |
| [net-next: stmmac: add dwmac-sun8i ethernet driver](https://lwn.net/Articles/724248/) | تطور الـ Allwinner networking stack في الـ mainline |
| [phylib: Add support for MDIO clause 45](https://lwn.net/Articles/384726/) | فهم الـ phylib اللي بيستخدمه sun4i-emac |
| [Add common OF device tree support for MDIO busses](https://lwn.net/Articles/326450/) | الـ `of_phy_connect()` وأصدقاؤها — critical للـ driver |
| [PHY Abstraction Layer — kernel documentation](https://static.lwn.net/kerneldoc/networking/phy.html) | الـ reference الرسمي للـ PHY layer اللي بيتعامل معاه الـ driver |

---

### توثيق الـ kernel الرسمي

```
Documentation/networking/phy.rst
Documentation/driver-api/dmaengine/client.rst
Documentation/driver-api/dmaengine/provider.rst
Documentation/driver-api/driver-model/platform.rst
Documentation/devicetree/bindings/net/allwinner,sun4i-emac.txt
Documentation/networking/netdevices.rst
```

الـ **`Documentation/devicetree/bindings/net/allwinner,sun4i-emac.txt`** بيوضح الـ DT properties الصحيحة زي `phy-handle`، `local-mac-address`، والـ clock name.

الـ **`Documentation/driver-api/dmaengine/client.rst`** مهم لفهم `dmaengine_prep_slave_single()` و`dma_async_issue_pending()` اللي بيستخدمهم `emac_dma_inblk_32bit()`.

روابط مباشرة من kernel.org:

- [DMAEngine API Guide](https://docs.kernel.org/driver-api/dmaengine/client.html)
- [DMAEngine controller documentation](https://docs.kernel.org/driver-api/dmaengine/provider.html)
- [Platform Devices and Drivers](https://www.kernel.org/doc/html/latest/driver-api/driver-model/platform.html)
- [ARM Allwinner SoCs — The Linux Kernel documentation](https://docs.kernel.org/arch/arm/sunxi.html)

---

### Kernel Commits المهمة

الـ driver دخل الـ mainline في **Linux 3.11** عن طريق الـ patch series الأصلية بواسطة Maxime Ripard:

- [PATCHv4 1/6 — net: Add EMAC ethernet driver found on Allwinner A10 SoC's](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1369921765-3367-2-git-send-email-maxime.ripard@free-electrons.com/)

الـ patch series الكاملة على netdev mailing list:

- [PATCH 0/5 — ARM: sunxi: Add support for A10 Ethernet controller](https://www.spinics.net/lists/netdev/msg229365.html)

إصلاح SRAM mapping bug:

- [PATCH 3/3 — net: allwinner: sun4i-emac: fix emac SRAM mapping](http://lists.infradead.org/pipermail/linux-arm-kernel/2015-January/320221.html)

---

### Mailing List Discussions

| الرابط | الموضوع |
|--------|---------|
| [linux-arm-kernel — SRAM mapping fix](http://lists.infradead.org/pipermail/linux-arm-kernel/2015-January/320221.html) | إصلاح مشكلة `sunxi_sram_claim()` اللي بيستخدمها الـ driver |
| [netdev — patch 0/5 original series](https://www.spinics.net/lists/netdev/msg229365.html) | نقاش إضافة الـ driver للـ mainline |
| [patchwork — sun8i-emac v2 series](https://patchwork.kernel.org/patch/9239065/) | مقارنة مفيدة بالجيل الجديد من الـ driver |

---

### توثيق Hardware الـ Allwinner

الـ community documentation على linux-sunxi.org هو المرجع الأفضل لفهم الـ registers:

| الصفحة | المحتوى |
|--------|---------|
| [EMAC Register Guide — linux-sunxi.org](https://linux-sunxi.org/A10/EMAC) | توثيق كامل لكل registers الـ EMAC بما فيها `EMAC_RX_IO_DATA_REG` والـ magic header |
| [Ethernet — linux-sunxi.org](https://linux-sunxi.org/Ethernet) | نظرة عامة على الـ Ethernet support في كل Allwinner SoCs |
| [A10 — linux-sunxi.org](https://linux-sunxi.org/A10) | توثيق الـ SoC الكامل بما فيه memory map والـ SRAM |
| [Linux Mainlining Effort — linux-sunxi.org](https://linux-sunxi.org/Linux_mainlining_effort) | تاريخ دخول الـ driver للـ mainline |

---

### eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Allwinner A1X — eLinux.org](https://elinux.org/A1x) | مواصفات الـ A10 hardware بما فيها الـ 10/100 Ethernet MAC |
| [Hack A10 devices — eLinux.org](https://elinux.org/Hack_A10_devices) | معلومات عملية عن تشغيل الـ A10 boards |
| [Supporting a new ARM platform: the Allwinner example (PDF)](https://elinux.org/images/5/51/Ripard--supporting_a_new_arm_platform_the_allwinner_example.pdf) | presentation من Maxime Ripard نفسه — كاتب الـ driver |

---

### KernelNewbies.org

| الصفحة | المحتوى |
|--------|---------|
| [Linux Ethernet Network Device Driver — A flow of code](https://kernelnewbies.org/KaruppuSwamy_Thangaraj/network-drivers-linux2.4) | شرح تفصيلي لـ `struct net_device` وإزاي الـ driver بيسجل نفسه |
| [Linux 3.11 Drivers/Arch changes](https://kernelnewbies.org/Linux_3.11-DriversArch) | الـ kernel version اللي دخل فيه الـ sun4i-emac للـ mainline |

---

### كتب موصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)

| الفصل | الصلة بالـ driver |
|-------|------------------|
| **Chapter 17 — Network Drivers** | الأساس — `net_device_ops`، `sk_buff`، `netif_rx()`، كل اللي بيعمله الـ driver |
| **Chapter 15 — Memory Mapping and DMA** | الـ DMA path في `emac_dma_inblk_32bit()` |
| **Chapter 10 — Interrupt Handling** | `emac_interrupt()`، `spin_lock_irqsave()` |
| **Chapter 12 — PCI Drivers** | platform driver model — مبادئ مشتركة |

متاح مجاناً: [LDD3 على lwn.net](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

| الفصل | الصلة |
|-------|-------|
| **Chapter 7 — Interrupts and Interrupt Handlers** | فهم `emac_interrupt()` والـ spinlock usage |
| **Chapter 8 — Bottom Halves and Deferring Work** | الفرق بين interrupt context وprocess context في الـ RX path |
| **Chapter 15 — The Process Address Space** | فهم `ioremap()` و`of_iomap()` |

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

| الفصل | الصلة |
|-------|-------|
| **Chapter 15 — Embedded Drivers** | platform drivers على embedded SoCs — نفس pattern الـ `emac_probe()` |
| **Chapter 16 — Open Source U-Boot** | bootloader و SRAM initialization اللي بيأثر على `sunxi_sram_claim()` |

#### Understanding Linux Network Internals — Christian Benvenuti

الكتاب الأكتر تخصصاً في الـ networking stack — بيغطي:
- كيف `netif_rx()` بتسلم الـ `sk_buff` للـ upper layers
- الـ `eth_type_trans()` وكيف بيتحدد الـ protocol
- الـ `net_device_stats` وكيف بيتحدثوا

---

### GitHub Repositories

| الـ repo | الاستخدام |
|---------|-----------|
| [torvalds/linux — sun4i-emac.c](https://github.com/torvalds/linux/blob/master/drivers/net/ethernet/allwinner/sun4i-emac.c) | الـ upstream source الحالي |
| [linux-sunxi/linux-sunxi](https://github.com/linux-sunxi/linux-sunxi) | الـ community kernel مع patches إضافية |

---

### Linux Kernel Driver Database

- [CONFIG_SUN4I_EMAC — cateee.net](https://cateee.net/lkddb/web-lkddb/SUN4I_EMAC.html) — بيوضح في أنهي kernel versions الـ config option ظهر

---

### Search Terms للبحث عن مزيد من المعلومات

```
# للبحث عن documentation الـ driver:
sun4i-emac allwinner a10 ethernet linux driver
CONFIG_SUN4I_EMAC kernel
allwinner,sun4i-a10-emac device tree

# للبحث عن networking internals:
linux kernel network device driver sk_buff netif_rx
linux phylib phy_connect mdio bus driver
linux platform_driver probe ethernet

# للبحث عن DMA في الـ driver:
linux dmaengine_prep_slave_single DMA_DEV_TO_MEM
dma_request_chan rx channel ethernet driver
dma_async_issue_pending kernel driver

# للبحث عن sunxi community:
linux-sunxi emac register map
allwinner a10 emac sram mapping
sunxi_sram_claim ethernet driver

# lore.kernel.org للمناقشات:
lore.kernel.org sun4i-emac
lore.kernel.org allwinner emac maxime ripard
```

---

### مصادر إضافية للـ Debugging

الـ `debug` module parameter في الـ driver بيستخدم الـ `netif_msg_*` framework:

```bash
# تفعيل debug messages أثناء runtime:
ethtool -s eth0 msglvl 0xffffffff

# مشاهدة الـ driver messages:
dmesg | grep sun4i-emac

# فحص الـ DMA channel:
cat /proc/interrupts | grep emac
cat /sys/kernel/debug/dmaengine/summary
```

مرجع الـ `msglvl` flags:

- `0x0001` — `NETIF_MSG_DRV` — driver initialization
- `0x0004` — `NETIF_MSG_INTR` — interrupt events
- `0x0020` — `NETIF_MSG_TX_DONE` — TX completions
- `0x0040` — `NETIF_MSG_RX_STATUS` — RX packet status
- `0x0100` — `NETIF_MSG_RX_ERR` — RX errors
- `0x1000` — `NETIF_MSG_IFUP` / `NETIF_MSG_IFDOWN` — interface up/down
## Phase 8: Writing simple module

### الفكرة

**الـ** `emac_start_xmit` هي الـ function اللي بتتسمى كل ما الـ network stack حابب يبعت packet من الـ sun4i-emac NIC. هنعمل **kretprobe** عليها — يعني بنعترض الـ return منها — عشان نشوف:
- إيه الـ `struct sk_buff` اللي اتبعت (الطول + الـ protocol).
- إيه الـ return value (نجاح ولا `NETDEV_TX_BUSY`).

ده آمن لأن الـ kretprobe مش بيحتاج يعدّل الـ function، وبيشتغل على مستوى الـ kernel symbol اللي exported.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kretprobe on emac_start_xmit — monitors every TX attempt on sun4i-emac.
 *
 * Tested on allwinner,sun4i-a10-emac driver (drivers/net/ethernet/allwinner/).
 */

#include <linux/module.h>      /* MODULE_*, module_init/exit         */
#include <linux/kprobes.h>     /* kretprobe, register_kretprobe etc. */
#include <linux/netdevice.h>   /* struct net_device, netdev_tx_t     */
#include <linux/skbuff.h>      /* struct sk_buff                      */
#include <linux/ptrace.h>      /* struct pt_regs (used by kprobe)    */

/* ------------------------------------------------------------------ */
/* entry_handler: runs right BEFORE emac_start_xmit returns           */
/* ------------------------------------------------------------------ */

/*
 * بنحتفظ بالـ skb pointer في الـ per-instance data عشان نعرف نقراه
 * في الـ ret_handler بعد ما الـ function ترجع.
 */
struct emac_xmit_data {
    struct sk_buff *skb;        /* saved pointer to the outgoing packet */
    struct net_device *dev;     /* saved net_device for interface name  */
};

/*
 * entry_handler: بيتنادى لما emac_start_xmit تبدأ تشتغل.
 * بنستخرج args من الـ regs ونحفظهم في الـ ri->data buffer.
 *
 * ABI (ARM/x86-64): arg0 → first param (skb), arg1 → second param (dev).
 * regs_get_kernel_argument() هي الـ portable helper للـ kernel ABI.
 */
static int emac_xmit_entry(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    struct emac_xmit_data *data = (struct emac_xmit_data *)ri->data;

    /* first argument to emac_start_xmit(skb, dev) */
    data->skb = (struct sk_buff *)regs_get_kernel_argument(regs, 0);
    data->dev = (struct net_device *)regs_get_kernel_argument(regs, 1);

    return 0; /* returning non-zero disables this instance */
}

/* ------------------------------------------------------------------ */
/* ret_handler: runs right AFTER emac_start_xmit returns              */
/* ------------------------------------------------------------------ */

/*
 * ret_handler: بيتنادى بعد ما الـ function ترجع.
 * regs_return_value() بتجيب الـ return value من الـ register المناسب.
 * بنطبع info مفيد عن الـ packet اللي بُعتت (أو مُش).
 */
static int emac_xmit_ret(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    struct emac_xmit_data *data = (struct emac_xmit_data *)ri->data;
    netdev_tx_t ret = (netdev_tx_t)regs_return_value(regs);
    struct sk_buff *skb = data->skb;
    const char *devname = data->dev ? data->dev->name : "?";

    if (!skb) {
        pr_info("sun4i_emac_probe: [%s] skb=NULL ret=%d\n", devname, ret);
        return 0;
    }

    /*
     * skb->len       → total packet length in bytes (including headers)
     * skb->protocol  → EtherType (e.g. 0x0800=IPv4, 0x0806=ARP)
     * ret==0         → NETDEV_TX_OK, ret==1 → NETDEV_TX_BUSY
     */
    pr_info("sun4i_emac_probe: [%s] TX len=%u proto=0x%04x ret=%s\n",
            devname,
            skb->len,
            be16_to_cpu(skb->protocol),
            ret == NETDEV_TX_OK   ? "OK"   :
            ret == NETDEV_TX_BUSY ? "BUSY" : "ERR");

    return 0;
}

/* ------------------------------------------------------------------ */
/* kretprobe struct                                                     */
/* ------------------------------------------------------------------ */

/*
 * بنربط الـ entry_handler والـ handler بالـ symbol name بتاع الـ function.
 * data_size بيحجز مكان في كل instance عشان نخزن الـ skb + dev pointers.
 * maxactive=16 يعني بيسمح بـ 16 concurrent call في نفس الوقت على الـ SMP.
 */
static struct kretprobe emac_xmit_kretprobe = {
    .kp.symbol_name = "emac_start_xmit",   /* exact kernel symbol name  */
    .entry_handler  = emac_xmit_entry,     /* called on function entry  */
    .handler        = emac_xmit_ret,       /* called on function return */
    .data_size      = sizeof(struct emac_xmit_data),
    .maxactive      = 16,                  /* max concurrent instances  */
};

/* ------------------------------------------------------------------ */
/* module_init / module_exit                                           */
/* ------------------------------------------------------------------ */

/*
 * module_init: بنسجّل الـ kretprobe هنا.
 * لو الـ symbol مش موجود في الـ kernel (لأن الـ driver مش loaded)
 * register_kretprobe بترجع -ENOENT وبنطبع error واضح.
 */
static int __init emac_probe_init(void)
{
    int ret;

    ret = register_kretprobe(&emac_xmit_kretprobe);
    if (ret < 0) {
        pr_err("sun4i_emac_probe: register_kretprobe failed, ret=%d\n", ret);
        pr_err("sun4i_emac_probe: is the sun4i-emac driver loaded?\n");
        return ret;
    }

    pr_info("sun4i_emac_probe: planted kretprobe on %s\n",
            emac_xmit_kretprobe.kp.symbol_name);
    return 0;
}

/*
 * module_exit: لازم نعمل unregister قبل ما الـ module يتنزل من الـ memory.
 * لو ما عملناش كده، الـ kernel هيتصل بـ handler موجودة في memory اتحررت
 * → kernel panic / use-after-free.
 */
static void __exit emac_probe_exit(void)
{
    unregister_kretprobe(&emac_xmit_kretprobe);
    pr_info("sun4i_emac_probe: kretprobe unregistered, missed=%lu\n",
            emac_xmit_kretprobe.nmissed);
}

module_init(emac_probe_init);
module_exit(emac_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kretprobe on sun4i-emac emac_start_xmit to trace TX packets");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | بيعرّف `struct kretprobe`, `register_kretprobe`, `regs_return_value` |
| `linux/netdevice.h` | بيعرّف `struct net_device`, `netdev_tx_t`, `NETDEV_TX_OK/BUSY` |
| `linux/skbuff.h` | بيعرّف `struct sk_buff` وحقل `len` و`protocol` |
| `linux/ptrace.h` | بيعرّف `struct pt_regs` اللي الـ handlers بياخدوه |
| `linux/module.h` | ماكروهات الـ module الأساسية |

---

#### الـ `emac_xmit_data` struct

**الـ** `kretprobe` بيشتغل على أكتر من CPU في نفس الوقت. الـ kernel بيحجز `data_size` bytes لكل **instance** منفصل، عشان كل CPU يخزن الـ args بتاعته بدون race condition. بنحتاج نحفظ الـ `skb` والـ `dev` لأنهم args في الـ entry وبنقرأهم في الـ return.

---

#### الـ `entry_handler`

**الـ** `regs_get_kernel_argument(regs, N)` بيجيب الـ N-th argument بطريقة portable بين الـ architectures (ARM, x86, RISC-V). بنستخدمه عشان ما نقعدش نعمل cast يدوي للـ regs. الـ function بترجع 0 يعني "proceed normally"، لو رجعت أي حاجة تانية الـ instance ده بيتـskip.

---

#### الـ `ret_handler`

**الـ** `regs_return_value(regs)` بتجيب الـ return value من الـ ABI-specific register (مثلاً `x0` على ARM64، `rax` على x86-64). بنطبع:
- اسم الـ interface (`eth0` مثلاً).
- حجم الـ packet بالـ bytes.
- الـ EtherType بـ hex (IPv4=`0x0800`, ARP=`0x0806`).
- هل الـ send نجح (`OK`) أو الـ FIFO كان مليان (`BUSY`).

---

#### الـ `kretprobe` struct

| حقل | القيمة | السبب |
|---|---|---|
| `kp.symbol_name` | `"emac_start_xmit"` | بيحدد الـ function بالاسم بدون الحاجة للـ address |
| `entry_handler` | `emac_xmit_entry` | بيتنادى قبل الـ return عشان نحفظ الـ args |
| `handler` | `emac_xmit_ret` | بيتنادى بعد الـ return عشان نقرأ النتيجة |
| `data_size` | `sizeof(emac_xmit_data)` | الـ kernel بيحجز ده لكل instance |
| `maxactive` | `16` | أقصى concurrent probes — لو أكتر، الـ kernel بيعدّ في `nmissed` |

---

#### `module_init` و `module_exit`

**الـ** `register_kretprobe` بيحط breakpoint على الـ function entry في الـ kernel memory. لو الـ sun4i-emac driver مش loaded (ولا الـ symbol مش موجود)، بترجع `-ENOENT`.

**الـ** `unregister_kretprobe` في الـ exit **ضروري جداً**: الـ handler pointer موجود في الـ module memory. لو الـ module اتنزل وما عملناش unregister، الـ kernel هيحاول يستدعي function في عنوان اتحرر → **kernel panic**. الـ `nmissed` بيقولنا كام مرة الـ probe اضطر يـskip لأن الـ `maxactive` اتملى.

---

### كيفية الاستخدام

```bash
# build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# load (sun4i-emac must be loaded first)
sudo insmod emac_xmit_probe.ko

# generate traffic and watch output
sudo dmesg -w | grep sun4i_emac_probe

# unload
sudo rmmod emac_xmit_probe
```

**مثال على الـ output:**

```
sun4i_emac_probe: planted kretprobe on emac_start_xmit
sun4i_emac_probe: [eth0] TX len=74 proto=0x0806 ret=OK
sun4i_emac_probe: [eth0] TX len=1514 proto=0x0800 ret=OK
sun4i_emac_probe: [eth0] TX len=60 proto=0x0800 ret=BUSY
sun4i_emac_probe: kretprobe unregistered, missed=0
```
