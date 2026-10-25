# STM32F103 + I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植

[TOC]

## 前言

​	I2C 串行总线与 OLED 点阵显示是嵌入式物联网项目中常用的人机交互方案。直接操作 OLED 显存编写绘图代码工作量大，而 U8G2 开源图形库封装了底层驱动，能够快速实现文字、图形、滚动动画等效果。

​     本次实训以 STM32F103 为主控，基于 HAL 库完成 U8G2 库移植，搭配 0.96 寸 I2C OLED 与 AHT20 温湿度传感器，实现信息显示、长文本滚动、图像绘制等功能。实验过程同时加深对 I2C 通信时序、汉字点阵编码原理的理解，区分软件仿真与硬件实测波形的差异，掌握小型嵌入式图形界面的开发流程。

## 一、实验目的、所用器材

本次实验要达成三件事：理解 I2C 协议的原理和时序，能独立完成一次完整的读/写事务；掌握 0.96 寸 OLED（SSD1306）的显存结构、寻址方式和汉字点阵的编码与取模；掌握 U8g2 在 STM32 上的裁剪、移植与编译方法，以及图形界面可视化技术。

| 类别 | 型号 / 版本 | 关键参数 |
| ---- | ----------- | -------- |
| 主控 | STM32F103C8T6 最小系统板 | 72 MHz，64 KB Flash，20 KB SRAM |
| 显示 | 0.96 寸 OLED，SSD1306，128×64，I2C 四针 | 7 位地址 `0x3C`，写地址 `0x78` |
| 传感器 | AHT20 温湿度模块 | 7 位地址 `0x38`，写地址 `0x70` |
| 测量 | USB 逻辑分析仪（8 通道 24 MHz）+ PulseView；100 MHz 双通道示波器 | 采样率 ≥ 信号频率的 10 倍 |
| 软件 | Keil MDK 5.38、STM32CubeMX 6.14、STM32F10x 标准库 V3.5.0、U8g2 v2.34 | HAL 库 `STM32Cube FW_F1` |

## 二、基本原理

### 1. I2C 协议：从两根线到一帧数据

I2C 只有 SCL、SDA 两根线，都是开漏输出，高电平靠外部上拉电阻提供，所以软件模拟时引脚必须配成 `GPIO_Mode_Out_OD`，写 1 只是释放总线。上拉电阻一般 100 kHz 用 4.7 kΩ，400 kHz 用 2.2 kΩ。

时序三条铁律：起始是 SCL 高时 SDA 由高变低，停止是 SCL 高时 SDA 由低变高，数据在 SCL 高电平期间必须稳定。每发完 8 位，接收方在第 9 个时钟拉低 SDA 表示应答。

![](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/i2c-17927466893321.jpeg)

一次写显存的事务是：`START + 地址字节 + ACK + 控制字节 + ACK + 数据 + ACK + … + STOP`。速率有标准模式 100 kHz 和快速模式 400 kHz，任务一软件模拟，任务三硬件 I2C 配成 400 kHz。

### 2. SSD1306：显存分页与影子刷新

SSD1306 显存切成 **8 页**，每页 8 行、128 列，一列 8 个像素打包成 1 字节，整屏正好 1 KB。这个按列存储的结构，正是 16×16 汉字占 2 页、32 字节的原因，也是取模方向一错字就散架的根源。

![](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/ssd.jpeg)

I2C 传输的第一字节是控制字节：`0x00` 后面是命令，`0x40` 后面是显存数据。手写驱动用页寻址，和"汉字占 2 页"天然匹配。为避免滚动撕裂，两个工程都用影子显存加整帧刷新，且只刷新有内容的那几页。

### 3. 汉字点阵与取模

16×16 点阵一个汉字 32 字节，bit=1 点亮。PCtoLCD2002 的关键设置是：**阴码、列行式、逆向、C51 格式**，正好对应 SSD1306 的显存结构，设置错了字就会翻转、镜像或变成乱码条纹。

![ad6e3333ff23d7a8ff01f924837ee356](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/ad6e3333ff23d7a8ff01f924837ee356.jpg)

### 4. U8g2 的分层与缓冲

U8g2 从上到下是绘图调用、图形层、设备层和通信回调，移植时 u8g2 本身一行不改，只需实现 `u8x8_byte_stm32_hw_i2c()` 和 `u8x8_gpio_and_delay_stm32()` 两个函数。缓冲模式选**全屏缓冲**，名字带 `_f` 后缀，占 1 KB RAM，滑动动画必须用它。

