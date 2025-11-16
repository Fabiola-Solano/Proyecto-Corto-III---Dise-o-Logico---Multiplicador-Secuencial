# 🎯 PROYECTO FPGA: Teclado 4×4 + Multiplicador Booth + Display 7seg

> **Sistema Digital Completo en Tang Nano 9K | SystemVerilog | Documentación + Diagramas ASM integrados**

![Status](https://img.shields.io/badge/Status-COMPLETO-brightgreen) ![Language](https://img.shields.io/badge/Language-SystemVerilog-blue) ![Platform](https://img.shields.io/badge/Platform-Tang%20Nano%209K-orange) ![Docs](https://img.shields.io/badge/Docs-COMPLETA-blue)

---

##  Resumen Ejecutivo

**Sistema digital embebido en FPGA** que integra:
-  **Entrada:** Teclado matricial 4×4 con antirrebote (1.2 ms)
-  **Procesamiento:** Multiplicador Booth radix-2 (10 ciclos)
-  **Salida:** 4 displays 7-seg multiplexados (5 kHz, sin ghosting)
-  **Sincronización:** CDC (Clock Domain Crossing) robusta sin metaestabilidad

**Ejemplo de uso:** `1 5 0 A 2 0 0 B` → **150 × 200 = 30.000** ✓

---

##  Inicio Rápido (5 minutos)

```bash
# 1. Cargar en placa
cd Segundo Proyecto/src/build
make load

# 2. Presionar: 1 5 0 A 1 5 0 B
#    ¡Resultado: [0, 1, 5, 0] = 150 × 150 = 22.500!

# 3. Presionar: C para reset
```

---

## Características Principales

| Característica | Especificación |
|---|---|
| **Plataforma** | Tang Nano 9K (FPGA GW1NR-LV9QN88PC6/I5) |
| **Frecuencia** | 27 MHz |
| **Entrada** | Teclado 4×4 matricial |
| **Rango** | 0–999 × 0–999 = hasta 998.001 |
| **Salida** | 4 displays 7-seg multiplexados |
| **Antirrebote** | Integrador por tecla, umbral 3 muestras (1.2 ms) |
| **Hold-off** | 150 ms (evita capturas duplicadas) |
| **Multiplicación** | 10 ciclos (370 ns) con Booth radix-2 |
| **Recursos** | ~8% LUTs, ~10% FF (muy eficiente) |
| **Lenguaje** | SystemVerilog |

---

##  Arquitectura (Vista Rápida)

```
┌─────────────────────────────────────────┐
│         FPGA TANG NANO 9K               │
│                                         │
│  ENTRADA          LÓGICA        SALIDA │
│  ┌──────────┐    ┌────────┐   ┌─────┐ │
│  │ KEYPAD   │ →  │ FSM +  │ → │ 4×  │ │
│  │ 4×4      │    │ BOOTH  │   │7seg │ │
│  │ (sync)   │    │(mult)  │   │ MUX │ │
│  └──────────┘    └────────┘   └─────┘ │
│                                        │
│  • Antirrebote      • Multiplicador    │
│  • CDC sync         • Conversión BCD   │
│  • Hold-off         • Blanking         │
└─────────────────────────────────────────┘
```

---

##  Documentación

| Documento | Para Quién | Tiempo |
|-----------|-----------|--------|
| **`QUICK_START.md`** | ¡COMIENZA AQUÍ! | 5 min |
| **`RESUMEN_EJECUTIVO.md`** | Evaluadores, managers | 10 min |
| **`INFORME_PROYECTO.md`** | Ingenieros, diseñadores | 45 min |
| **`DIAGRAMAS_ASM.md`** | Diseñadores de lógica | 30 min |
| **`GUIA_USUARIO_TROUBLESHOOTING.md`** | Usuarios, soporte técnico | 30 min |
| **`INDICE_MAESTRO.md`** | Navegación de docs | 5 min |


---

##  Estructura del Código

```
Segundo Proyecto/src/
├── design/
│   ├── top.sv                 (glue logic, CDC, hold-off)
│   ├── keypad_reader.sv       (escaneo 4×4 + antirrebote)
│   ├── fsm_capture.sv         (FSM captura + multiplicador)
│   ├── booth_multiplier.sv    (multiplicador radix-2, 10 ciclos)
│   ├── display_driver.sv      (multiplexador 4×7seg)
│   └── fifo.sv                (registro de últimas 4 teclas)
│
├── sim/
│   └── tb_top.sv              (testbench con ejemplos)
│
├── build/
│   └── Makefile               (síntesis, P&R, carga)
│
└── constr/
    └── tangnano9k.cst         (constraints físicos)
```

**Todas las líneas comentadas** explicando la intención de cada bloque. ✓

---

## Comandos Principales

```bash
cd Segundo Proyecto/src/build

# Cargar en placa (lo más importante)
make load

# Recompilar todo
make synth pnr bitstream load

# Simulación
make test
gtkwave tb_top.vcd

# Limpiar archivos
make clean
```

---

##  Validación

### Simulación
```bash
make test           # Ejecuta testbench
gtkwave tb_top.vcd  # Visualiza formas de onda
```

### Hardware
```bash
make load                    # Cargar bitstream
# Presionar: 9 9 9 A 9 9 9 B
# Resultado esperado: [8, 0, 0, 1] = 998.001 ✓
```

---

## Conceptos Clave Implementados

- ✓ **Escaneo matricial** con antirrebote por integrador
- ✓ **Clock Domain Crossing (CDC)** con sincronizador 2FF
- ✓ **Hold-off counter** para evitar capturas duplicadas
- ✓ **FSM de 3 estados** para captura de operandos
- ✓ **Multiplicador Booth radix-2** (10 ciclos, 20 bits resultado)
- ✓ **Conversión secuencial binario→BCD** (6 dígitos máximo)
- ✓ **Multiplexado dinámico** con blanking (5 kHz scan rate)
- ✓ **Parámetros calibrables** (remapeo, diagnóstico, test)


---

## Estadísticas

| Métrica | Valor |
|---------|-------|
| Líneas código (design) | ~2500 |
| Módulos principales | 6 |
| Máquinas de estado | 3 |
| Dominios de reloj | 2 |
| Utilizacion LUT | 8% (700/8640) |
| Utilizacion FF | 10% (200/2000) |
| Latencia multiplicación | 370 ns (10 ciclos @ 27MHz) |
| Tiempo síntesis completa | 50–85 s |

---

##  Casos de Uso

1. **Calculadora Embebida** – Operaciones aritméticas en tiempo real
2. **Panel de Control** – Entrada segura de parámetros numéricos
3. **Sistema Educativo** – Aprendizaje de FPGA y diseño digital
4. **Interfaz HMI** – Captura numérica con feedback visual
5. **Aplicaciones Industriales** – Mediciones con cálculos locales



---

## Referencias

- **Harris & Harris.** *Digital Design and Computer Architecture. RISC-V Edition.* 2022.
- **Booth, A.D.** *A Signed Binary Multiplication Technique.* 1951.
- **Yosys Documentation:** https://yosyshq.net/yosys/
- **nextpnr-gowin:** https://github.com/YosysHQ/nextpnr




---


**Algoritmo de Booth:**
Basado en *A Signed Binary Multiplication Technique* (Booth, 1951)
