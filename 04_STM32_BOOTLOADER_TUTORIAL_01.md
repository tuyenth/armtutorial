# XÂY DỰNG BOOTLOADER CHO ARM #

> Sử dụng [STM32\_BARE\_METAL\_PROJECT](https://code.google.com/p/armtutorial/wiki/03_BARE_METAL_PROJECT) thay đổi một chút ít ta có thể tạo ra được một chương trình bootloader đơn giản.

> Trên bộ nhớ FLASH của STM32, bootloader được thiết kế với phần code không vượt quá 64K bắt đầu ở vi trí 0x08000000. Kế đến là chương trình ứng dụng, ta định nghĩa các tham số như sau:

```
    #define FLASH_BASE           0x08000000                         /* Vùng cho bootloader */ 
    #define BOOTLOADER_MAX_SIZE  0x00010000                         /* 64K                 */
    #define IMAGE_ADDR           (FLASH_BASE + BOOTLOADER_MAX_SIZE) /* Vùng ứng dụng       */              
```

# CHƯƠNG TRÌNH BOOTLOADER #

> ## 1> gcc\_arm.ld : ##

```

   PROVIDE(__image_addr = 0x08010000);

   MEMORY
   {
     FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 64K
     RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 16K
   }

   ENTRY(Reset_Handler)

   SECTIONS
   {
      .text :
      {
	KEEP(*(.isr_vector))
        *(.text*)

	KEEP(*(.init))
	KEEP(*(.fini))
        ...  
   }
```

> Linker script của bootloader tương tự với [STM32\_BARE\_METAL\_PROJECT](https://code.google.com/p/armtutorial/wiki/03_BARE_METAL_PROJECT), ở phần khai báo memory, nên giới hạn tham số LENGTH của FLASH để ta có thể kiểm soát được trường hợp code size của bootloader bị tràn và chồng lấn với vùng ứng dụng.

```
   PROVIDE(__image_addr = 0x08010000);
```

> Tạo một symbol có tên là `__image_addr` đánh dấu vị trí 0x08010000 cho vùng ứng dụng.

> ## 2> startup\_stm32f4xx.S ##

> Hoàn toàn tương tự với [STM32\_BARE\_METAL\_PROJECT](https://code.google.com/p/armtutorial/wiki/03_BARE_METAL_PROJECT)

> ## 3> system\_stm32f4xx.c ##

> Hoàn toàn tương tự với [STM32\_BARE\_METAL\_PROJECT](https://code.google.com/p/armtutorial/wiki/03_BARE_METAL_PROJECT)

> ## 4> main.c ##

> Chương trình chính, bootloader có thể thực hiện một số thao tác trước khi nhảy đến ứng dụng, ví dụ như kiểm tra điều kiện update firmware có thỏa mãn hay không:

> + Nếu không thỏa mãn điều kiện update (ứng dụng mới nhất): bootloader sẽ nhảy đến thực thi chương trình ứng dụng.

> + Nếu thỏa mãn điều kiện update : Sau khi được sự đồng ý của người dùng, bootloader sẽ load firmware (binary, hex files...) từ các thiết bị nhớ, SD card, SPI Nor Flash, thông qua UART, Ethernet, CAN... Sau đó lập trình firmware vào FLASH và tiến hành chạy ứng dụng.

> Ở demo đơn giản này bootloader sẽ nhảy thẳng đến chương trình ứng dụng, với mục đích kiểm chứng việc khởi tạo các thành phần trên STM32F4 có thành công hay không.

> Để chương trình nhảy đến một vị trí trên vùng nhớ có mã thực thi, ta có thể sử dụng đoạn code với cú pháp sau:


```
   int main(void) 
   {
     /* Initialize LEDs mounted on EVAL board */
     STM_EVAL_LEDInit(LED3);
     STM_EVAL_LEDInit(LED4);

     STM_EVAL_LEDOff(LED3);
     STM_EVAL_LEDOff(LED4);

     /* Jump to application */
     asm( "ldr   r1, =(__image_addr + 4)\n\t"
          "ldr   r0, [r1]\n\t"
          "bx    r0" );

     return 0;
   }
```

> Một điểm cần lưu ý rằng, giá trị tại ô nhớ tại vị trí `__image_addr` không phải là entry point(`Reset_Handler`) của ứng dụng, nó thật sự là giá trị con trỏ stack pointer (SP). Giá trị ô nhớ tại vị trí `(__image_addr + 4)` mới đúng là vị trí của hàm `Reset_Handler`. Việc nhảy đến hàm này có thể xem như sự kiện reset giả lập sự kiện reset theo cách nhìn của chương trình ứng dụng.

# CHƯƠNG TRÌNH ỨNG DỤNG #

> ## 1> gcc\_arm.ld : ##

```

   PROVIDE(__image_addr = 0x08010000);

   MEMORY
   {
      FLASH (rx) : ORIGIN = 0x08010000, LENGTH = (2M - 64K)
      RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 16K
   }

   ENTRY(Reset_Handler)

   SECTIONS
   {
      .text :
      {
	KEEP(*(.isr_vector))
        *(.text*)

	KEEP(*(.init))
	KEEP(*(.fini))
        ...  
   }
```

> Thông số FLASH được thay đổi:

> + ORIGIN = IMAGE\_ADDR

> + LENGTH = (2M - 64K)

> ## 2> startup\_stm32f4xx.S ##

> Hoàn toàn tương tự với [STM32\_BARE\_METAL\_PROJECT](https://code.google.com/p/armtutorial/wiki/03_BARE_METAL_PROJECT)

> ## 3> system\_stm32f4xx.c ##

> Lowlevel init, sau khi khởi động CPU clock, ngoại vi... Ta cần map interrupt vector table đến vị trí đầu của chương trình ứng dụng:

```
     /* Do remap interrupt vector */
     SCB->VTOR = IMAGE_ADDR;  
```

> ## 4> main.c ##

```
      /* Initialize LEDs mounted on EVAL board */
      STM_EVAL_LEDInit(LED3);
      STM_EVAL_LEDInit(LED4);

      /* Init LEDs */
      STM_EVAL_LEDOn(LED3);
      STM_EVAL_LEDOff(LED4);

      /* EXTI0 config */
      EXTILine0_Config();

      while(1);
```

# CHẠY CHƯƠNG TRÌNH #

> - Board demo: STM32F429 DISCO

> - Biên dịch và load chương trình "STM32F4\_Bootloader\_01" (Sử dụng công cụ debug của emBlocks)

> - Biên dịch và load chương trình "STM32F4\_Application\_01"

> Sau khi nhấn nút reset, chương trình bootloader chạy, hàm main khởi động và tắt 2 đèn LED3 (green) và LED4 (red), sau đó nhảy đến chương trình ứng dụng. Hàm main của chương trình ứng dụng bật LED3 và tắt LED4, đồng thời khởi tạo trình phục vụ ngắt ngoài EXTI0 (User Button).

> Như vậy:

> - Nếu bootloader khởi động được chương trình ứng dụng, thì đèn LED3 (green) sẽ sáng.

> - Nếu việc remap interrupt vector, khởi tạo ngắt đúng, khi ấn và nhả nút User Button thì đèn LED4 (red) sẽ đổi trạng thái (toggle).

> - Đây là demo đơn giản, bootloader hoàn chỉnh cần phải có tính năng load và program firmware vào FLASH cho STM32, hy vọng chúng ta sẽ có cơ hội phát triển thêm trong thời gian tới.
> > ![http://armtutorial.googlecode.com/svn/trunk/image/STM32F429DISCO.png](http://armtutorial.googlecode.com/svn/trunk/image/STM32F429DISCO.png)

# DOWNLOAD CHƯƠNG TRÌNH DEMO #


> [STF32F4\_Bootloader\_Tutorial\_01.rar](http://armtutorial.googlecode.com/svn/trunk/src/STF32F4_Bootloader_Tutorial_01.rar)
