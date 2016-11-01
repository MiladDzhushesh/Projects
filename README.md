
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "stm32f3xx_hal.h"
#include "stm32f3_discovery.h"
#include "stm32f3_discovery_accelerometer.h"
#include "stm32f3_discovery_gyroscope.h"
#include "common.h"
extern uint32_t myTickCount;
GPIO_PinState buffer[16][20];
int32_t bar1x, bar2x, bar3x, bar1y, bar2y, bar3y, jumpx, jumpy, jumpState, score;
int32_t gameState = 0;
// barx range 0~15
// jumpState range 0~8
void jumpInit(void);
void jumpParameterUpdate(void);
void jumpClearBuffer(void);
void jumpUpdateBuffer(void);
int16_t Accel(void);
void bufferToLED(void);
void gameLCDmode(void);
void lcd_CostomizedChar(int32_t y, int32_t x);
void gameLCDsetUp(void);
void lcd_set(GPIO_PinState RS, GPIO_PinState D7, GPIO_PinState D6,
             GPIO_PinState D5, GPIO_PinState D4, GPIO_PinState D3,
             GPIO_PinState D2, GPIO_PinState D1, GPIO_PinState D0);
void gameLogic(void);
void jumpEndingAnimation(void);

void jumpInit(void){
  srand(myTickCount);
  bar1y = 1;
  bar2y = 7;
  bar3y = 13;
  bar1x = rand() % 15;
  bar2x = rand() % 15;
  bar3x = rand() % 15;
  jumpx = bar1x + 2;
  jumpy = bar1y + 1;
  jumpState = 8;
  score = 0;

}

void gameLogic(void){
  switch (gameState) {
    case 0:
      lcd_clrscr();
      lcd_puts(" <Jumppy Pixel>");
      lcd_set(0, 1, 1, 0, 0, 0, 0, 0, 0);  // set the cursor to position 40(second line 1st char)
      lcd_puts("by Mark & Milad");
      gameState = 1;
      break;
    case 1: // wait user botton
      if(BSP_PB_GetState(BUTTON_USER) == 1){
        gameState = 2;
        lcd_clrscr();
        jumpInit();
        gameLCDsetUp(); // put 8 customized chars on the lcd
        jumpClearBuffer();
      }
      break;
    case 2:
      jumpParameterUpdate();
      jumpClearBuffer();
      jumpUpdateBuffer();
      bufferToLED();
      break;
    case 3:
      jumpEndingAnimation();
      jumpClearBuffer();
      jumpUpdateBuffer();
      bufferToLED();
      break;
    case 4:
      lcd_clrscr();
      lcd_puts(" <Jumppy Pixel>");
      char score_s[4];
      score_s[3] = 0;
      score_s[2] = (score % 10) + 48;
      score_s[1] = ((score / 10) % 10) + 48;
      score_s[0] = ((score / 100) % 10) + 48;
      lcd_set(0, 1, 1, 0, 0, 0, 0, 0, 0);  // set the cursor to position 40(second line 1st char)
      lcd_puts("   Score: ");
      lcd_puts(score_s);
      gameState = 5;
      break;
    case 5:
      if(BSP_PB_GetState(BUTTON_USER) == 1)
        gameState = 0;
      break;
  }
}

void jumpEndingAnimation(void){
  if((bar1y > 15) && (bar2y > 15) && (bar3y > 15)){
    gameState = 4;
    return;
  }
  bar1y ++;
  bar2y ++;
  bar3y ++;
}

