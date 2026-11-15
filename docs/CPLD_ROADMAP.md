# Концепция развития: ALF-NEMOBUS v2.0 (CPLD Edition)

Данный документ описывает перспективную модернизацию платы **ALF-NEMOBUS** с целью минимизации количества корпусов за счет применения ПЛИС (CPLD) семейства **Altera MAX3000A (`EPM3032A` / `EPM3064A` в корпусе PLCC-44 или TQFP-44)**, доступных с донорских плат (например, Igrosoft).

---

## 1. Архитектурная концепция

В версии 2.0 вся дискретная логика дешифрации, регистров банков и арбитража шины переносится внутрь одной микросхемы CPLD:
* **Что остается на плате:**
  1. **CPLD `EPM3064A-TC44` / `PC44`** (34 доступных вывода I/O, входы толерантны к 5V).
  2. **Стабилизатор питания `AMS1117-3.3`** (питание ядра `VCCINT` и `VCCIO` ПЛИС 3.3 В).
  3. **PNP-транзистор `BC856` (`Q1`)** — силовой ключ для гарантированной подачи полных `+5V` в линию `IORQGE` (`/IOGE/`) по принципу доработки **Prusak (Мод Прусака)** и схемотехнического решения **@Alex_Eburg (Алекс Екб)** *«Аппаратная блокировка портов ZX-Spectrum. Формирование сигнала IORQGE»*.
  4. **Пятивольтовые параллельные Flash-ПЗУ (`SST39SF010` / `SST39SF040` / `27C080`)** — работают напрямую с 5-вольтовой шиной Z80 без преобразователей уровней.
  5. **Два буфера `74HCT245` (`U5`, `U6`)** — позволяют сэкономить выводы CPLD, оставляя за ПЛИС только выдачу сигналов `/{slash}1FRD` и `/{slash}FERD`.

* **Почему выбраны 5-вольтовые параллельные ПЗУ (а не 3.3V Flash или SPI Flash):**
  * **Отказ от 3.3V параллельных ПЗУ (LPC/FWH и 3.3V NOR):** большинство чипов от материнских плат Pentium 4 используют протокол LPC/FWH (несовместимый с Z80), а параллельные 3.3V чипы не имеют 5V-толерантных входов и требуют установки трех дополнительных корпусов шинных конвертеров `74LVC245` на шину адреса и управляющих сигналов.
  * **Отказ от SPI Flash (XIP over `/WAIT`):** чтение одного байта по SPI (40 тактов) требует удержания процессора сигналом `/WAIT` в течение ~2.8 мкс (падение скорости выполнения кода в 3–4 раза) и занимает 14 дополнительных выводов младших адресов `A0–A13` на ПЛИС, что превышает лимит 44-выводного корпуса.

---

## 2. Базовый шаблон логики на языке Verilog (для Quartus II)

```verilog
module alf_nemobus_cpld (
    input  wire        rst_n,      // /*RST/ (J1-20)
    input  wire [7:0]  a_low,      // A0..A7 (или используемые биты A0, A1, A7)
    input  wire        a14,        // /*A14  (J1-1)
    input  wire        a15,        // /*A15  (J2-1)
    input  wire [7:0]  d,          // /*D0../*D7
    input  wire        iorq_n,     // /*IORQ/ (J2-17)
    input  wire        mreq_n,     // /*MREQ/ (J2-16)
    input  wire        rd_n,       // /*RD/   (J2-18)
    input  wire        wr_n,       // /*WR/   (J2-19)
    input  wire        m1_n,       // /*M1/   (J2-24)

    output wire        rd_1f_n,    // Управление буфером U5 (CE, pin 19)
    output wire        rd_fe_n,    // Управление буфером U6 (CE, pin 19)
    output wire        ioge_drv_n, // На базу PNP-транзистора Q1 через R14 (220 Ом)
    output wire        rom_sys_ce_n,  // ~CE системного ПЗУ U8
    output wire        rom_cart_ce_n, // ~CE игрового ПЗУ U15
    output reg  [2:0]  sys_bank,   // Банки A14..A16 системного ПЗУ U8
    output reg  [5:0]  cart_bank,  // Банки A14..A19 игрового ПЗУ U15
    output reg         cart_mode   // 0 = System ROM (U8), 1 = Cartridge ROM (U15)
);

    // 1. Дешифрация чтения портов #1F и #FE (строго в цикле чтения I/O, не M1)
    wire io_read  = (~iorq_n) & (~rd_n) & m1_n;
    wire io_write = (~iorq_n) & (~wr_n) & m1_n;

    // Условие U2: A7=0, A1=1, A0=1 (#1F) и A7=1, A1=1, A0=0 (#FE)
    wire sel_1f = (~a_low[7]) & a_low[1] & a_low[0];
    wire sel_fe = ( a_low[7]) & a_low[1] & (~a_low[0]);

    assign rd_1f_n = ~(io_read & sel_1f);
    assign rd_fe_n = ~(io_read & sel_fe);

    // 2. Формирование сигнала IORQGE (Мод Прусака / решение @Alex_Eburg: активный '0' открывает PNP-ключ Q1)
    assign ioge_drv_n = rd_1f_n & rd_fe_n;

    // 3. Запись в регистр банков ПЗУ (/PBANK: запись при A7=0, A1=1, A0=1)
    wire pbank_wr = io_write & sel_1f;

    always @(posedge pbank_wr or negedge rst_n) begin
        if (!rst_n) begin
            sys_bank  <= 3'b000;
            cart_bank <= 6'b000000;
            cart_mode <= 1'b0;
        end else begin
            sys_bank  <= d[2:0];
            cart_bank <= d[5:0];
            cart_mode <= d[7];
        end
    end

    // 4. Внутреннее формирование сигнала выборки области ПЗУ #0000-#3FFF (/*CSR/)
    wire rom_area = (~a14) & (~a15) & (~mreq_n);
    assign rom_sys_ce_n  = ~(rom_area & (~cart_mode));
    assign rom_cart_ce_n = ~(rom_area & cart_mode);

endmodule
```
