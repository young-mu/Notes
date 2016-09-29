## Tiva registers

### system control

take EEPROM as an example, you can refer it in `nuttx/arch/arm/src/tiva/chip/tm4c129_syscontrl.h`

You can get the register value by using the following commands `mb`/`mh`/`mw` in nsh

```bash
nsh> mb 400fe358
  400fe358 = 0x01
```

```bash
nsh> mh 400fe358
  400fe358 = 0x0001
```

```bash
nsh> mw 400fe358
  400fe358 = 0x00000001
```

#### 1. PP_xxx 

**Peripheral Present**

`#define TIVA_SYSCON_PPEEPROM_OFFSET    0x0358`

- `0` - module is **not** present
- `1` - module is present

#### 2. SR_xxx

**Software Reset**

`#define TIVA_SYSCON_SREEPROM_OFFSET    0x0558`

- `0` - module is not reset
- `1` - reset the module

#### 3. RCGC_xxx

**Run mode Clock Gating Control** -- `enableclks()`

`#define TIVA_SYSCON_RCGCEEPROM_OFFSET  0x0658`

- `0` - module is disabled
- `1` - enable and provide a clock to the module in **run** mode

#### 4. SCGC_xxx

**Sleep mode Clock Gating**

`#define TIVA_SYSCON_SCGCEEPROM_OFFSET  0x0758`

- `0` - module is disabled
- `1` - enable and provide a clock to the module in **sleep** mode

#### 5. DCGC_xxx

**Deep-sleep mode Clock Gating Control**

`#define TIVA_SYSCON_DCGCEEPROM_OFFSET  0x0858`

- `0` - module is disabled
- `1` - enable and provide a clock to the module in **deep-sleep** mode

#### 6. PC_xxx

**Power Control** -- `enablepwr()`

`#define TIVA_SYSCON_PCEEPROM_OFFSET    0x0958`

- `0` - module is **not** powered and does not receive a clock.
- `1` - module is powered but does not receive a clock.

#### 7. PR_xxx

**Peripheral Ready**

`#define TIVA_SYSCON_PREEPROM_OFFSET    0x0a58`

- `0` - module is **not** ready for access
- `1` - module is ready for access

By default, the eeprom's PC register is 1 and RCGC register is 0, so the PR register is 0. After set RCGC register as 1, the PR register will be 1.

```bash
nsh> mw 400fe958
  400fe958 = 0x00000001
nsh> mw 400fe658
  400fe658 = 0x00000000
nsh> mw 400fea58
  400fea58 = 0x00000000
nsh> mw 400fe658=1
  400fe658 = 0x00000000 -> 0x00000001
nsh> mw 400fea58
  400fea58 = 0x00000001
```