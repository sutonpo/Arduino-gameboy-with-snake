# Arduino-gameboy-with-snake

這是一個使用 **Arduino 和 LCD12864（ST7920 控制器） 的專案。

## 🔧 使用元件

- Arduino UNO
- LCD12864 (SPI 模式)
- 10k Potentiometer
- Breadboard and jump wires

🖼️ 專案展示

![LCD 顯示畫面](images/demo.jpg)


## 📜 功能說明

- 顯示貪食蛇遊戲畫面
- 支援遊戲選單
- 調整亮度 via potentiometer

## 🧠 使用的函式庫

- `U8g2` - 用於簡單顯示圖文資訊（支援 ST7920 控制器）
- `SPI.h`

## 🚀 開始使用

### 1. 環境準備

- 安裝 Arduino IDE
- 安裝 U8g2 函式庫（管理員中搜尋安裝）

### 2. 燒錄範例程式碼

將以下程式碼燒錄至 Arduino UNO：

```cpp
#include <U8g2lib.h>  // LCD Library : https://github.com/olikraus/U8g2_Arduino
// On Arduino IDE, click Sketch/Include library/Manage libraries/ then type Freertos and install Richard Barry's library
#include <Arduino_FreeRTOS.h> 
#include <SPI.h>
#define RIGHT 4 // Pin 2 connect to RIGHT button
#define LEFT 5  // Pin 3 connect to LEFT button
#define UP 6    // Pin 4 connect to UP button
#define DOWN 7  // Pin 5 connect to DOWN button
#define MAX_LENGTH 100  // Max snake's length, need less than 89 because arduino don't have more memory
void TaskDisplayLCD( void *pvParameters );
void TaskHandleButton( void *pvParameters );
TaskHandle_t TaskHandle_1;
TaskHandle_t TaskHandle_2;
static const short joystickVCC = 3; // virtual VCC for the joystick (Analog 1) (to make the joystick connectable right next to the arduino nano)
static const short joystickGND = 4; // virtual GND for the joystick (Analog 0) (to make the joystick connectable right next to the arduino nano)

#define JOYSTICK_X A1
#define JOYSTICK_Y A0
#define JOYSTICK_SW 2  // Assuming SW is connected to digital pin 8
U8G2_ST7920_128X64_1_SW_SPI u8g2(U8G2_R0, /* clock=*/ 13, /* data=*/ 11, /* CS=*/ 10, /* reset=*/ 9);
//diameter of the gameItemSize
unsigned int gameItemSize = 4;
volatile unsigned int snakeSize = 4;
volatile unsigned int snakeDir = 1;
volatile int SPEED = 1; // Input from button
int initxValue = analogRead(JOYSTICK_X);
int inityValue = analogRead(JOYSTICK_Y);
struct gameItem {
  volatile unsigned int X; // x position
  volatile unsigned int Y;  //y position
};

//array to store all snake body part positions
gameItem snake[MAX_LENGTH];

//snake food item
gameItem snakeFood;

void get_key(){
  int xValue = analogRead(JOYSTICK_X);
  int yValue = analogRead(JOYSTICK_Y);
  //Serial.println(xValue);
  //Serial.println(yValue);
  if (xValue < 50 && snakeDir != 1) {
    snakeDir = 1;  // left
    //Serial.println(xValue);
    Serial.println("Right");
  } else if (xValue > 700 && snakeDir != 0) {
    snakeDir = 0;  // right
    //Serial.println(xValue);
    Serial.println("Left");
  } else if (yValue < 300 && snakeDir != 3) {
    snakeDir = 2;  // down
    Serial.println("Down");
  } else if (yValue > 700 && snakeDir != 2) {
    snakeDir = 3;  // up
    Serial.println("Up");
  }
}

void drawGameOver() {
  char _point[]= {' ',' ',' ','\n'}; // put '1' instead of '0' , it will ignore :v
  snakeSize -= 5;
  if (snakeSize < 10)
  {
    _point[2] = snakeSize + '0';
  }
  else if(snakeSize >= 10 && snakeSize < 100)
  {
    _point[1] = (snakeSize / 10) + '0';
    _point[2] = (snakeSize % 10) + '0';
  }
  else if (snakeSize >= 100)
  {
    _point[2] = (snakeSize % 10) + '0';
    snakeSize /= 10;
    _point[0] = (snakeSize / 10) + '0';
    _point[1] = (snakeSize % 10) + '0';
  }
  while(1)
  {
     u8g2.firstPage();
    do {
      u8g2.drawStr(30,20,"-<<POINT>>-");//y,x
      u8g2.drawStr(50,40,_point);//y,x
    }   while (u8g2.nextPage() );
  }
}

void drawSnake() {
  for (unsigned int i = 0; i < snakeSize; i++) {
    u8g2.drawFrame(snake[i].X, snake[i].Y, gameItemSize, gameItemSize);
  }
}

void drawFood() {
  u8g2.drawBox(snakeFood.X, snakeFood.Y, gameItemSize, gameItemSize);
}

void spawnSnakeFood() {
  //generate snake Food position and avoid generate on position of snake
  unsigned int i = 1;
  do {
    snakeFood.X = random(2, 126);
    while(snake[i].X == snakeFood.X || i != snakeSize)
    {
      snakeFood.X = random(2, 126);
      i++;
    }    
  } while (snakeFood.X % 4 != 0);
  i = 1;
  do {
    snakeFood.Y = random(2, 62);
    while(snake[i].Y == snakeFood.Y || i != snakeSize)
    {
      snakeFood.Y = random(2, 62);
      i++;
    }    
  } while (snakeFood.Y % 4 != 0);
}

void handleColisions() {
  //check if snake eats food
  if (snake[0].X == snakeFood.X && snake[0].Y == snakeFood.Y) {
    //increase snakeSize
    snakeSize++;
    //regen food
    spawnSnakeFood();
  }

  //check if snake collides with itself
  else {
    for (unsigned int i = 1; i < snakeSize; i++) {
      if (snake[0].X == snake[i].X && snake[0].Y == snake[i].Y) {
        drawGameOver();
      }
    }
  }
  //check for wall collisions
  if ((snake[0].X < 0) || (snake[0].Y < 0) || (snake[0].X > 124) || (snake[0].Y > 60)) {
    drawGameOver();
  }
}

void updateValues() {
  //update all body parts of the snake excpet the head
  unsigned int i;
  for (i = snakeSize - 1; i > 0; i--)
    snake[i] = snake[i - 1];

  //Now update the head
  //move left
  if (snakeDir == 0)
    snake[0].X -= gameItemSize;
    
  //move right
  else if (snakeDir == 1)
    snake[0].X += gameItemSize;

  //move down
  else if (snakeDir == 2)
    snake[0].Y += gameItemSize;

  //move up
  else if (snakeDir == 3) 
    snake[0].Y -= gameItemSize;
}

void playGame() {
  handleColisions();
  updateValues();
  u8g2.firstPage();
  do {
    drawSnake();
    drawFood();
    delay(1);
  } while (u8g2.nextPage());
}

void setup() {
  //Set 4 button pins
  Serial.begin(9600);
  pinMode(LEFT, INPUT_PULLUP);
  pinMode(RIGHT, INPUT_PULLUP);
  pinMode(DOWN, INPUT_PULLUP);
  pinMode(UP, INPUT_PULLUP);
  pinMode(JOYSTICK_X, INPUT);
  pinMode(JOYSTICK_Y, INPUT);
  pinMode(joystickVCC, OUTPUT);
	digitalWrite(joystickVCC, HIGH);

	pinMode(joystickGND, OUTPUT);
	digitalWrite(joystickGND, LOW);
  pinMode(JOYSTICK_SW, INPUT_PULLUP);
  u8g2.begin(); 
  u8g2.setFont(u8g2_font_ncenB10_tr);
  volatile char ModeSelection[] = {SPEED+'0','\n'};
  while(analogRead(JOYSTICK_SW) == 0)
  {
    if(analogRead(JOYSTICK_X)<57)
    {
      while(analogRead(JOYSTICK_SW)==0);
      if(SPEED < 4)
        {
          SPEED++;
          ModeSelection[0] = SPEED + '0';
        }
    }
    else if(analogRead(JOYSTICK_Y)>700)
    {
      while(analogRead(JOYSTICK_SW)==0);
      if(SPEED > 1 )
          {
            SPEED--;
            ModeSelection[0] = SPEED + '0';
          }
    }
     
    u8g2.firstPage();
    do {
      u8g2.drawStr(17,20,"Speed Mode");//y,x
      u8g2.drawStr(62,40,ModeSelection);//y,x
    }while ( u8g2.nextPage() );
    
    
    delay(50);
  }

  SPEED *= 50;
  randomSeed(analogRead(0)); // Change random number base on analog noise
  xTaskCreate(TaskDisplayLCD, "Task1", 100, NULL, 1, &TaskHandle_1);
  xTaskCreate(TaskHandleButton, "Task2", 100, NULL, 1, &TaskHandle_2);
  Serial.println(digitalRead(UP));
}

void loop() {


}

void TaskDisplayLCD(void *pvParameters)  // This is a task.
{
  (void) pvParameters;

  for (;;) // A Task shall never return or exit.
  {
    playGame();
    vTaskDelay( SPEED / portTICK_PERIOD_MS ); // load each 200ms
  }
}

void TaskHandleButton(void *pvParameters)  // This is a task.
{
  (void) pvParameters;

  for (;;) // A Task shall never return or exit.
  {
    get_key();
    vTaskDelay(100/ portTICK_PERIOD_MS ); //load each 100ms
  }
}
