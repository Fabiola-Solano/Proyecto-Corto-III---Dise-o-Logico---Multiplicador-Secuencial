# Proyecto 3: SISTEMA DE CAPTURA NUMÉRICA CON TECLADO 4×4 Y MULTIPLICADOR SECUENCIAL

**Teclado 4×4 + Displays 4×7seg con Multiplicación (Booth Radix-2) ** 
**Plataforma:** Tang Nano 9K (GW1NR-LV9QN88PC6/I5)  
**Frecuencia de Reloj:** 27 MHz  
**Lenguaje:** SystemVerilog  
**Fecha:** Noviembre 2025

---

## 1. RESUMEN EJECUTIVO

Este proyecto integra un sistema digital embebido en FPGA capaz de:

- **Entrada:** Captura de dígitos decimales desde un teclado matricial 4×4 con antirrebote
- **Procesamiento:** Multiplicación secuencial de dos números de hasta 3 dígitos cada uno (0–999) utilizando el algoritmo de Booth radix-2
- **Salida:** Visualización de operandos y resultado en cuatro displays de 7 segmentos multiplexados
- **Sincronización:** CDC (Clock Domain Crossing) entre dominios de reloj (clk_scan ≈ 10 kHz y clk = 27 MHz)

**Características principales:**
- Antirrebote integrado por tecla (umbral = 3 muestras, ≈ 1.2 ms)
- Hold-off de 150 ms para evitar capturas duplicadas por pulsación prolongada
- Conversión secuencial a BCD para resultados hasta 998.001
- Multiplexado con blanking dinámico para evitar ghosting
- Parámetros calibrables (remapeo de dígitos, test diagnóstico)

---

## 2. DESCRIPCIÓN GENERAL DEL SISTEMA

### 2.1 Diagrama de Bloques Funcional

```
┌─────────────────────────────────────────────────────────────────┐
│                      SISTEMA FPGA (27 MHz)                      │
│                                                                 │
│  ┌──────────────┐         ┌─────────────┐   ┌──────────────┐  │
│  │  KEYPAD_     │         │  FSM_CAPTURE│   │  DISPLAY_    │  │
│  │  READER      │────────▶│  & BOOTH    │──▶│  DRIVER      │  │
│  │              │         │  MULTIPLIER │   │              │  │
│  │(clk_scan)    │         │(clk)        │   │ (4×7seg)     │  │
│  └──────────────┘         │             │   └──────────────┘  │
│        △                  │  ┌────────┐ │                      │
│        │                  │  │ FIFO   │ │                      │
│   [TECLADO]              │  │ (hist) │ │                      │
│                          └──┴────────┴─┘                       │
│                                                                │
│    Top-Level Glue Logic (CDC, Remapeo, Hold-off)              │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Dominios de Reloj y Sincronización

| Dominio | Frecuencia | Propósito | CDC | 
|---------|-----------|----------|-----|
| clk_scan | ≈ 10 kHz | Escaneo y antirrebote del teclado | Sincronizador 2FF → clk |
| clk | 27 MHz | Lógica de control, multiplicación, displays | N/A |

**Sincronización CDC:** Desde `clk_scan` a `clk` se utiliza un sincronizador de dos flip-flops para `key_valid` y `key_code`, seguido de un detector de flanco para generar un pulso alineado en el dominio rápido.

---

## 3. MÓDULOS PRINCIPALES

### 3.1 `keypad_reader.sv` – Lector de Teclado 4×4

**Función:** Escaneo matricial + antirrebote integrado

#### Funcionamiento:
1. **Escaneo (col_idx):** Activa una columna a la vez (línea en bajo, las demás en alto)
2. **Muestreo:** Lee las filas cuando la columna está activa. Si fila=0, tecla presionada
3. **Antirrebote:** Integrador por tecla (`db_cnt[0:15]`, 8 bits)
   - Si muestreo positivo: `db_cnt++`
   - Si muestreo negativo: `db_cnt--`
   - Umbral de estabilidad: `db_cnt ≥ 3` (≈ 1.2 ms a 10 kHz de scan)
4. **Detección one-shot:** Compara `stable` con `stable_prev` para emitir `newpress`
5. **Prioridad:** Codifica el índice más bajo de `newpress` (evita ghosting)

#### Mapeo Físico (Estándar):
```
         Col0   Col1   Col2   Col3
  Row0    1      2      3      A(10)
  Row1    4      5      6      B(11)
  Row2    7      8      9      C(12)
  Row3   *(14)   0      #(15)  D(13)