## 三、任务一：标准库 + 手写驱动

### 1. 总线层：软件模拟 I2C 与自动探测

软件模拟 I2C 就是四个动作：起始、停止、发字节、读应答：

```c
static void I2C_Start(void)
{
    SDA_H(); SCL_H(); I2C_Delay();
    SDA_L();                    /* ← 关键：SCL 高电平期间拉低 SDA */
    I2C_Delay(); SCL_L(); I2C_Delay();
}

static void I2C_Stop(void)
{
    SDA_L(); I2C_Delay(); SCL_H(); I2C_Delay();
    SDA_H();                    /* ← 关键：SCL 高电平期间释放 SDA */
    I2C_Delay();
}

/* 发一个字节，返回从机应答：0 = ACK，1 = NACK */
static uint8_t I2C_SendByte(uint8_t byte)
{
    uint8_t i, nack;
    for (i = 0; i < 8; i++)
    {
        if (byte & 0x80) { SDA_H(); } else { SDA_L(); }   /* 先放数据 */
        byte <<= 1;
        I2C_Delay();
        SCL_H();                                          /* 再拉高时钟，从机采样 */
        I2C_Delay(); SCL_L(); I2C_Delay();
    }
    SDA_H(); I2C_Delay(); SCL_H(); I2C_Delay();
    nack = (uint8_t)SDA_READ();     /* 第 9 个时钟：0 = 从机应答 */
    SCL_L(); I2C_Delay();
    return nack;
}

/* 总线解锁：补 9 个 SCL 时钟，让被打断的从机把剩下的位吐完并放开 SDA */
void I2C_Bus_Recover(void)
{
    uint8_t i;
    SDA_H();
    for (i = 0; i < 9; i++) { SCL_H(); I2C_Delay(); SCL_L(); I2C_Delay(); }
    I2C_Stop();
}
```

总线解锁用于上次通信被打断、从机一直拉住 SDA 的情况。屏的接线和地址不确定，就让程序自己探测：

```c
uint8_t OLED_Init(void)
{
    static const uint8_t addrs[2] = { 0x3D, 0x3C };
    ...
    for (p = 0; p < I2C_PIN_PAIR_COUNT; p++)          /* 6 组候选引脚 */
    {
        I2C_Bus_PinPairUse(p);
        I2C_Bus_Recover();
        if (I2C_Bus_PairHasPullup(p) == 0) continue;  /* 两根线都被拉低 = 这组没器件 */

        for (a = 0; a < 2; a++)                       /* 每组把两个地址都试一遍 */
        {
            if (I2C_Bus_Write(addrs[a], 0, 0) == 0)   /* 只发地址，应答就算找到 */
            { pair = p; addr = addrs[a]; break; }
        }
        if (pair != 0xFF) break;
    }
    if (pair == 0xFF) return 0;

    OLED_WriteCmds(OLED_InitCmd, sizeof(OLED_InitCmd));
    OLED_Clear();
    OLED_Refresh();
    return s_oled_addr;
}
```

经验是顺序不能反：先切换引脚加解锁总线，再做电平判断；先在一组引脚上把两个地址试完，再去下一组。主程序里留了 `g_stage` 等调试变量，Keil 的 Watch 窗口直接看卡在哪一步。

### 2. 显示层：影子显存与中英混排

所有绘图只改影子显存，画完再按页送出：

```c
static unsigned char OLED_GRAM[OLED_PAGES][OLED_WIDTH];   /* 影子显存 8×128 = 1 KB */

static void OLED_WriteCmds(const uint8_t *cmd, uint16_t len)   /* 控制字 0x00 + 命令 */
{
    OLED_BusSelect();
    if (I2C_Bus_WriteEx(s_oled_addr, 0x00, cmd, len) != 0) { g_oled_nack++; }
    else                                                   { g_oled_good++; }
}

static void OLED_WriteData(const uint8_t *buf, uint16_t len)   /* 控制字 0x40 + 数据 */
{
    OLED_BusSelect();
    if (I2C_Bus_WriteEx(s_oled_addr, 0x40, buf, len) != 0) { g_oled_nack++; }
    else                                                   { g_oled_good++; }
}

static void OLED_SetPage(uint8_t page)
{
    uint8_t cmd[3];
    cmd[0] = (uint8_t)(0xB0 | (page & 0x07));   /* 页地址 */
    cmd[1] = 0x00;                              /* 列地址低 4 位 = 0 */
    cmd[2] = 0x10;                              /* 列地址高 4 位 = 0 */
    OLED_WriteCmds(cmd, 3);
}

void OLED_RefreshRange(uint8_t startPage, uint8_t endPage)      /* 只刷有内容的页 */
{
    uint8_t p;
    for (p = startPage; p <= endPage; p++)
    {
        OLED_SetPage(p);
        OLED_WriteData(&OLED_GRAM[p][0], OLED_WIDTH);
    }
}
```

