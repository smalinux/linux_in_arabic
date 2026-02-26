## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الملف ده جزء من **MMC/SD/SDIO subsystem** في Linux kernel، وبالتحديد هو ملف header خاص بـ **SDIO (Secure Digital Input Output)** — النسخة المتطورة من كارت الـ SD اللي مش بس بتخزن data، لكن بتشغّل devices كاملة زي Wi-Fi و Bluetooth و GPS.

---

### الصورة الكبيرة — شرح بالمثال

تخيل عندك كارت SD في تليفونك أو laptop. الكارت ده عادةً بيخزن صور وملفات بس. لكن في نوع تاني اسمه **SDIO card** — ده بيحط جوا نفس الفتحة الصغيرة دي **hardware كامل**: Wi-Fi chip أو Bluetooth module أو GPS receiver.

السؤال: لما الـ kernel يشوف SDIO card جديد، إزاي يعرف مين المصنّع ده؟ وإيه نوع الـ device؟

الإجابة موجودة في الملف ده.

---

### القصة كاملة — المشكلة اللي بيحلها الملف

لما تحط SDIO card في الجهاز، الـ kernel بيقرأ منه **Vendor ID** و **Device ID** — رقمين صغيرين محفوظين في الـ card نفسه. زي ما كل USB device عنده VID/PID، كل SDIO device عنده نفس الفكرة.

المشكلة: كل driver بيحتاج يعرف أي device بيشتغل معاه. لو كل driver كتب الأرقام دي بنفسه، هتلاقي تكرار وأخطاء في كل مكان.

**الحل**: ملف واحد مركزي — `sdio_ids.h` — فيه كل الأرقام دي كـ `#define` macros واضحة الاسم. أي driver يعمل `#include` للملف ده ويستخدم الاسم بدل الرقم الـ hex.

---

### محتوى الملف بالتفصيل

#### 1. الـ SDIO Standard Classes
```c
#define SDIO_CLASS_NONE   0x00  /* Not a SDIO standard interface */
#define SDIO_CLASS_BT_A   0x02  /* Type-A BlueTooth std interface */
#define SDIO_CLASS_WLAN   0x07  /* WLAN interface */
```
دي تصنيفات قياسية من **SD Association spec** — بتقول إيه نوع الـ function الجوا الكارت بصرف النظر عن المصنّع.

#### 2. الـ Vendor IDs و Device IDs
```c
#define SDIO_VENDOR_ID_BROADCOM        0x02d0
#define SDIO_DEVICE_ID_BROADCOM_4329   0x4329
#define SDIO_DEVICE_ID_BROADCOM_4330   0x4330
```
كل مصنّع عنده **Vendor ID** واحد، وتحته devices كتير كل واحد بـ Device ID خاص بيه.

المصنّعين الموجودين في الملف:
| Vendor | أمثلة Devices |
|--------|--------------|
| **Broadcom / Cypress** | Wi-Fi chips شائعة في Android و Raspberry Pi |
| **Marvell** | Libertas WLAN، chips للـ embedded systems |
| **Atheros** | AR6003، AR6004 — Wi-Fi للـ mobile |
| **Realtek** | RTW8723، RTW8822 — Wi-Fi+BT combo chips |
| **MediaTek** | MT7663، MT7961 — Wi-Fi 6 chips |
| **Texas Instruments** | WL1271، WL1251 — WLAN modules |
| **Siano** | Mobile DTV (تليفزيون على الموبايل) |
| **Intel** | IWMC3200 — combo WiMAX+Wi-Fi |
| **RSI** | Redpine Signals — IoT chips |
| **Microchip WILC** | WILC1000 — IoT Wi-Fi |

---

### الملفات المرتبطة اللي المفروض تعرفها

```
include/linux/mmc/
├── sdio_ids.h        ← الملف ده (قاموس الـ IDs)
├── sdio.h            ← SDIO API للـ drivers
├── sdio_func.h       ← struct sdio_func وعمليات I/O
├── card.h            ← struct mmc_card (يمثل الـ card في الـ kernel)
├── host.h            ← struct mmc_host (يمثل الـ controller)
└── mmc.h             ← MMC/SD commands و registers

drivers/mmc/core/
├── sdio.c            ← core SDIO logic (enumeration, init)
├── sdio_bus.c        ← SDIO bus driver matching
└── sdio_io.c         ← SDIO read/write functions

drivers/net/wireless/
├── broadcom/brcmfmac/  ← Broadcom SDIO Wi-Fi driver
├── marvell/mwifiex/    ← Marvell SDIO Wi-Fi driver
├── ath6kl/             ← Atheros AR6003/AR6004 driver
└── realtek/rtw88/      ← Realtek SDIO Wi-Fi driver
```

---

### إزاي الـ Driver Matching بيشتغل

```
SDIO Card يتحط في الجهاز
        ↓
kernel يقرأ Vendor ID + Device ID من CIS (Card Information Structure)
        ↓
SDIO bus يعمل match مع كل registered driver
        ↓
Driver عنده جدول: sdio_device_id[] فيه نفس الأرقام من sdio_ids.h
        ↓
لو match → driver يتحمّل ويشغّل الـ device
```

مثال من driver حقيقي:
```c
/* من drivers/net/wireless/broadcom/brcmfmac/sdio.c */
static const struct sdio_device_id brcmf_sdmmc_ids[] = {
    { SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4329) },
    { SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4330) },
    /* ... */
};
```

من غير `sdio_ids.h`، الـ driver كان هيكتب `0x02d0` و `0x4329` مباشرةً — أرقام مش واضحة ومش مركزية.

---

### ملخص سريع

**الـ** `sdio_ids.h` هو الـ "قاموس الرسمي" للـ SDIO ecosystem في Linux — بيوفّر اسم معبّر لكل رقم Vendor/Device، ويضمن إن كل الـ drivers تتكلم بنفس اللغة من غير تكرار أو تعارض.
## Phase 2: شرح الـ SDIO IDs Framework

### المشكلة اللي بيحلها الـ Framework ده

لما بتوصّل جهاز SDIO (زي Wi-Fi chip أو Bluetooth module) على بورد embedded، الكيرنل محتاج يعرف:
1. مين صنع الـ chip ده؟ (Vendor)
2. إيه هو الـ model بالظبط؟ (Device)
3. بيعمل إيه؟ هل هو WLAN ولا BT ولا GPS؟ (Class/Interface)

من غير framework موحد، كل driver كان هيحط الـ IDs دي بشكل عشوائي، وكان ممكن تتعارض أو تتكرر. فا الـ kernel محتاج **مصدر حقيقة واحد** لكل IDs دي.

---

### الحل اللي بياخده الكيرنل

الكيرنل بيحل المشكلة بـ **header file مركزي** واحد بس — هو `sdio_ids.h` — بيعرّف فيه:

- **SDIO Function Classes**: أرقام standardized من الـ SDIO spec نفسه، بتوصف نوع الـ interface.
- **Vendor IDs**: رقم 16-bit لكل شركة، assigned من **SD Association**.
- **Device IDs**: رقم 16-bit لكل chip، assigned من الشركة نفسها.

الـ driver بعدين بيستخدم الـ macros دي جوا `sdio_device_id` table عشان الكيرنل يعمل **automatic matching** بين hardware وdriver.

---

### Big Picture Architecture

```
+--------------------------------------------------+
|              SD Association Standard              |
|   (assigns Vendor IDs, defines Class numbers)    |
+--------------------------------------------------+
                        |
                        v
+--------------------------------------------------+
|         include/linux/mmc/sdio_ids.h             |
|                                                  |
|  #define SDIO_CLASS_WLAN        0x07             |
|  #define SDIO_VENDOR_ID_BROADCOM 0x02d0          |
|  #define SDIO_DEVICE_ID_BROADCOM_4329 0x4329     |
|  ... (كل الـ vendors والـ devices)               |
+--------------------------------------------------+
        |                        |
        v                        v
+---------------+      +---------------------+
|  SDIO Bus     |      |   Device Driver     |
|  Driver       |      |  (e.g. brcmfmac)   |
|               |      |                     |
| reads CIS from|      | static const        |
| card at probe |      | struct sdio_device_id|
| time          |      | brcmf_sdmmc_ids[] = |
|               |      | {                   |
| extracts:     |      |  {SDIO_DEVICE(      |
|  - vendor ID  |      |   SDIO_VENDOR_ID_   |
|  - device ID  |      |   BROADCOM,         |
|  - class      |      |   SDIO_DEVICE_ID_   |
|               |      |   BROADCOM_4329)},  |
+-------+-------+      +----------+----------+
        |                         |
        +----------+--------------+
                   |
                   v
        +----------+----------+
        |   Kernel Bus Match  |
        |   sdio_bus_match()  |
        |                     |
        |  vendor == vendor?  |
        |  device == device?  |
        |  class  == class?   |
        +----------+----------+
                   |
                   v
        +----------+----------+
        |  Driver .probe()    |
        |  gets called        |
        +---------------------+
```

---

### تشبيه من الواقع

فكّر في **بطاقة الهوية القومية** في أي دولة:

| الحياة الواقعية | الـ SDIO Framework |
|---|---|
| هيئة قومية بتصدر أرقام هويات | **SD Association** بتحدد نظام الـ IDs |
| رقم قومي فريد لكل مواطن | **SDIO_VENDOR_ID** + **SDIO_DEVICE_ID** لكل chip |
| المهنة في البطاقة (دكتور / مهندس) | **SDIO_CLASS** (WLAN / BT / GPS) |
| الـ database الحكومي المركزي | `sdio_ids.h` — header file واحد |
| موظف بيتحقق من البطاقة | `sdio_bus_match()` في الكيرنل |
| الموظف بيوجّهك للقسم الصح | الكيرنل بياخد الـ driver المناسب |

---

### الـ Core Abstraction

الـ abstraction الأساسية اللي الـ framework ده بيقدمها هي مفهوم الـ **SDIO Device Identity Tuple**:

```c
/*
 * Every SDIO device is uniquely identified by this triple:
 *   (vendor_id, device_id, class)
 * defined as simple #define macros in sdio_ids.h
 */

/* Example: Broadcom 4329 Wi-Fi chip */
#define SDIO_VENDOR_ID_BROADCOM     0x02d0   /* who made it */
#define SDIO_DEVICE_ID_BROADCOM_4329 0x4329  /* which chip exactly */
#define SDIO_CLASS_WLAN             0x07     /* what it does */
```

الـ driver بيحوّل الـ tuple ده لـ matching table باستخدام الـ macro الموجود في `<linux/mmc/sdio_func.h>`:

```c
/* Driver's device table — uses IDs from sdio_ids.h */
static const struct sdio_device_id my_driver_ids[] = {
    { SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4329) },
    { SDIO_DEVICE_CLASS(SDIO_CLASS_WLAN) }, /* match any WLAN device */
    { /* end of list */ }
};
MODULE_DEVICE_TABLE(sdio, my_driver_ids);
```

---

### الـ Framework بيملك إيه؟ وبيفوّض إيه؟

| الجانب | **sdio_ids.h يملكه** | **بيفوّضه للـ driver** |
|---|---|---|
| تعريف الـ Vendor IDs | نعم — macros ثابتة | لا |
| تعريف الـ Device IDs | نعم — macros ثابتة | لا |
| تعريف الـ Class numbers | نعم — من الـ spec | لا |
| بناء الـ matching table | لا | الـ driver يبني `sdio_device_id[]` |
| تسجيل الـ driver | لا | الـ driver يكال `sdio_register_driver()` |
| إدارة الـ power / clock | لا | الـ driver + MMC core |
| الـ probe / remove logic | لا | الـ driver بالكامل |
| قراءة الـ CIS من الـ card | لا | الـ MMC/SDIO bus core |

الـ `sdio_ids.h` هو **قاموس الهويات فقط** — مش بيتدخل في أي logic. الحكم الفعلي بيحصل في `drivers/mmc/core/sdio_bus.c` عن طريق دالة `sdio_bus_match()`، وتنفيذ الـ driver بيحصل في كل driver على حدة (زي `drivers/net/wireless/broadcom/brcm80211/brcmfmac/`).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — كل الـ Macros والـ IDs في جداول

#### SDIO Standard Class Codes

| Macro | Value | الوظيفة |
|---|---|---|
| `SDIO_CLASS_NONE` | `0x00` | مش interface قياسي |
| `SDIO_CLASS_UART` | `0x01` | UART |
| `SDIO_CLASS_BT_A` | `0x02` | Bluetooth Type-A |
| `SDIO_CLASS_BT_B` | `0x03` | Bluetooth Type-B |
| `SDIO_CLASS_GPS` | `0x04` | GPS |
| `SDIO_CLASS_CAMERA` | `0x05` | Camera |
| `SDIO_CLASS_PHS` | `0x06` | PHS (Personal Handy-phone) |
| `SDIO_CLASS_WLAN` | `0x07` | Wi-Fi |
| `SDIO_CLASS_ATA` | `0x08` | Embedded ATA storage |
| `SDIO_CLASS_BT_AMP` | `0x09` | Bluetooth AMP Type-A |

