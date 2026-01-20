[README (9).md](https://github.com/user-attachments/files/24731705/README.9.md)
# 01_ST7735S_SPI_160x80

<img width="957" height="899" alt="004" src="https://github.com/user-attachments/assets/991a4ec9-5b22-494e-a85c-affe61f94816" />


<img width="957" height="899" alt="005" src="https://github.com/user-attachments/assets/ec4d5a44-347e-41ec-9cd7-51d63cf64d89" />

코드 수정
/* USER CODE END Header */
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include <string.h>
#include <stdlib.h>
/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */

// ★ 오프셋 설정 (0.96" 160x80 LCD, 핀 위쪽) ★
// 안 맞으면 다른 값 시도: (24,0), (26,1), (1,26), (0,24)
#define X_OFFSET   0 //24
#define Y_OFFSET   26

// ST7735 Commands
#define ST7735_SWRESET 0x01
#define ST7735_SLPOUT  0x11
#define ST7735_NORON   0x13
#define ST7735_INVOFF  0x20
#define ST7735_INVON   0x21
#define ST7735_DISPON  0x29
#define ST7735_CASET   0x2A
#define ST7735_RASET   0x2B
#define ST7735_RAMWR   0x2C
#define ST7735_COLMOD  0x3A
#define ST7735_MADCTL  0x36
#define ST7735_FRMCTR1 0xB1
#define ST7735_FRMCTR2 0xB2
#define ST7735_FRMCTR3 0xB3
#define ST7735_INVCTR  0xB4
#define ST7735_PWCTR1  0xC0
#define ST7735_PWCTR2  0xC1
#define ST7735_PWCTR3  0xC2
#define ST7735_PWCTR4  0xC3
#define ST7735_PWCTR5  0xC4
#define ST7735_VMCTR1  0xC5
#define ST7735_GMCTRP1 0xE0
#define ST7735_GMCTRN1 0xE1

// Colors (RGB565)
#define BLACK       0x0000
#define WHITE       0xFFFF
#define RED         0xF800
#define GREEN       0x07E0
#define BLUE        0x001F
#define EYE_COLOR   0x07E0   // Green
#define EYE_BRIGHT  0xAFE5
#define EYE_DIM     0x0320

// 눈 위치 (화면 좌표)
#define LX              40      // 왼쪽 눈 중심 X
#define RX              120     // 오른쪽 눈 중심 X
#define CY              40      // 눈 중심 Y (화면 중앙)

// 눈 크기
#define EYE_W           30
#define EYE_H           50
#define EYE_R           10

// GPIO 매크로
#define LCD_CS_LOW()   HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_RESET)
#define LCD_CS_HIGH()  HAL_GPIO_WritePin(GPIOB, GPIO_PIN_6, GPIO_PIN_SET)
#define LCD_DC_LOW()   HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_RESET)
#define LCD_DC_HIGH()  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_6, GPIO_PIN_SET)
#define LCD_RES_LOW()  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET)
#define LCD_RES_HIGH() HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET)

// Expression 타입
typedef enum {
    EXPR_NORMAL, EXPR_HAPPY, EXPR_SAD, EXPR_ANGRY,
    EXPR_SURPRISED, EXPR_SLEEPY, EXPR_WINK_LEFT, EXPR_WINK_RIGHT,
    EXPR_BLINK, EXPR_LOVE, EXPR_DIZZY,
    EXPR_LOOK_LEFT, EXPR_LOOK_RIGHT, EXPR_LOOK_UP, EXPR_LOOK_DOWN
} Expression_t;

static Expression_t current_expr = EXPR_NORMAL;
static uint32_t last_blink = 0;
static uint32_t last_action = 0;

// ============================================================================
// LCD 기본 함수
// ============================================================================

static void LCD_Cmd(uint8_t cmd) {
    LCD_DC_LOW();
    LCD_CS_LOW();
    HAL_SPI_Transmit(&hspi1, &cmd, 1, HAL_MAX_DELAY);
    LCD_CS_HIGH();
}

static void LCD_Data(uint8_t data) {
    LCD_DC_HIGH();
    LCD_CS_LOW();
    HAL_SPI_Transmit(&hspi1, &data, 1, HAL_MAX_DELAY);
    LCD_CS_HIGH();
}

// ★ 오프셋 적용된 SetWindow ★
static void LCD_SetWindow(uint16_t x0, uint16_t y0, uint16_t x1, uint16_t y1) {
    LCD_Cmd(ST7735_CASET);
    LCD_Data(0); LCD_Data(x0 + X_OFFSET);
    LCD_Data(0); LCD_Data(x1 + X_OFFSET);

    LCD_Cmd(ST7735_RASET);
    LCD_Data(0); LCD_Data(y0 + Y_OFFSET);
    LCD_Data(0); LCD_Data(y1 + Y_OFFSET);

    LCD_Cmd(ST7735_RAMWR);
}

// 고속 색상 출력
static void LCD_WriteColorFast(uint16_t color, uint32_t count) {
    uint8_t hi = color >> 8;
    uint8_t lo = color & 0xFF;

    LCD_DC_HIGH();
    LCD_CS_LOW();
    while(count--) {
        HAL_SPI_Transmit(&hspi1, &hi, 1, HAL_MAX_DELAY);
        HAL_SPI_Transmit(&hspi1, &lo, 1, HAL_MAX_DELAY);
    }
    LCD_CS_HIGH();
}

// 사각형 채우기
static void LCD_FillRect(int16_t x, int16_t y, int16_t w, int16_t h, uint16_t color) {
    if(x >= LCD_WIDTH || y >= LCD_HEIGHT || w <= 0 || h <= 0) return;
    if(x < 0) { w += x; x = 0; }
    if(y < 0) { h += y; y = 0; }
    if(x + w > LCD_WIDTH) w = LCD_WIDTH - x;
    if(y + h > LCD_HEIGHT) h = LCD_HEIGHT - y;
    if(w <= 0 || h <= 0) return;

    LCD_SetWindow(x, y, x + w - 1, y + h - 1);
    LCD_WriteColorFast(color, (uint32_t)w * h);
}

// 수평선
static void LCD_HLine(int16_t x, int16_t y, int16_t w, uint16_t color) {
    if(y < 0 || y >= LCD_HEIGHT || w <= 0) return;
    if(x < 0) { w += x; x = 0; }
    if(x + w > LCD_WIDTH) w = LCD_WIDTH - x;
    if(w <= 0) return;

    LCD_SetWindow(x, y, x + w - 1, y);
    LCD_WriteColorFast(color, w);
}

// 채워진 원
static void LCD_FillCircle(int16_t x0, int16_t y0, int16_t r, uint16_t color) {
    int16_t x = r, y = 0;
    int16_t err = 1 - r;

    while(x >= y) {
        LCD_HLine(x0 - x, y0 + y, x * 2 + 1, color);
        LCD_HLine(x0 - x, y0 - y, x * 2 + 1, color);
        LCD_HLine(x0 - y, y0 + x, y * 2 + 1, color);
        LCD_HLine(x0 - y, y0 - x, y * 2 + 1, color);

        y++;
        if(err < 0) err += 2 * y + 1;
        else { x--; err += 2 * (y - x + 1); }
    }
}

// 둥근 사각형
static void LCD_RoundRect(int16_t x, int16_t y, int16_t w, int16_t h, int16_t r, uint16_t color) {
    if(w > 2*r) LCD_FillRect(x + r, y, w - 2*r, h, color);
    if(h > 2*r) {
        LCD_FillRect(x, y + r, r, h - 2*r, color);
        LCD_FillRect(x + w - r, y + r, r, h - 2*r, color);
    }
    LCD_FillCircle(x + r, y + r, r, color);
    LCD_FillCircle(x + w - r - 1, y + r, r, color);
    LCD_FillCircle(x + r, y + h - r - 1, r, color);
    LCD_FillCircle(x + w - r - 1, y + h - r - 1, r, color);
}

// 두꺼운 선
static void LCD_ThickLine(int16_t x0, int16_t y0, int16_t x1, int16_t y1, int16_t t, uint16_t color) {
    int16_t dx = (x1 > x0) ? (x1 - x0) : (x0 - x1);
    int16_t dy = (y1 > y0) ? (y1 - y0) : (y0 - y1);

    if(dy <= 2) {
        int16_t minX = (x0 < x1) ? x0 : x1;
        int16_t maxX = (x0 > x1) ? x0 : x1;
        LCD_FillRect(minX, (y0 + y1)/2 - t/2, maxX - minX + 1, t, color);
        return;
    }
    if(dx <= 2) {
        int16_t minY = (y0 < y1) ? y0 : y1;
        int16_t maxY = (y0 > y1) ? y0 : y1;
        LCD_FillRect((x0 + x1)/2 - t/2, minY, t, maxY - minY + 1, color);
        return;
    }

    int16_t sx = (x0 < x1) ? 1 : -1;
    int16_t sy = (y0 < y1) ? 1 : -1;
    int16_t err = dx - dy;

    while(1) {
        LCD_FillRect(x0 - t/2, y0 - t/2, t, t, color);
        if(x0 == x1 && y0 == y1) break;
        int16_t e2 = 2 * err;
        if(e2 > -dy) { err -= dy; x0 += sx; }
        if(e2 < dx) { err += dx; y0 += sy; }
    }
}

// ============================================================================
// LCD 초기화
// ============================================================================

static void LCD_Init(void) {
    LCD_RES_LOW(); HAL_Delay(50);
    LCD_RES_HIGH(); HAL_Delay(50);

    LCD_Cmd(ST7735_SWRESET); HAL_Delay(150);
    LCD_Cmd(ST7735_SLPOUT); HAL_Delay(150);

    LCD_Cmd(ST7735_FRMCTR1);
    LCD_Data(0x01); LCD_Data(0x2C); LCD_Data(0x2D);
    LCD_Cmd(ST7735_FRMCTR2);
    LCD_Data(0x01); LCD_Data(0x2C); LCD_Data(0x2D);
    LCD_Cmd(ST7735_FRMCTR3);
    LCD_Data(0x01); LCD_Data(0x2C); LCD_Data(0x2D);
    LCD_Data(0x01); LCD_Data(0x2C); LCD_Data(0x2D);

    LCD_Cmd(ST7735_INVCTR); LCD_Data(0x07);
    LCD_Cmd(ST7735_PWCTR1); LCD_Data(0xA2); LCD_Data(0x02); LCD_Data(0x84);
    LCD_Cmd(ST7735_PWCTR2); LCD_Data(0xC5);
    LCD_Cmd(ST7735_PWCTR3); LCD_Data(0x0A); LCD_Data(0x00);
    LCD_Cmd(ST7735_PWCTR4); LCD_Data(0x8A); LCD_Data(0x2A);
    LCD_Cmd(ST7735_PWCTR5); LCD_Data(0x8A); LCD_Data(0xEE);
    LCD_Cmd(ST7735_VMCTR1); LCD_Data(0x0E);

    LCD_Cmd(ST7735_INVOFF);

    // ★ MADCTL: 핀 위쪽, 가로 모드 ★
    // 안 맞으면 0xC8, 0x60, 0x68, 0xA0, 0xA8 등 시도
    LCD_Cmd(ST7735_MADCTL); LCD_Data(0x60); //C0);

    LCD_Cmd(ST7735_COLMOD); LCD_Data(0x05);  // 16-bit color

    LCD_Cmd(ST7735_GMCTRP1);
    uint8_t gp[] = {0x02,0x1c,0x07,0x12,0x37,0x32,0x29,0x2d,0x29,0x25,0x2B,0x39,0x00,0x01,0x03,0x10};
    for(int i=0;i<16;i++) LCD_Data(gp[i]);
    LCD_Cmd(ST7735_GMCTRN1);
    uint8_t gn[] = {0x03,0x1d,0x07,0x06,0x2E,0x2C,0x29,0x2D,0x2E,0x2E,0x37,0x3F,0x00,0x00,0x02,0x10};
    for(int i=0;i<16;i++) LCD_Data(gn[i]);

    LCD_Cmd(ST7735_NORON); HAL_Delay(10);
    LCD_Cmd(ST7735_DISPON); HAL_Delay(100);
}

// 전체 화면 클리어
static void LCD_Clear(uint16_t color) {
    LCD_FillRect(0, 0, LCD_WIDTH, LCD_HEIGHT, color);
}

// ============================================================================
// 눈 그리기 함수
// ============================================================================

static void Eye_Normal(int16_t cx, int16_t ox, int16_t oy) {
    LCD_RoundRect(cx - EYE_W/2, CY - EYE_H/2, EYE_W, EYE_H, EYE_R, EYE_COLOR);
    LCD_FillCircle(cx - 5 + ox, CY - 8 + oy, 4, EYE_BRIGHT);
}

static void Eye_Closed(int16_t cx) {
    LCD_FillRect(cx - EYE_W/2 + 3, CY - 3, EYE_W - 6, 6, EYE_COLOR);
}

static void Eye_Half(int16_t cx, uint8_t pct) {
    int16_t h = (EYE_H * pct) / 100;
    if(h < 10) { Eye_Closed(cx); return; }
    int16_t top = CY + EYE_H/2 - h;
    LCD_RoundRect(cx - EYE_W/2, top, EYE_W, h, EYE_R/2, EYE_COLOR);
}

static void Eye_Happy(int16_t cx) {
    for(int16_t i = -EYE_W/2 + 2; i <= EYE_W/2 - 2; i++) {
        int32_t n = (int32_t)i * i * 100 / ((EYE_W/2) * (EYE_W/2));
        int16_t y = CY + 5 - (12 * (100 - n) / 100);
        LCD_FillRect(cx + i, y - 3, 2, 5, EYE_COLOR);
    }
}

static void Eye_Sad(int16_t cx) {
    LCD_RoundRect(cx - EYE_W/2, CY - EYE_H/2 + 6, EYE_W, EYE_H - 6, EYE_R, EYE_COLOR);
    LCD_ThickLine(cx - EYE_W/2 - 2, CY - EYE_H/2 - 2,
                  cx + EYE_W/2 + 2, CY - EYE_H/2 + 8, 4, EYE_COLOR);
}

static void Eye_Angry(int16_t cx, uint8_t is_left) {
    LCD_RoundRect(cx - EYE_W/2, CY - EYE_H/2 + 8, EYE_W, EYE_H - 12, EYE_R - 2, EYE_COLOR);
    if(is_left) {
        LCD_ThickLine(cx - EYE_W/2 - 3, CY - EYE_H/2 + 6,
                      cx + EYE_W/2 + 3, CY - EYE_H/2 - 6, 5, EYE_COLOR);
    } else {
        LCD_ThickLine(cx - EYE_W/2 - 3, CY - EYE_H/2 - 6,
                      cx + EYE_W/2 + 3, CY - EYE_H/2 + 6, 5, EYE_COLOR);
    }
}

static void Eye_Surprised(int16_t cx) {
    LCD_FillCircle(cx, CY, EYE_H/2 - 2, EYE_COLOR);
    LCD_FillCircle(cx, CY, EYE_H/2 - 10, EYE_DIM);
    LCD_FillCircle(cx - 5, CY - 6, 5, EYE_BRIGHT);
    LCD_FillCircle(cx + 3, CY + 3, 3, EYE_BRIGHT);
}

static void Eye_Heart(int16_t cx) {
    int16_t s = 14;
    LCD_FillCircle(cx - s/2, CY - s/3, s/2, EYE_COLOR);
    LCD_FillCircle(cx + s/2, CY - s/3, s/2, EYE_COLOR);
    for(int16_t r = 0; r < s; r++) {
        LCD_FillRect(cx - (s - r), CY - s/3 + r, (s - r) * 2 + 1, 1, EYE_COLOR);
    }
}

static void Eye_X(int16_t cx) {
    int16_t s = EYE_H/2 - 6;
    LCD_ThickLine(cx - s, CY - s, cx + s, CY + s, 5, EYE_COLOR);
    LCD_ThickLine(cx + s, CY - s, cx - s, CY + s, 5, EYE_COLOR);
}

// ============================================================================
// 표정 설정
// ============================================================================

static void Draw_Expression(Expression_t expr, int16_t ox, int16_t oy) {
    LCD_Clear(BLACK);

    switch(expr) {
        case EXPR_NORMAL:
            Eye_Normal(LX, ox, oy);
            Eye_Normal(RX, ox, oy);
            break;
        case EXPR_HAPPY:
            Eye_Happy(LX);
            Eye_Happy(RX);
            break;
        case EXPR_SAD:
            Eye_Sad(LX);
            Eye_Sad(RX);
            break;
        case EXPR_ANGRY:
            Eye_Angry(LX, 1);
            Eye_Angry(RX, 0);
            break;
        case EXPR_SURPRISED:
            Eye_Surprised(LX);
            Eye_Surprised(RX);
            break;
        case EXPR_SLEEPY:
            Eye_Half(LX, 30);
            Eye_Half(RX, 30);
            break;
        case EXPR_WINK_LEFT:
            Eye_Closed(LX);
            Eye_Normal(RX, 0, 0);
            break;
        case EXPR_WINK_RIGHT:
            Eye_Normal(LX, 0, 0);
            Eye_Closed(RX);
            break;
        case EXPR_BLINK:
            Eye_Closed(LX);
            Eye_Closed(RX);
            break;
        case EXPR_LOVE:
            Eye_Heart(LX);
            Eye_Heart(RX);
            break;
        case EXPR_DIZZY:
            Eye_X(LX);
            Eye_X(RX);
            break;
        case EXPR_LOOK_LEFT:
            Eye_Normal(LX, -6, 0);
            Eye_Normal(RX, -6, 0);
            break;
        case EXPR_LOOK_RIGHT:
            Eye_Normal(LX, 6, 0);
            Eye_Normal(RX, 6, 0);
            break;
        case EXPR_LOOK_UP:
            Eye_Normal(LX, 0, -6);
            Eye_Normal(RX, 0, -6);
            break;
        case EXPR_LOOK_DOWN:
            Eye_Normal(LX, 0, 6);
            Eye_Normal(RX, 0, 6);
            break;
    }
}

static void Anim_SetExpr(Expression_t expr) {
    current_expr = expr;
    Draw_Expression(expr, 0, 0);
}

// ============================================================================
// 애니메이션
// ============================================================================

static void Anim_Blink(void) {
    LCD_Clear(BLACK);
    Eye_Half(LX, 50);
    Eye_Half(RX, 50);

    LCD_Clear(BLACK);
    Eye_Closed(LX);
    Eye_Closed(RX);
    HAL_Delay(50);

    LCD_Clear(BLACK);
    Eye_Half(LX, 50);
    Eye_Half(RX, 50);

    Draw_Expression(current_expr, 0, 0);
}

static void Anim_WinkL(void) {
    LCD_Clear(BLACK);
    Eye_Closed(LX);
    Eye_Normal(RX, 0, 0);
    HAL_Delay(200);
    Draw_Expression(EXPR_NORMAL, 0, 0);
}

static void Anim_WinkR(void) {
    LCD_Clear(BLACK);
    Eye_Normal(LX, 0, 0);
    Eye_Closed(RX);
    HAL_Delay(200);
    Draw_Expression(EXPR_NORMAL, 0, 0);
}

static void Anim_LookAround(void) {
    Anim_SetExpr(EXPR_LOOK_LEFT);
    HAL_Delay(300);
    Anim_SetExpr(EXPR_NORMAL);
    HAL_Delay(100);
    Anim_SetExpr(EXPR_LOOK_RIGHT);
    HAL_Delay(300);
    Anim_SetExpr(EXPR_NORMAL);
}

static void Anim_Idle(void) {
    uint32_t t = HAL_GetTick();

    if(t - last_blink > 2500 + (rand() % 2000)) {
        Anim_Blink();
        last_blink = t;
    }

    if(t - last_action > 6000 + (rand() % 4000)) {
        switch(rand() % 5) {
            case 0: Anim_SetExpr(EXPR_LOOK_LEFT); HAL_Delay(350); break;
            case 1: Anim_SetExpr(EXPR_LOOK_RIGHT); HAL_Delay(350); break;
            case 2: Anim_WinkL(); break;
            case 3: Anim_WinkR(); break;
            case 4: Anim_LookAround(); break;
        }
        Anim_SetExpr(EXPR_NORMAL);
        last_action = t;
    }
}

static void Anim_Demo(void) {
    Anim_SetExpr(EXPR_NORMAL);    HAL_Delay(1000);
    Anim_Blink();                  HAL_Delay(500);
    Anim_SetExpr(EXPR_HAPPY);     HAL_Delay(1000);
    Anim_SetExpr(EXPR_SAD);       HAL_Delay(1000);
    Anim_SetExpr(EXPR_ANGRY);     HAL_Delay(1000);
    Anim_SetExpr(EXPR_SURPRISED); HAL_Delay(1000);
    Anim_WinkL();                  HAL_Delay(400);
    Anim_WinkR();                  HAL_Delay(400);
    Anim_SetExpr(EXPR_LOVE);      HAL_Delay(1000);
    Anim_SetExpr(EXPR_SLEEPY);    HAL_Delay(1000);
    Anim_SetExpr(EXPR_DIZZY);     HAL_Delay(1000);
    Anim_LookAround();             HAL_Delay(500);
}

결과 

https://github.com/user-attachments/assets/6a1d34fc-ac2b-49de-8ef5-bea010c4c3f1