写字节的小函数越界直接丢弃，文字就能自然从屏幕边缘滑进滑出：

```c
static void OLED_SetByte(int16_t x, uint8_t page, uint8_t data)
{
    if ((x < 0) || (x >= OLED_WIDTH) || (page >= OLED_PAGES)) return;   /* 裁切 */
    OLED_GRAM[page][x] = data;
}

uint8_t OLED_ShowChar(int16_t x, int16_t y, char ch)     /* 8×16 ASCII，占两页 */
{
    ...
    page = (uint8_t)(y >> 3);                 /* y 必须是 8 的倍数，才能整字节覆盖 */
    p = OLED_F8x16[(uint8_t)ch - 0x20];
    for (i = 0; i < 8; i++)
    {
        OLED_SetByte((int16_t)(x + i), page,               p[i]);
        OLED_SetByte((int16_t)(x + i), (uint8_t)(page + 1), p[i + 8]);
    }
    return 8;
}

static void OLED_DrawCNBitmap(int16_t x, int16_t y, const unsigned char *data)  /* 16×16 汉字 */
{
    uint8_t i;
    uint8_t page = (uint8_t)(y >> 3);
    for (i = 0; i < 16; i++)
    {
        OLED_SetByte((int16_t)(x + i), page,               data[i]);
        OLED_SetByte((int16_t)(x + i), (uint8_t)(page + 1), data[i + 16]);
    }
}
```

有了这两个原语，中英混排就是"看首字节是不是 ASCII，决定走 8 像素还是 16 像素"：

```c
uint16_t OLED_ShowStr(int16_t x, int16_t y, const char *str)
{
    const unsigned char *s = (const unsigned char *)str;
    int16_t x0 = x;
    uint8_t n;
    const CNFont_t *font;
    ...
    while (*s != '\0')
    {
        if (*s < 0x80)                       /* ASCII：宽 8 */
        {
            (void)OLED_ShowChar(x, y, (char)(*s));
            x += 8; s++;
        }
        else                                 /* 汉字：查表，宽 16 */
        {
            font = OLED_FindCN(s, &n);
            if (font != 0) OLED_DrawCNBitmap(x, y, font->data);
            else           OLED_DrawMissing(x, y);   /* 缺字画空心方框，一眼看出漏了哪个 */
            x += 16; s += n;
        }
    }
    return (uint16_t)(x - x0);               /* 返回实际宽度，方便后面紧接单位 */
}
```

信息页的汉字都存在 `CNFont_t` 表里，字模用 PCtoLCD2002 取出，操作过程见任务三。浮点数自己转字符串，避免 `%.1f` 需要额外开 MicroLIB 和浮点 printf：

```c
    x = (int16_t)OLED_ShowStr(0, 32, "温度 ");
    x = (int16_t)(x + (int16_t)OLED_ShowFloat(x, 32, temp, 1));
    (void)OLED_ShowCN(x, 32, "℃");
```

### 3. 传感器层与主循环

AHT20 的流程是触发测量 `0xAC 0x33 0x00`、等 80 ms、读 7 字节、按 20 位满量程换算：