---

#### Vendor IDs

| Macro | Value | الشركة |
|---|---|---|
| `SDIO_VENDOR_ID_STE` | `0x0020` | ST-Ericsson |
| `SDIO_VENDOR_ID_INTEL` | `0x0089` | Intel |
| `SDIO_VENDOR_ID_CGUYS` | `0x0092` | C-guys Inc. |
| `SDIO_VENDOR_ID_TI` | `0x0097` | Texas Instruments |
| `SDIO_VENDOR_ID_ATHEROS` | `0x0271` | Qualcomm Atheros |
| `SDIO_VENDOR_ID_REALTEK` | `0x024c` | Realtek |
| `SDIO_VENDOR_ID_BROADCOM` | `0x02d0` | Broadcom |
| `SDIO_VENDOR_ID_MARVELL` | `0x02df` | Marvell |
| `SDIO_VENDOR_ID_MEDIATEK` | `0x037a` | MediaTek |
| `SDIO_VENDOR_ID_SIANO` | `0x039a` | Siano Mobile Silicon |
| `SDIO_VENDOR_ID_RSI` | `0x041b` | Redpine Signals |
| `SDIO_VENDOR_ID_CYPRESS` | `0x04b4` | Cypress Semiconductor |
| `SDIO_VENDOR_ID_MICROCHIP_WILC` | `0x0296` | Microchip (WILC) |
| `SDIO_VENDOR_ID_TI_WL1251` | `0x104c` | Texas Instruments (WL1251) |

---

#### Device IDs — STE

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_STE_CW1200` | `0x2280` | CW1200 Wi-Fi chip |

#### Device IDs — Intel

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_INTEL_IWMC3200WIMAX` | `0x1402` | iWMC 3200 WiMAX |
| `SDIO_DEVICE_ID_INTEL_IWMC3200WIFI` | `0x1403` | iWMC 3200 Wi-Fi |
| `SDIO_DEVICE_ID_INTEL_IWMC3200TOP` | `0x1404` | iWMC 3200 Top |
| `SDIO_DEVICE_ID_INTEL_IWMC3200GPS` | `0x1405` | iWMC 3200 GPS |
| `SDIO_DEVICE_ID_INTEL_IWMC3200BT` | `0x1406` | iWMC 3200 Bluetooth |
| `SDIO_DEVICE_ID_INTEL_IWMC3200WIMAX_2G5` | `0x1407` | iWMC 3200 WiMAX 2.5G |

#### Device IDs — Atheros

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_ATHEROS_AR6003_00` | `0x0300` | AR6003 rev 00 |
| `SDIO_DEVICE_ID_ATHEROS_AR6003_01` | `0x0301` | AR6003 rev 01 |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_00` | `0x0400` | AR6004 rev 00 |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_01` | `0x0401` | AR6004 rev 01 |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_02` | `0x0402` | AR6004 rev 02 |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_18` | `0x0418` | AR6004 rev 18 |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_19` | `0x0419` | AR6004 rev 19 |
| `SDIO_DEVICE_ID_ATHEROS_AR6005` | `0x050A` | AR6005 |
| `SDIO_DEVICE_ID_ATHEROS_QCA9377` | `0x0701` | QCA9377 |

#### Device IDs — Broadcom

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_BROADCOM_NINTENDO_WII` | `0x044b` | Nintendo Wii Wi-Fi |
| `SDIO_DEVICE_ID_BROADCOM_43241` | `0x4324` | BCM43241 |
| `SDIO_DEVICE_ID_BROADCOM_4329` | `0x4329` | BCM4329 |
| `SDIO_DEVICE_ID_BROADCOM_4330` | `0x4330` | BCM4330 |
| `SDIO_DEVICE_ID_BROADCOM_4334` | `0x4334` | BCM4334 |
| `SDIO_DEVICE_ID_BROADCOM_4335_4339` | `0x4335` | BCM4335/4339 combo |
| `SDIO_DEVICE_ID_BROADCOM_4339` | `0x4339` | BCM4339 |
| `SDIO_DEVICE_ID_BROADCOM_4345` | `0x4345` | BCM4345 |
| `SDIO_DEVICE_ID_BROADCOM_4354` | `0x4354` | BCM4354 |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_89359` | `0x4355` | Cypress 89359 |
| `SDIO_DEVICE_ID_BROADCOM_4356` | `0x4356` | BCM4356 |
| `SDIO_DEVICE_ID_BROADCOM_4359` | `0x4359` | BCM4359 |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_4373` | `0x4373` | Cypress 4373 |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_43012` | `0xa804` | Cypress 43012 |
| `SDIO_DEVICE_ID_BROADCOM_43143` | `0xa887` | BCM43143 |
| `SDIO_DEVICE_ID_BROADCOM_43340` | `0xa94c` | BCM43340 |
| `SDIO_DEVICE_ID_BROADCOM_43341` | `0xa94d` | BCM43341 |
| `SDIO_DEVICE_ID_BROADCOM_43362` | `0xa962` | BCM43362 |
| `SDIO_DEVICE_ID_BROADCOM_43364` | `0xa9a4` | BCM43364 |
| `SDIO_DEVICE_ID_BROADCOM_43430` | `0xa9a6` | BCM43430 |
| `SDIO_DEVICE_ID_BROADCOM_43439` | `0xa9af` | BCM43439 |
| `SDIO_DEVICE_ID_BROADCOM_43455` | `0xa9bf` | BCM43455 |
| `SDIO_DEVICE_ID_BROADCOM_43751` | `0xaae7` | BCM43751 |
| `SDIO_DEVICE_ID_BROADCOM_43752` | `0xaae8` | BCM43752 |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_43439` | `0xbd3d` | Cypress 43439 (vendor=0x04b4) |

#### Device IDs — Marvell

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_MARVELL_LIBERTAS` | `0x9103` | Libertas Wi-Fi |
| `SDIO_DEVICE_ID_MARVELL_8688_WLAN` | `0x9104` | 88W8688 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8688_BT` | `0x9105` | 88W8688 BT |
| `SDIO_DEVICE_ID_MARVELL_8786_WLAN` | `0x9116` | 88W8786 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8787_WLAN` | `0x9119` | 88W8787 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8787_BT` | `0x911a` | 88W8787 BT |
| `SDIO_DEVICE_ID_MARVELL_8787_BT_AMP` | `0x911b` | 88W8787 BT AMP |
| `SDIO_DEVICE_ID_MARVELL_8797_F0` | `0x9128` | 88W8797 Function 0 |
| `SDIO_DEVICE_ID_MARVELL_8797_WLAN` | `0x9129` | 88W8797 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8797_BT` | `0x912a` | 88W8797 BT |
| `SDIO_DEVICE_ID_MARVELL_8897_WLAN` | `0x912d` | 88W8897 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8897_BT` | `0x912e` | 88W8897 BT |
| `SDIO_DEVICE_ID_MARVELL_8887_F0` | `0x9134` | 88W8887 Function 0 |
| `SDIO_DEVICE_ID_MARVELL_8887_WLAN` | `0x9135` | 88W8887 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8887_BT` | `0x9136` | 88W8887 BT |
| `SDIO_DEVICE_ID_MARVELL_8801_WLAN` | `0x9139` | 88W8801 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8997_F0` | `0x9140` | 88W8997 Function 0 |
| `SDIO_DEVICE_ID_MARVELL_8997_WLAN` | `0x9141` | 88W8997 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8997_BT` | `0x9142` | 88W8997 BT |
| `SDIO_DEVICE_ID_MARVELL_8977_WLAN` | `0x9145` | 88W8977 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8977_BT` | `0x9146` | 88W8977 BT |
| `SDIO_DEVICE_ID_MARVELL_8987_WLAN` | `0x9149` | 88W8987 WLAN |
| `SDIO_DEVICE_ID_MARVELL_8987_BT` | `0x914a` | 88W8987 BT |
| `SDIO_DEVICE_ID_MARVELL_8978_WLAN` | `0x9159` | 88W8978 WLAN |

#### Device IDs — MediaTek

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_MEDIATEK_MT7663` | `0x7663` | MT7663 Wi-Fi |
| `SDIO_DEVICE_ID_MEDIATEK_MT7668` | `0x7668` | MT7668 Wi-Fi |
| `SDIO_DEVICE_ID_MEDIATEK_MT7961` | `0x7961` | MT7961 Wi-Fi |

#### Device IDs — Realtek

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_REALTEK_RTW8723BS` | `0xb723` | RTW8723BS |
| `SDIO_DEVICE_ID_REALTEK_RTW8821BS` | `0xb821` | RTW8821BS |
| `SDIO_DEVICE_ID_REALTEK_RTW8822BS` | `0xb822` | RTW8822BS |
| `SDIO_DEVICE_ID_REALTEK_RTW8821CS` | `0xc821` | RTW8821CS |
| `SDIO_DEVICE_ID_REALTEK_RTW8822CS` | `0xc822` | RTW8822CS |
| `SDIO_DEVICE_ID_REALTEK_RTW8723DS_2ANT` | `0xd723` | RTW8723DS (2 antenna) |
| `SDIO_DEVICE_ID_REALTEK_RTW8723DS_1ANT` | `0xd724` | RTW8723DS (1 antenna) |
| `SDIO_DEVICE_ID_REALTEK_RTW8821DS` | `0xd821` | RTW8821DS |
| `SDIO_DEVICE_ID_REALTEK_RTW8723CS` | `0xb703` | RTW8723CS |

#### Device IDs — Siano

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_SIANO_NOVA_B0` | `0x0201` | Nova B0 |
| `SDIO_DEVICE_ID_SIANO_NICE` | `0x0202` | Nice |
| `SDIO_DEVICE_ID_SIANO_VEGA_A0` | `0x0300` | Vega A0 |
| `SDIO_DEVICE_ID_SIANO_VENICE` | `0x0301` | Venice |
| `SDIO_DEVICE_ID_SIANO_MING` | `0x0302` | Ming |
| `SDIO_DEVICE_ID_SIANO_PELE` | `0x0500` | Pele |
| `SDIO_DEVICE_ID_SIANO_RIO` | `0x0600` | Rio |
| `SDIO_DEVICE_ID_SIANO_DENVER_2160` | `0x0700` | Denver 2160 |
| `SDIO_DEVICE_ID_SIANO_DENVER_1530` | `0x0800` | Denver 1530 |
| `SDIO_DEVICE_ID_SIANO_NOVA_A0` | `0x1100` | Nova A0 |
| `SDIO_DEVICE_ID_SIANO_STELLAR` | `0x5347` | Stellar |

#### Device IDs — باقي الـ Vendors

| Macro | Value | الجهاز |
|---|---|---|
| `SDIO_DEVICE_ID_CGUYS_EW_CG1102GC` | `0x0004` | C-guys EW-CG1102GC |
| `SDIO_DEVICE_ID_TI_WL1271` | `0x4076` | TI WL1271 Wi-Fi |
| `SDIO_DEVICE_ID_MICROCHIP_WILC1000` | `0x5347` | WILC1000 |
| `SDIO_DEVICE_ID_RSI_9113` | `0x9330` | RSI 9113 |
| `SDIO_DEVICE_ID_RSI_9116` | `0x9116` | RSI 9116 |
| `SDIO_DEVICE_ID_TI_WL1251` | `0x9066` | TI WL1251 Wi-Fi |

---

### 1. مجموعات الـ Macros وغرضها

الملف ده **header-only** — مفيش structs ولا functions. بس فيه مجموعتين من الـ macros:

**المجموعة الأولى: `SDIO_CLASS_*`**
- بتعرّف الـ **standard function class codes** اللي محددة في SDIO specification.
- الـ driver بيقارن الـ class code اللي بيجي من الـ hardware بالـ macros دي عشان يعرف نوع الـ interface.

**المجموعة التانية: `SDIO_VENDOR_ID_*` و `SDIO_DEVICE_ID_*`**
- بتعرّف الـ **vendor/product IDs** اللي بتيجي من الـ SDIO CIS (Card Information Structure).
- الـ driver بيستخدمها في `sdio_device_id` tables عشان يعمل matching مع الـ hardware.

---

### 2. العلاقة مع الـ Kernel Structs

