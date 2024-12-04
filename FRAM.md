#  第一題
![image](https://hackmd.io/_uploads/rJpQxqDz1l.png)
![image](https://hackmd.io/_uploads/B1QNxqDMJx.png)
![image](https://hackmd.io/_uploads/BJYNl5PGyl.png)
![image](https://hackmd.io/_uploads/H1zUxcwM1e.png)
`make`後產生`hello.ko`
![image](https://hackmd.io/_uploads/S1bql9vMkl.png)
```insmod hello.ko```後使用```dmesg```查看會印出Hello, world!

```clike=
static int __init hello_init(void) {
    printk(KERN_INFO "Hello, world!\n");
    return 0;
}
```
![image](https://hackmd.io/_uploads/r1G2bcwGJl.png)

```rmmod hello```卸載模組後會印出Goodbye, world!
```clike=
static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye, world!\n");
}
```
![image](https://hackmd.io/_uploads/H1gqb5vfJx.png)

`hello.c`內容為
![image](https://hackmd.io/_uploads/BJaaW9wfye.png)

`Makefile`為
![image](https://hackmd.io/_uploads/B1NyG9Pzkg.png)
其中obj-**m** += hello.o：
•	這表示將 ```hello.o``` 編譯為可載入的模組（Loadable Kernel Module）。
•	使用 obj-m 編譯的模組可以在需要時使用 ```insmod``` 或 ```modprobe``` 載入，並且可以用 ```rmmod``` 卸載。
•	編譯完成後，生成的檔案是 ```hello.ko```，這是一個可載入的模組檔案。
•	適用於需要動態載入和卸載的模組。

![image](https://hackmd.io/_uploads/HyO0V9Dzyg.png)


把```hello.c```複製到```drivers/misc```這個目錄
![image](https://hackmd.io/_uploads/rJm_G9wzJl.png)
修改```drivers/misc```中的```Makefile```
![image](https://hackmd.io/_uploads/BJiOfqDGye.png)
添加```hello.o```在obj-y這行，我們使用 obj-**y** 而非 obj-m，這會讓module成為kernel的一部分
![image](https://hackmd.io/_uploads/SJqcf5vM1l.png)
接著編譯kernel
![image](https://hackmd.io/_uploads/rk1x7cwMye.png)
reboot後選擇最新的version![image](https://hackmd.io/_uploads/rywb75wGJg.png)
使用```dmesg```查看訊息，有印出Hello, world!, 確實module已成功作為kernel的一部分被編譯進去，這是module loads automatically at boot的情況。
![image](https://hackmd.io/_uploads/Sk34XqPz1x.png)




另一種情況我們可以使用via manual insertion (```modprobe```)，手動載入module
![image](https://hackmd.io/_uploads/S1WOmqvfyg.png)

```modprobe``` 會自動查找 `/lib/modules/$(uname -r)/` 目錄下的模組，並加載module的所有依賴。`dmesg`中查看訊息確實有Hello, world!
前面多加個 `-r`表示卸載hello這個module，`dmesg`中查看訊息確實有Goodbye, world!

特別注意兩點:
* 	注意到當重啟系統後， hello 模組是作為可載入模組（而不是內建模組）編譯的，它將不會自動載入，因此不會顯示 Hello, world! 或 Goodbye, world! 的訊息。

* 	注意到obj-y 將模組編譯為核心的一部分時，模組就成為了內建模組（built-in module），而不是可載入模組（loadable module）。內建模組是靜態編譯進kernel的，隨著kernel一起啟動，並且<font color="yellow">**無法動態卸載**</font>。這也是為什麼使用 ```sudo modprobe -r hello```不能卸載模組的原因
![image](https://hackmd.io/_uploads/BJhKQcDfJe.png)









#  第二題

使用CCS Theia 1.5.1作為IDE

import OutOfBox_MSP430FR4133 workspace

![image](https://hackmd.io/_uploads/SkRbirmMkg.png)


![image](https://hackmd.io/_uploads/rySOcSQM1l.png)
`TempSensorMode.c`中`tempSensor()`可達到此效果
```c=
void tempSensor() {
  // Initialize the ADC Module
  /*
   * Base Address for the ADC Module
   * Use Timer trigger 1 as sample/hold signal to start conversion
   * USE MODOSC 5MHZ Digital Oscillator as clock source
   * Use default clock divider of 1
   */
  ADC_init(ADC_BASE, ADC_SAMPLEHOLDSOURCE_2, ADC_CLOCKSOURCE_ADCOSC,
           ADC_CLOCKDIVIDER_1);

  ADC_enable(ADC_BASE);

  // Configure Memory Buffer
  /*
   * Base Address for the ADC Module
   * Use input A12 Temp Sensor
   * Use positive reference of Internally generated Vref
   * Use negative reference of AVss
   */
  ADC_configureMemory(ADC_BASE, ADC_INPUT_TEMPSENSOR, ADC_VREFPOS_INT,
                      ADC_VREFNEG_AVSS);

  ADC_clearInterrupt(ADC_BASE, ADC_COMPLETED_INTERRUPT);

  // Enable the Memory Buffer Interrupt
  ADC_enableInterrupt(ADC_BASE, ADC_COMPLETED_INTERRUPT);

  ADC_startConversion(ADC_BASE, ADC_REPEATED_SINGLECHANNEL);

  // Enable internal reference and temperature sensor
  PMM_enableInternalReference();
  PMM_enableTempSensor();

  // TimerA1.1 (125ms ON-period) - ADC conversion trigger signal
  Timer_A_initUpMode(TIMER_A1_BASE, &initUpParam_A1);

  // Initialize compare mode to generate PWM1
  Timer_A_initCompareMode(TIMER_A1_BASE, &initCompParam);

  // Start timer A1 in up mode
  Timer_A_startCounter(TIMER_A1_BASE, TIMER_A_UP_MODE);

  // Delay for reference settling
  __delay_cycles(300000);

  // Enter LPM3.5 mode with interrupts enabled
  while (*tempSensorRunning) {
    __bis_SR_register(LPM3_bits | GIE); // LPM3 with interrupts enabled
    __no_operation();                   // Only for debugger

    if (*tempSensorRunning) {
      // Calculate Temperature in degree C and F
      signed short temp = (ADCMEM0 - CALADC_15V_30C);
      *degC = ((long)temp * 10 * (85 - 30) * 10) /
                  ((CALADC_15V_85C - CALADC_15V_30C) * 10) +
              300;
      *degF = (*degC) * 9 / 5 + 320;
      displayTemp();
    }
  }

  // Loop in LPM3 to while buttons are held down and debounce timer is running
  while (TA0CTL & MC__UP) {
    __bis_SR_register(LPM3_bits | GIE); // Enter LPM3
    __no_operation();
  }

  if (*mode == TEMPSENSOR_MODE) {
    // Disable ADC, TimerA1, Internal Ref and Temp used by TempSensor Mode
    ADC_disableConversions(ADC_BASE, ADC_COMPLETECONVERSION);
    ADC_disable(ADC_BASE);

    Timer_A_stop(TIMER_A1_BASE);

    PMM_disableInternalReference();
    PMM_disableTempSensor();
    PMM_turnOffRegulator();

    __bis_SR_register(LPM4_bits | GIE); // re-enter LPM3.5
    __no_operation();
  }
}
```

關鍵在52~59行，顯示了讀取的溫度如何轉換成華氏攝氏的溫度

其中`displayTemp()`函式

```c=
void displayTemp() {
  clearLCD();

  // Pick C or F depending on tempUnit state
  int deg;
  if (*tempUnit == 0) {
    showChar('C', pos6);
    deg = *degC;
  } else {
    showChar('F', pos6);
    deg = *degF;
  }

  // Handle negative values
  if (deg < 0) {
    deg *= -1;
    // Negative sign
    LCDMEM[pos1 + 1] |= 0x04;
  }

  // Handles displaying up to 999.9 degrees
  if (deg >= 1000)
    showChar((deg / 1000) % 10 + '0', pos2);
  if (deg >= 100)
    showChar((deg / 100) % 10 + '0', pos3);
  if (deg >= 10)
    showChar((deg / 10) % 10 + '0', pos4);
  if (deg >= 1)
    showChar((deg / 1) % 10 + '0', pos5);

  // Decimal point
  LCDMEM[pos4 + 1] |= 0x01;

  // Degree symbol
  LCDMEM[pos5 + 1] |= 0x04;
}
```
可在LCD上顯示華氏和攝氏溫度，透過S2 button做切換

![image](https://hackmd.io/_uploads/HkAZjFvfJg.png)
在`TempSensorMode.c`新增兩個用來寫入FRAM的函式
```c=
void writeCelsiusToFRAM(unsigned long temperature) {
  SYSCFG0 &= ~DFWP; // Disable FRAM write protection
  *(unsigned long *)FRAM_ADDRESS = temperature;
  SYSCFG0 |= DFWP; // Enable FRAM write protection
}
void writeFahrenheitToFRAM(unsigned long temperature) {
  SYSCFG0 &= ~DFWP; // Disable FRAM write protection
  *(unsigned long *)FRAM_ADDRESS = temperature;
  SYSCFG0 |= DFWP; // Enable FRAM write protection
}
```
其中FRAM_ADDRESS為使用者自定義的一個位址
```c
#define FRAM_ADDRESS 0x1900
```
特別注意要寫入之前要關掉寫入保護，寫好再開啟寫入保護

`displayTemp()`會多兩行

```c=
void displayTemp() {
  clearLCD();

  // Pick C or F depending on tempUnit state
  int deg;
  if (*tempUnit == 0) {
    showChar('C', pos6);
    deg = *degC;
    writeCelsiusToFRAM(*degC);

  } else {
    showChar('F', pos6);
    deg = *degF;
    writeFahrenheitToFRAM(*degF);
  }

  ........
}
```

![image](https://hackmd.io/_uploads/BkME6KDf1x.png)
測試斷電後資料依然存在FRAM中

換一個FRAM_ADDRESS，斷電後再看看原本FRAM_ADDRESS的資料是否存在，附檔影片中可以看到原本的資料依然存在，不會因為斷電而消失。

![image](https://hackmd.io/_uploads/rJKXRKvzye.png)

設定華氏跟攝氏的閥值
```clike=
int threshold_Celsius    = 280;
int threshold_Fahrenheit = 830;
```
接著新增兩個函式，代表超過此閥值會亮紅燈，否則不會亮燈
```c=
void chkCelsius(int temperature) {

  if (temperature > threshold_Celsius) {
    P1OUT |= BIT0; // 亮燈
  } else {
    P1OUT &= ~BIT0; // 關燈
  }
}
void chkFahrenheit(int temperature) {

  if (temperature > threshold_Fahrenheit) {
    P1OUT |= BIT0; // 亮燈
  } else {
    P1OUT &= ~BIT0; // 關燈
  }
}
```
最終的`displayTemp()`的長相為
```c=
void displayTemp() {
  clearLCD();

  // Pick C or F depending on tempUnit state
  int deg;
  if (*tempUnit == 0) {
    showChar('C', pos6);
    deg = *degC;
    writeCelsiusToFRAM(*degC);
    chkCelsius(*degC);

  } else {
    showChar('F', pos6);
    deg = *degF;
    writeFahrenheitToFRAM(*degF);
    chkFahrenheit(*degF);
  }
  ......
}

```