```c
uint8_t AHT20_Read(float *temp, float *humi)
{
    uint8_t  cmd[3] = { 0xAC, 0x33, 0x00 };
    uint8_t  buf[7];
    uint32_t raw_h, raw_t;
    ...
    if (AHT20_WriteCmd(cmd, 3) != 0) return 2;         /* 发命令失败 */
    Delay_ms(80);                                      /* 手册 75ms，留足 */

    for (retry = 0; retry < 5; retry++)
    {
        AHT20_BusSelect();
        if (I2C_Bus_Read(AHT20_I2C_ADDR, buf, 7) != 0) return 3;
        if ((buf[0] & AHT20_STATUS_BUSY) == 0) break;  /* bit7 = 忙 */
        Delay_ms(10);
    }

    raw_h = ((uint32_t)buf[1] << 12) | ((uint32_t)buf[2] << 4) | (buf[3] >> 4);
    *humi = (float)raw_h * 100.0f / 1048576.0f;                  /* 2^20 */

    raw_t = ((uint32_t)(buf[3] & 0x0F) << 16) | ((uint32_t)buf[4] << 8) | buf[5];
    *temp = (float)raw_t * 200.0f / 1048576.0f - 50.0f;
    return 0;
}
```

$$
RH = \frac{D_{humi}}{2^{20}} \times 100\%,\qquad T = \frac{D_{temp}}{2^{20}} \times 200 - 50
$$

注意 `buf[3]` 高 4 位属于湿度、低 4 位属于温度；另外悬空引脚会假应答，初始化还要确认状态字非 0 才算器件真的在。

主循环是信息页 5 秒、滚动页交替，信息页每 0.5 秒刷新温湿度，读不到就显示 `--.-`：

```c
/* 第一行：学号。"学号" 2 个汉字 = 32 px，12 位学号 = 96 px，加起来正好 128 px */
OLED_ShowStr(0, 0, "学号" STUDENT_ID);
OLED_ShowStr(0, 16, "姓名 " STUDENT_NAME);          /* 第二行：姓名 */

if (err == 0)                                       /* 第三、四行：温湿度 */
{
    x = (int16_t)OLED_ShowStr(0, 32, "温度 ");
    x = (int16_t)(x + (int16_t)OLED_ShowFloat(x, 32, temp, 1));
    (void)OLED_ShowCN(x, 32, "℃");
    ...
}
else
{
    OLED_ShowStr(0, 32, "温度 --.-");               /* 传感器没接也不留空白 */
    OLED_ShowStr(0, 48, "湿度 --.-");
}
OLED_Refresh();                                     /* 8 页一次性送出 */
```

汉字 16 px、ASCII 8 px，"学号"两字加 12 位学号正好 **32 + 96 = 128**，铺满一行。

### 4. 滑动显示：软件滚动

整屏 1032 字节，400 kHz 下刷一帧约 23.2 ms，100 kHz 要 92.9 ms：

$$
t_{frame} = \frac{1032 \times 9}{400000} \approx 23.2\ \text{ms}
$$

软件滚动每帧重画、整体左移，靠越界裁切让文字滑出屏幕：

```c
static void ScrollPageSoft(void)
{
    int16_t x;
    int16_t w = (int16_t)OLED_StrWidth(SCROLL_TEXT);   /* 整句宽 352 px */
    uint8_t loop;

    OLED_Clear();
    OLED_Refresh();                     /* 先把信息页残留清干净 */

    for (loop = 0; loop < SCROLL_LOOPS; loop++)
    {
        for (x = OLED_WIDTH; x >= -w; x = (int16_t)(x - SCROLL_STEP_PX))
        {
            OLED_Clear();
            OLED_ShowStrCenter(0,  SCROLL_TITLE);    /* 静态标题 */
            OLED_ShowStr(x, 24, SCROLL_TEXT);        /* 滚动的那一行 */
            OLED_ShowStrCenter(48, SCROLL_FOOTER);   /* 静态脚注 */

            OLED_RefreshRange(0, 1);                 /* 只刷有字的 6 页 */
            OLED_RefreshRange(3, 4);
            OLED_RefreshRange(6, 7);
            Delay_ms(SCROLL_STEP_MS);                /* 20 ms */
        }
    }
}
```

一句话宽 352 像素、屏宽 128，所以要走 `(352+128)/2 = 240` 步，每步约 50 ms，一遍**约 12 秒**。

### 5.上板效果

<img src="STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/e2a552e369e857da4b65249b975fda47.jpg" alt="e2a552e369e857da4b65249b975fda47" style="zoom: 33%;" />

## 四、任务三：CubeMX + U8g2

### 1. HAL 框架与 U8g2 裁剪

用 CubeMX 6.14 新建工程，芯片选 `STM32F103C8Tx`