الملف ده بيُغذّي الـ `sdio_device_id` struct اللي موجود في `include/linux/mmc/sdio_func.h`:

```c
struct sdio_device_id {
    __u8   class;       /* Standard interface or SDIO_ANY_ID */
    __u16  vendor;      /* Vendor or SDIO_ANY_ID */
    __u16  device;      /* Device ID or SDIO_ANY_ID */
    kernel_ulong_t driver_data; /* arbitrary driver data */
};
```

**كل driver بيعمل table زي ده:**

```c
/* Example: driver using sdio_ids.h macros in its ID table */
static const struct sdio_device_id brcmf_sdmmc_ids[] = {
    BRCMF_SDIO_DEVICE(SDIO_DEVICE_ID_BROADCOM_4329),
    BRCMF_SDIO_DEVICE(SDIO_DEVICE_ID_BROADCOM_4330),
    /* ... */
    { /* end sentinel */ }
};
MODULE_DEVICE_TABLE(sdio, brcmf_sdmmc_ids);
```

---

### 3. ASCII Diagram — العلاقة بين الملف والـ Subsystem

```
sdio_ids.h
┌──────────────────────────────────────────────┐
│  SDIO_CLASS_*        (0x00 - 0x09)           │
│  SDIO_VENDOR_ID_*    (16-bit vendor codes)   │
│  SDIO_DEVICE_ID_*    (16-bit device codes)   │
└──────────────┬───────────────────────────────┘
               │  #include
               ▼
┌──────────────────────────────────────────────┐
│           SDIO Driver (e.g. brcmfmac)        │
│                                              │
│  static const struct sdio_device_id ids[] = │
│  {                                           │
│    { SDIO_DEVICE(VENDOR_ID, DEVICE_ID) },    │
│    { }                                       │
│  };                                          │
│  MODULE_DEVICE_TABLE(sdio, ids);             │
└──────────────┬───────────────────────────────┘
               │  registered via
               ▼
┌──────────────────────────────────────────────┐
│           sdio_bus (SDIO Bus Driver)         │
│                                              │
│  sdio_bus_match()                            │
│    compares card CIS {vendor, device, class} │
│    against driver's sdio_device_id table     │
└──────────────┬───────────────────────────────┘
               │  match found → probe()
               ▼
┌──────────────────────────────────────────────┐
│           sdio_func (Physical SDIO Function) │
│  .vendor  = SDIO_VENDOR_ID_BROADCOM (0x02d0) │
│  .device  = SDIO_DEVICE_ID_BCM4329 (0x4329)  │
│  .class   = SDIO_CLASS_WLAN (0x07)           │
└──────────────────────────────────────────────┘
```

---

### 4. Lifecycle — من الـ Enumeration للـ Driver Probe

```
Card Inserted
     │
     ▼
MMC Core enumerates SDIO card
     │
     ▼
Read CIS (Card Information Structure)
     │  extracts: vendor_id, device_id, class
     ▼
Create sdio_func struct per function
     │  func->vendor = <from CIS>
     │  func->device = <from CIS>
     │  func->class  = <from CIS>
     ▼
sdio_bus_match()
     │  loops over all registered sdio_drivers
     │  compares func IDs against sdio_device_id table
     │  (uses SDIO_VENDOR_ID_* / SDIO_DEVICE_ID_* macros)
     ▼
Match Found?
  YES ──► call driver->probe(func, id)
  NO  ──► card stays unbound
     │
     ▼
Driver probe() runs
     │  uses func->vendor / func->device to distinguish
     │  chip variant if needed
     ▼
Driver registered and functional
     │
     ▼
Card Removed
     │
     ▼
driver->remove(func) called
     ▼
sdio_func released
```

---

### 5. Call Flow — sdio_bus_match بالتفصيل

```
sdio_bus_match(dev, drv)
        │
        ├── get sdio_func *func   from dev
        ├── get sdio_driver *sdrv from drv
        │
        └── sdio_match_device(func, sdrv)
                │
                └── for each id in sdrv->id_table:
                        │
                        ├── id->class  != SDIO_ANY_ID?
                        │     └── compare with func->class
                        │
                        ├── id->vendor != SDIO_ANY_ID?
                        │     └── compare with func->vendor
                        │          (== SDIO_VENDOR_ID_BROADCOM ?)
                        │
                        └── id->device != SDIO_ANY_ID?
                              └── compare with func->device
                                   (== SDIO_DEVICE_ID_BROADCOM_4329 ?)
                                        │
                                        YES → return id (match!)
                                        NO  → next id
```

---

### 6. ملاحظة: Vendor ID Collision

في الملف في نقطة دقيقة: نفس الـ device value `0x5347` موجود لـ vendor-ين مختلفين:

```
SDIO_DEVICE_ID_SIANO_STELLAR     (vendor=0x039a) → 0x5347
SDIO_DEVICE_ID_MICROCHIP_WILC1000(vendor=0x0296) → 0x5347
```

ده طبيعي — الـ device ID معناه بس في سياق الـ vendor. الـ matching بيشيل الاتنين مع بعض (vendor + device)، فمفيش تعارض.

---

### 7. ملاحظة: Broadcom/Cypress Split

بعض الـ Cypress chips بتيجي بـ vendor ID إما `SDIO_VENDOR_ID_BROADCOM` (0x02d0) أو `SDIO_VENDOR_ID_CYPRESS` (0x04b4):

```
/* Same chip, two possible vendor IDs */
SDIO_VENDOR_ID_BROADCOM + SDIO_DEVICE_ID_BROADCOM_CYPRESS_43439 (0xa9af)
SDIO_VENDOR_ID_CYPRESS  + SDIO_DEVICE_ID_BROADCOM_CYPRESS_43439 (0xbd3d)
```

الـ brcmfmac driver بيحتاج يعمل entry لكل حالة في الـ id_table عشان يشتغل مع الاتنين.

---

### 8. Locking Strategy

الملف ده **بس definitions** — مفيش لocking. الـ locking بيحصل في:
- `sdio_bus_match()` — بيتعمل under RCU read lock (bus-level)
- `sdio_func` structs — محميين بـ `host->lock` في الـ MMC core
- الـ driver probe/remove — بيتعملوا under `device_lock` من kernel device model

الـ macros نفسها compile-time constants — thread-safe by definition.
## Phase 4: شرح الـ Functions

> **ملاحظة:** الملف `include/linux/mmc/sdio_ids.h` هو **header-only** بيحتوي على `#define` macros بس — مفيش functions أو APIs قابلة للاستدعاء. الـ macros دي بتعمل دور **ID database** للـ SDIO subsystem في الكيرنل.

---

### جدول الـ Macros — Cheatsheet

#### أ) SDIO Class Macros

| Macro | Value | الوصف |
|---|---|---|
| `SDIO_CLASS_NONE` | `0x00` | مش interface standard |
| `SDIO_CLASS_UART` | `0x01` | UART standard interface |
| `SDIO_CLASS_BT_A` | `0x02` | Bluetooth Type-A |
| `SDIO_CLASS_BT_B` | `0x03` | Bluetooth Type-B |
| `SDIO_CLASS_GPS` | `0x04` | GPS interface |
| `SDIO_CLASS_CAMERA` | `0x05` | Camera interface |
| `SDIO_CLASS_PHS` | `0x06` | PHS interface |
| `SDIO_CLASS_WLAN` | `0x07` | WLAN interface |
| `SDIO_CLASS_ATA` | `0x08` | Embedded SDIO-ATA |
| `SDIO_CLASS_BT_AMP` | `0x09` | Bluetooth AMP Type-A |

#### ب) Vendor ID Macros

| Macro | Value | الشركة |
|---|---|---|
| `SDIO_VENDOR_ID_STE` | `0x0020` | ST-Ericsson |
| `SDIO_VENDOR_ID_INTEL` | `0x0089` | Intel |
| `SDIO_VENDOR_ID_CGUYS` | `0x0092` | C-guys |
| `SDIO_VENDOR_ID_TI` | `0x0097` | Texas Instruments |
| `SDIO_VENDOR_ID_ATHEROS` | `0x0271` | Atheros |
| `SDIO_VENDOR_ID_BROADCOM` | `0x02d0` | Broadcom |
| `SDIO_VENDOR_ID_MARVELL` | `0x02df` | Marvell |
| `SDIO_VENDOR_ID_REALTEK` | `0x024c` | Realtek |
| `SDIO_VENDOR_ID_MEDIATEK` | `0x037a` | MediaTek |
| `SDIO_VENDOR_ID_MICROCHIP_WILC` | `0x0296` | Microchip WILC |
| `SDIO_VENDOR_ID_SIANO` | `0x039a` | Siano |
| `SDIO_VENDOR_ID_RSI` | `0x041b` | Redpine Signals |
| `SDIO_VENDOR_ID_CYPRESS` | `0x04b4` | Cypress |
| `SDIO_VENDOR_ID_TI_WL1251` | `0x104c` | TI WL1251 |

#### ج) Device ID Macros (مختصرة حسب الـ vendor)

**STE:**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_STE_CW1200` | `0x2280` |

**Intel (IWMC3200 series):**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_INTEL_IWMC3200WIMAX` | `0x1402` |
| `SDIO_DEVICE_ID_INTEL_IWMC3200WIFI` | `0x1403` |
| `SDIO_DEVICE_ID_INTEL_IWMC3200TOP` | `0x1404` |
| `SDIO_DEVICE_ID_INTEL_IWMC3200GPS` | `0x1405` |
| `SDIO_DEVICE_ID_INTEL_IWMC3200BT` | `0x1406` |
| `SDIO_DEVICE_ID_INTEL_IWMC3200WIMAX_2G5` | `0x1407` |

**Atheros (AR6xxx/QCA series):**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_ATHEROS_AR6003_00` | `0x0300` |
| `SDIO_DEVICE_ID_ATHEROS_AR6003_01` | `0x0301` |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_00` | `0x0400` |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_01` | `0x0401` |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_02` | `0x0402` |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_18` | `0x0418` |
| `SDIO_DEVICE_ID_ATHEROS_AR6004_19` | `0x0419` |
| `SDIO_DEVICE_ID_ATHEROS_AR6005` | `0x050A` |
| `SDIO_DEVICE_ID_ATHEROS_QCA9377` | `0x0701` |

**Broadcom/Cypress (bcmdhd/brcmfmac series):**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_BROADCOM_NINTENDO_WII` | `0x044b` |
| `SDIO_DEVICE_ID_BROADCOM_43241` | `0x4324` |
| `SDIO_DEVICE_ID_BROADCOM_4329` | `0x4329` |
| `SDIO_DEVICE_ID_BROADCOM_4330` | `0x4330` |
| `SDIO_DEVICE_ID_BROADCOM_4334` | `0x4334` |
| `SDIO_DEVICE_ID_BROADCOM_4335_4339` | `0x4335` |
| `SDIO_DEVICE_ID_BROADCOM_4339` | `0x4339` |
| `SDIO_DEVICE_ID_BROADCOM_4345` | `0x4345` |
| `SDIO_DEVICE_ID_BROADCOM_4354` | `0x4354` |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_89359` | `0x4355` |
| `SDIO_DEVICE_ID_BROADCOM_4356` | `0x4356` |
| `SDIO_DEVICE_ID_BROADCOM_4359` | `0x4359` |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_4373` | `0x4373` |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_43012` | `0xa804` |
| `SDIO_DEVICE_ID_BROADCOM_43143` | `0xa887` |
| `SDIO_DEVICE_ID_BROADCOM_43340` | `0xa94c` |
| `SDIO_DEVICE_ID_BROADCOM_43341` | `0xa94d` |
| `SDIO_DEVICE_ID_BROADCOM_43362` | `0xa962` |
| `SDIO_DEVICE_ID_BROADCOM_43364` | `0xa9a4` |
| `SDIO_DEVICE_ID_BROADCOM_43430` | `0xa9a6` |
| `SDIO_DEVICE_ID_BROADCOM_43439` | `0xa9af` |
| `SDIO_DEVICE_ID_BROADCOM_43455` | `0xa9bf` |
| `SDIO_DEVICE_ID_BROADCOM_43751` | `0xaae7` |
| `SDIO_DEVICE_ID_BROADCOM_43752` | `0xaae8` |
| `SDIO_DEVICE_ID_BROADCOM_CYPRESS_43439` | `0xbd3d` |

**Marvell (libertas/mwifiex series):**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_MARVELL_LIBERTAS` | `0x9103` |
| `SDIO_DEVICE_ID_MARVELL_8688_WLAN` | `0x9104` |
| `SDIO_DEVICE_ID_MARVELL_8688_BT` | `0x9105` |
| `SDIO_DEVICE_ID_MARVELL_8786_WLAN` | `0x9116` |
| `SDIO_DEVICE_ID_MARVELL_8787_WLAN` | `0x9119` |
| `SDIO_DEVICE_ID_MARVELL_8787_BT` | `0x911a` |
| `SDIO_DEVICE_ID_MARVELL_8787_BT_AMP` | `0x911b` |
| `SDIO_DEVICE_ID_MARVELL_8797_F0` | `0x9128` |
| `SDIO_DEVICE_ID_MARVELL_8797_WLAN` | `0x9129` |
| `SDIO_DEVICE_ID_MARVELL_8797_BT` | `0x912a` |
| `SDIO_DEVICE_ID_MARVELL_8897_WLAN` | `0x912d` |
| `SDIO_DEVICE_ID_MARVELL_8897_BT` | `0x912e` |
| `SDIO_DEVICE_ID_MARVELL_8887_F0` | `0x9134` |
| `SDIO_DEVICE_ID_MARVELL_8887_WLAN` | `0x9135` |
| `SDIO_DEVICE_ID_MARVELL_8887_BT` | `0x9136` |
| `SDIO_DEVICE_ID_MARVELL_8801_WLAN` | `0x9139` |
| `SDIO_DEVICE_ID_MARVELL_8997_F0` | `0x9140` |
| `SDIO_DEVICE_ID_MARVELL_8997_WLAN` | `0x9141` |
| `SDIO_DEVICE_ID_MARVELL_8997_BT` | `0x9142` |
| `SDIO_DEVICE_ID_MARVELL_8977_WLAN` | `0x9145` |
| `SDIO_DEVICE_ID_MARVELL_8977_BT` | `0x9146` |
| `SDIO_DEVICE_ID_MARVELL_8987_WLAN` | `0x9149` |
| `SDIO_DEVICE_ID_MARVELL_8987_BT` | `0x914a` |
| `SDIO_DEVICE_ID_MARVELL_8978_WLAN` | `0x9159` |