void jumpParameterUpdate(void){
  // update jumpx
  if((Accel()) > 1500){
    if((Accel()) > 2500)
      jumpx ++;
    jumpx ++;
  }else if((Accel()) < -1500){
    if((Accel()) < -2500)
      jumpx --;
    jumpx --;
  }
  if (jumpx >= 20){
    jumpx = 0;
  }
  if (jumpx <= -1){
    jumpx = 19;
  }

  if(jumpState > 0){
    if(jumpy <= 9){
      jumpy ++;
    }else{
      bar1y --;
      bar2y --;
      bar3y --;
      if(bar1y < 0){
        bar1y = bar3y + 6;
        bar1x = rand() % 15;
        score ++;
      }
      if(bar2y < 0){
        bar2y = bar1y + 6;
        bar2x = rand() % 15;
        score ++;
      }
      if(bar3y < 0){
        bar3y = bar2y + 6;
        bar3x = rand() % 15;
        score ++;
      }
    }
    jumpState --;
  }else if(jumpState == 0){
    jumpState --;
  }else if(((jumpy == bar1y + 1) && (jumpx >= bar1x) && (jumpx <= bar1x + 4)) // falling
        || ((jumpy == bar2y + 1) && (jumpx >= bar2x) && (jumpx <= bar2x + 4))
        || ((jumpy == bar3y + 1) && (jumpx >= bar3x) && (jumpx <= bar3x + 4))){
    jumpState = 9;
    //jumpy ++;
  }else{
    if (jumpy <= 0){
      gameState = 3;
      return;
    }
    jumpy --;
  }
}

void jumpUpdateBuffer(void){
  if(gameState < 3){
    buffer[jumpy][jumpx] = 1;
  }
  int32_t i = 0;
  for (i = 0; i <= 4; i ++){
    if((bar1y <= 15) && (bar1y >= 0))
      buffer[bar1y][bar1x + i] = 1;

    if((bar2y <= 15) && (bar2y >= 0))
      buffer[bar2y][bar2x + i] = 1;

    if((bar3y <= 15) && (bar3y >= 0))
      buffer[bar3y][bar3x + i] = 1;
  }
}

void jumpClearBuffer(void){
  memset(buffer, 0, 320 * sizeof(GPIO_PinState));
}

int16_t Accel(void){
  int16_t xyz[3];
  BSP_ACCELERO_GetXYZ(xyz);
  return -xyz[0];
}

void gameLCDsetUp(void){
  HAL_Delay(1);
  lcd_set(0, 1, 0, 0, 0, 0, 1, 1, 0);  // set the cursor to position 6
  lcd_set(1, 0, 0, 0, 0, 0, 0, 0, 0);  // put costomized char 1 here
  lcd_set(1, 0, 0, 0, 0, 0, 0, 0, 1);  // put costomized char 2 here
  lcd_set(1, 0, 0, 0, 0, 0, 0, 1, 0);  // put costomized char 3 here
  lcd_set(1, 0, 0, 0, 0, 0, 0, 1, 1);  // put costomized char 4 here
  lcd_set(0, 1, 1, 0, 0, 0, 1, 1, 0);  // set the cursor to position 46 (second line 6th char)
  lcd_set(1, 0, 0, 0, 0, 0, 1, 0, 0);  // put costomized char 5 here
  lcd_set(1, 0, 0, 0, 0, 0, 1, 0, 1);  // put costomized char 6 here
  lcd_set(1, 0, 0, 0, 0, 0, 1, 1, 0);  // put costomized char 7 here
  lcd_set(1, 0, 0, 0, 0, 0, 1, 1, 1);  // put costomized char 8 here
}

void bufferToLED(void){
  lcd_set(0, 0, 1, 0, 0, 0, 0, 0, 0);  // set the CGRAM to the firest costomized char
  lcd_CostomizedChar(15, 0);
  lcd_CostomizedChar(15, 5);
  lcd_CostomizedChar(15, 10);
  lcd_CostomizedChar(15, 15);
  lcd_CostomizedChar(7, 0);
  lcd_CostomizedChar(7, 5);
  lcd_CostomizedChar(7, 10);
  lcd_CostomizedChar(7, 15);
}
void lcd_CostomizedChar(int32_t y, int32_t x){
  int32_t index = 0;
  for (index = y; index >= (y - 7); index --){
    lcd_set(1, 0, 0, 0, buffer[index][x], buffer[index][x + 1], buffer[index][x + 2], buffer[index][x + 3], buffer[index][x + 4]);
  }
}

void bufferTest(int mode){
  jumpClearBuffer();
  jumpInit();
  jumpUpdateBuffer();
  gameLCDsetUp();
  bufferToLED();
}
ADD_CMD("testbuffer", bufferTest, "bufferTest");

void logicTest(int mode){
  jumpParameterUpdate();
  jumpClearBuffer();
  jumpUpdateBuffer();
  bufferToLED();
}
ADD_CMD("logic",logicTest,"logicTest");