![image-20261023234418298](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261023234418298.png)

RCC 用外部 8 MHz 晶振，SYS 开 Serial Wire

![image-20261023234616276](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261023234616276.png)

时钟树 PLL ×9 到 72 MHz（APB1 36 MHz、APB2 72 MHz），**I2C1 配成 Fast Mode 400 kHz（PB6/PB7）**

![image-20261023234817360](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261023234817360.png)

![image-20261023234748901](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261023234748901.png)

PC13 配成 GPIO_Output 。

![image-20261023234916264](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261023234916264.png)

生成后在 `main()` 里只加两行，主循环里只调一个 `UI_Task()`：

```c
  /* USER CODE BEGIN 2 */
  OLED_Init();   /* 注册回调 + 初始化 SSD1306 + 清屏 */
  UI_Init();     /* 应用层：从开机画面开始轮播 */
  /* USER CODE END 2 */

  while (1)
  {
    UI_Task();                        /* 内部用 HAL_GetTick() 做非阻塞动画和换屏 */
    if ((HAL_GetTick() - led_tick) >= 500u)
    {
      led_tick += 500u;
      HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
  }
```

完整 U8g2 会撑爆 64 KB Flash，只保留 SSD1306 128×64 I2C 全屏缓冲这一条路径：

| 原文件 | 原大小 | 裁剪后 | 做法 |
| ------ | -----: | -----: | ---- |
| `u8g2_d_setup.c` | 467 KB | 1.1 KB | 只留 `u8g2_Setup_ssd1306_i2c_128x64_noname_f()` |
| `u8g2_d_memory.c` | 41 KB | 0.8 KB | 只留 `u8g2_m_16_8_f()`（1 KB 显存） |
| `u8g2_fonts.c` | **39 MB** | 25 KB | 只留 5×7、6×12、7×14B、10×20 四个 ASCII 字体 |
| `u8x8_d_*.c` | 100+ 个驱动 | 1 个 | 只留 `u8x8_d_ssd1306_128x64_noname.c` |

最终工程 **Flash 24.9 KB / 64 KB，RAM 3.0 KB / 20 KB，0 Error 0 Warning**。

### 2. 移植层：两个回调

第一个回调把 U8g2 攒的数据存进缓冲，到 `END_TRANSFER` 一次性用 HAL 发出：

```c
static uint8_t i2c_tx_buf[64];      /* ssd13xx 一次最多 24 字节 + 1 控制字，64 足够 */
static uint8_t i2c_tx_len;

uint8_t u8x8_byte_stm32_hw_i2c(u8x8_t *u8x8, uint8_t msg, uint8_t arg_int, void *arg_ptr)
{
  uint8_t *data = (uint8_t *)arg_ptr;
  (void)u8x8;

  switch (msg)
  {
    case U8X8_MSG_BYTE_SEND:
      while (arg_int > 0)
      {
        if (i2c_tx_len < sizeof(i2c_tx_buf)) i2c_tx_buf[i2c_tx_len++] = *data;
        data++; arg_int--;
      }
      break;
    case U8X8_MSG_BYTE_INIT:   break;      /* 总线由 MX_I2C1_Init() 初始化 */
    case U8X8_MSG_BYTE_SET_DC: break;      /* I2C 没有 DC 引脚 */
    case U8X8_MSG_BYTE_START_TRANSFER:  i2c_tx_len = 0; break;
    case U8X8_MSG_BYTE_END_TRANSFER:       /* 一次 START...STOP 把攒的字节全发出去 */
      HAL_I2C_Master_Transmit(&hi2c1, OLED_I2C_ADDR8, i2c_tx_buf, i2c_tx_len, 100);
      break;
    default: return 0;
  }
  return 1;
}

uint8_t u8x8_gpio_and_delay_stm32(u8x8_t *u8x8, uint8_t msg, uint8_t arg_int, void *arg_ptr)
{
  (void)u8x8; (void)arg_ptr;
  switch (msg)
  {
    case U8X8_MSG_GPIO_AND_DELAY_INIT:  break;
    case U8X8_MSG_DELAY_MILLI:   HAL_Delay(arg_int); break;
    case U8X8_MSG_DELAY_10MICRO: delay_us((uint32_t)arg_int * 10u); break;
    case U8X8_MSG_DELAY_100NANO:
    case U8X8_MSG_DELAY_NANO:    __NOP(); break;
    case U8X8_MSG_GPIO_CS: case U8X8_MSG_GPIO_DC: case U8X8_MSG_GPIO_RESET:
    case U8X8_MSG_GPIO_I2C_CLOCK: case U8X8_MSG_GPIO_I2C_DATA:
      break;                                /* I2C 接口用不到这些引脚 */
    default: return 0;                      /* ← 返回 0 = 失败！漏了分支屏就黑 */
  }
  return 1;
}
```