**Realtek (rtw88 series):**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_REALTEK_RTW8723BS` | `0xb723` |
| `SDIO_DEVICE_ID_REALTEK_RTW8821BS` | `0xb821` |
| `SDIO_DEVICE_ID_REALTEK_RTW8822BS` | `0xb822` |
| `SDIO_DEVICE_ID_REALTEK_RTW8821CS` | `0xc821` |
| `SDIO_DEVICE_ID_REALTEK_RTW8822CS` | `0xc822` |
| `SDIO_DEVICE_ID_REALTEK_RTW8723DS_2ANT` | `0xd723` |
| `SDIO_DEVICE_ID_REALTEK_RTW8723DS_1ANT` | `0xd724` |
| `SDIO_DEVICE_ID_REALTEK_RTW8821DS` | `0xd821` |
| `SDIO_DEVICE_ID_REALTEK_RTW8723CS` | `0xb703` |

**MediaTek:**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_MEDIATEK_MT7663` | `0x7663` |
| `SDIO_DEVICE_ID_MEDIATEK_MT7668` | `0x7668` |
| `SDIO_DEVICE_ID_MEDIATEK_MT7961` | `0x7961` |

**Microchip WILC:**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_MICROCHIP_WILC1000` | `0x5347` |

**Siano (DVB/DTV receivers):**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_SIANO_NOVA_B0` | `0x0201` |
| `SDIO_DEVICE_ID_SIANO_NICE` | `0x0202` |
| `SDIO_DEVICE_ID_SIANO_VEGA_A0` | `0x0300` |
| `SDIO_DEVICE_ID_SIANO_VENICE` | `0x0301` |
| `SDIO_DEVICE_ID_SIANO_MING` | `0x0302` |
| `SDIO_DEVICE_ID_SIANO_PELE` | `0x0500` |
| `SDIO_DEVICE_ID_SIANO_RIO` | `0x0600` |
| `SDIO_DEVICE_ID_SIANO_DENVER_2160` | `0x0700` |
| `SDIO_DEVICE_ID_SIANO_DENVER_1530` | `0x0800` |
| `SDIO_DEVICE_ID_SIANO_NOVA_A0` | `0x1100` |
| `SDIO_DEVICE_ID_SIANO_STELLAR` | `0x5347` |

**RSI (Redpine Signals):**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_RSI_9113` | `0x9330` |
| `SDIO_DEVICE_ID_RSI_9116` | `0x9116` |

**TI WL1251:**

| Macro | Value |
|---|---|
| `SDIO_DEVICE_ID_TI_WL1251` | `0x9066` |

---

### 1. مجموعة SDIO Class Macros

**الغرض:** بتحدد نوع الـ **standard function interface** اللي الجهاز بيدعمه حسب مواصفات SDIO. الكيرنل بيستخدمها في الـ `sdio_func` struct لتصنيف الـ function قبل ما يبحث عن driver مناسب.

---

#### `SDIO_CLASS_NONE` — `0x00`

```c
#define SDIO_CLASS_NONE  0x00  /* Not a SDIO standard interface */
```

الـ function مش بتتبع أي standard interface class. بيُستخدم لما الجهاز بيكون proprietary وما بيدعمش أي class standard من SDIO spec.

- **Parameters:** لا يوجد — macro ثابت.
- **Return:** integer literal `0x00`.
- **Key details:** لما الـ `sdio_func->class` بيكون `SDIO_CLASS_NONE`، الكيرنل بيعتمد على الـ `vendor/device ID` بدل الـ class لمطابقة الـ driver.
- **Who uses it:** `drivers/mmc/core/sdio_bus.c` في دالة `sdio_bus_match()`.

---

#### `SDIO_CLASS_UART` — `0x01`

```c
#define SDIO_CLASS_UART  0x01  /* standard UART interface */
```

بيعرّف الـ function إنها UART standard. أي chip SDIO بيعمل serial communication عبر SDIO بيستخدم الـ class ده.

---

#### `SDIO_CLASS_BT_A` / `SDIO_CLASS_BT_B` — `0x02` / `0x03`

```c
#define SDIO_CLASS_BT_A  0x02  /* Type-A BlueTooth std interface */
#define SDIO_CLASS_BT_B  0x03  /* Type-B BlueTooth std interface */
```

Type-A وType-B بيختلفوا في طريقة الـ power management والـ transport layer. معظم الـ combo chips (Wi-Fi+BT) زي Marvell و Broadcom بتستخدم الـ BT_A.

---

#### `SDIO_CLASS_GPS` — `0x04`

```c
#define SDIO_CLASS_GPS  0x04  /* GPS standard interface */
```

بيعرّف الـ function إنها GPS receiver. Intel IWMC3200GPS بيستخدمه.

---

#### `SDIO_CLASS_WLAN` — `0x07`

```c
#define SDIO_CLASS_WLAN  0x07  /* WLAN interface */
```

**الأكتر استخداماً** في الكيرنل. معظم الـ Wi-Fi chips اللي شغاله على SDIO بتعرّف نفسها بالـ class ده. الـ drivers زي `brcmfmac` و`mwifiex` بتبحث عنه.

---

#### `SDIO_CLASS_ATA` — `0x08`

```c
#define SDIO_CLASS_ATA  0x08  /* Embedded SDIO-ATA std interface */
```

بيعرّف الـ function إنها storage interface من نوع ATA embedded على SDIO.

---

#### `SDIO_CLASS_BT_AMP` — `0x09`

```c
#define SDIO_CLASS_BT_AMP  0x09  /* Type-A Bluetooth AMP interface */
```

AMP = **Alternate MAC/PHY**. بيستخدمه Bluetooth 3.0 لنقل البيانات بسرعة أعلى. Marvell 8787 بيدعمه.

---

### كيف بيشتغل الـ Matching؟

```
ASCII: SDIO Device Enumeration Flow
─────────────────────────────────────────────────────
SDIO card inserted
        │
        ▼
MMC core reads CIS (Card Information Structure)
        │
        ├── Vendor ID  ──► SDIO_VENDOR_ID_*
        ├── Device ID  ──► SDIO_DEVICE_ID_*
        └── Class      ──► SDIO_CLASS_*
        │
        ▼
sdio_bus_match() في sdio_bus.c
        │
        ├── [1] بيطابق sdio_device_id table في الـ driver
        │         ├── بيقارن vendor + device
        │         └── أو بيقارن class بس (generic driver)
        │
        └── [2] لو اتطابق ► probe() بتتنفذ
─────────────────────────────────────────────────────
```

---

### 2. مجموعة Vendor ID Macros

**الغرض:** بتعرّف الـ **manufacturer ID** اللي مسجل في CIS (Card Information Structure) على الـ SDIO card. الكيرنل بيقراه تلقائياً أثناء الـ enumeration.

---

#### `SDIO_VENDOR_ID_BROADCOM` — `0x02d0`

```c
#define SDIO_VENDOR_ID_BROADCOM  0x02d0
```

الـ Vendor ID الرسمي لـ Broadcom. بيُستخدم مع كل الـ `SDIO_DEVICE_ID_BROADCOM_*` macros في `drivers/net/wireless/broadcom/brcm80211/brcmfmac/`.

- **Key detail:** بعض الـ Cypress chips (اللي اتشترتها Broadcom) بتستخدم نفس الـ vendor ID ده، وبعضها بيستخدم `SDIO_VENDOR_ID_CYPRESS` (`0x04b4`) — لازم تتحقق من الـ device ID.

---

#### `SDIO_VENDOR_ID_MARVELL` — `0x02df`

```c
#define SDIO_VENDOR_ID_MARVELL  0x02df
```

بيُستخدم في `drivers/net/wireless/marvell/mwifiex/` و`drivers/net/wireless/marvell/libertas/`. الـ Marvell chips بتفرق بين WLAN و BT functions بالـ device ID.

---

#### `SDIO_VENDOR_ID_CYPRESS` — `0x04b4`

```c
#define SDIO_VENDOR_ID_CYPRESS  0x04b4
```

بعد ما Infineon اشترت Cypress، بعض الـ chips الجديدة زي `CYW43439` بتستخدم الـ vendor ID ده بدل `SDIO_VENDOR_ID_BROADCOM`.

---

#### `SDIO_VENDOR_ID_TI_WL1251` — `0x104c`

```c
#define SDIO_VENDOR_ID_TI_WL1251  0x104c
```

مختلف عن `SDIO_VENDOR_ID_TI` (`0x0097`) — الـ WL1251 بيستخدم vendor ID مختلف. لازم الـ driver يفرق بينهم صح.

---

### 3. مجموعة Device ID Macros

**الغرض:** بتعرّف الـ **product ID** الخاص بكل chip داخل نفس الـ vendor. بالاتنين مع Vendor ID بيتكون الـ pair اللي بيميز الجهاز بشكل unambiguous.

---

#### Marvell — F0 Function IDs

```c
#define SDIO_DEVICE_ID_MARVELL_8797_F0   0x9128
#define SDIO_DEVICE_ID_MARVELL_8887_F0   0x9134
#define SDIO_DEVICE_ID_MARVELL_8997_F0   0x9140
```

الـ **F0** function هي الـ common function في كل SDIO card، وبتحتوي على الـ CIS. الـ mwifiex driver بيستخدم الـ F0 ID عشان يتعرف على الـ chipset قبل ما يفرق بين WLAN و BT functions.

---

#### Broadcom/Cypress — Naming Convention

```c
/* Broadcom-branded chips */
#define SDIO_DEVICE_ID_BROADCOM_43439    0xa9af
/* Cypress-branded version of same chip */
#define SDIO_DEVICE_ID_BROADCOM_CYPRESS_43439  0xbd3d
```

نفس الـ chipset (BCM43439/CYW43439) بس vendor ID مختلف (Broadcom vs Cypress) → device ID مختلف كمان. الـ brcmfmac driver عنده entries للاتنين في `sdio_ids[]` table.

---

#### Realtek — Antenna Variants

```c
#define SDIO_DEVICE_ID_REALTEK_RTW8723DS_2ANT  0xd723  /* 2 antennas */
#define SDIO_DEVICE_ID_REALTEK_RTW8723DS_1ANT  0xd724  /* 1 antenna */
```

نفس الـ chipset (RTW8723DS) بس بـ 2 variants مختلفين في عدد الـ antennas → كل واحد عنده device ID مختلف. الـ driver بيعمل conditional config بناءً على الـ ID.

---

### 4. كيفية الاستخدام في الـ Driver

الـ macros دي بتتستخدم في بناء `sdio_device_id` table في كل driver:

```c
/* مثال من brcmfmac */
static const struct sdio_device_id brcmf_sdmmc_ids[] = {
    /* vendor ID + device ID → بتعرّف الـ chip */
    {SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_43143)},
    {SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4329)},
    {SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4330)},
    /* ... */
    {SDIO_DEVICE(SDIO_VENDOR_ID_CYPRESS,  SDIO_DEVICE_ID_BROADCOM_CYPRESS_43439)},
    { /* end: all zeroes */ }
};
MODULE_DEVICE_TABLE(sdio, brcmf_sdmmc_ids);
```

```
Flow: Driver Registration → Device Matching
──────────────────────────────────────────────────────────
module_init()
    │
    ▼
