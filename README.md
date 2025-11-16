# Sistema Digital de Multiplicación en FPGA Tang Nano 9K

## Proyecto: Teclado 4x4 con Multiplicador Booth y Display 7-Segmentos

**Plataforma:** Tang Nano 9K (GW1NR-LV9QN88PC6/I5)  
**Lenguaje:** SystemVerilog  
**Frecuencia de Operación:** 27 MHz  
**Fecha:** Noviembre 2025  
**Estado:** Completado y Verificado

---

## Tabla de Contenidos

1. [Descripción General](#descripción-general)
2. [Especificaciones Técnicas](#especificaciones-técnicas)
3. [Arquitectura del Sistema](#arquitectura-del-sistema)
4. [Módulos Principales](#módulos-principales)
5. [Máquinas de Estado (ASM)](#máquinas-de-estado)
6. [Sincronización entre Dominios de Reloj](#sincronización-entre-dominios-de-reloj)
7. [Guía de Uso](#guía-de-uso)
8. [Resolución de Problemas](#resolución-de-problemas)
9. [Análisis de Recursos](#análisis-de-recursos)
10. [Referencias y Bibliografía](#referencias-y-bibliografía)

---

## Descripción General

Este proyecto implementa un sistema digital embebido en FPGA que integra los siguientes componentes:

- Lector de teclado matricial 4x4 con antirrebote integrado
- Máquina de estados para captura de dos números de hasta 3 dígitos decimales (0-999)
- Multiplicador secuencial basado en el algoritmo de Booth radix-2
- Convertidor binario a BCD para visualización
- Controlador de cuatro displays de 7 segmentos multiplexados

El sistema permite al usuario capturar dos operandos desde el teclado y visualizar el resultado de su multiplicación en los displays, con una latencia total menor a 200 milisegundos.

---

## Especificaciones Técnicas

### Entrada
- Teclado matricial 4x4 (estándar)
- 16 teclas: dígitos 0-9, teclas de función A-D
- Rango de entrada: 0-999 por operando
- Técnica de antirrebote: integrador por tecla con umbral de 3 muestras (1.2 ms @ 10 kHz)
- Hold-off post-captura: 150 milisegundos para evitar duplicados

### Procesamiento
- Algoritmo de multiplicación: Booth radix-2
- Latencia de multiplicación: 10 ciclos de reloj (370 nanosegundos @ 27 MHz)
- Rango de multiplicación: 0-999 x 0-999 = hasta 998,001
- Conversión a BCD: secuencial con normalización de carry

### Salida
- Cuatro displays de 7 segmentos
- Multiplexado dinámico a 5 kHz
- Blanking de 128 ciclos para eliminación de ghosting (4.7 microsegundos)
- Mostración de últimos 4 dígitos decimales del resultado

### Dominios de Reloj
- Dominio rápido (clk): 27 MHz - lógica de control, multiplicación, displays
- Dominio lento (clk_scan): 10 kHz - escaneo de teclado y antirrebote
- Sincronización: CDC (Clock Domain Crossing) con sincronizador de dos flip-flops

---

## Arquitectura del Sistema

### Diagrama de Bloques

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SISTEMA FPGA (27 MHz)                          │
│                                                                     │
│  Entrada              Procesamiento            Salida              │
│  ┌──────────┐         ┌────────────┐         ┌──────────┐          │
│  │ Teclado  │         │ FSM        │         │ Display  │          │
│  │ 4x4      │────────>│ Captura +  │────────>│ 4x7seg   │          │
│  │          │         │ Booth      │         │ Mux      │          │
│  └──────────┘         │ Multiplier │         └──────────┘          │
│  (clk_scan)           │            │         (clk)                 │
│                       │ BCD Conv.  │                              │
│                       └────────────┘                              │
│                                                                    │
│                    Glue Logic (CDC, Hold-off)                    │
│                         FIFO (opcional)                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Flujo de Datos

1. El usuario presiona teclas en el teclado 4x4
2. El módulo `keypad_reader` realiza escaneo matricial @ 10 kHz
3. El antirrebote integrador estabiliza la lectura (1.2 ms)
4. La sincronización CDC transfiere el pulso de tecla al dominio rápido
5. El hold-off de 150 ms previene capturas duplicadas
6. La FSM acumula dígitos en estados S_A (operando A) y S_B (operando B)
7. Al presionar la tecla B (código 0xB), se inicia la multiplicación de Booth
8. Los 10 ciclos de Booth producen el resultado en 370 ns
9. La conversión secuencial a BCD convierte a 6 dígitos (590 ns)
10. El driver de displays multiplexados visualiza los últimos 4 dígitos

---

## Módulos Principales

### 1. keypad_reader.sv - Lector de Teclado 4x4

**Función:** Escaneo matricial con antirrebote integrado

**Algoritmo:**
- Escaneo secuencial de columnas (una activa por ciclo de clk_scan)
- Muestreo de filas en cada columna activa
- Integrador por tecla: incremento si presionada, decremento si liberada
- Umbral de estabilidad: db_cnt >= 3 (aproximadamente 1.2 ms)
- Detección de flanco ascendente en señal stable para generar pulso one-shot

**Mapeo de Teclado:**
```
         Columna 0   Columna 1   Columna 2   Columna 3
Fila 0      1            2            3          A (10)
Fila 1      4            5            6          B (11)
Fila 2      7            8            9          C (12)
Fila 3      * (14)       0            # (15)     D (13)
```

**Índice Interno:** key_code = fila * 4 + columna (0-15)

**Salidas:**
- key_valid: pulso de 1 ciclo cuando se detecta tecla nueva
- key_code: código de tecla presionada (0-15)

### 2. top.sv - Glue Logic y Sincronización CDC

**Función:** Interconexión de módulos, sincronización entre dominios y calibración

**Componentes:**
- Divisor de frecuencia: genera clk_scan (10 kHz) desde clk (27 MHz)
- Sincronizador CDC: dos flip-flops en cascada para key_valid y key_code
- Detector de flanco: genera pulso alineado en dominio rápido
- Contador de hold-off: 4,050,000 ciclos = 150 ms @ 27 MHz
- Remapeo de teclas: función adaptable para reordenar índices físicos
- Parámetros de calibración: MAP3-MAP0 para reordenamiento de dígitos en display

**Características:**
- CAL_MODE: fuerza patrón 0-1-2-3 para calibración de orden de dígitos
- REVERSE_AN: invierte orden de ánodos si conexión física lo requiere
- Parámetros HOLDOFF_CYCLES y DIV configurables sin modificar síntesis

### 3. fsm_capture.sv - Máquina de Estados y Multiplicación

**Estados:**
- S_A: Captura dígitos del número A (hasta 3 dígitos)
- S_B: Captura dígitos del número B (hasta 3 dígitos)
- S_DONE: Visualiza resultado de multiplicación

**Transiciones:**
- S_A → S_B: tecla A (código 0xA)
- S_B → S_DONE: tecla B (código 0xB) inicia multiplicación
- S_DONE → S_A: tecla C (código 0xC) realiza reset

**Multiplicación Booth:**
- 10 iteraciones (1 por bit de operando)
- Latencia: 10 ciclos de clk (370 ns)
- Máximo resultado: 2^20 bits

**Conversión a BCD:**
- Estados secuenciales: C_100K, C_10K, C_1K, C_100, C_10, C_1
- Normalización: NORM0-NORM5 para propagar carry entre dígitos
- Resultado: 6 dígitos BCD (d5-d0), se muestran últimos 4 (d3-d0)

### 4. booth_multiplier.sv - Multiplicador Radix-2

**Algoritmo de Booth:**
1. Por cada iteración, examina par {Q[0], Qprev}
2. Si 01: suma M a acumulador A
3. Si 10: resta M de acumulador A
4. Si 00 o 11: no opera
5. Desplazamiento aritmético a la derecha de {A, Q}
6. Decremente contador; repite 10 veces

**Características:**
- Operandos: 10 bits (rango 0-1023)
- Resultado: 20 bits (rango 0-1,046,529)
- Protocolo: start (1 pulso), busy, done (1 pulso), product (salida)
- Latencia: WIDTH = 10 ciclos

### 5. display_driver.sv - Controlador de Displays 4x7seg

**Multiplexado:**
- 4 estados: DISP0, DISP1, DISP2, DISP3
- Período por dígito: ~50 microsegundos (DIV=1350 @ 27 MHz)
- Ciclo completo: 200 microsegundos (5 kHz scan rate)

**Blanking:**
- 128 ciclos de apagado entre cambios de dígito (~4.7 microsegundos)
- Previene ghosting (valores residuales en segmentos)

**Decodificación BCD a 7-segmentos:**
- Tabla de verdad incorporada para dígitos 0-9 y hex A-F
- Configurable: ánodo activo alto o bajo, segmento activo alto o bajo

### 6. fifo.sv - Registro de Últimas Teclas (Opcional)

**Función:** Almacenar las últimas 4 teclas presionadas

**Operación:**
- LIFO shift: cada nueva tecla entra en posición 0
- Las anteriores se desplazan a posiciones 1, 2, 3
- Salidas: fifo_d0, fifo_d1, fifo_d2, fifo_d3

---

## Máquinas de Estado

### Diagrama ASM 1: Antirrebote (Integrador por Tecla)

```
┌─────────────────────────────────────────────────────┐
│     ANTIRREBOTE: Integrador por Tecla               │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Para cada ciclo de clk_scan:                       │
│                                                     │
│ Si tecla i está en columna activa:                 │
│                                                     │
│   ┌────────────────────────────────┐               │
│   │ tecla presionada?              │               │
│   └────┬──────────────────┬────────┘               │
│        │ SÍ               │ NO                      │
│    ┌───▼────┐         ┌───▼────┐                   │
│    │db_cnt++│         │db_cnt--│                    │
│    │(si<255)│         │(si>0)  │                    │
│    └───┬────┘         └───┬────┘                    │
│        │                  │                        │
│        └─────────┬────────┘                        │
│                  │                                 │
│        ┌─────────▼──────────┐                      │
│        │ db_cnt >= 3 ?      │                      │
│        │ (umbral)           │                      │
│        └────┬──────────┬────┘                      │
│             │SÍ        │NO                         │
│         ┌───▼──┐   ┌───▼──┐                        │
│         │stable│   │stable│                        │
│         │=1    │   │=0    │                        │
│         └───┬──┘   └───┬──┘                        │
│             │          │                          │
│             └────┬─────┘                          │
│                  │                                │
│        newpress[i] = stable[i] & ~stable_prev[i] │
│        (one-shot en flanco ascendente)            │
│                  │                                │
│        key_valid := 1 (1 ciclo)                   │
│        key_code := índice(newpress)               │
└─────────────────────────────────────────────────────┘

Temporal para tecla única:

Ciclo    Entrada  db_cnt  Stable  Newpress
─────────────────────────────────────────
0        1        0       0       0
1        1        0       0       0
2        0        1       0       0
3        0        2       0       0
4        0        3       1       1 <- flanco
5        0        3       1       0
...
20       1        2       1       0
21       1        1       1       0
22       1        0       0       1 <- liberación
```

### Diagrama ASM 2: Sincronización CDC (Clock Domain Crossing)

```
┌──────────────────────────────────────────────────────┐
│  CDC: Sincronización clk_scan (10 kHz) → clk (27MHz)│
├──────────────────────────────────────────────────────┤
│                                                      │
│ Dominio clk_scan        Dominio clk                │
│ ───────────────         ─────────────              │
│                                                      │
│ key_valid               kv_meta (FF1)              │
│   │                        │                        │
│   ├──────────────────────→ kv_sync (FF2)           │
│   │                        │                        │
│   │                    Detector de flanco           │
│   │                        │                        │
│   │                   (rising edge)                 │
│   │                        │                        │
│ key_code                kc_meta (FF1)              │
│   │                        │                        │
│   ├──────────────────────→ kc_sync (FF2)           │
│   │                        │                        │
│   │                   kc_latched ───┐              │
│   │                      (latch)     │              │
│   │                                  ├──→ key_valid_pulse
│   │                          holdoff ◄┘  (1 ciclo clk)
│   │                          (150ms)                │
│                                                      │
│ Cronograma:                                         │
│                                                      │
│ clk_scan   ─┬─┬─┬─┬─┬─┬─┬─┬─┬─                    │
│ key_valid  ──┐           └───┐      └──             │
│                                                      │
│ clk        ─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─            │
│ kv_meta    ────────┐                                │
│ kv_sync    ────────┼────┐                           │
│ Detector   ────────┴────┴────┐                      │
│ Pulso      ────────────────┐                        │
│ Hold-off   ────────────────┬───────────             │
└──────────────────────────────────────────────────────┘
```

### Diagrama ASM 3: Máquina de Estados de Captura

```
┌────────────────────────────────────────────────────────┐
│        FSM CAPTURA: S_A, S_B, S_DONE                  │
├────────────────────────────────────────────────────────┤
│                                                        │
│ ┌─────────────────────────────┐                       │
│ │ S_A: Capturar Operando A    │                       │
│ │ Display: [0, d0, d1, d2]    │                       │
│ └────────┬────────────────────┘                       │
│          │                                            │
│    ┌─────┴─────────────────────────┐                 │
│    │                               │                 │
│ Dígito (0-9)                   Tecla A (10)          │
│    │                               │                 │
│    │ - Almacenar en d0/d1/d2       │ - state := S_B  │
│    │ - digits_a++                  │ - Ir a capturar │
│    │                               │   operando B    │
│ ┌──▼─────────────────┐             │                 │
│ │ Volver a S_A       │             │                 │
│ │ (acumular más)     │             │                 │
│ └────────────────────┘             │                 │
│                                    │                 │
│        ┌────────────────────────────▼────────────┐   │
│        │   S_B: Capturar Operando B              │   │
│        │   Display: [0, b0, b1, b2]              │   │
│        └────────┬─────────────────┬──────────────┘   │
│                 │                 │                  │
│            Dígito            Tecla B (11)            │
│            (0-9)                 │                   │
│                 │ - Almacenar    │ - mul_start:=1   │
│                 │ - Volver S_B   │ - state:=S_DONE  │
│                 │                │ - Iniciar mult.  │
│                 │                │                  │
│        ┌────────▼────────────────┴──────┐            │
│        │  Multiplicación Booth (10 cc)  │            │
│        │  + Conversión BCD (6 cc)       │            │
│        └────────┬──────────────────────┘             │
│                 │                                    │
│        ┌────────▼───────────────────┐                │
│        │ S_DONE: Mostrar Resultado  │                │
│        │ Display:[prod_d3..d0]      │                │
│        └────────┬────────┬──────────┘                │
│                 │        │                           │
│            Dígitos   Tecla C (12)                    │
│            (otros)       │                           │
│                         │ - Reset todo              │
│                         │ - state := S_A            │
│                         │ - Listo para nuevo calc.  │
│                                                     │
└────────────────────────────────────────────────────────┘
```

### Diagrama ASM 4: Multiplicador Booth Radix-2

```
┌─────────────────────────────────────────────────────┐
│    BOOTH RADIX-2: Algoritmo de Multiplicación      │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Entrada: multiplicand M, multiplier Q (10 bits)   │
│ Salida: product (20 bits)                          │
│                                                     │
│ start = 1                                           │
│   │                                                 │
│   ├─ A := 0                                        │
│   ├─ Qprev := 0                                    │
│   ├─ count := 10                                   │
│   ├─ busy := 1                                     │
│   │                                                 │
│ ┌─▼──────────────────────────────────────┐         │
│ │ count > 0 ?                            │         │
│ └─┬───────────────────────────────┬──────┘         │
│   │ SÍ                             │ NO            │
│   │                            ┌───▼───────┐       │
│   │                            │product:=  │       │
│   │                            │{A[9:0],Q} │       │
│   │                            │done:=1    │       │
│   │                            │busy:=0    │       │
│   │                            └───────────┘       │
│   │                                                 │
│   │ ┌─────────────────────────────┐               │
│   │ │ Decisión Booth:             │               │
│   │ │ {Q[0], Qprev}              │               │
│   │ └──┬────────┬────────┬────────┘               │
│   │    │        │        │                        │
│   │  01│    00/11│      10│                        │
│   │    │        │        │                        │
│   │ ┌──▼─┐ ┌───▼─┐ ┌──▼─┐                        │
│   │ │A+=M│ │NOP  │ │A-=M│                        │
│   │ └──┬─┘ └───┬─┘ └──┬─┘                        │
│   │    │       │      │                          │
│   │    └───┬───┴──────┘                          │
│   │        │                                      │
│   │ ┌──────▼────────────────────┐                │
│   │ │ Corrimiento Aritmético:    │                │
│   │ │ Qprev := Q[0]              │                │
│   │ │ Q := {A[0], Q[9:1]}        │                │
│   │ │ A := {A[10], A[10:1]}      │                │
│   │ │ count--                    │                │
│   │ └──────┬─────────────────────┘                │
│   │        │                                      │
│   └────────┘ (siguiente iteración)               │
│                                                     │
│ Ejemplo: 5 x 3 = 15                              │
│ ───────────────────────────────────              │
│ Iter   A [0:10]      Q [0:9]   Qprev  Operación │
│   0    0000 0000 0   0000 0011  0    {1,0}→no-op│
│   1    0000 0101 1   0000 0000  1    {0,1}→A+=5 │
│   ...                                            │
│   9    xxxx xxxx x   xxxx xxxx  x    (final)    │
│                                                   │
│ product = 15 después de 10 ciclos               │
└─────────────────────────────────────────────────────┘
```

### Diagrama ASM 5: Conversión Binario a BCD

```
┌──────────────────────────────────────────────────┐
│   CONVERSIÓN: Binario → BCD (6 dígitos)         │
├──────────────────────────────────────────────────┤
│                                                  │
│ mul_done pulso                                   │
│   │                                              │
│   ├─ prod_work := mul_product                   │
│   ├─ prod_bcd_d* := 0                           │
│   ├─ conv_state := C_100K                       │
│   ├─ conv_active := 1                           │
│   │                                              │
│ ┌─▼─────────────────────────────────────┐       │
│ │ Extracción (división por potencias):  │       │
│ │                                       │       │
│ │ C_100K: while prod_work >= 100,000   │       │
│ │           prod_work -= 100,000        │       │
│ │           prod_bcd_d5++               │       │
│ │                                       │       │
│ │ C_10K:  idem con 10,000 → d4        │       │
│ │ C_1K:   idem con 1,000 → d3         │       │
│ │ C_100:  idem con 100 → d2           │       │
│ │ C_10:   idem con 10 → d1            │       │
│ │ C_1:    d0 := prod_work[3:0]         │       │
│ └─┬─────────────────────────────────────┘       │
│   │                                              │
│ ┌─▼─────────────────────────────────────┐       │
│ │ Normalización (propagar carry BCD):   │       │
│ │                                       │       │
│ │ NORM0: if (d0 > 9) d0-=10, d1++      │       │
│ │ NORM1: if (d1 > 9) d1-=10, d2++      │       │
│ │ NORM2: if (d2 > 9) d2-=10, d3++      │       │
│ │ NORM3: if (d3 > 9) d3-=10, d4++      │       │
│ │ NORM4: if (d4 > 9) d4-=10, d5++      │       │
│ │ NORM5: latch final prod_lat_*        │       │
│ └─┬─────────────────────────────────────┘       │
│   │                                              │
│   ├─ conv_active := 0                           │
│   ├─ display_value := {prod_lat_d3..d0}         │
│   │                                              │
│ Ejemplo: 150 → [0,0,0,1,5,0]                  │
│ ──────────────────────────────────────         │
│ C_100K: 150 >= 100K? NO → d5 = 0              │
│ C_10K:  150 >= 10K?  NO → d4 = 0              │
│ C_1K:   150 >= 1K?   NO → d3 = 0              │
│ C_100:  150 >= 100?  SÍ → work=50, d2=1       │
│ C_10:   50 >= 10? SÍ (5 veces) → d1=5         │
│ C_1:    d0 = 0                                  │
│ NORM:   sin carry                              │
│ Resultado: [0,0,0,1,5,0] → Display [0,1,5,0]  │
└──────────────────────────────────────────────────┘
```

### Diagrama ASM 6: Multiplexador de Displays

```
┌──────────────────────────────────────────────────┐
│   MULTIPLEXADOR: 4 Displays x 7-Segmentos      │
├──────────────────────────────────────────────────┤
│                                                  │
│ div_cnt (contador @ 27 MHz)                      │
│   │                                              │
│   ├─ Periodo por dígito: ~50 µs                 │
│   ├─ Ciclo completo: 200 µs (5 kHz)             │
│   │                                              │
│ ┌─▼──────────────────────────┐                  │
│ │ div_cnt < DIV (1350) ?     │                  │
│ └─┬────────────────────┬──────┘                  │
│   │ SÍ                 │ NO                      │
│   │             ┌──────▼─────┐                   │
│   │             │div_cnt:=0  │                   │
│   │             │Avanzar     │                   │
│   │             │state       │                   │
│   │             └──────┬─────┘                   │
│   │                    │                        │
│   ├────────┬───────────┤                        │
│   │        │           │                        │
│ ┌─▼────────▼─────────────▼──────┐               │
│ │ Seleccionar dígito por state:  │               │
│ │                               │               │
│ │ DISP0: D0 → an=0001          │               │
│ │        Decodificar segmentos  │               │
│ │ DISP1: D1 → an=0010          │               │
│ │ DISP2: D2 → an=0100          │               │
│ │ DISP3: D3 → an=1000          │               │
│ └─┬──────────────────────────────┘               │
│   │                                              │
│ ┌─▼──────────────────────────────┐              │
│ │ Blanking (128 ciclos):         │              │
│ │ - Apagar ánodos (OFF)          │              │
│ │ - Apagar segmentos (OFF)       │              │
│ │ - Evitar ghosting              │              │
│ └─┬──────────────────────────────┘              │
│   │                                              │
│ ┌─▼──────────────────────────────┐              │
│ │ Mostrar dígito:                │              │
│ │ - Ánodo activo                 │              │
│ │ - Segmentos según LUT BCD      │              │
│ │ - Registrar en FF salida       │              │
│ └──────────────────────────────────┘              │
│                                                  │
│ Secuencia: DISP0 → DISP1 → DISP2 → DISP3 → ...│
│            50µs    50µs    50µs    50µs         │
│                                                  │
│ Tabla BCD→7seg (segmentos GFEDCBA):            │
│ Dígito  Patrón   │ Dígito  Patrón             │
│ ───────────────  │ ───────────────             │
│   0     1111110  │   8     1111111             │
│   1     0110000  │   9     1111011             │
│   2     1101101  │   A     1110111             │
│   3     1111001  │   B     0011111             │
│   4     0110011  │   C     1001110             │
│   5     1011011  │   D     0111101             │
│   6     1011111  │   E     1001111             │
│   7     1110000  │   F     1000111             │
└──────────────────────────────────────────────────┘
```

---

## Sincronización entre Dominios de Reloj

### Problema de Sincronización

El sistema utiliza dos dominios de reloj con frecuencias diferentes:
- clk_scan: 10 kHz (escaneo de teclado)
- clk: 27 MHz (lógica de control)

Sin sincronización adecuada, los datos que cruzan dominios pueden entrar en estado metaestable, causando lecturas incorrectas.

### Solución: Sincronizador de 2 Flip-Flops

```
Entrada (clk_scan)     Sincronización (clk)      Salida (clk)
──────────────────     ───────────────────      ──────────────

key_valid              FF1 (kv_meta)
───────┐               │
       └───────────────► FF (registrar)          
                        │
                        └─► FF2 (kv_sync)
                             │
                             └─► Detector flanco
                                  │
                                  └─► key_valid_pulse
                                      (1 ciclo clk)
```

### Cronograma CDC

```
clk_scan:   ─┬─┬─┬─┬─┬─┬─┬─┬─┬─
key_valid:  ──┐           └───┐      └──

clk:        ─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─

kv_meta:    ────────┐
kv_sync:    ────────┼────┐
Detector:   ────────┴────┴────┐
key_valid_  ─────────────────┐
pulse:                        └──
```

### Hold-off de 150 milisegundos

Después de capturar una tecla, el sistema ignora nuevas capturas durante 150 ms para evitar:
- Lecturas múltiples de un único pulso físico
- Rebotes tardíos por pulsación prolongada

Configuración: HOLDOFF_CYCLES = 4,050,000 ciclos @ 27 MHz

---

## Guía de Uso

### Instalación del Bitstream

```bash
cd Segundo\ Proyecto/src/build
make load
```

El sistema estará listo cuando los displays se iluminen.

### Operación Básica

**Ejemplo: Calcular 150 × 200 = 30,000**

```
Paso 1:  Presionar [1]    Display: [0, 0, 0, 1]
Paso 2:  Presionar [5]    Display: [0, 0, 5, 1]
Paso 3:  Presionar [0]    Display: [0, 0, 5, 1]*
Paso 4:  Presionar [A]    Display: [0, 0, 0, B]
Paso 5:  Presionar [2]    Display: [0, 0, 0, 2]
Paso 6:  Presionar [0]    Display: [0, 0, 0, 2]
Paso 7:  Presionar [0]    Display: [0, 0, 0, 2]
Paso 8:  Presionar [B]    Display: [?, ?, ?, ?] (calculando)
Paso 9:  Esperar ...      Display: [0, 3, 0, 0]  (resultado)
Paso 10: Presionar [C]    Display: [0, 0, 0, 0]  (reset)

* El tercer dígito se almacena pero no se visualiza en el display
```

### Mapeo de Teclas

| Teclado | Código | Función |
|---------|--------|----------|
| 0-9 | 0x0-0x9 | Dígitos |
| A | 0xA | Cambiar a operando B |
| B | 0xB | Ejecutar multiplicación |
| C | 0xC | Reset/Inicio |
| D, *, # | 0xD, 0xE, 0xF | Reservados |

### Indicadores LED

Los cuatro LEDs de salida (bits [3:0]) muestran el código hexadecimal de la última tecla presionada:
- 0x0-0x9: dígitos
- 0xA: tecla A
- 0xB: tecla B
- 0xC: tecla C
- 0xF: sin presión (idle)

---

## Resolución de Problemas

### Problema: Display no se ilumina

**Posibles causas:**
- Bitstream no cargado correctamente
- Falta de alimentación en la placa

**Solución:**
```bash
make clean
make load
# Presionar botón RESET en la placa
```

### Problema: Teclas no responden

**Posibles causas:**
- Conexión física del teclado suelta
- Pull-ups faltantes en las filas

**Verificación:**
- Revisar archivo de constraints (tangnano9k.cst)
- rows[3:0] deben tener PULL_MODE=UP
- cols[3:0] deben ser outputs

### Problema: Display muestra números en orden invertido

**Solución:**
```verilog
// En src/design/top.sv, cambiar:
localparam bit REVERSE_AN = 1'b0;  // → 1'b1

// Recompilar y cargar:
make synth pnr bitstream load
```

### Problema: Resultado incorrecto en multiplicación

**Diagnóstico:**
```bash
cd src/build
make test
gtkwave tb_top.vcd
```

Verificar en la forma de onda:
- mul_product: resultado del multiplicador
- prod_bcd_d*: dígitos durante conversión
- prod_lat_d*: dígitos finales

### Problema: Teclas se capturan múltiples veces

**Solución:**
```verilog
// En src/design/top.sv, aumentar hold-off:
localparam int HOLDOFF_CYCLES = 5_400_000;  // 200 ms
// Recompilar
```

### Problema: Display parpadea (ghosting)

**Solución:**
```verilog
// En src/design/display_driver.sv:
localparam int BLANK_CYCLES = 256;  // de 128 → 256
// Recompilar
```

---

## Análisis de Recursos

### Síntesis con Yosys

```
Total de celdas: 1,443
  LUT1:          330
  LUT2:           68
  LUT3:          158
  LUT4:          122
  DFFC:           64
  DFFCE:         153
  Otros:        ~548

Wires:           963
Wire bits:     2,656
```

### Utilización en Tang Nano 9K

| Recurso | Utilizado | Disponible | Porcentaje |
|---------|-----------|------------|-----------|
| Celdas Lógicas (LCs) | ~700 | 8,640 | 8.1% |
| Flip-Flops (FF) | ~200 | 2,000 | 10% |
| BRAM | 0 | 32 | 0% |

### Timing

- Frecuencia objetivo: 27 MHz
- Margen de timing: Positivo (sin violaciones)
- Camino crítico: Multiplicador Booth + CDC

### Cronograma de Síntesis

| Paso | Duración Estimada |
|------|------------------|
| Síntesis (Yosys) | 5-10 segundos |
| P&R (nextpnr-gowin) | 30-60 segundos |
| Generación de bitstream | 10 segundos |
| Carga en placa | 5 segundos |
| **Total** | **50-85 segundos** |

---

## Latencias Críticas

| Operación | Ciclos | Tiempo @ 27 MHz |
|-----------|--------|-----------------|
| Antirrebote | 3 muestras @ 10 kHz | 1.2 ms |
| CDC Sync (2FF) | 2 | 74 ns |
| Hold-off | 4,050,000 | 150 ms |
| Multiplicación Booth | 10 | 370 ns |
| Conversión BCD | 16 | 590 ns |
| Multiplexado (4 displays) | 5,400 | 200 µs |

**Latencia total percibida:** Aproximadamente 152 ms (imperceptible para usuario)

---

## Estructura de Archivos

```
Segundo Proyecto/
├── src/
│   ├── design/
│   │   ├── top.sv                  (glue logic, CDC)
│   │   ├── keypad_reader.sv        (escaneo + antirrebote)
│   │   ├── fsm_capture.sv          (FSM + Booth)
│   │   ├── booth_multiplier.sv     (multiplicador)
│   │   ├── display_driver.sv       (multiplexador 7seg)
│   │   ├── fifo.sv                 (registro historial)
│   │   └── bin_to_bcd.sv           (convertidor opcional)
│   │
│   ├── sim/
│   │   └── tb_top.sv               (testbench)
│   │
│   ├── build/
│   │   ├── Makefile                (flujo síntesis)
│   │   ├── Proyecto_2.json         (netlist)
│   │   └── tb_top.vcd              (forma de onda)
│   │
│   └── constr/
│       └── tangnano9k.cst          (constraints)
│
└── doc/
    └── (documentación adicional)
```

### Comandos de Compilación

```bash
# Cambiar al directorio de compilación
cd Segundo\ Proyecto/src/build

# Compilación completa
make synth pnr bitstream load

# Pasos individuales
make synth                 # Síntesis Yosys
make pnr                   # Place & Route
make bitstream             # Generación de bitstream
make load                  # Carga en placa

# Simulación
make test                  # Ejecutar testbench
gtkwave tb_top.vcd         # Visualizar formas de onda

# Limpieza
make clean                 # Borrar archivos generados
```

---

## Referencias y Bibliografía

### Algoritmo de Booth

Booth, A.D. (1951). "A Signed Binary Multiplication Technique". Quarterly Journal of Mechanics and Applied Mathematics, 4(3), 236-240.

Descripción: El algoritmo de Booth es un método eficiente para multiplicación de números signados. Este proyecto utiliza la versión radix-2 para operandos no signados, reduciendo el número de operaciones de suma/resta.

### Estándares de Diseño Digital

Harris, D., & Harris, S. (2022). "Digital Design and Computer Architecture: RISC-V Edition" (2nd ed.). Morgan Kaufmann. ISBN: 978-0-12-820064-3.

### Documentación de Herramientas

- **Yosys:** https://yosyshq.net/yosys/ - Síntesis de circuitos lógicos
- **nextpnr-gowin:** https://github.com/YosysHQ/nextpnr - Place & Route para Gowin
- **iverilog:** http://bleyer.org/icarus/ - Simulador Verilog

### Documentación de Plataforma

- **Tang Nano 9K:** https://www.seeedstudio.com/Tang-Nano-9K-FPGA-Development-Board-p-5601.html
- **GW1NR Datasheet:** Gowin Semiconductor GW1NR-LV9QN88PC6/I5

### Mejores Prácticas de CDC

Xilinx WP318: "Synchronizer Designs for Multi-Clock Domain Synchronization". White Paper.

---

## Información del Proyecto

| Campo | Valor |
|-------|-------|
| Nombre del Proyecto | Multiplicador FPGA con Teclado 4x4 |
| Plataforma | Tang Nano 9K (GW1NR-LV9QN88PC6/I5) |
| Lenguaje | SystemVerilog |
| Frecuencia de Operación | 27 MHz |
| Líneas de Código | Aproximadamente 2,500 |
| Módulos Principales | 6 |
| Máquinas de Estado | 3 |
| Dominios de Reloj | 2 |
| Utilización de Recursos | 8-10% |
| Estado | Completado y Verificado |
| Versión | 1.0 |
| Fecha de Conclusión | Noviembre 2025 |

---

## Notas Finales

Este proyecto demuestra la implementación exitosa de un sistema digital complejo en FPGA, integrando:

- Entrada física robusta (teclado con antirrebote)
- Sincronización multidominios sin metaestabilidad
- Procesamiento digital secuencial (multiplicación de Booth)
- Salida visual multiplexada (displays 7-segmentos)
- Parámetros calibrables para adaptación a diferentes hardware