两个函数返回 0 就代表失败，U8g2 会放弃刷新；屏幕不亮多半是漏了延时分支。初始化只有四步：

```c
void OLED_Init(void)
{
  u8g2_Setup_ssd1306_i2c_128x64_noname_f(&u8g2, U8G2_R0,
                                         u8x8_byte_stm32_hw_i2c,
                                         u8x8_gpio_and_delay_stm32);
  u8g2_SetI2CAddress(&u8g2, OLED_I2C_ADDR8);   /* 0x3D 的模块改一处即可 */
  u8g2_InitDisplay(&u8g2);
  u8g2_SetPowerSave(&u8g2, 0);                 /* 退出休眠，点亮屏幕 */
  u8g2_ClearBuffer(&u8g2);
  u8g2_SendBuffer(&u8g2);
}
```

### 3. 四个显示要求

**① U8g2 的 demo 例程，**一屏之内用满十种绘图 API：

```c
  u8g2_DrawFrame(&u8g2, 3, 17, 28, 15);                  /* 空心矩形 */
  u8g2_DrawRFrame(&u8g2, 35, 17, 28, 15, 4);             /* 圆角矩形 */
  u8g2_DrawBox(&u8g2, 67, 17, 28, 15);                   /* 实心矩形 */
  u8g2_DrawRBox(&u8g2, 99, 17, 26, 15, 4);               /* 圆角实心 */
  u8g2_DrawCircle(&u8g2, 13, 46, (u8g2_uint_t)(6 + ((t / 250u) % 3u)), U8G2_DRAW_ALL);
  u8g2_DrawDisc(&u8g2, 38, 46, 6, U8G2_DRAW_ALL);        /* 实心圆 */
  u8g2_DrawTriangle(&u8g2, 52, 54, 63, 38, 74, 54);
  u8g2_DrawXBMP(&u8g2, 111, 42, 8, 8, bmp_heart8);       /* 位图 */
```

**② 学号姓名。**中文字库太大放不下，只取用到的 39 个汉字做成 16×16 点阵，共 1.2 KB。字模全部用 PCtoLCD2002 取出：设置阴码、列行式、逆向、C51 格式，字体宋体 16 号，粘入 39 个汉字生成 32 字节数组，再用脚本整理成查表结构和 `\xNN` 转义宏，最后电脑端预览核对方向位序再烧录。注意 `u8g2_DrawStr` 的 y 坐标是基线，要用 `u8g2_GetAscent()` 对齐中英文。

```c
#define CN_WANGSIYU "\xE7\x8E\x8B\xE6\x80\x9D\xE9\x9B\xA8"   /* 王思雨 */
#define CN_CQJTU    "\xE9\x87\x8D\xE5\xBA\x86\xE4\xBA\xA4\xE9\x80\x9A" \
                    "\xE5\xA4\xA7\xE5\xAD\xA6"               /* 重庆交通大学 */

/* 查表 + DrawXBMP，一个字一个 16×16 位图 */
u8g2_uint_t CN_DrawStr(u8g2_t *u8g2, u8g2_uint_t x, u8g2_uint_t y, const char *utf8)
{
  const uint8_t *bmp;
  while (*utf8)
  {
    bmp = cn16_lookup(utf8_next(&utf8));      /* UTF-8 → Unicode 码点 → 查点阵 */
    if (bmp != 0) u8g2_DrawXBMP(u8g2, x, y, 16, 16, bmp);
    x += 16;
  }
  return x;
}

static void draw_info(void)
{
  u8g2_ClearBuffer(&u8g2);
  u8g2_DrawRFrame(&u8g2, 0, 0, SCREEN_W, SCREEN_H, 4);       /* 圆角边框 */
  CN_DrawStr(&u8g2, 8, 4, CN_XUEHAO);                        /* 学号 */
  u8g2_SetFont(&u8g2, FONT_BODY);
  u8g2_DrawStr(&u8g2, 8 + 32 + 6, 4 + u8g2_GetAscent(&u8g2), "632407030504");
  CN_DrawStr(&u8g2, 8, 24, CN_XINGMING);                     /* 姓名 */
  CN_DrawStr(&u8g2, 8 + 32 + 6, 24, CN_WANGSIYU);
  CN_DrawStrCenter(&u8g2, 44, SCREEN_W, CN_CQJTU);            /* 学校 */
}
```