sdio_register_driver(&brcmf_sdmmc_driver)
    │           brcmf_sdmmc_driver.id_table = brcmf_sdmmc_ids[]
    │
    ▼
[Device Inserted / Already Present]
    │
    ▼
sdio_bus_match(dev, drv)
    │
    ├── loop over id_table[]
    │       ├── compare: sdio_func->vendor == id->vendor
    │       └── compare: sdio_func->device == id->device
    │
    ├── match found?
    │       YES → sdio_bus_probe() → driver->probe(func, id)
    │       NO  → try next driver
──────────────────────────────────────────────────────────
```

---

### 5. ملاحظات مهمة للـ Embedded Developer

| الموضوع | التفاصيل |
|---|---|
| **Guard Macro** | `#ifndef LINUX_MMC_SDIO_IDS_H` يمنع double-inclusion |
| **Cypress/Broadcom Overlap** | نفس الـ chips ممكن تظهر بـ vendor IDs مختلفة حسب إصدار الـ chip |
| **F0 Function** | Function 0 دايماً موجودة وبتحتوي CIS — بعض الـ drivers بتعمل match عليها |
| **Combo Chips** | chips زي Marvell 8797 بتعمل expose functions منفصلة لـ WLAN وBT لكن بنفس الـ Vendor ID |
| **SDIO Spec Alignment** | الـ Class macros بتطابق SDIO Specification Part E1 Section 5.1 |
| **`MODULE_DEVICE_TABLE`** | الـ macros دي بتتضمن في `MODULE_DEVICE_TABLE(sdio, ...)` عشان الـ udev/modprobe يعرف يـ autoload الـ driver الصح |
## Phase 5: دليل الـ Debugging الشامل

---

### السياق: `include/linux/mmc/sdio_ids.h`

الملف ده بيحدد **SDIO Vendor/Device IDs** و **SDIO Class codes** — ما فيهوش runtime code، بس هو نقطة البداية لأي debugging لـ SDIO device enumeration و driver binding.

---

## Software Level

### 1. debugfs Entries

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل SDIO/MMC hosts
ls /sys/kernel/debug/mmc*/

# اقرأ state الـ host
cat /sys/kernel/debug/mmc0/ios

