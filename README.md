# zte-f6600-libre

Firmware **100 % libre** para el **ZTE ZXHN F6600**, un ONT GPON con WiFi 6, y otros equipos con el mismo SoC:
Linux mainline, OpenWrt y drivers propios, sin blobs ni módulos del fabricante.

> **Estado: alfa.** El código todavía no está publicado; este repositorio por ahora solo tiene este README.
> Se va a subir cuando la primera alfa esté grabada en flash y validada.

## Hardware

| | |
|---|---|
| Equipo | ZTE ZXHN F6600 (ONT GPON + router WiFi 6) |
| SoC | ZTE **ZX279128S**: 2 × Cortex-A9 a 1 GHz, PL310 L2, GIC |
| RAM / flash | 256 MiB DDR / SPI-NAND ESMT F50L1G41LB (128 MiB), dos bancos de firmware |
| WiFi | MediaTek **MT7915** (2,4 + 5 GHz, 802.11ax) por PCIe |
| LAN | 4 × Gigabit detrás de un switch fabric interno (TM/PP/NPP) |
| PON | MAC GPON integrado, láser driver Semtech **GN25L95** (I2C) |
| Telefonía | FXS con SLIC Silicon Labs ProSLIC |

El ZXHN F680 V6 usa el mismo SoC (ZX279128), así que gran parte del trabajo le sirve también.

## Estado de la primera alfa

Kernel **Linux 6.18.57 LTS** propio más userspace **OpenWrt 25.12.5**, armado con `apk` a partir de los
paquetes oficiales `arm_fa526` y cargado como initramfs. **Grabada en el banco 1 de la flash** y validada con cortes
de energía reales (arranca sola, LuCI en ~40–55 s); el banco 0 conserva el firmware de fábrica como red de seguridad.

| Componente | Estado |
|---|---|
| Arranque, consola UART (PL011, variante ZTE), timer, GIC, L2, reinicio | ✅ |
| Ethernet: driver propio del switch fabric (TM DMA, anillos RX/TX, MDIO, FDB) | ✅ 1 Gbps; 25 MB/s TCP; probado en 1 de 4 puertos |
| VLAN 802.1Q y bridges | ✅ |
| PCIe (DesignWare) + WiFi MT7915 con `mt76` | ✅ 2,4 y 5 GHz, WPA2, calibración de la unidad leída del eFuse |
| SPI-NAND (driver propio del controlador SPIFC) | ✅ lectura; ⏳ escritura deshabilitada a propósito |
| Firewall (firewall4/nftables), NAT, DHCP/DNS (dnsmasq), LuCI | ✅ |
| WAN PPPoE (VLAN 10) | ✅ configurado, ⏳ sin probar con un proveedor real |
| Segundo núcleo (SMP) | ⚠️ funciona, pero se congela de forma intermitente cuando el WiFi por PCIe trabaja en paralelo con CPU1. La alfa arranca con `maxcpus=1` mientras se investiga |
| GPON (MAC, PLOAM/ranging, óptica, OMCI, GEM/T-CONT) | 🚧 drivers propios del MAC GPON, GPIO e I²C escritos; SerDes PON enganchada y doble cierre del láser verificados sin fibra; primera prueba con OLT real pendiente |
| USB 3 (DWC3) | ❌ todavía no |
| Telefonía FXS (SLIC + DSP + RTP) | ❌ todavía no |

Hitos de estabilidad ya validados en hardware, con campañas automáticas de arranque y cortes de energía reales:

- 10 de 10 arranques limpios con un núcleo.
- Prueba de 30 min: 0 % de pérdida de ping, 1.329 descargas de 8 MiB verificadas por hash (~25 MB/s).
- Arranque desde flash tras cortes de energía reales, con LuCI completo (incluido firewall).
- DHCP, y ARP hacia hosts inexistentes, sin trabar la recepción.
- Tráfico TCP sostenido sin tormentas de interrupciones.

## Objetivo final

1. **Todo el SoC en mainline**: clocks, resets, SMP, NAND con escritura, USB, PCIe y Ethernet, con DTS upstream.
2. **GPON libre de punta a punta**: driver del MAC GPON (PLOAM, ranging O1–O5, AES), óptica GN25L95 con DDM,
   y un stack **OMCI propio** (`omcid`) que se registre en un OLT real y configure GEM/T-CONT y VLANs.
   Sin `cspd`, sin `gpon_omci` y sin módulos del fabricante.
3. **Telefonía FXS** libre: driver del SLIC y audio, con las credenciales SIP que entrega el OLT por OMCI.
4. **Imagen OpenWrt instalable**, con actualización segura entre los dos bancos de flash.
5. **Enviar upstream** lo que corresponda: Linux, OpenWrt y mt76.

## Cómo se hace

Ingeniería inversa de hardware propio: lectura de los módulos del kernel de fábrica (licenciados GPL/Dual,
sin stripear), de U-Boot y del kernel 4.1.25 de fábrica; volcados de registros por UART; y pruebas sobre el
equipo real. La verificación se automatiza con campañas de N arranques por RAM, con cortes de energía por un
enchufe inteligente.

### Herramientas e IA usadas

Proyecto dirigido por Juan Manuel Reyes ([@juanmreyesa](https://github.com/juanmreyesa)), con ayuda de varios
asistentes de IA que se reparten el análisis, el código y la revisión cruzada:

| IA | Uso |
|---|---|
| **Claude** (Anthropic, Claude Code: Opus) | Coordinación, diseño de experimentos en hardware, depuración de drivers, integración |
| **OpenAI Codex** (`gpt-6-sol`, `gpt-5.6-sol`) | Ingeniería inversa del kernel y de los módulos de fábrica, SMP, SPI-NAND, fabric, PCIe, revisiones |
| **Grok** (xAI, Grok Build) | ABI de CMAPI, `ioctl_dev`, VLAN/GEM y FXS del stack de fábrica |
| **Kimi K3** (Moonshot AI, vía OpenCode Go) | Análisis de la tormenta de IRQ del fabric |
| **GLM 5.3** (Zhipu AI, vía OpenCode Go) | Síntesis del plan GPON |
| **DeepSeek V4.1 Flash** (vía OpenCode, gratis) | Ingeniería inversa del MAC GPON (PLOAM, ranging) e IP del firmware de fábrica |
| **GLM 5.3 Flash** (vía OpenCode, gratis) | Óptica GN25L95, DHCP/firewall en OpenWrt |
| **Nemotron 3 Ultra** (NVIDIA, vía OpenCode, gratis) | Camino OMCI/GEM en el TM, U-Boot |
| **MiMo V2.6 Flash** (Xiaomi, vía OpenCode, gratis) | Device tree, clocks y dependencias del bloque PON |

Herramientas: Ghidra (+ GhidraMCP), REA, binutils ARM, OpenWrt `apk`, scripts propios de UART, XMODEM y TFTP.

Ninguna salida de IA se da por buena sin evidencia en el binario o en el hardware: cada hallazgo se marca como
visto (V) o inferido (I), y los arreglos se validan con campañas de arranques en el equipo real.

## Licencia

Software libre. Las partes de kernel van bajo **GPL-2.0**, como Linux; las herramientas y scripts propios,
bajo GPL-2.0-or-later. No se distribuyen binarios, firmwares ni datos de calibración del fabricante.