![image-20261024163433705](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261024163433705.png)

![image-20261024163447984](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261024163447984.png)

**③ 上下、左右滑动。**都用裁剪窗口加时间戳算坐标，9 秒和 8 秒一轮。`CN_DrawStrAt()` 用带符号的 int16_t 坐标，因为 `u8g2_DrawXBMP()` 的参数是无符号类型，负数会被当成巨大的正数。

```c
static void draw_hscroll(uint32_t t)
{
  u8g2_uint_t w = CN_StrWidth(CN_STR_ALL);               /* 整句宽 448 px */
  int16_t span = (int16_t)(w + SCREEN_W);
  int16_t x = (int16_t)(SCREEN_W - (int16_t)(((uint32_t)t * (uint32_t)span) / 9000u));

  u8g2_ClearBuffer(&u8g2);
  u8g2_DrawFrame(&u8g2, 0, 0, SCREEN_W, SCREEN_H);
  u8g2_SetClipWindow(&u8g2, 0, 22, SCREEN_W - 1, 41);    /* 20 px 高的滑动窗口 */
  CN_DrawStrAt(&u8g2, x, 22, CN_STR_ALL);                /* 带符号坐标，可画到屏幕外 */
  u8g2_SetMaxClipWindow(&u8g2);                          /* 恢复全屏裁剪 */
}

static void draw_vscroll(uint32_t t)
{
  uint16_t total  = VSCROLL_LINE_CNT * VSCROLL_LINE_H;              /* 5 × 18 */
  uint32_t offset = ((uint32_t)t * (uint32_t)total / 8000u) % total; /* 8 s 一轮 */

  u8g2_ClearBuffer(&u8g2);
  u8g2_DrawFrame(&u8g2, 0, 0, SCREEN_W, SCREEN_H);
  u8g2_SetClipWindow(&u8g2, 0, VSCROLL_TOP, SCREEN_W - 1, VSCROLL_BOTTOM);
  for (i = 0; i < VSCROLL_LINE_CNT; i++)
  {
    y = (int16_t)(VSCROLL_TOP + i * VSCROLL_LINE_H - (int16_t)offset);
    if (y > -16 && y < (int16_t)(VSCROLL_BOTTOM + 16))   /* 只画窗口附近的几行 */
      CN_DrawStrAt(&u8g2, 4, y, VSCROLL_LINES[i]);
  }
  u8g2_SetMaxClipWindow(&u8g2);
}。
```

**④ 图案与动态效果。** 64×64 心形，4 帧循环，一张 8 元素的节奏表每 110 ms 换帧，看起来就是心跳效果。

```c
static void draw_logo(uint32_t t)
{
  static const uint8_t beat[8] = {1, 2, 3, 2, 1, 2, 3, 2};   /* 心跳节奏 */
  uint8_t frame = beat[(t / 110u) % 8u];

  u8g2_ClearBuffer(&u8g2);
  u8g2_SetDrawColor(&u8g2, 1);
  u8g2_DrawXBMP(&u8g2, (SCREEN_W - LOGO_FRAME_W) / 2, 0,
                LOGO_FRAME_W, LOGO_FRAME_H, logo_frames[frame]);
}
```

六个画面由非阻塞调度器轮播，每屏有自己的停留时间和刷新间隔，全程没有一处 `delay`：

```c
void UI_Task(void)
{
  uint32_t now = HAL_GetTick();
  uint32_t t;

  if ((now - s_enter_ms) >= s_duration[s_screen])   /* 停留时间到，换下一屏 */
  {
    s_screen = (ui_screen_t)((s_screen + 1) % SCR_COUNT);
    s_enter_ms = now;
    s_last_draw_ms = 0;
  }
  if (s_last_draw_ms != 0 && (now - s_last_draw_ms) < s_interval[s_screen])
  {
    return;                                          /* 还没到刷新时间，直接返回 */
  }
  s_last_draw_ms = now;
  t = now - s_enter_ms;                              /* 本屏已经显示了多久 */

  switch (s_screen) { ... }

  OLED_Refresh();
}
```