```

**Índice interno:** `key_code = row*4 + col` (0..15)

#### Diagrama ASM del Antirrebote:

```
                    ┌─────────────────┐
                    │    IDLE         │
                    │  (col_mask=0)   │
                    └────────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                        │
         (sample_onehot)         (!sample_onehot)
                │                        │
         ┌──────▼──────┐          ┌──────▼──────┐
         │ INCREMENT    │          │ DECREMENT    │
         │ db_cnt[i]    │          │ db_cnt[i]    │
         │ (si <255)    │          │ (si >0)      │
         └──────┬──────┘          └──────┬──────┘
                │                        │
    ┌───────────┴─────────────┬─────────┴──────────────┐
    │                         │                        │
    │  db_cnt[i]>=3?         │   db_cnt[i]>=3?        │
    │  stable[i]:=1          │   stable[i]:=0         │
    │                         │                        │
    └────────────┬────────────┴────────────┬───────────┘
                 │                        │
          newpress[i]:= stable[i] & ~stable_prev[i]
                 │
             key_valid:=1
             key_code:= lowest_index(newpress)
```

---

### 3.2 `top.sv` – Lógica de Interconexión y Sincronización

**Función:** CDC, hold-off, remapeo de teclas, parámetros de calibración

#### Sincronización de `key_valid`:

```
clk_scan         ──┬──┬──┬──┬──┬──┬──┬──┬──
                    │  │  │  │  │  │  │  │
key_valid (raw)  ───┘  │  └──┐  └────────
                       │     │
kv_meta (2FF)    ───┐  └──┐  └────┐  └─
                    │     │      │
kv_sync (2FF)    ───┤─┐   └─┐    └─┐
                    │ │     │      │
kv_sync_prev     ───┘ │     └──┐   └──
                      │        │
flanco detect    ──────┘        └─────
                      
kv_edge_pending  ──────┐──────────┐──
                       │          │
key_valid_pulse  ──────┘          └──
(alineado clk)
                      
holdoff_active   ──────┬──────────────
(150ms)                │
                       └──────────────

────────────────────────────────────────
  clk  ─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─
       │ │ │ │ │ │ │ │ │ │ │ │ │ │
```

#### Diagrama ASM del Hold-off:

```
        ┌──────────────────┐
        │   kv_sync=0      │
        │   holdoff=0      │
        └────────┬─────────┘
                 │
     kv_sync & ~kv_sync_prev (flanco)
                 │
         ┌───────▼────────┐
         │  holdoff=1?    │
         │ kv_edge_pend=1?│
         └───┬────────┬───┘
             │ Sí     │ No
        ┌────▼──┐ ┌───▼────┐
        │ Arm   │ │ Ignore  │
        │ Pulse │ │ (busy)  │
        └────┬──┘ └────┬───┘
             │         │
             └────┬────┘
                  │
          ┌───────▼──────────┐
          │ kv_edge_pending  │
          │ := 1 (next cycle)│
          └────────┬─────────┘
                   │
        (next cycle)
                   │
          ┌────────▼──────────┐
          │ key_valid_pulse:=1│
          │ holdoff_cnt:=0    │
          │ holdoff_active:=1 │
          └────────┬──────────┘
                   │
        ┌──────────┴──────────┐
        │  holdoff_cnt<      │
        │  HOLDOFF_CYCLES?   │
        │  4_050_000         │
        └───┬──────────┬──────┘
            │ Sí       │ No
        ┌───▼──┐   ┌───▼──────┐
        │ ++   │   │ holdoff:=0│
        └───┬──┘   └───┬───────┘
            │          │
            └──────┬───┘
                   │
               Volver a
             kv_sync=0