# اقرأ registers الـ host (لو الـ driver بيدعمه)
cat /sys/kernel/debug/mmc0/regs
```

**تفسير output الـ `ios`:**

```
clock:          50000000 Hz     # clock speed
actual clock:   50000000 Hz
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)
chip select:    0 (don't care)
power mode:     2 (on)
bus width:      2 (4 bits)      # SDIO standard
timing spec:    2 (sd high-speed)
signal voltage: 0 (3.30 V)
driver type:    0 (driver type B)
```

```bash
# شوف SDIO functions المتعرفة
ls /sys/kernel/debug/mmc0/mmc0\:*/
```

---

### 2. sysfs Entries

```bash
# اعرض كل MMC/SDIO devices
ls /sys/bus/sdio/devices/

# اقرأ vendor ID لـ device
cat /sys/bus/sdio/devices/mmc1\:0001\:1/vendor
# output: 0x02d0  (Broadcom مثلاً)

# اقرأ device ID
cat /sys/bus/sdio/devices/mmc1\:0001\:1/device
# output: 0x4330  (BCM4330)

# اقرأ class الـ SDIO function
cat /sys/bus/sdio/devices/mmc1\:0001\:1/class
# output: 0x07  (WLAN)

# شوف الـ driver المرتبط
cat /sys/bus/sdio/devices/mmc1\:0001\:1/driver/module/name

# عرض كل attributes
find /sys/bus/sdio/devices/ -maxdepth 2 -name "vendor" -exec sh -c \
  'echo "$(dirname $1): $(cat $1)"' _ {} \;
```

**جدول SDIO Class codes من الـ header:**

| Class Value | Macro | المعنى |
|---|---|---|
| `0x00` | `SDIO_CLASS_NONE` | Non-standard interface |
| `0x01` | `SDIO_CLASS_UART` | UART |
| `0x02` | `SDIO_CLASS_BT_A` | Bluetooth Type-A |
| `0x03` | `SDIO_CLASS_BT_B` | Bluetooth Type-B |
| `0x04` | `SDIO_CLASS_GPS` | GPS |
| `0x05` | `SDIO_CLASS_CAMERA` | Camera |
| `0x06` | `SDIO_CLASS_PHS` | PHS |
| `0x07` | `SDIO_CLASS_WLAN` | WLAN |
| `0x08` | `SDIO_CLASS_ATA` | Embedded ATA |
| `0x09` | `SDIO_CLASS_BT_AMP` | Bluetooth AMP |

---

### 3. ftrace — Tracepoints و Events

```bash
# شوف كل MMC tracepoints المتاحة
ls /sys/kernel/debug/tracing/events/mmc/

# فعّل كل أحداث MMC (للبداية)
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable

# أو بشكل انتقائي — أهم events لـ SDIO IDs و enumeration:
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_start/enable
echo 1 > /sys/kernel/debug/tracing/events/mmc/mmc_request_done/enable

# فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل action (مثلاً: insert SDIO card)
# اقرأ النتايج
cat /sys/kernel/debug/tracing/trace | head -60

# وقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on

# filter على SDIO commands بس (CMD5 = SDIO_SEND_OP_COND)
echo 'opcode==5' > /sys/kernel/debug/tracing/events/mmc/mmc_request_start/filter
```

**أهم MMC commands لـ SDIO enumeration:**

| CMD | الوظيفة |
|---|---|
| CMD5 | `SDIO_SEND_OP_COND` — بيكتشف لو في SDIO |
| CMD3 | `SEND_RELATIVE_ADDR` — بيدي الـ card عنوان |
| CMD7 | `SELECT_CARD` |
| CMD52 | `IO_RW_DIRECT` — بيقرأ/يكتب CCCR |
| CMD53 | `IO_RW_EXTENDED` — data transfer |

---

### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug لـ MMC/SDIO subsystem كله
echo 'module mmc_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module brcmfmac +p' > /sys/kernel/debug/dynamic_debug/control  # Broadcom مثلاً
echo 'module mwifiex_sdio +p' > /sys/kernel/debug/dynamic_debug/control  # Marvell

# فعّل debug لملف معين (لو عارف اسمه)
echo 'file drivers/mmc/core/sdio.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/mmc/core/sdio_bus.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف إيه اللي فعّال دلوقتي
cat /sys/kernel/debug/dynamic_debug/control | grep mmc

# زوّد verbosity الـ kernel messages
dmesg -n 8

# تابع logs بشكل real-time
dmesg -w | grep -iE 'sdio|mmc|vendor|device'
```

---

### 5. Kernel Config Options للـ Debugging

```bash
# اعرض الـ config الحالية
zcat /proc/config.gz | grep -E 'MMC|SDIO|DEBUG'
```

| Config Option | الوظيفة |
|---|---|
| `CONFIG_MMC_DEBUG` | يفعّل debug messages في MMC core |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل dynamic debug framework |
| `CONFIG_MMC_UNSAFE_RESUME` | مفيد لتتبع suspend/resume issues |
| `CONFIG_MMC_SDHCI_PCI` | لو الـ host على PCIe |
| `CONFIG_MMC_TEST` | يضيف test driver للـ MMC |
| `CONFIG_BRCMFMAC_SDIO` | Broadcom SDIO WLAN driver |
| `CONFIG_LIBERTAS_SDIO` | Marvell Libertas SDIO |
| `CONFIG_ATH6KL_SDIO` | Atheros AR6003/AR6004 |
| `CONFIG_RTW88_8723DS` | Realtek RTW8723DS SDIO |

لتفعيل debugging عند build:
```bash
# في .config
CONFIG_MMC_DEBUG=y
CONFIG_DYNAMIC_DEBUG=y
```

---

### 6. رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `mmc1: error -110 whilst initialising SDIO card` | timeout أثناء SDIO init — الـ card مش بترد | افحص الاتصال الفيزيائي، ارفع clock timeout |
| `mmc1: SDIO card does not support CCCR format` | الـ CCCR register بيرجع قيمة غلط | firmware مشكلة أو hardware معيوب |
| `sdio: vendor 0x02d0, device 0x4330: no module found` | ما فيش driver مطابق للـ Vendor/Device ID | `modprobe brcmfmac` أو راجع الـ ID في الـ driver |
| `mmc1: Card stuck in wrong state! Resetting!` | الـ card وقعت في حالة undefined | hardware أو noise على SDIO bus |
| `mmc1: tuning execution failed` | عدم استقرار signal عند high-speed mode | خفّض clock أو افحص الـ PCB routing |
| `sdio_enable_func failed (-110)` | الـ function ما استجابتش | الـ firmware ما اتحملش أو مشكلة power |
| `mmc1: couldn't set UHS signaling, error -22` | مشكلة في تغيير voltage | الـ regulator ما بيدعمش 1.8V |
| `brcmfmac: brcmf_sdio_probe failed` | Broadcom firmware load فشل | افحص `/lib/firmware/brcm/` |
| `mwifiex: SDIO card 0x9129 not found` | Marvell 8797 مش متعرف | راجع الـ SDIO_DEVICE_ID في الـ driver match table |

---

### 7. Strategic Points لـ `dump_stack()` / `WARN_ON()`

الأماكن دي في kernel code أهم نقاط للـ instrumentation:

```c
/* في drivers/mmc/core/sdio_bus.c — عند الـ probe */
static int sdio_bus_probe(struct device *dev)
{
    struct sdio_driver *drv = to_sdio_driver(dev->driver);
    struct sdio_func *func = dev_to_sdio_func(dev);

    /* نقطة 1: تحقق إن الـ IDs متطابقة */
    WARN_ON(!sdio_match_device(func->card, drv->id_table));
    /* dump_stack(); // uncomment للـ debugging */
    ...
}

/* في drivers/mmc/core/sdio.c — عند قراءة CIS */
static int sdio_read_cis(struct sdio_func *func)
{
    /* نقطة 2: لو Vendor ID جه غلط */
    WARN_ON(func->vendor == 0x0000 || func->vendor == 0xFFFF);
    ...
}
```

**أهم نقاط strategically:**

```
drivers/mmc/core/sdio.c        → mmc_sdio_init_card()     # بداية card init
drivers/mmc/core/sdio_bus.c    → sdio_bus_match()          # ID matching
drivers/mmc/core/sdio_io.c     → sdio_enable_func()        # function enable
drivers/mmc/core/sdio_cis.c    → sdio_read_cis_node()      # قراءة Vendor/Device IDs من CIS
```

---

## Hardware Level

### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel

```bash
# قارن الـ Vendor ID اللي قرأه الـ kernel مع الـ datasheet
cat /sys/bus/sdio/devices/mmc1\:0001\:1/vendor
# المفروض يطابق SDIO_VENDOR_ID_* في الـ header

# قارن الـ Device ID
cat /sys/bus/sdio/devices/mmc1\:0001\:1/device

# اتحقق من الـ CIS (Card Information Structure) مباشرة
# CMD52 بيقرأ CCCR register 0x09 (Vendor ID low byte)
# Vendor ID = CCCR[0x09] | (CCCR[0x0A] << 8)

# شوف الـ class المُعلن من الـ card
cat /sys/bus/sdio/devices/mmc1\:0001\:1/class
# لازم يطابق SDIO_CLASS_* المناسب للـ device

# تحقق من عدد الـ functions
cat /sys/bus/sdio/devices/mmc1\:0001\:1/num_info
```

**مثال: Broadcom BCM4330 (SDIO_DEVICE_ID_BROADCOM_4330 = 0x4330)**

```
Expected: vendor=0x02d0, device=0x4330, class=0x07 (WLAN)
```

---

### 2. Register Dump Techniques

```bash
# قراءة CCCR registers مباشرة عبر CMD52
# (محتاج tool زي mmc-utils أو sdio-utils)

# باستخدام mmcli (lو متاح)
mmcli -L   # list all MMC devices

# قراءة raw SDIO registers (Broadcom مثلاً)
# Driver بيعمل expose لبعض registers في debugfs:
cat /sys/kernel/debug/brcmfmac/mmc*/registers 2>/dev/null

# لـ Marvell mwifiex
cat /sys/kernel/debug/mwifiex/mmc*/info

# عمل dump لـ CCCR عبر /dev/mmcblkX (experimental)
# CCCR = أول 22 bytes من Function 0 space
dd if=/dev/mmcblk0 bs=1 count=22 skip=0 2>/dev/null | xxd
```

**تفسير CCCR registers:**

```
Offset  Field               المعنى
0x00    CCCR/SDIO revision  SDIO spec version
0x01    SD spec revision
0x02    I/O Enable          enable بكل function
0x03    I/O Ready           هل الـ function ready؟
0x04    Int Enable
0x05    Int Pending
0x06    I/O Abort
0x07    Bus Interface Control (bus width: bits 1:0)
0x08    Card Capability
0x09    Common CIS Pointer (low byte)
0x09-0x0B  CIS Pointer      عنوان الـ CIS في CIS space
```

---

### 3. Logic Analyzer / Oscilloscope Tips

**إعداد الـ Logic Analyzer لـ SDIO:**

```
Signals to probe:
┌─────────────────────────────────────────────┐
│  CLK  — SDIO Clock (up to 50 MHz SD-HS)    │
│  CMD  — Command/Response line               │
│  DAT0 — Data line 0 (always present)        │
│  DAT1 — Data line 1 (IRQ in 1-bit mode)    │
│  DAT2 — Data line 2                         │
│  DAT3 — Data line 3 (card detect)          │
└─────────────────────────────────────────────┘
```

**إعدادات الـ sampling:**

| الـ Mode | الـ Max Clock | الـ Sampling Rate المطلوبة |
|---|---|---|
| Default Speed | 25 MHz | ≥ 100 MHz |
| High Speed | 50 MHz | ≥ 200 MHz |
| SDR50/SDR104 | 100/208 MHz | ≥ 500 MHz |

**نقاط ترقب:**
- CMD5 (SDIO_SEND_OP_COND): أول command بعد power-on — لو ما بيجيش رد، الـ card مش موجود
- الـ Response R4 لـ CMD5 بيحتوي على OCR و number of I/O functions
- CMD52 بـ Function 0 لقراءة CCCR (Vendor/Device IDs)
- توقف الـ CLK فجأة = الـ host قرر إن في مشكلة

**مشاكل شائعة في الـ signal:**
- Signal integrity: level ما بيوصلش لـ 3.3V → بيتسبب في misread للـ IDs
- Glitches على CMD → responses غلط → driver مش بيتعرف على الـ Vendor ID

---

### 4. مشاكل الـ Hardware الشائعة ← patterns في الـ kernel log

| المشكلة | Pattern في dmesg | الحل |
|---|---|---|
| Card مش موجود فيزيائياً | `mmc1: SDIO card is not present` | افحص socket/connector |
| مشكلة Power (VDD) | `mmc1: error -110 whilst initialising` + لا CMD5 response | افحص regulator، قس الـ voltage |
| Clock ما بيشتغلش | لا activity خالص على الـ bus | افحص clock source، PCB track |
| DAT lines مش متصلة | `mmc1: card stuck in wrong state` | افحص الـ 4-bit data lines |
| Pull-up مش صح | IDs بتيجي `0xFFFF` أو `0x0000` | افحص pull-up resistors على CMD و DAT lines |
| Voltage mismatch | `mmc1: failed to switch to 1.8V` | افحص الـ level shifter |
| Firmware ما اتحملش | `brcmfmac: firmware: failed to load` | حط الـ firmware في `/lib/firmware/` |

---

## Practical Commands

### أوامر جاهزة للـ Copy-Paste

#### فحص SDIO device enumeration:

```bash
#!/bin/bash
# sdio_debug.sh — فحص شامل لـ SDIO device

echo "=== SDIO Devices ==="
for dev in /sys/bus/sdio/devices/*/; do
    echo "Device: $(basename $dev)"
    echo "  Vendor:  $(cat $dev/vendor 2>/dev/null)"
    echo "  Device:  $(cat $dev/device 2>/dev/null)"
    echo "  Class:   $(cat $dev/class 2>/dev/null)"
    echo "  Driver:  $(readlink $dev/driver 2>/dev/null | xargs basename)"
    echo ""
done

echo "=== MMC IOS State ==="
for ios in /sys/kernel/debug/mmc*/ios; do
    echo "--- $ios ---"
    cat "$ios" 2>/dev/null
done
```

#### تفعيل full MMC debugging:

```bash
# script واحد يفعّل كل حاجة
echo 8 > /proc/sys/kernel/printk
echo 1 > /sys/kernel/debug/tracing/events/mmc/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo 'module mmc_core +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module mmc_block +pflmt' > /sys/kernel/debug/dynamic_debug/control
dmesg -w &
```

#### مطابقة الـ IDs مع الـ kernel header:

```bash
# compare detected IDs against known IDs in sdio_ids.h
VENDOR=$(cat /sys/bus/sdio/devices/mmc1\:0001\:1/vendor)
DEVICE=$(cat /sys/bus/sdio/devices/mmc1\:0001\:1/device)
echo "Detected: vendor=$VENDOR device=$DEVICE"

# ابحث في الـ header
grep -i "$VENDOR" /usr/src/linux/include/linux/mmc/sdio_ids.h
grep -i "$DEVICE" /usr/src/linux/include/linux/mmc/sdio_ids.h
```

#### reset SDIO card يدوياً:

```bash
# إزالة و إعادة bind الـ device
echo "mmc1:0001:1" > /sys/bus/sdio/drivers/brcmfmac/unbind
echo "mmc1:0001:1" > /sys/bus/sdio/drivers/brcmfmac/bind

# أو reset الـ MMC host كله
echo 1 > /sys/class/mmc_host/mmc1/soft_reset 2>/dev/null || \
  echo "soft_reset not supported by this host"
```

---

### مثال على Output وتفسيره

#### Output لـ `dmesg | grep -i sdio`:

```
[    3.142356] mmc1: new ultra high speed SDR50 SDIO card at address 0001
[    3.143001] mmc1: card 0001: SDIO
[    3.143102]   0: Function 1, 0x02d0:0x4330 (brcmfmac)   ← vendor:device
[    3.143215] brcmfmac: F1 signature read @0x18000000=0x1534433
[    3.143890] brcmfmac: brcmf_sdio_probe: completed
```

**التفسير:**
- `0x02d0:0x4330` = `SDIO_VENDOR_ID_BROADCOM` : `SDIO_DEVICE_ID_BROADCOM_4330` — تطابق صح
- `brcmfmac` اتـ bind تلقائياً لأن الـ IDs موجودين في `brcmf_sdmmc_ids[]` في الـ driver
- لو شفت `no module found` بدل `(brcmfmac)` → الـ ID مش في الـ driver table

#### Output لـ `cat /sys/kernel/debug/mmc1/ios`:

```
clock:          50000000 Hz     # ← SDIO High Speed OK
actual clock:   50000000 Hz     # ← clock فعلاً بالسرعة دي
vdd:            21 (3.3 ~ 3.4 V)
bus mode:       2 (push-pull)   # ← SDIO standard
chip select:    0 (don't care)
power mode:     2 (on)          # ← 0=off, 1=up, 2=on
bus width:      2 (4 bits)      # ← 0=1bit, 2=4bits, 3=8bits
timing spec:    2 (sd high-speed)
signal voltage: 0 (3.30 V)      # ← 0=3.3V, 1=1.8V, 2=1.2V
```

لو شفت `power mode: 0 (off)` والـ card المفروض موجودة → مشكلة في detection أو power.
## Phase 6: سيناريوهات من الحياة العملية

---

### سيناريو 1: Wi-Fi مش بيتعرف على Android TV Box بـ Allwinner H616

**العنوان:** Broadcom 43455 SDIO Wi-Fi مش بيظهر في kernel على TV box

**السياق:**
منتج: Android TV box بـ Allwinner H616 SoC.
الـ Wi-Fi chip: Broadcom BCM43455 متوصل بـ SDIO.
البيئة: custom Android kernel مبني من AOSP مع BSP من vendor.

**المشكلة:**
بعد build جديد للـ kernel، الـ Wi-Fi مش بيشتغل خالص. `dmesg` بيديك:
```
mmc1: new ultra high speed SDR104 SDIO card at address 0001
mmc1: card 0001 removed
```
الـ card بتتعرف وبعدين بتتشال فوراً. مفيش driver بيـ bind.

**التحليل:**
الـ driver الخاص بـ BCM43455 هو `brcmfmac`. بيعمل match عن طريق `sdio_device_id` table:
```c
/* في driver/net/wireless/broadcom/brcm80211/brcmfmac/sdio.c */
static const struct sdio_device_id brcmf_sdmmc_ids[] = {
    ...
    SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_43455),
    ...
};
```
الـ macro ده بيرجع لـ `sdio_ids.h`:
```c
#define SDIO_VENDOR_ID_BROADCOM   0x02d0
#define SDIO_DEVICE_ID_BROADCOM_43455  0xa9bf
```
الـ kernel بيقرأ الـ vendor/device IDs من الـ SDIO CIS (Card Information Structure) ويعمل compare مع الـ table.

المشكلة: في custom BSP، المطور اتكلم بـ `CONFIG_BRCMFMAC=m` بس منساش يحط الـ firmware في `/lib/firmware/brcm/brcmfmac43455-sdio.bin`. الـ driver اتـ load، حاول يفتح الـ firmware، fail، وعمل `mmc_remove_host`.

**الحل:**
```bash
# تحقق إن الـ driver بيتعرف على الـ IDs صح
cat /sys/bus/sdio/devices/mmc1\:0001\:1/device   # لازم يطلع 0xa9bf
cat /sys/bus/sdio/devices/mmc1\:0001\:1/vendor   # لازم يطلع 0x02d0

# تحقق من الـ firmware
ls /lib/firmware/brcm/brcmfmac43455-sdio*
# لو مش موجود، copy الفايلات
cp brcmfmac43455-sdio.bin /lib/firmware/brcm/
cp brcmfmac43455-sdio.txt /lib/firmware/brcm/
```
في الـ Device Tree، تأكد إن `brcmf` node موجود:
```dts
/* arch/arm64/boot/dts/allwinner/sun50i-h616-tv-box.dts */
wifi_pwrseq: wifi-pwrseq {
    compatible = "mmc-pwrseq-simple";
    reset-gpios = <&r_pio 0 7 GPIO_ACTIVE_LOW>;
};

&mmc1 {
    vmmc-supply = <&reg_dldo1>;
    mmc-pwrseq = <&wifi_pwrseq>;
    bus-width = <4>;
    non-removable;
    status = "okay";

    brcmf: wifi@1 {
        reg = <1>;
        compatible = "brcm,bcm4329-fmac";
    };
};
```

**الدرس المستفاد:**
الـ `sdio_ids.h` بيحدد بس الـ IDs — مسؤولية الـ driver إنه يـ probe صح وأي failure في الـ firmware أو الـ DT بيخلي الـ card تتشال. لازم دايماً تفصل بين مشكلة الـ ID matching ومشكلة الـ firmware loading.

---

### سيناريو 2: Bluetooth وWi-Fi conflict على RK3562 Industrial Gateway

**العنوان:** Marvell 8997 WLAN و BT بيتعارضوا في probe على gateway صناعي

**السياق:**
منتج: Industrial IoT gateway بـ Rockchip RK3562.
الـ combo chip: Marvell 88W8997 (WLAN + BT على نفس SDIO interface بـ multiple functions).
البيئة: Yocto Linux, kernel 6.1.

**المشكلة:**
الـ Wi-Fi بيشتغل تمام، بس الـ Bluetooth مش بيظهر خالص. `hciconfig` مفيش output. الـ `dmesg` بيقول:
```
mwifiex_sdio mmc0:0001:1: WLAN FW is active
mwifiex_sdio mmc0:0001:2: probe failed: -ENODEV
```
الـ function 2 (BT) مش بتـ probe.

**التحليل:**
في `sdio_ids.h`:
```c
#define SDIO_DEVICE_ID_MARVELL_8997_F0    0x9140  /* function 0 */
#define SDIO_DEVICE_ID_MARVELL_8997_WLAN  0x9141  /* function 1 */
#define SDIO_DEVICE_ID_MARVELL_8997_BT    0x9142  /* function 2 */
```
الـ chip 88W8997 بيظهر كـ multi-function SDIO device:
- Function 1: WLAN — بيتـ handle بـ `mwifiex` driver
- Function 2: BT — بيتـ handle بـ `btmrvl_sdio` driver

الـ `btmrvl_sdio` driver بيعمل match على `SDIO_DEVICE_ID_MARVELL_8997_BT = 0x9142`:
```c
static const struct sdio_device_id btmrvl_sdio_ids[] = {
    ...
    { SDIO_DEVICE(SDIO_VENDOR_ID_MARVELL, SDIO_DEVICE_ID_MARVELL_8997_BT) },
    ...
};
```
المشكلة: `CONFIG_BT_MRVL_SDIO` كانت `=n` في الـ Yocto kernel config.

**الحل:**
```bash
# تحقق إن الـ function 2 موجود
ls /sys/bus/sdio/devices/
# المفروض تلاقي mmc0:0001:1 و mmc0:0001:2

cat /sys/bus/sdio/devices/mmc0\:0001\:2/device  # 0x9142

# في kernel config
grep BT_MRVL /boot/config-$(uname -r)
# لو مش موجود:
```
في الـ Yocto `linux-rockchip.cfg`:
```
CONFIG_BT_MRVL=m
CONFIG_BT_MRVL_SDIO=m
```
```bash
bitbake linux-rockchip -c menuconfig
# Enable: Networking → Bluetooth → Marvell Bluetooth drivers → SDIO
```

**الدرس المستفاد:**
الـ multi-function SDIO chips زي الـ Marvell 8997 بيحتاجوا kernel configs منفصلة لكل function. الـ `sdio_ids.h` بيفرق بين `_F0`, `_WLAN`, `_BT` — كل واحدة محتاج driver مستقل.

---

### سيناريو 3: Atheros AR6004 على STM32MP1 IoT Sensor مش بيتعرف

**العنوان:** Device ID مش في الـ table — Atheros AR6004 variant جديدة على custom IoT board

**السياق:**
منتج: IoT sensor board بـ STMicroelectronics STM32MP157.
الـ Wi-Fi chip: Atheros AR6004 بس variant جديدة من الـ supplier (revision 0x03).
البيئة: OpenSTLinux, kernel 5.15.

**المشكلة:**
الـ Wi-Fi مش بيشتغل خالص. `dmesg`:
```
mmc2: new high speed SDIO card at address 0001
ath6kl_sdio: probe of mmc2:0001:1 failed with error -ENODEV
```

**التحليل:**
في `sdio_ids.h`، الـ Atheros AR6004 entries:
```c
#define SDIO_VENDOR_ID_ATHEROS             0x0271
#define SDIO_DEVICE_ID_ATHEROS_AR6004_00   0x0400
#define SDIO_DEVICE_ID_ATHEROS_AR6004_01   0x0401
#define SDIO_DEVICE_ID_ATHEROS_AR6004_02   0x0402
#define SDIO_DEVICE_ID_ATHEROS_AR6004_18   0x0418
#define SDIO_DEVICE_ID_ATHEROS_AR6004_19   0x0419
```
الـ device جديدة بتـ report نفسها كـ `0x0403`. مش موجودة في الـ `sdio_device_id` table في `ath6kl_sdio.c`. الـ kernel probe بيعمل match على الـ table ولو مش لاقي يـ return `-ENODEV`.

نتحقق من الـ actual device ID:
```bash
cat /sys/bus/sdio/devices/mmc2\:0001\:1/device
# output: 0x0403
cat /sys/bus/sdio/devices/mmc2\:0001\:1/vendor
# output: 0x0271
```
مؤكد: الـ ID `0x0403` مش في الـ table.

**الحل:**
خطوتين:

1. إضافة الـ ID الجديدة في `sdio_ids.h`:
```c
/* في include/linux/mmc/sdio_ids.h */
#define SDIO_DEVICE_ID_ATHEROS_AR6004_03   0x0403  /* new hw revision */
```

2. إضافة الـ entry في driver table في `drivers/net/wireless/ath/ath6kl/sdio.c`:
```c
static const struct sdio_device_id ath6kl_sdio_devices[] = {
    ...
    {SDIO_DEVICE(SDIO_VENDOR_ID_ATHEROS, SDIO_DEVICE_ID_ATHEROS_AR6004_02)},
    {SDIO_DEVICE(SDIO_VENDOR_ID_ATHEROS, SDIO_DEVICE_ID_ATHEROS_AR6004_03)}, /* add this */
    ...
};
```

```bash
# rebuild و test
make M=drivers/net/wireless/ath/ath6kl modules
insmod ath6kl_sdio.ko
dmesg | grep ath6kl
```

**الدرس المستفاد:**
الـ `sdio_ids.h` هو الـ single source of truth للـ vendor/device IDs. أي hardware revision جديدة محتاج entry جديدة هنا **وفي** الـ driver table. مش كفاية تضيف في مكان واحد بس.

---

### سيناريو 4: TI WL1271 على i.MX8 Automotive ECU — SDIO Class Mismatch

**العنوان:** Wi-Fi SDIO card بيـ enumerate بـ wrong class على i.MX8M automotive ECU

**السياق:**
منتج: Automotive ECU بـ NXP i.MX8M Plus.
الـ Wi-Fi: Texas Instruments WL1271 متوصل بـ SDIO.
البيئة: Automotive Linux (AGL), kernel 6.1, custom BSP.

**المشكلة:**
الـ Wi-Fi بيـ enumerate بس الـ driver مش بيـ bind. الـ `dmesg`:
```
mmc1: new high speed SDIO card at address 0001
mmc1: card 0001 is a SDIO card with class 0x00
wl1271_sdio: no matching device found
```
والعجيب إن نفس الـ chip شغال على board تانية.

**التحليل:**
في `sdio_ids.h`:
```c
#define SDIO_CLASS_NONE    0x00  /* Not a SDIO standard interface */
#define SDIO_CLASS_WLAN    0x07  /* WLAN interface */

#define SDIO_VENDOR_ID_TI         0x0097
#define SDIO_DEVICE_ID_TI_WL1271  0x4076
```
الـ WL1271 driver في `drivers/net/wireless/ti/wlcore/sdio.c` بيعمل match على:
```c
static const struct sdio_device_id wl1271_devices[] = {
    { SDIO_DEVICE(SDIO_VENDOR_ID_TI, SDIO_DEVICE_ID_TI_WL1271) },
    {}
};
```
الـ matching هنا بيبصص على الـ vendor + device ID فقط، مش على الـ class. بس الـ card بتـ report class `0x00` (SDIO_CLASS_NONE) بدل `0x07` (WLAN).

السبب: الـ SDIO CIS على الـ custom board اتكتب غلط في الـ manufacturing — الـ OTP data مش صح. الـ kernel بيقرأ الـ class من الـ CIS، وبيـ log الـ mismatch.

التأكيد:
```bash
# قرأ الـ CIS manually
cat /sys/bus/sdio/devices/mmc1\:0001\:1/class  # يطلع 0x00 بدل 0x07
```

**الحل:**
الـ driver بيعمل match على vendor+device فقط مش على class، فالمشكلة مش في الـ matching. لكن الـ `SDIO_CLASS_NONE` خلى الـ kernel يـ log warning وبعد investigation طلع إن في chip آخر على نفس الـ bus بيـ intercept.

الحل الفعلي: إصلاح الـ Device Tree لتحديد الـ SDIO bus frequency صح:
```dts
/* arch/arm64/boot/dts/freescale/imx8mp-ecu.dts */
&usdhc2 {
    pinctrl-names = "default", "state_100mhz", "state_200mhz";
    bus-width = <4>;
    max-frequency = <25000000>;  /* كان 50MHz فبيحصل glitch */
    non-removable;
    status = "okay";
};
```

**الدرس المستفاد:**
الـ `SDIO_CLASS_*` constants في `sdio_ids.h` مهمة للـ diagnostics. لما تلاقي `SDIO_CLASS_NONE` على Wi-Fi card فابحث في الـ CIS programming أو الـ bus electrical issues مش في الـ driver code.

---

### سيناريو 5: Realtek RTW8723BS على AM62x Board Bring-up — Duplicate Vendor ID

**العنوان:** Siano و Microchip WILC1000 بيتعارضوا مع Realtek في الـ ID space أثناء bring-up

**السياق:**
منتج: Custom industrial board بـ Texas Instruments AM62x SoC.
الـ Wi-Fi: Realtek RTW8723BS متوصل بـ SDIO.
البيئة: TI SDK Linux, kernel 6.1, أول bring-up للـ board.

**المشكلة:**
أثناء الـ bring-up، المهندس لاحظ إن الـ SDIO Wi-Fi بيـ probe بـ Microchip WILC1000 driver بدل Realtek! الـ `dmesg`:
```
mmc0: new high speed SDIO card at address 0001
wilc_sdio mmc0:0001:1: chipid 0x00005347
wilc_sdio: probe succeeded (wrongly!)
```

**التحليل:**
في `sdio_ids.h`:
```c
#define SDIO_VENDOR_ID_MICROCHIP_WILC    0x0296
#define SDIO_DEVICE_ID_MICROCHIP_WILC1000  0x5347

#define SDIO_VENDOR_ID_SIANO             0x039a
#define SDIO_DEVICE_ID_SIANO_STELLAR     0x5347  /* نفس الـ device ID! */

#define SDIO_VENDOR_ID_REALTEK           0x024c
#define SDIO_DEVICE_ID_REALTEK_RTW8723BS 0xb723
```
لاحظ: `SDIO_DEVICE_ID_MICROCHIP_WILC1000` و `SDIO_DEVICE_ID_SIANO_STELLAR` كلاهم `0x5347`! لكن بـ vendor IDs مختلفة.

المشكلة الفعلية: الـ RTW8723BS اتـ solder غلط — الـ PCB layout استخدم نفس footprint لـ WILC1000 من design قديم. الـ chip على البورد فعلاً WILC1000 مش RTW8723BS.

```bash
# تأكيد
cat /sys/bus/sdio/devices/mmc0\:0001\:1/vendor  # 0x0296 مش 0x024c
cat /sys/bus/sdio/devices/mmc0\:0001\:1/device  # 0x5347
```

بعد تأكيد الـ vendor ID: `0x0296` = Microchip, مش Realtek `0x024c`.

**الحل:**
المشكلة hardware — الـ chip الغلط اتـ populate. بس لو كانت software، الـ fix في الـ DT:
```dts
/* arch/arm64/boot/dts/ti/k3-am62x-industrial.dts */
&sdhci1 {
    status = "okay";
    bus-width = <4>;
    non-removable;
    cap-power-off-card;

    wifi@1 {
        /* تحديد الـ chip الصح */
        compatible = "realtek,rtw8723bs-sdio";
        reg = <1>;
        interrupt-parent = <&gpio0>;
        interrupts = <23 IRQ_TYPE_LEVEL_HIGH>;
    };
};
```

وللتأكد من الـ IDs قبل الـ bringup:
```bash
# script للتحقق من الـ SDIO IDs في بداية الـ bringup
echo "Checking SDIO device IDs..."
for dev in /sys/bus/sdio/devices/*/; do
    vendor=$(cat "$dev/vendor" 2>/dev/null)
    device=$(cat "$dev/device" 2>/dev/null)
    class=$(cat  "$dev/class"  2>/dev/null)
    echo "  Device: $(basename $dev) => vendor=$vendor device=$device class=$class"
done
```

**الدرس المستفاد:**
في الـ board bring-up، أول حاجة تعملها: اقرأ الـ vendor ID من `/sys/bus/sdio/devices/` وقارنه بالـ `sdio_ids.h` **قبل** ما تشتغل على الـ software. الـ `0x5347` بيظهر مرتين في الـ file (Microchip + Siano) — الـ vendor ID هو اللي بيفرق. اكتشاف الـ wrong chip مبكراً بيوفر أيام من الـ debug.
## Phase 7: مصادر ومراجع

---

### 1. مقالات LWN.net

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| SDIO support coming | [lwn.net/Articles/242744](https://lwn.net/Articles/242744/) | إعلان Pierre Ossman بإضافة SDIO للـ kernel |
| MMC updates | [lwn.net/Articles/253888](https://lwn.net/Articles/253888/) | تحديثات MMC layer وإضافة SDIO + SPI support |
| 1/4 MMC layer | [lwn.net/Articles/82765](https://lwn.net/Articles/82765/) | أول patch لـ MMC framework |
| new driver: MMC framework | [lwn.net/Articles/7176](https://lwn.net/Articles/7176/) | patch الـ MMC الأصلي ضد kernel 2.4.19 |
| Add support for Meson MX SDIO MMC | [lwn.net/Articles/734836](https://lwn.net/Articles/734836/) | مثال على إضافة controller driver |
| mmc: core: add SD4.0 support | [lwn.net/Articles/642336](https://lwn.net/Articles/642336/) | إضافة SD 4.0 للـ kernel |
| StarFive SDIO/eMMC driver support | [lwn.net/Articles/923377](https://lwn.net/Articles/923377/) | driver حديث لـ VisionFive 2 board |

---

### 2. التوثيق الرسمي للـ Kernel

**Documentation/ paths داخل kernel source tree:**

```
Documentation/driver-api/mmc/index.rst        -- نقطة الدخول الرئيسية لـ MMC subsystem
Documentation/driver-api/mmc/mmc-dev-attrs.rst -- device attributes
Documentation/driver-api/mmc/mmc-dev-parts.rst -- device partitions
Documentation/driver-api/mmc/mmc-async-req.rst -- asynchronous requests
Documentation/driver-api/mmc/mmc-tools.rst     -- أدوات userspace
```

**روابط online:**

- [kernel.org: MMC/SD/SDIO card support](https://www.kernel.org/doc/html/latest/driver-api/mmc/index.html)
- [kernel.org: SD and MMC Device Partitions](https://www.kernel.org/doc/html/latest/driver-api/mmc/mmc-dev-parts.html)
- [static.lwn.net: kernel MMC/SD/SDIO docs](https://static.lwn.net/kerneldoc/driver-api/mmc/index.html)

**الملف المدروس في kernel source:**

```
include/linux/mmc/sdio_ids.h    -- SDIO vendor/device IDs وclass definitions
include/linux/mmc/sdio.h        -- SDIO protocol constants وregister definitions
include/linux/mmc/sdio_func.h   -- sdio_func struct وAPI للـ SDIO drivers
drivers/mmc/core/sdio.c         -- SDIO core implementation
drivers/mmc/core/sdio_bus.c     -- SDIO bus driver model
drivers/mmc/core/sdio_io.c      -- SDIO I/O functions
```

---

### 3. Kernel Commits المهمة

| الوصف | الرابط |
|-------|--------|
| sdio_ids.h على GitHub (torvalds/linux) | [github.com/torvalds/linux/.../sdio_ids.h](https://github.com/torvalds/linux/blob/master/include/linux/mmc/sdio_ids.h) |
| Broadcom WLAN SDIO IDs patch (Patchwork) | [patchwork.kernel.org/patch/3398481](https://patchwork.kernel.org/patch/3398481/) |
| sdio_ids.h على Bootlin Elixir (v6.6.1) | [elixir.bootlin.com/linux/latest/source/include/linux/mmc/sdio_ids.h](https://elixir.bootlin.com/linux/latest/source/include/linux/mmc/sdio_ids.h) |
| sdio_ids.h على Android kernel (Google Git) | [android.googlesource.com: sdio_ids.h](https://android.googlesource.com/kernel/common/+/1669a941f7c4844ae808cf441db51dde9e94db07/include/linux/mmc/sdio_ids.h) |
| linux/commits history (GitHub) | [github.com/torvalds/linux/commits/master](https://github.com/torvalds/linux/commits/master/) |

---

### 4. كتب مقترحة

| الكتاب | الفصل ذو الصلة | الملاحظة |
|--------|----------------|---------|
| **Linux Device Drivers, 3rd Ed.** (LDD3) | Chapter 14: The Linux Device Model | يشرح bus/driver/device model اللي بيتبنى عليه SDIO bus |
| **Linux Kernel Development** — Robert Love | Chapter 14: The Block I/O Layer | يفيد لفهم I/O stack |
| **Embedded Linux Primer** — Christopher Hallinan | Chapter 11: MTD Subsystem | يغطي storage subsystems في embedded systems |
| **Essential Linux Device Drivers** — Sreekrishnan Venkateswaran | Chapter 15: Multimedia Cards | يغطي MMC/SD/SDIO بشكل مباشر |

---

### 5. موارد elinux.org

| الصفحة | الرابط |
|--------|--------|
| Tests:SDIO-KS7010 — اختبار SDIO مع Spectec WiFi card | [elinux.org/Tests:SDIO-KS7010](https://elinux.org/Tests:SDIO-KS7010) |
| Tests:SDIO-with-UHS — اختبار SDIO مع UHS support | [elinux.org/Tests:SDIO-with-UHS](https://elinux.org/Tests:SDIO-with-UHS) |
| Libertas SDIO — Marvell Libertas driver وفيرموير SDIO | [elinux.org/Libertas_SDIO](https://elinux.org/Libertas_SDIO) |
| Didj and Explorer MMC Patch | [elinux.org/Didj_and_Explorer_MMC_Patch](https://elinux.org/Didj_and_Explorer_MMC_Patch) |

---

### 6. موارد إضافية

- **Bootlin Elixir Cross-Referencer** — للبحث في kernel source:
  [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/include/linux/mmc)

- **systemd hwdb لـ SDIO IDs** — قاعدة بيانات SDIO devices:
  [github.com/systemd/systemd: sdio.ids](https://github.com/systemd/systemd/blob/main/hwdb.d/sdio.ids)

- **Linux Kernel Driver DataBase — CONFIG_MMC:**
  [cateee.net/lkddb: MMC](https://cateee.net/lkddb/web-lkddb/MMC.html)

- **STM32 MPU: MMC overview:**
  [wiki.st.com/stm32mpu/wiki/MMC_overview](https://wiki.st.com/stm32mpu/wiki/MMC_overview)

---

### 7. Search Terms للبحث عن معلومات أكتر

```
# للبحث في kernel mailing list وLWN
"SDIO function driver" site:lwn.net
"sdio_register_driver" linux kernel
"SDIO_VENDOR_ID" site:github.com/torvalds/linux

# للبحث عن drivers بعيني
"sdio_ids.h" vendor_id device_id linux
"sdio_func" linux kernel driver example
SDIO subsystem linux kernel internals

# للبحث في hardware datasheets
"SDIO specification" SD Association
"CCCR" SDIO card capability register
"FBR" SDIO function basic register

# للبحث عن debugging وtools
"mmc_test" linux kernel module
sdio debugfs linux
"/sys/bus/sdio" linux device tree
```

---

### 8. ملف `sdio_ids.h` — ملخص سريع للمحتوى

الملف `include/linux/mmc/sdio_ids.h` يحتوي على:

| النوع | المحتوى |
|-------|---------|
| **SDIO_CLASS_*** | 10 standard interface classes (UART, BT, GPS, WLAN, ...) |
| **Vendors** | STE, Intel, Atheros, Broadcom, Cypress, Marvell, MediaTek, Microchip, Realtek, Siano, RSI, TI |
| **Device IDs** | ~60+ device IDs لأجهزة WiFi وBluetooth وDVB-T وغيرها |
| **الاستخدام** | `sdio_device_id` table في كل SDIO driver لـ device matching |

**مثال على الاستخدام في driver:**

```c
/* استخدام IDs من sdio_ids.h في driver */
static const struct sdio_device_id brcmf_sdmmc_ids[] = {
    SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4329),
    SDIO_DEVICE(SDIO_VENDOR_ID_BROADCOM, SDIO_DEVICE_ID_BROADCOM_4330),
    { /* end of list */ }
};
MODULE_DEVICE_TABLE(sdio, brcmf_sdmmc_ids);
```
## Phase 8: Writing simple module

الملف `sdio_ids.h` بيحتوي بس على macros للـ vendor/device IDs والـ SDIO class codes — مفيش functions أو tracepoints مباشرة فيه. لكن الـ IDs دي بتتستخدم جوا الـ SDIO subsystem لما بيتعمل probe لأي device. الهوك الأنسب هنا هو `sdio_bus_match` أو الأحسن: kprobe على `sdio_claim_host` — دي function بتتنادى كل ما driver SDIO بيبدأ يكلم device، وبتعدي `struct sdio_func *` اللي فيها الـ vendor ID والـ device ID بالظبط اللي معرّفين في الملف.

---

### الـ Module

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * sdio_probe_hook.c
 * kprobe on sdio_claim_host — prints vendor/device IDs
 * from sdio_ids.h when any SDIO driver claims its device.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/mmc/sdio_func.h>   /* struct sdio_func */
#include <linux/mmc/sdio_ids.h>    /* SDIO_VENDOR_ID_* / SDIO_DEVICE_ID_* */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("kprobe on sdio_claim_host — log vendor/device IDs");

/* ------------------------------------------------------------------ */
/* Helper: map known vendor IDs (from sdio_ids.h) to a readable name  */
/* ------------------------------------------------------------------ */
static const char *vendor_name(u16 vid)
{
    switch (vid) {
    case SDIO_VENDOR_ID_BROADCOM:        return "Broadcom";
    case SDIO_VENDOR_ID_MARVELL:         return "Marvell";
    case SDIO_VENDOR_ID_ATHEROS:         return "Atheros";
    case SDIO_VENDOR_ID_REALTEK:         return "Realtek";
    case SDIO_VENDOR_ID_MEDIATEK:        return "MediaTek";
    case SDIO_VENDOR_ID_TI:              return "TI";
    case SDIO_VENDOR_ID_TI_WL1251:       return "TI-WL1251";
    case SDIO_VENDOR_ID_INTEL:           return "Intel";
    case SDIO_VENDOR_ID_SIANO:           return "Siano";
    case SDIO_VENDOR_ID_RSI:             return "RSI";
    case SDIO_VENDOR_ID_MICROCHIP_WILC:  return "Microchip-WILC";
    case SDIO_VENDOR_ID_CYPRESS:         return "Cypress";
    case SDIO_VENDOR_ID_STE:             return "STE";
    case SDIO_VENDOR_ID_CGUYS:           return "C-guys";
    default:                             return "Unknown";
    }
}

/* ------------------------------------------------------------------ */
/* kprobe pre-handler: called just before sdio_claim_host() executes  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * sdio_claim_host(struct sdio_func *func)
     * On x86-64: first arg is in RDI; on ARM64: in X0.
     * regs_get_kernel_argument(regs, 0) is portable across arches.
     */
    struct sdio_func *func =
        (struct sdio_func *)regs_get_kernel_argument(regs, 0);

    if (!func)
        return 0;

    pr_info("sdio_hook: sdio_claim_host called — "
            "func#%u  vendor=0x%04x (%s)  device=0x%04x  class=0x%02x\n",
            func->num,
            func->vendor,
            vendor_name(func->vendor),
            func->device,
            func->class);

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/* kprobe struct                                                        */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "sdio_claim_host",  /* function to hook */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init sdio_hook_init(void)
{
    int ret = register_kprobe(&kp);

    if (ret < 0) {
        pr_err("sdio_hook: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("sdio_hook: kprobe planted on %s @ %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit sdio_hook_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("sdio_hook: kprobe removed from %s\n", kp.symbol_name);
}

module_init(sdio_hook_init);
module_exit(sdio_hook_exit);
```

---

### شرح كل جزء

| الجزء | الهدف |
|---|---|
| `#include <linux/mmc/sdio_ids.h>` | بنجيب الـ macros (vendor/device IDs) عشان نستخدمها في switch |
| `#include <linux/mmc/sdio_func.h>` | بنجيب `struct sdio_func` اللي فيها `.vendor` و `.device` و `.class` |
| `vendor_name()` | بتحوّل الـ vendor ID الـ hex لاسم مقروء — بتستخدم نفس الـ macros من `sdio_ids.h` |
| `handler_pre()` | ده الـ callback اللي بيتنفذ قبل ما `sdio_claim_host` تشتغل؛ بيسحب الـ `sdio_func*` من الـ registers |
| `regs_get_kernel_argument(regs, 0)` | portable way عشان نجيب الـ argument الأول من الـ registers على أي architecture |
| `struct kprobe kp` | بتحدد اسم الـ function المستهدفة والـ callback |
| `register_kprobe` / `unregister_kprobe` | بتزرع وبتشيل الـ hook بأمان في `init` و `exit` |

---

### ليه `sdio_claim_host`؟

`sdio_claim_host` بتتنادى من كل SDIO driver قبل ما يبعت أي command للـ device — يعني دي نقطة مركزية guaranteed تمر بيها كل SDIO transaction. ده بيخلينا نشوف **أي device اتعمله probe** وبيانات الـ vendor/device ID اللي معرّفة في `sdio_ids.h` من غير ما نعدّل في أي driver أصلي.

---

### Makefile للتجربة

```makefile
obj-m += sdio_probe_hook.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

**تشغيل:**

```bash
sudo insmod sdio_probe_hook.ko
# شيل SDIO device أو لود driver عشان تشوف الـ log
sudo dmesg | grep sdio_hook
sudo rmmod sdio_probe_hook
```

**مثال على الـ output:**

```
sdio_hook: kprobe planted on sdio_claim_host @ ffffffffc0a12340
sdio_hook: sdio_claim_host called — func#1  vendor=0x02d0 (Broadcom)  device=0x4335  class=0x00
```