**上板效果**

U8G2的demo例程

<img src="STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/3f14eac47595f83632ba33fade709647.jpg" alt="3f14eac47595f83632ba33fade709647" style="zoom:50%;" />

<img src="STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/511f031aa3df7c139bf0ea81929d991c.jpg" alt="511f031aa3df7c139bf0ea81929d991c" style="zoom:50%;" />

学号姓名显示

<img src="STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/443dccdc19c73620356685c70e1fd433.jpg" alt="443dccdc19c73620356685c70e1fd433" style="zoom:50%;" />

上下左右滑动显示

<img src="STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/2c71b03314d410a3f1308174da631b11.jpg" alt="2c71b03314d410a3f1308174da631b11" style="zoom:50%;" />

<img src="STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/98b8d7d66234972cfba6708697023a12.jpg" alt="98b8d7d66234972cfba6708697023a12" style="zoom:50%;" />

显示一个动态图案

<img src="STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/5f53f1081f23ad3d9871c8e063a6e2d0.jpg" alt="5f53f1081f23ad3d9871c8e063a6e2d0" style="zoom:50%;" />



### 4. 思考题：Keil 虚拟逻辑分析仪采集 SDA 波形是什么样

Keil 的 `Logic Analyzer` 不是测引脚电压，而是读某个寄存器的值随时间画波形。任务三的 SCL/SDA 由 I2C1 外设硬件驱动，CPU 没写过这两个 GPIO 的 ODR 位，所以 Logic Analyzer 里只会看到一条恒定直线，看不到 START、ACK 等任何内容；纯仿真环境下还会因为没有 I2C 从机而让 `HAL_I2C_Master_Transmit()` 卡到超时。波形只能在真实硬件上抓。

![image-20261024180308260](STM32F103 I2C OLED 显示实验 —— 汉字点阵、AHT20 温湿度与 U8g2 移植.assets/image-20261024180308260.png)

抓不到时的替代手段：

| 方法            | 怎么做                                                       | 能看到什么                                   | 局限             |
| --------------- | ------------------------------------------------------------ | -------------------------------------------- | ---------------- |
| 真实逻辑分析仪  | Saleae / DSLogic，SCL、SDA 各一通道，软件加 I2C 解码         | 完整帧：地址、R/W、ACK、控制字节、命令、数据 | 要硬件，必须共地 |
| 示波器双通道    | CH1=SCL、CH2=SDA，触发 SCL 下降沿，单次捕获                  | 电平、上升沿 RC、ACK 是否真被拉低            | 一次只看一小段   |
| GPIO 打点法     | 在 START、每个字节、STOP 处翻转一个空闲 IO                   | 各阶段耗时、帧间隔                           | 看不到具体数据位 |
| 串口 / SWO 日志 | `HAL_GetTick()` 打时间戳，配合 `HAL_I2C_GetError()`、`HAL_I2C_IsDeviceReady()` | 传输成败、重试次数、每步耗时                 | 会占用总线时间   |
| 自环 / 从机回环 | 用另一块 STM32 当 I2C 从机收数据再回显                       | 数据内容是否正确                             | 需要第二块板     |

## 五、实验总结

本次实验以 STM32F103 为主控，基于 I2C 总线完成 U8G2 图形库移植，驱动 0.96 寸 OLED 屏并读取 AHT20 温湿度数据，实现学号姓名静态显示、温湿度实时显示、长字符串滑动显示功能。实验巩固了 I2C 通信原理、OLED 汉字点阵编码以及缓冲区刷屏机制。

调试过程中，遇到 I2C 地址错误、汉字取模格式不对、屏幕闪烁等问题，采用分模块单独测试的方式逐一排查解决，掌握了嵌入式软硬件调试的基本思路。本次未开展逻辑分析仪、示波器的信号测试，缺少对 I2C 底层时序的观测。

通过本次实训，我熟悉了 HAL 库开发与 U8G2 的使用，将物联网感知与屏幕显示结合起来，提升了嵌入式开发实践能力，为后续物联网项目开发积累了经验。
