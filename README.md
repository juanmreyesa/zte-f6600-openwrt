# zte-f6600-openwrt

**English** · [Español](#español)

Open-source firmware for the **ZTE ZXHN F6600**, a GPON ONT with Wi-Fi 6, and other devices built on the same SoC:
mainline Linux, OpenWrt and purpose-written drivers. No vendor kernel modules and no vendor userspace; the only
binary pieces are the redistributable MT7915 Wi-Fi firmware files from `linux-firmware`.

> **Status: alpha.** The source code is not published yet; for now this repository only holds this README.

## Hardware

| | |
|---|---|
| Device | ZTE ZXHN F6600 (GPON ONT + Wi-Fi 6 router) |
| SoC | ZTE **ZX279128S**: 2 × Cortex-A9 @ 1 GHz, PL310 L2 cache, GIC |
| RAM / flash | 256 MiB DDR / ESMT F50L1G41LB SPI-NAND (128 MiB), two firmware banks |
| Wi-Fi | MediaTek **MT7915** (2.4 + 5 GHz, 802.11ax) over PCIe |
| LAN | 4 × Gigabit behind an internal switch fabric (TM/PP/NPP) |
| PON | Integrated GPON MAC, Semtech **GN25L95** laser driver (I²C) |
| Telephony | FXS port with a Silicon Labs ProSLIC |

The ZXHN F680 V6 uses the same SoC family (ZX279128), so much of this work applies to it as well.

## Current alpha

A **Linux 6.18.57 LTS** kernel with **OpenWrt 25.12.5** userspace (assembled with `apk` from the official
packages). It is installed in **bank 1** of the flash and boots on its own; bank 0 keeps the factory firmware as a
fallback. Every release is validated on the real device with automated boot campaigns and real power cuts.

| Component | Status |
|---|---|
| Boot, UART console (ZTE PL011 variant), timers, GIC, L2, reset, hardware watchdog | ✅ (watchdog recovers any hang in ~30 s) |
| Ethernet: own switch-fabric driver (TM DMA rings, MDIO, FDB), 4 ports | ✅ Gigabit |
| 802.1Q VLANs and bridges | ✅ |
| PCIe (DesignWare) + MT7915 Wi-Fi with `mt76` | ✅ 2.4 and 5 GHz, Wi-Fi 6, per-unit calibration read from eFuse |
| SPI-NAND (own SPIFC controller driver) | ✅ read and write |
| `sysupgrade` from OpenWrt / LuCI | ✅ verified writes, vendor header preserved |
| Persistent configuration (overlay on `/etc`, JFFS2) | ✅ survives reboots, power cuts and upgrades; factory reset with `firstboot` |
| Firewall (firewall4/nftables), NAT, DHCP/DNS, LuCI | ✅ |
| PPPoE WAN (VLAN 10) | ✅ configured, ⏳ not yet tested with a real ISP |
| USB 3 (DWC3/xHCI) | ✅ controller up; ⏳ no real device tested yet |
| Second CPU core (SMP) | ⚠️ works, but the SoC freezes when Wi-Fi/PCIe work runs on CPU1; the alpha runs with `maxcpus=1` while this is investigated |
| GPON (MAC, PLOAM/ranging, optics, OMCI, GEM/T-CONT) | 🚧 GPON MAC, GPIO, I²C and OMCI drivers written and tested without fibre; first OLT test pending |
| Hardware forwarding offload (switch fabric) | 🔬 under reverse engineering |
| FXS telephony (SLIC + DSP + RTP) | ❌ not yet |

Measured forwarding performance (iperf3, single core, all traffic through the CPU):

| Mode | TCP |
|---|---|
| Bridge | ~417 Mbit/s |
| Router (NAT + firewall) | ~352 Mbit/s |

## Goals

1. **The whole SoC in mainline Linux**: clocks, resets, SMP, NAND, USB, PCIe and Ethernet, with upstream device trees.
2. **An open GPON stack end to end**: GPON MAC driver (PLOAM, O1–O5 ranging), GN25L95 optics with DDM, and an
   **in-house OMCI daemon** that registers with a real OLT and configures GEM ports, T-CONTs and VLANs, without
   `cspd`, `gpon_omci` or any vendor module. Firmware updates pushed by the OLT are captured for inspection but
   never applied.
3. **Bridge and router modes**, selectable locally or by the operator through OMCI.
4. **Hardware forwarding offload** in the switch fabric to reach line rate.
5. **FXS telephony**: SLIC and audio drivers, using the SIP credentials delivered by the OLT over OMCI.
6. **Upstream** whatever fits: Linux, OpenWrt and mt76.

## How it is done

Reverse engineering of hardware we own: the factory kernel modules (GPL/Dual-licensed and not stripped), U-Boot and
the factory 4.1.25 kernel are disassembled; registers are inspected over UART and live (including running the
factory firmware from RAM to compare its configuration); everything is checked on the real device. Validation is
automated with RAM-boot campaigns and power cuts through a smart plug.

### Tools and AI

Project led by Juan Manuel Reyes ([@juanmreyesa](https://github.com/juanmreyesa)), with several AI assistants
sharing analysis, code and cross-review:

| AI | Used for |
|---|---|
| **Claude** (Anthropic, Claude Code: Opus) | Coordination, hardware experiments, driver debugging and performance work, integration |
| **OpenAI Codex** (`gpt-6-sol`, `gpt-5.6-sol`) | Reverse engineering of the factory kernel and modules: SMP, SPI-NAND, fabric, PCIe, switch offload; reviews |
| **Grok** (xAI, Grok Build) | Factory-stack ABIs: CMAPI, `ioctl_dev`, VLAN/GEM, FXS |
| **Kimi K3** (Moonshot AI, via OpenCode Go) | Fabric IRQ storm analysis |
| **GLM 5.3** (Zhipu AI, via OpenCode Go) | GPON plan synthesis |
| **DeepSeek V4.1 Flash** (via OpenCode) | GPON MAC reverse engineering (PLOAM, ranging) |
| **GLM 5.3 Flash** (via OpenCode) | GN25L95 optics, DHCP/firewall in OpenWrt |
| **Nemotron 3 Ultra** (NVIDIA, via OpenCode) | OMCI/GEM path in the traffic manager, U-Boot |
| **MiMo V2.6 Flash** (Xiaomi, via OpenCode) | Device tree, clocks and PON block dependencies |
| **LongCat 2.5** (Meituan, via OpenCode) | Switch-fabric offload analysis |

Tools: Ghidra (+ GhidraMCP), REA, ARM binutils, the OpenWrt SDK and `apk`, and in-house UART, TFTP and
benchmarking scripts.

No AI output is taken at face value: every finding is labelled as *seen* in the binary or on the hardware, or as
*inferred*, and every fix is validated on the real device.

## License

Kernel code is licensed under **GPL-2.0**, like Linux; in-house tools and scripts under GPL-2.0-or-later.
No vendor binaries, firmware images or calibration data are distributed.

---

# Español

[English](#zte-f6600-openwrt) · **Español**

Firmware de código abierto para el **ZTE ZXHN F6600**, un ONT GPON con WiFi 6, y otros equipos con el mismo SoC:
Linux mainline, OpenWrt y drivers escritos para el proyecto. No usa módulos de kernel ni programas del fabricante;
las únicas piezas binarias son los archivos de firmware redistribuibles del WiFi MT7915, de `linux-firmware`.

> **Estado: alfa.** El código todavía no está publicado; por ahora este repositorio solo contiene este README.

## Hardware

| | |
|---|---|
| Equipo | ZTE ZXHN F6600 (ONT GPON + router WiFi 6) |
| SoC | ZTE **ZX279128S**: 2 × Cortex-A9 a 1 GHz, caché L2 PL310, GIC |
| RAM / flash | 256 MiB DDR / SPI-NAND ESMT F50L1G41LB (128 MiB), dos bancos de firmware |
| WiFi | MediaTek **MT7915** (2,4 + 5 GHz, 802.11ax) por PCIe |
| LAN | 4 × Gigabit detrás de un switch fabric interno (TM/PP/NPP) |
| PON | MAC GPON integrado, driver de láser Semtech **GN25L95** (I²C) |
| Telefonía | Puerto FXS con un ProSLIC de Silicon Labs |

El ZXHN F680 V6 usa la misma familia de SoC (ZX279128), así que buena parte de este trabajo también le sirve.

## Alfa actual

Kernel **Linux 6.18.57 LTS** con userspace **OpenWrt 25.12.5** (armado con `apk` a partir de los paquetes oficiales).
Está instalado en el **banco 1** de la flash y arranca solo; el banco 0 conserva el firmware de fábrica como respaldo.
Cada versión se valida en el equipo real con campañas automáticas de arranque y cortes de energía reales.

| Componente | Estado |
|---|---|
| Arranque, consola UART (variante ZTE del PL011), timers, GIC, L2, reinicio, watchdog por hardware | ✅ (el watchdog recupera cualquier cuelgue en ~30 s) |
| Ethernet: driver propio del switch fabric (anillos DMA del TM, MDIO, FDB), 4 puertos | ✅ Gigabit |
| VLAN 802.1Q y bridges | ✅ |
| PCIe (DesignWare) + WiFi MT7915 con `mt76` | ✅ 2,4 y 5 GHz, WiFi 6, calibración propia de la unidad leída del eFuse |
| SPI-NAND (driver propio del controlador SPIFC) | ✅ lectura y escritura |
| `sysupgrade` desde OpenWrt / LuCI | ✅ escritura verificada, preserva la cabecera del fabricante |
| Configuración persistente (overlay sobre `/etc`, JFFS2) | ✅ sobrevive reinicios, cortes de energía y actualizaciones; reset de fábrica con `firstboot` |
| Firewall (firewall4/nftables), NAT, DHCP/DNS, LuCI | ✅ |
| WAN PPPoE (VLAN 10) | ✅ configurado, ⏳ sin probar todavía con un proveedor real |
| USB 3 (DWC3/xHCI) | ✅ controlador funcionando; ⏳ sin probar con un dispositivo real |
| Segundo núcleo (SMP) | ⚠️ funciona, pero el SoC se congela cuando el trabajo del WiFi/PCIe corre en CPU1; la alfa arranca con `maxcpus=1` mientras se investiga |
| GPON (MAC, PLOAM/ranging, óptica, OMCI, GEM/T-CONT) | 🚧 drivers del MAC GPON, GPIO, I²C y OMCI escritos y probados sin fibra; falta la primera prueba con un OLT |
| Aceleración por hardware del reenvío (switch fabric) | 🔬 en ingeniería inversa |
| Telefonía FXS (SLIC + DSP + RTP) | ❌ todavía no |

Rendimiento de reenvío medido (iperf3, un núcleo, todo el tráfico por la CPU):

| Modo | TCP |
|---|---|
| Puente | ~417 Mbit/s |
| Router (NAT + firewall) | ~352 Mbit/s |

## Objetivos

1. **Todo el SoC en Linux mainline**: clocks, resets, SMP, NAND, USB, PCIe y Ethernet, con device trees upstream.
2. **Un stack GPON abierto de punta a punta**: driver del MAC GPON (PLOAM, ranging O1–O5), óptica GN25L95 con DDM
   y un **daemon OMCI propio** que se registre en un OLT real y configure puertos GEM, T-CONT y VLAN, sin `cspd`,
   `gpon_omci` ni módulos del fabricante. Las actualizaciones de firmware que envíe el OLT se guardan para
   analizarlas, pero nunca se aplican.
3. **Modos puente y router**, elegibles en el equipo o por el operador vía OMCI.
4. **Aceleración por hardware** en el switch fabric para llegar a velocidad de línea.
5. **Telefonía FXS**: drivers del SLIC y del audio, con las credenciales SIP que entrega el OLT por OMCI.
6. **Enviar upstream** lo que corresponda: Linux, OpenWrt y mt76.

## Cómo se hace

Ingeniería inversa de hardware propio: se desensamblan los módulos del kernel de fábrica (licenciados GPL/Dual y sin
stripear), U-Boot y el kernel 4.1.25 de fábrica; los registros se inspeccionan por UART y en vivo (incluso corriendo
el firmware de fábrica en RAM para comparar su configuración), y todo se comprueba en el equipo real. La validación se
automatiza con campañas de arranque por RAM y cortes de energía mediante un enchufe inteligente.

### Herramientas e IA

Proyecto dirigido por Juan Manuel Reyes ([@juanmreyesa](https://github.com/juanmreyesa)), con varios asistentes de IA
que se reparten el análisis, el código y la revisión cruzada:

| IA | Uso |
|---|---|
| **Claude** (Anthropic, Claude Code: Opus) | Coordinación, experimentos en hardware, depuración y rendimiento de drivers, integración |
| **OpenAI Codex** (`gpt-6-sol`, `gpt-5.6-sol`) | Ingeniería inversa del kernel y los módulos de fábrica: SMP, SPI-NAND, fabric, PCIe, offload del switch; revisiones |
| **Grok** (xAI, Grok Build) | ABI del stack de fábrica: CMAPI, `ioctl_dev`, VLAN/GEM, FXS |
| **Kimi K3** (Moonshot AI, vía OpenCode Go) | Análisis de la tormenta de IRQ del fabric |
| **GLM 5.3** (Zhipu AI, vía OpenCode Go) | Síntesis del plan GPON |
| **DeepSeek V4.1 Flash** (vía OpenCode) | Ingeniería inversa del MAC GPON (PLOAM, ranging) |
| **GLM 5.3 Flash** (vía OpenCode) | Óptica GN25L95, DHCP/firewall en OpenWrt |
| **Nemotron 3 Ultra** (NVIDIA, vía OpenCode) | Camino OMCI/GEM en el gestor de tráfico, U-Boot |
| **MiMo V2.6 Flash** (Xiaomi, vía OpenCode) | Device tree, clocks y dependencias del bloque PON |
| **LongCat 2.5** (Meituan, vía OpenCode) | Análisis del offload del switch fabric |

Herramientas: Ghidra (+ GhidraMCP), REA, binutils ARM, el SDK y `apk` de OpenWrt, y scripts propios de UART, TFTP y
medición de rendimiento.

Ninguna salida de IA se da por buena sin evidencia: cada hallazgo se marca como *visto* en el binario o en el
hardware, o como *inferido*, y cada arreglo se valida en el equipo real.

## Licencia

El código de kernel se publica bajo **GPL-2.0**, como Linux; las herramientas y scripts propios, bajo
GPL-2.0-or-later. No se distribuyen binarios, imágenes de firmware ni datos de calibración del fabricante.