```

#### Remapeo de Teclas:

La función `remap_key(4-bit index)` mapea el índice físico directo del escáner al código estándar:

```verilog
function automatic logic [3:0] remap_key(input logic [3:0] idx);
    case (idx)
        4'd0:  remap_key = 4'd1;   // → 1
        4'd1:  remap_key = 4'd2;   // → 2
        ...
        4'd15: remap_key = 4'hD;   // → D
    endcase
endfunction
```

Además, existe un ajuste: `key_code_std = remap_key(4'd15 - kc_latched)` para reflejar el índice físico si fuera necesario.

---

### 3.3 `fsm_capture.sv` – FSM de Captura y Multiplicación

**Función:** Máquina de estados para capturar operandos y ejecutar multiplicación Booth

#### Estados Principales:

```
                ┌──────────────────┐
                │      S_A         │
                │ (Capturar dígitos│
                │  número A)       │
                └────────┬─────────┘
                         │
              digit (0..9) │  tecla A
                         │  (10)
                    ┌────▼────┐
                    │ Acumular │ (10)
                    │ d0,d1,d2 │ ─────┐
                    └────┬────┘       │
                         │           │
         ┌────────────────┴───────────▼──┐
         │                               │
         │   digits_a >= 3 ?             │
         │                               │
         └───┬──────────────────┬────────┘
             │ Sí (o esperar)   │ No
             │                  │
        ┌────▼────────────┐ ┌───▼────┐
        │ Pasar a S_B     │ │ Volver  │
        │ (Tecla A=10)    │ │ a S_A   │
        └────┬────────────┘ └────────┘
             │
        ┌────▼──────────────────┐
        │      S_B              │
        │ (Capturar número B)   │
        └────┬──────────────────┘
             │
         digit (0..9) │  tecla B (11)
             │        │
        ┌────▼──┐ ┌───▼─────────────┐
        │Acumul │ │ digits_b >= 3 ? │
        │d0..d2 │ └───┬─────────┬───┘
        └────┬──┘     │ Sí      │ No
             │        │         │
             │        │    ┌────▼────┐
             │        │    │ Volver  │
             │        │    │ a S_B   │
             │        │    └────────┘
             │   ┌────▼────────────────────┐
             │   │ Iniciar Multiplicación  │
             │   │ mul_start := 1          │
             │   │ Pasar a S_DONE          │
             │   └────┬───────────────────┘
             │        │
             └────┬───┘
                  │
        ┌─────────▼──────────┐
        │     S_DONE         │
        │  (Resultado)       │
        │  Mostrar prod_lat  │
        └─────────┬──────────┘
                  │
          tecla C (12) = Reset
                  │
          ┌───────▼────────┐
          │ Limpiar todo   │
          │ digits_a:=0    │
          │ digits_b:=0    │
          │ prod_*:=0      │
          │ Ir a S_A       │
          └────────────────┘
```

#### Diagrama ASM de Conversión a BCD (Secuencial):

```
        ┌──────────────────┐
        │    mul_done=1    │
        │  prod_work:=prod │
        └────────┬─────────┘
                 │
        ┌────────▼────────────────┐
        │  prod_bcd_d*:=0         │
        │  conv_state := C_100K   │
        │  conv_active := 1       │
        └────────┬────────────────┘
                 │
   ┌─────────────┴─────────────┐
   │                           │
   ▼   prod_work >= 100K?      ▼
 [C_100K]                   [C_100K]
   │ Sí                       │ No
   ├─ prod_work -= 100K      │
   ├─ prod_bcd_d5++          │
   └──┐                       │
      │                       │
   (loop)                     │
      │                       │
   prod_work < 100K?    ──────┘
      │
      ├─ conv_state := C_10K
      │
      ▼
   [C_10K]  (similar: 10K)
      │
      ├─ [C_1K]   (similar: 1K)
      │
      ├─ [C_100]  (similar: 100)
      │
      ├─ [C_10]   (similar: 10)
      │
      ├─ [C_1]    prod_bcd_d0 := prod_work[3:0]
      │           conv_state := NORM0
      │
      ▼
   [NORM0..NORM5]  (propagar carry BCD)
      │
      ├─ NORM0: if (d0>9) d0-=10; d1++
      ├─ NORM1: if (d1>9) d1-=10; d2++
      ├─ NORM2: if (d2>9) d2-=10; d3++
      ├─ NORM3: if (d3>9) d3-=10; d4++
      ├─ NORM4: if (d4>9) d4-=10; d5++
      ├─ NORM5: latch prod_lat_*
      │          conv_state := C_IDLE
      │          conv_active := 0
      │
      ▼
   display_value = {prod_lat_d3, ..., d0}
```

---

### 3.4 `booth_multiplier.sv` – Multiplicador Booth Radix-2

**Función:** Multiplicación secuencial de dos números no signados (WIDTH bits cada uno)

#### Algoritmo de Booth Radix-2:

```
┌──────────────────────────────────────────┐
│ INPUT: M (multiplicando), Q (multiplier) │
│ WIDTH = 10 bits → 10 iteraciones         │
└──────────────────────────────────────────┘
         │
         │ Inicializar:
         │   A := 0
         │   Qprev := 0
         │   count := WIDTH
         │
         ▼
    ┌─────────────────┐
    │  count > 0 ?    │
    └──┬──────────┬───┘
       │ Sí       │ No
       │          └─ DONE: product = {A[WIDTH-1:0], Q}
       │
       ▼
    [Iter. n]
    ┌─────────────────────────────────┐
    │ Decidir operación:              │
    │ {Q[0], Qprev} = ?               │
    └──┬──┬──┬──┬───────────────────┘
       │  │  │  │
    01 │  10│  │ 00, 11
       │  │  │  │
    ┌──▼┐┌─▼┐│ ┌──▼──────────┐
    │A+=M││A-=M│ │ A sin cambio│
    │    ││    │ │             │
    └──┬┘└─┬┘│ └──┬──────────┘
       │   │ │    │
       └───┼─┼────┘
           │ │
           ▼ ▼
    ┌──────────────────────────┐
    │ Corrimiento Aritmético:  │
    │                          │
    │ Qprev := Q[0]            │
    │ Q := {A[0], Q[WIDTH:1]}  │
    │ A := {A[WIDTH], A[WIDTH:1]}
    └──────┬───────────────────┘
           │
           │ count--
           │
           ▼
    ┌──────────────────┐
    │ count = 0 ?      │
    └───┬──────────┬───┘
        │ No       │ Sí
        │          └─ DONE
        │
        └────────────────┐
                         ▼ (siguiente iteración)
```

#### Características:
- **Latencia:** 10 ciclos (WIDTH) para operandos de 10 bits
- **Resultado:** 20 bits (2×WIDTH)
- **Control:** Pulso `start` inicia; `busy` indica ocupado; `done` pulsa al terminar
- **Máximo operando:** 1023 × 1023 = 1.046.529 (cabe en 20 bits)

---

### 3.5 `display_driver.sv` – Controlador de Displays 4×7seg Multiplexados

**Función:** Multiplexado dinámico + decodificación BCD→7seg + blanking

#### Diagrama ASM del Multiplexador:

```
        ┌──────────────────────┐
        │   div_cnt := 0       │
        │   state := DISP0     │
        │   blank_cnt := 0     │
        └────────┬─────────────┘
                 │
        ┌────────▼─────────────────┐
        │  div_cnt < DIV (1350) ?   │
        └──┬──────────────────┬─────┘
           │ Sí               │ No
           │                  │
           │          ┌───────▼──────────┐
           │          │  div_cnt := 0    │
           │          │  Avanzar state:  │
           │          │  DISP0→1→2→3→0   │
           │          │  blank_cnt := 0  │
           │          └────────┬─────────┘
           │                   │
        blank_cnt < 128?    (transición a siguiente)
           │                   │
        ┌──▼──┐  ┌────┐  ┌────▼────┐
        │ ++  │  │Apag│  │ blank   │
        └──┬──┘  └─┬──┘  │ cnt++   │
           │      │      └──┬─────┘
           │  (stay)        │
           │      │         │
           └──┬───┴────┬────┘
              │        │
         ┌────▼────┐┌──▼────────┐
         │ Blanking││ Mostrar   │
         │ (OFF)   ││ cur_bcd   │
         └─────────┘└───┬──────┘
                        │
                 Decodificar BCD
                        │
                 ┌───────▼────────┐
                 │ Mostrar digito  │
                 │ en posición     │
                 │ state (D0..D3)  │
                 └────────────────┘
```

#### Tabla de Decodificación BCD→7seg:

| Dígito | Segmentos (GFEDCBA) | Valor |
|--------|----------------------|-------|
| 0 | 1111110 | 0x3E |
| 1 | 0110000 | 0x30 |
| 2 | 1101101 | 0x6D |
| 3 | 1111001 | 0x79 |
| 4 | 0110011 | 0x33 |
| 5 | 1011011 | 0x5B |
| 6 | 1011111 | 0x5F |
| 7 | 1110000 | 0x70 |
| 8 | 1111111 | 0x7F |
| 9 | 1111011 | 0x7B |

#### Tiempo de Multiplexado:
- **DIV = 1350 ciclos @ 27 MHz** → 50 µs por estado
- **4 displays** → 200 µs por ciclo completo (5 kHz)
- **Blanking:** 128 ciclos ≈ 4.7 µs (tiempo muerto)

---

### 3.6 `fifo.sv` – Registro de Últimas Teclas

**Función:** Almacenar las últimas 4 teclas presionadas (desplazamiento LIFO)

```
En cada key_valid_pulse:

Entrada (wr_data) → mem[0] (D0 - más reciente)
mem[0] (D0)      → mem[1] (D1)
mem[1] (D1)      → mem[2] (D2)
mem[2] (D2)      → mem[3] (D3 - más antigua)

Salidas: fifo_d0, fifo_d1, fifo_d2, fifo_d3
```

---

## 4. INTERFAZ FÍSICA Y PARÁMETROS DE SÍNTESIS

### 4.1 Puertos del Módulo Top

```verilog
module top (
    input  logic clk,              // 27 MHz
    input  logic rst_n,            // Reset activo bajo
    input  logic [3:0] rows,       // Filas del teclado (pull-up)
    output logic [3:0] cols,       // Columnas del teclado (activas en bajo)
    output logic seg_a, seg_b, ... // Segmentos (cátodo común)
    output logic [3:0] an,         // Ánodos (display)
    output logic [3:0] leds        // Debug (última tecla)
);
```

### 4.2 Restricciones (Constraints)

```
rows[3:0]: LVCMOS33, PULL_MODE=UP (entrada con pull-up interno)
cols[3:0]: LVCMOS33, PULL_MODE=DOWN (salida con pull-down leve)
seg_*:     LVCMOS33 (salida)
an[3:0]:   LVCMOS33 (salida)
```

### 4.3 Parámetros Calibrables (en `top.sv`)

| Parámetro | Valor | Propósito |
|-----------|-------|----------|
| `HOLDOFF_CYCLES` | 4.050.000 | Hold-off de 150 ms |
| `DIV_CNT` | 1349 | Reloj scan ≈ 10 kHz |
| `CAL_MODE` | 0 | 0: modo normal, 1: mostrar 0-1-2-3 |
| `MAP3..MAP0` | 3-0 | Remapeo de nibbles en displays |
| `REVERSE_AN` | 1 | Invertir orden de ánodos (reversión física) |

---

## 5. FLUJO DE DATOS Y SINCRONIZACIÓN

### 5.1 Captura de Dígito (Ejemplo: `1 5 0 A`)

```
Tiempo (ms)    Evento                    Estado         Display
────────────────────────────────────────────────────────────────
0              Pulsar '1'                S_A
               keypad detects            db_cnt[0]++
~1.2           stable[0]=1, newpress    key_valid_pulse
               FSM recibe key_code=0x1   num_a_d0=1     [0 0 0 1]
               
~10            Pulsar '5'
~11.2          stable[5]=1               key_valid_pulse
               FSM recibe key_code=0x5   num_a_d1=5     [0 0 5 1]
               
~20            Pulsar '0'
~21.2          stable[13]=1              key_valid_pulse
               FSM recibe key_code=0x0   num_a_d2=0     [0 0 5 1]*
               
~30            Pulsar 'A' (0xA)
~31.2          stable[3]=1               key_valid_pulse
               FSM recibe key_code=0xA   state -> S_B    [0 0 0 B]
               (B = espera siguiente)
               
~40            Pulsar '1', '5', '0'     (similar a A)
~45            Pulsar 'B' (0xB)          key_code=0xB
               mul_start=1               
               state -> S_DONE
~55            mul_done=1                prod_ab=150
               conv_state=C_100K
~65-~120       Conversión secuencial      prod_lat_d*
               NORM0..NORM5              display_value
                                         [0 1 5 0]

* En S_A, el display muestra siempre el dígito acumulado
```

### 5.2 Temporización Crítica

```
Signal                  Domain      Path crítico
──────────────────────────────────────────────
clk_scan               clk_scan    10 kHz (≈100 µs por período)
key_valid (raw)        clk_scan    Salida de antirrebote (1 período)
key_valid (sync)       clk          Sincronizador 2FF + flanco (~74 ns @ 27 MHz)
key_valid_pulse        clk          1 período de reloj (≈37 ns)
holdoff_active         clk          Comparador 22 bits (máximo ≈8 ns)
display_cycle          clk          DIV=1350 → ~50 µs (20 kHz)
```

---

## 6. ANÁLISIS DE RECURSOS SINTETIZADOS

### 6.1 Uso de Celdas (Síntesis Yosys)

```
Total de celdas:      1443
├─ LUT1:              330
├─ LUT2:              68
├─ LUT3:              158
├─ LUT4:              122
├─ DFFC (D flip-flop): 64
├─ DFFCE (D con enable): 153
└─ Otros:             ~548

Estimado de LCs (Logic Cells):
  Capacidad Tang Nano 9K: ~8640 LCs
  Utilización:           ~16-18%
```

### 6.2 Distribución de Recursos por Módulo (Estimado)

| Módulo | Función | LUTs | FFs | Notas |
|--------|---------|------|-----|-------|
| keypad_reader | Escaneo + antirrebote | ~150 | ~60 | 16 contadores de 8 bits |
| fsm_capture | FSM + Booth + BCD | ~400 | ~300 | Multiplicador secuencial, conversión |
| display_driver | Multiplexador | ~250 | ~80 | DIV counter, blanking |
| top (CDC + FIFO) | Sincronización | ~200 | ~120 | Hold-off counter, 2FF sync, FIFO |
| Interconexión | Cableado | ~50 | ~40 | Muxes, decodificadores |

---

## 7. SIMULACIÓN Y VERIFICACIÓN

### 7.1 Testbench (`tb_top.sv`)

El testbench inyecta estímulos en `rows[3:0]` simulando pulsaciones de tecla:

```verilog
// Ejemplo: simular presión de tecla en (row=0, col=0) = índice 0 (tecla '1')
@(posedge clk_scan)
    rows <= 4'b1110; // Simula fila 0 presionada (activo bajo)

// Esperar antirrebote
repeat(50) @(posedge clk_scan);

rows <= 4'b1111; // Soltar

// Verificar señales
@(posedge clk)
    if (key_valid_pulse && key_code_std == 4'd1) 
        $display("✓ Tecla '1' capturada correctamente");
```

### 7.2 Formas de Onda Clave a Verificar

```
clk_scan:            Reloj de scan (10 kHz, periodo ~100 µs)
col_o[3:0]:          Scan de columnas (rotación 1110→1101→1011→0111)
row_i[3:0]:          Simulación de pulsación (activo bajo)
db_cnt[i]:           Integrador (debe alcanzar 3 para estabilizar)
stable[i]:           Debounce estable (flanco subida con retraso)
newpress:            Pulso one-shot (1 ciclo clk_scan)
key_valid_pulse:     Pulso sincronizado a clk (1 ciclo clk)
key_code_std:        Índice remapeado (0..15)
num_a_d0/d1/d2:      Dígitos acumulados
mul_busy:            Indicador de ocupado (10 ciclos)
mul_done:            Pulso de finalización
prod_bcd_d*:         Dígitos BCD siendo convertidos
display_value:       Valor mostrado en displays (16 bits)
```

---

## 8. PROBLEMAS RESUELTOS Y SOLUCIONES

### 8.1 Rebotes y Capturas Duplicadas

**Problema:** Teclas que "rebotan" causaban múltiples capturas de un solo pulso físico.

**Solución:**
1. Integrador por tecla (`db_cnt[0:15]`) con umbral (3 muestras)
2. Hold-off de 150 ms a nivel de `top.sv` para ignorar repeticiones por pulsación larga
3. One-shot (`newpress`) que solo emite en flanco de estabilización

**Resultado:** Captura determinista, una tecla por pulsación física.

---

### 8.2 Ghosting y Valores Antiguos en Displays

**Problema:** Transiciones abruptas entre dígitos causaban parpadeos o valores intermedios visibles.

**Solución:**
1. Blanking dinámico entre cambios de dígito (128 ciclos ≈ 4.7 µs)
2. Salidas multiplexadas registradas para evitar glitches
3. Reloj de scan y multiplicador sincronizados con clk principal

**Resultado:** Transiciones suaves sin parpadeo visible.

---

### 8.3 Resultado Parcial Visible Durante Conversión a BCD

**Problema:** Mientras se realizaba la conversión secuencial a BCD, los dígitos intermedios se mostraban en pantalla.

**Solución:**
1. Conversión completamente secuencial (estados C_100K → NORM5)
2. Latch final (`prod_lat_d*`) que captura el resultado solo cuando `conv_active=0`
3. `display_value` siempre muestra `prod_lat_*` en estado S_DONE

**Resultado:** Resultado final atomicio, sin transiciones intermedias visibles.

---

### 8.4 Orden Físico de Dígitos y Mapeo de Teclas

**Problema:** Dígitos mostrados en orden inverso; teclas no correspondían con posiciones esperadas.

**Solución:**
1. Parámetros `MAP3..MAP0` para reordeno de nibbles
2. Función `remap_key()` adaptable a física del hardware
3. Modo `CAL_MODE=1` que fuerza patrón 0-1-2-3 para calibración
4. Inversor físico `REVERSE_AN` para ánodos si fuera necesario

**Resultado:** Mapeo flexible, calibrable sin tocar código.

---

### 8.5 Sincronización Entre Dominios de Reloj

**Problema:** `key_valid` (10 kHz) y `key_code` llegaban de forma metaestable a dominio de 27 MHz.

**Solución:**
1. Sincronizador de dos flip-flops (2FF CDC)
2. Detector de flanco posterior para generar pulso alineado
3. Hold-off contador activado inmediatamente después del pulso

**Resultado:** Sin metaestabilidad, sincronización robusta.

---

## 9. INSTRUCCIONES DE SÍNTESIS Y CARGA

### 9.1 Síntesis con Yosys

```bash
cd src/build
make synth
```

Genera:
- `Proyecto_2.json` (netlist Yosys)
- `synthesis_tangnano9k.log` (reporte de síntesis)

### 9.2 Place & Route (nextpnr-gowin)

```bash
make pnr
```

Genera:
- `Proyecto_2_pnr.json` (P&R final)
- `pnr_tangnano9k.log` (timing, utilización)

### 9.3 Generación de Bitstream

```bash
make bitstream
```

Genera:
- `Proyecto_2_tangnano9k.fs` (bitstream)

### 9.4 Carga en la Placa

```bash
make load
```

Requiere `openFPGALoader` instalado.

### 9.5 Simulación (iverilog + GTKWave)

```bash
make test
```

Genera:
- `tb_top.vcd` (forma de onda para GTKWave)

---

## 10. MEJORAS FUTURAS

1. **Operación Configurable:** Permitir selección entre suma, resta, multiplicación, división
2. **Teclado Virtual en Display:** Mostrar matriz 4×4 en uno de los displays para retroalimentación
3. **Memoria de Operaciones:** Almacenar últimos 10 cálculos
4. **Comunicación Serial:** Interfaz UART/SPI para telemetría en tiempo real
5. **Optimización de Área:** Usar multiplicador de hardware (DSP) si disponibl la placa
6. **Detección de Errores:** Checksum o validación de entrada

---

## 11. REFERENCIAS Y ESPECIFICACIONES

- **Harris & Harris.** *Digital Design and Computer Architecture. RISC-V Edition.* Morgan Kaufmann, 2022.
- **Gowin Semiconductor.** GW1NR-9K FPGA Datasheet
- **Yosys Documentation:** https://yosyshq.net/yosys/
- **nextpnr-gowin:** https://github.com/YosysHQ/nextpnr
- **Algoritmo de Booth:** Booth, A. D. (1951). *A Signed Binary Multiplication Technique*

---

## 12. APÉNDICES

### A. Listado de Archivos

```
src/design/
├── top.sv                    (Glue logic, CDC, hold-off)
├── keypad_reader.sv          (Escaneo + antirrebote)
├── fsm_capture.sv            (FSM + multiplicador Booth + conversión BCD)
├── booth_multiplier.sv       (Multiplicador radix-2 secuencial)
├── display_driver.sv         (Multiplexador 4×7seg)
├── fifo.sv                   (Registro de últimas 4 teclas)
├── bin_to_bcd.sv             (Convertidor combinacional, opcional)
└── contador_5642as.sv.bak    (Obsoleto)

src/sim/
└── tb_top.sv                 (Testbench)

src/build/
├── Makefile                  (Flujo de síntesis)
├── Proyecto_2.json           (Netlist)
└── Proyecto_2_pnr.json       (P&R)

constr/
└── tangnano9k.cst            (Restricciones físicas)
```

### B. Tiempos Típicos de Ejecución

| Tarea | Tiempo | Notas |
|-------|--------|-------|
| Síntesis (Yosys) | ~5-10 s | Depende del PC |
| P&R (nextpnr) | ~30-60 s | Depende de la síntesis |
| Bitstream | ~10 s | Gowin pack |
| Carga (openFPGALoader) | ~5 s | Placa USB |
| Simulación (10 µs) | ~1-2 s | iverilog + VCD |

### C. Preguntas Frecuentes (FAQ)

**P: ¿Puedo cambiar la frecuencia del teclado?**  
R: Sí, modifica `div_cnt <= 12'd1349` en `top.sv` para `clk_scan`.

**P: ¿Cómo recalibro el orden de dígitos?**  
R: Establece `CAL_MODE = 1'b1` en `top.sv`, ejecuta, observa el patrón 0-1-2-3, luego ajusta `MAP3..MAP0`.

**P: ¿Es posible anidar más dígitos?**  
R: Sí, expande los contadores `digits_a` y `digits_b`, aumenta el convertidor BCD, y añade displays al multiplexador.

**P: ¿Qué pasa si presiono varias teclas a la vez?**  
R: El sistema prioriza la de menor índice físico (encode_1hot). Ghosting se previene con los diodos del teclado.

---

## 13. CONCLUSIÓN

Este proyecto demuestra la integración exitosa de múltiples subsistemas digitales en una FPGA pequeña:

- **Entrada robusta:** Antirrebote integrado + sincronización CDC
- **Procesamiento:** Multiplicador secuencial Booth con conversión a BCD
- **Salida:** Multiplexado eficiente con eliminación de ghosting
- **Confiabilidad:** Parámetros calibrables, diagnóstico, sin metaestabilidad

El diseño es escalable, modular y puede servir como base para calculadoras embebidas, teclados especializados o interfaces numéricas en sistemas embebidos.
