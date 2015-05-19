# STM32F4 Bare Metal Project #
> Chương trình được khởi tạo từ đầu người ta gọi là bare-metal project, một chương trình standalone cho ARM tối thiểu phải có 2 thành phần:

```
   1> Source code.
   2> Linker script.
```

> Nếu chỉ có source code, trình biên dịch chỉ có thể tạo ra được các file đối tượng (object), thông thường có đuôi .o , để tạo thành file chạy (elf), trình biên dịch sẽ link các file đối tượng này và map chúng vào vùng nhớ vật lý để ARM có thể chạy được. Thông tin phân bố vùng nhớ của ARM được mô tả bởi linker script.

> Ví dụ ta có chương trình cộng 2 số (add.S) :

```
        .text                     @ Code section
     start:                       @ Label, not really required
        mov   r0, #5              @ Load register r0 with the value 5
        mov   r1, #4              @ Load register r1 with the value 4
        add   r2, r1, r0          @ Add r0 and r1 and store in r2

     stop:   b stop               @ Infinite loop to stop execution
```

> Build source :

```
  $ arm-none-eabi-as -o add.o add.S
  $ arm-none-eabi-nm add.o
    00000000 t start
    0000000c t stop
```

> Ở file object, trình biên dịch sẽ gán tạm thời địa chỉ offset cho các symbol (bắc đầu từ 0x0), để tạo ra file chạy (elf) trình biên dịch cần phải link và map vị trí các symbol này vào bố cục bộ nhớ của ARM (memory layout).

> Ví dụ: Bộ nhớ của ARM được bắt đầu ở địa chỉ 0x08000000, ta dùng lệnh sau để sinh ra file chạy:

```
  $ arm-none-eabi-ld -Ttext=0x08000000 -o add.elf add.o
  $ arm-none-eabi-nm add.elf 
    08000000 t start
    0800000c t stop
```

> Option -Ttext=0x08000000 biểu thị section .text được bắt đầu ở địa chỉ 0x08000000, tương tự, có thể khai báo thông qua file linker script (script.ld) như sau:

```
    SECTIONS {
         . = 0x08000000;
         .text : { * (.text); }
    }
```

> Ký hiệu . được gọi là location pointer, với khai báo như trên, section .text được bắt đầu ở vị trí 0x08000000,  wildcard { **(.text); } biểu thị tập hợp tất cả section .text của các file đối tượng cần link đến.**

```
  $ arm-none-eabi-ld -T script.ld -o add.elf add.o
  $ arm-none-eabi-nm add.elf 
    08000000 t start
    0800000c t stop
```

# STM32 GNU Project #

> Một chương trình đơn giản cho STM32 bao gồm các thành phần sau:

```
    - startup_stm32f4xx.S : Startup code.
    - gcc_arm.ld          : STM32 Linker script.
    - system_stm32f4xx.c  : STM32 system init.
    - main.c              : Chương trình ứng dụng.   
```

> A> Startup code:

> Khi có sự kiện reset, trình phục vụ ngắt `Reset_Handler` sẽ được thực thi, `Reset_Handler` thực hiện các tác vụ sau:

> - Copy .data section từ FLASH sang RAM, .data section là phân đoạn chứa các biến được khởi động trị lúc khai báo (ví dụ: char c = 'a';), do RAM là bộ nhớ mất dữ liệu khi tắt nguồn điện vì vậy các trị khởi động trước này được lưu trữ vào bộ nhớ FLASH, ROM... `Reset_Handler` copy phân vùng này sang RAM trước khi thực thi các chương trình. Địa chỉ bắt đầu chứa phân vùng data trong FLASH được gọi là LMA (Load Memory Address), địa chỉ đầu tiên cần copy đến RAM được gọi là run-time address hoặc là VMA (Virtual Memory Address).

> - Khởi động các biến trên vùng .bss , vùng này nằm trên RAM đây là vùng chứa các biến không khởi động trị trong lúc khai báo (ví dụ : int d;), `Reset_Handler` thực thi việc xóa các biến này về zero. Đây là thao tác option, `Reset_Handler` có thể thự hiện hoặc không. Vì thế, chúng ta cần lưu ý không được hiểu nhầm khi một biến mới cấp phát có giá trị = zero là do trình biên dịch thực hiện. Một số trình biên dịch thông minh sẽ cho ra cảnh báo nếu ta vô tình lấy giá trị của một biến không được khởi động trước.

> - Khởi tạo con trỏ stack (SP).

> - Khởi tạp vùng heap (nếu chương trình có sử dụng thư viện cấp phát bộ nhớ động dynamic memory allocation)

> - Thực hiện khởi động hardware (low level init), bao gồm enable cache, CPU clock, peripheral... hoặc gọi đến 1 chương trình con khác để khởi động ngoại vi, ở ví dụ cho STM32, `Reset_Handler` gọi đến các hàm trong file system\_stm32f4xx.c.

> - Sau khi thực hiện xong các bước trên `Reset_Handler` nhảy đến hàm main.

> ## Phân đoạn stack: ##

```
       .section .stack
       .align 3
       #ifdef __STACK_SIZE
         .equ    Stack_Size, __STACK_SIZE
       #else
         .equ    Stack_Size, 0x400
       #endif
       .globl    __StackTop
       .globl    __StackLimit
       __StackLimit:
       .space    Stack_Size
       .size __StackLimit, . - __StackLimit
       __StackTop:
       .size __StackTop, . - __StackTop
```

> Vị trí bắt đầu của stack được biểu thị bởi symbol `__StackTop`, thông thường vùng stack được bố trí ở địa chỉ cao nhất của RAM. Khi gọi chương trình con, CPU thực hiện lưu dữ liệu tạm thời, con trỏ hiện tại của PC của chương trình vào stack, con trỏ stack SP sẽ giảm dần (PUSH) và tiến đến `__StackLimit`, khi thoát khỏi chương trình, các thông tin lưu trữ được lấy ra khỏi stack, con trỏ SP sẽ tăng dần và tiến về `__StackTop`. Hiện tượng tràn stack xảy ra khi con trỏ SP <= `__StackLimit` và chồng lấn lên các section khác trong RAM.

> `__StackTop` cũng là giá trị khởi động của SP khi reset hệ thống, cấu trúc ARM cho phép khởi động giá trị ban đầu SP bằng cách gán `__StackTop` vào giá trị của vector ngắt đầu tiên (ở địa chỉ 0x08000000 đối với STM32).

> ASM derective .space `Stack_Size` thực hiện cấp phát 1 vùng có kích thước `Stack_Size` cho phân đoạn stack.

> Khai báo trương ứng trong linker script : gcc\_arm.ld

```
     MEMORY
     {
       FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 2M
       RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 16K
     }

     ENTRY(Reset_Handler)
     
     ...

     .heap :
     {
	__end__ = .;
	end = __end__;
	*(.heap*)
	__HeapLimit = .;
     } > RAM

     /* .stack_dummy section doesn't contains any symbols. It is only
      * used for linker to calculate size of stack sections, and assign
      * values to stack symbols later */
     .stack_dummy :
     {
	*(.stack)
     } > RAM

     /* Set stack top to end of RAM, and stack limit move down by
      * size of stack_dummy section */
     __StackTop = ORIGIN(RAM) + LENGTH(RAM);
     __StackLimit = __StackTop - SIZEOF(.stack_dummy);
     PROVIDE(__stack = __StackTop);    
       
```

> `__StackTop` nằm ở vị trí cuối cùng của RAM, được gán vào địa chỉ 0x20004000

> ## Phân đoạn heap: ##

> Các cấp phát phân đoạn này tương tự như stack:

```
       .section .heap
       .align 3
       #ifdef __HEAP_SIZE
         .equ    Heap_Size, __HEAP_SIZE
       #else
         .equ    Heap_Size, 0xC00
       #endif
       .globl    __HeapBase
       .globl    __HeapLimit
       __HeapBase:
       .if    Heap_Size
       .space    Heap_Size
       .endif
       .size __HeapBase, . - __HeapBase
       __HeapLimit:
       .size __HeapLimit, . - __HeapLimit
```

> ## Phân đoạn .text ##

> Phân đoạn bao gồm mã nhị phân chạy chương trình (instruction code...) bao gồm các hằng số (phân đoạn .rodata), các biến có trị khởi động trước (.data) và một số phân đoạn (.ctors), (.dtors)...

```
    SECTIONS
    {
	.text :
	{
	   KEEP(*(.isr_vector))
	   *(.text*)

	   KEEP(*(.init))
	   KEEP(*(.fini))

	   /* .ctors */
	   *crtbegin.o(.ctors)
	   *crtbegin?.o(.ctors)
	   *(EXCLUDE_FILE(*crtend?.o *crtend.o) .ctors)
	   *(SORT(.ctors.*))
	   *(.ctors)

	   /* .dtors */
 	    *crtbegin.o(.dtors)
 	    *crtbegin?.o(.dtors)
 	    *(EXCLUDE_FILE(*crtend?.o *crtend.o) .dtors)
 	    *(SORT(.dtors.*))
 	    *(.dtors)

	    *(.rodata*)

	    KEEP(*(.eh_frame*))
	} > FLASH
       ... 
    }

```


> ## Phân đoạn vector table: ##

```
.section .isr_vector
    .align 2
    .globl __isr_vector
__isr_vector:
    .long    __StackTop                        /* Top of Stack */
    .long    Reset_Handler                     /* Reset Handler */
    .long    NMI_Handler                       /* NMI Handler */
    .long    HardFault_Handler                 /* Hard Fault Handler */
    .long    MemManage_Handler                 /* MPU Fault Handler */
    .long    BusFault_Handler                  /* Bus Fault Handler */
    .long    UsageFault_Handler                /* Usage Fault Handler */
    .long    0                                 /* Reserved */
    .long    0                                 /* Reserved */
    .long    0                                 /* Reserved */
    .long    0                                 /* Reserved */
    .long    SVC_Handler                       /* SVCall Handler */
    .long    DebugMon_Handler                  /* Debug Monitor Handler */
    .long    0                                 /* Reserved */
    .long    PendSV_Handler                    /* PendSV Handler */
    .long    SysTick_Handler                   /* SysTick Handler */

                                               // External Interrupts
    .long    WWDG_IRQHandler                   // Window WatchDog
    .long    PVD_IRQHandler                    // PVD through EXTI Line detection
    .long    TAMP_STAMP_IRQHandler             // Tamper and TimeStamps through the EXTI line
    .long    RTC_WKUP_IRQHandler               // RTC Wakeup through the EXTI line
    .long    FLASH_IRQHandler                  // FLASH
    .long    RCC_IRQHandler                    // RCC
    .long    EXTI0_IRQHandler                  // EXTI Line0
    .long    EXTI1_IRQHandler                  // EXTI Line1
    .long    EXTI2_IRQHandler                  // EXTI Line2
    .long    EXTI3_IRQHandler                  // EXTI Line3
    .long    EXTI4_IRQHandler                  // EXTI Line4
    .long    DMA1_Stream0_IRQHandler           // DMA1 Stream 0
    .long    DMA1_Stream1_IRQHandler           // DMA1 Stream 1
    .long    DMA1_Stream2_IRQHandler           // DMA1 Stream 2
    .long    DMA1_Stream3_IRQHandler           // DMA1 Stream 3
    .long    DMA1_Stream4_IRQHandler           // DMA1 Stream 4
    .long    DMA1_Stream5_IRQHandler           // DMA1 Stream 5
    .long    DMA1_Stream6_IRQHandler           // DMA1 Stream 6
    .long    ADC_IRQHandler                    // ADC1, ADC2 and ADC3s
    .long    CAN1_TX_IRQHandler                // CAN1 TX
    .long    CAN1_RX0_IRQHandler               // CAN1 RX0
    .long    CAN1_RX1_IRQHandler               // CAN1 RX1
    .long    CAN1_SCE_IRQHandler               // CAN1 SCE
    .long    EXTI9_5_IRQHandler                // External Line[9:5]s
    .long    TIM1_BRK_TIM9_IRQHandler          // TIM1 Break and TIM9
    .long    TIM1_UP_TIM10_IRQHandler          // TIM1 Update and TIM10
    .long    TIM1_TRG_COM_TIM11_IRQHandler     // TIM1 Trigger and Commutation and TIM11
    .long    TIM1_CC_IRQHandler                // TIM1 Capture Compare
    .long    TIM2_IRQHandler                   // TIM2
    .long    TIM3_IRQHandler                   // TIM3
    .long    TIM4_IRQHandler                   // TIM4
    .long    I2C1_EV_IRQHandler                // I2C1 Event
    .long    I2C1_ER_IRQHandler                // I2C1 Error
    .long    I2C2_EV_IRQHandler                // I2C2 Event
    .long    I2C2_ER_IRQHandler                // I2C2 Error
    .long    SPI1_IRQHandler                   // SPI1
    .long    SPI2_IRQHandler                   // SPI2
    .long    USART1_IRQHandler                 // USART1
    .long    USART2_IRQHandler                 // USART2
    .long    USART3_IRQHandler                 // USART3
    .long    EXTI15_10_IRQHandler              // External Line[15:10]s
    .long    RTC_Alarm_IRQHandler              // RTC Alarm (A and B) through EXTI Line
    .long    OTG_FS_WKUP_IRQHandler            // USB OTG FS Wakeup through EXTI line
    .long    TIM8_BRK_TIM12_IRQHandler         // TIM8 Break and TIM12
    .long    TIM8_UP_TIM13_IRQHandler          // TIM8 Update and TIM13
    .long    TIM8_TRG_COM_TIM14_IRQHandler     // TIM8 Trigger and Commutation and TIM14
    .long    TIM8_CC_IRQHandler                // TIM8 Capture Compare
    .long    DMA1_Stream7_IRQHandler           // DMA1 Stream7
    .long    FSMC_IRQHandler                   // FSMC
    .long    SDIO_IRQHandler                   // SDIO
    .long    TIM5_IRQHandler                   // TIM5
    .long    SPI3_IRQHandler                   // SPI3
    .long    UART4_IRQHandler                  // UART4
    .long    UART5_IRQHandler                  // UART5
    .long    TIM6_DAC_IRQHandler               // TIM6 and DAC1&2 underrun errors
    .long    TIM7_IRQHandler                   // TIM7
    .long    DMA2_Stream0_IRQHandler           // DMA2 Stream 0
    .long    DMA2_Stream1_IRQHandler           // DMA2 Stream 1
    .long    DMA2_Stream2_IRQHandler           // DMA2 Stream 2
    .long    DMA2_Stream3_IRQHandler           // DMA2 Stream 3
    .long    DMA2_Stream4_IRQHandler           // DMA2 Stream 4
    .long    ETH_IRQHandler                    // Ethernet
    .long    ETH_WKUP_IRQHandler               // Ethernet Wakeup through EXTI line
    .long    CAN2_TX_IRQHandler                // CAN2 TX
    .long    CAN2_RX0_IRQHandler               // CAN2 RX0
    .long    CAN2_RX1_IRQHandler               // CAN2 RX1
    .long    CAN2_SCE_IRQHandler               // CAN2 SCE
    .long    OTG_FS_IRQHandler                 // USB OTG FS
    .long    DMA2_Stream5_IRQHandler           // DMA2 Stream 5
    .long    DMA2_Stream6_IRQHandler           // DMA2 Stream 6
    .long    DMA2_Stream7_IRQHandler           // DMA2 Stream 7
    .long    USART6_IRQHandler                 // USART6
    .long    I2C3_EV_IRQHandler                // I2C3 event
    .long    I2C3_ER_IRQHandler                // I2C3 error
    .long    OTG_HS_EP1_OUT_IRQHandler         // USB OTG HS End Point 1 Out
    .long    OTG_HS_EP1_IN_IRQHandler          // USB OTG HS End Point 1 In
    .long    OTG_HS_WKUP_IRQHandler            // USB OTG HS Wakeup through EXTI
    .long    OTG_HS_IRQHandler                 // USB OTG HS
    .long    DCMI_IRQHandler                   // DCMI
    .long    CRYP_IRQHandler                   // CRYP crypto
    .long    HASH_RNG_IRQHandler               // Hash and Rng
    .long    FPU_IRQHandler                    // FPU

    .size    __isr_vector, . - __isr_vector
```

> Vector table là một bảng chứa các địa chỉ của các trình phục vụ ngắt, khi có sự kiện ngắt (exception), CPU tự động nhảy đến các trình phục vụ ngắt để thực thi các công việc cụ thể. STM32 mặc định map vector table ở vị trí đầu của FLASH 0x08000000 (alias 0x00000000), người dùng có thể remap vector table này đến vị trí khác thông qua thanh ghi VTOR:

```
    #ifdef VECT_TAB_SRAM
        SCB->VTOR = SRAM_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal SRAM */
    #else
        SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET; /* Vector Table Relocation in Internal FLASH */
    #endif
```

> Trong đó VECT\_TAB\_OFFSET (bội số của 0x200) là giá trị offset mà chúng ta cần di chuyển vector table đến.

> Ở khai báo phân đoạn .isr\_vector trên, mỗi vector có giá trị được gán bởi địa chỉ hàm phục vụ ngắt được gọi đến, trình biên dịch cho phép ta tạm thời định nghĩa các hàm này (dành chỗ trước) với thuộc tính weak, khi người dùng muốn định nghĩa lại các hàm này, có thể thực thi ở các source .c, trình biên dịch sẽ tự động thay thế nội dung các hàm tạm thời bởi các hàm của người dùng. Các hàm tạm thời được khai báo với cú pháp sau:

```
    /* Macro to define default handlers. Default handler
     * will be weak symbol and just dead loops. They can be
     * overwritten by other handlers */
    .macro    def_irq_handler    handler_name
    .align 1
    .thumb_func
    .weak    \handler_name
    .type    \handler_name, %function
    \handler_name :
     b    .
    .size    \handler_name, . - \handler_name
    .endm

    def_irq_handler    NMI_Handler
    def_irq_handler    HardFault_Handler
    def_irq_handler    MemManage_Handler
    .... 
```

> Macro định nghĩa các hàm tạm thời với thuộc tính weak, khi có sự kiện ngắt, nếu người dùng ko định nghĩa hàm thay thế thì khi có sự kiện ngắt CPU sẽ bị treo bởi lệnh nhảy loop tại chỗ : b .

> Phân đoạn .isr\_vector được layout ở phần đầu của FLASH (tại vị trí VTOR chỉ định), cách khai báo linker script gcc\_arm.ld như sau:

```
    MEMORY
    {
      FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 2M
      RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 16K
    }
    
    ENTRY(Reset_Handler)

    SECTIONS
    {
	.text :
	{
	    KEEP(*(.isr_vector))
	    *(.text*)
            ... 
        } > FLASH
       ... 
    }
   
```

> Sau đây là nội dung của file .map sau khi biên dịch thành công :

```
      .text          0x08000000      0x9d8
      *(.isr_vector)
      .isr_vector    0x08000000      0x188 obj\debug\startup_stm32f4xx.o
                     0x08000000                __isr_vector
```

> Phân đoạn .text có kích thước 0x9d8, và phân đoạn .isr\_sector được bắt đầu ở địa chỉ 0x08000000.

> ## Phân đoạn .data ##

> Phân đoạn .data bắt đầu từ cuối phân đoạn .text (LMA), và nằm trong vùng RAM lúc chạy chương trình (VMA), cú pháp AT biểu thị trường hợp LMA khác với VMA khi chạy chương trình.


```
    MEMORY
    {
      FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 2M
      RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 16K
    }
    
    ENTRY(Reset_Handler)

    SECTIONS
    {
	.text :
	{
	    KEEP(*(.isr_vector))
	    *(.text*)
            ... 
        } > FLASH
       ...

       	__etext = .;

        .data : AT (__etext)
	{
	    __data_start__ = .;
	    *(vtable)
	    *(.data*)

	    . = ALIGN(4);
	    /* preinit data */
	    PROVIDE_HIDDEN (__preinit_array_start = .);
	    KEEP(*(.preinit_array))
	    PROVIDE_HIDDEN (__preinit_array_end = .);

	    . = ALIGN(4);
	    /* init data */
	    PROVIDE_HIDDEN (__init_array_start = .);
	    KEEP(*(SORT(.init_array.*)))
	    KEEP(*(.init_array))
	    PROVIDE_HIDDEN (__init_array_end = .);


	    . = ALIGN(4);
	    /* finit data */
	    PROVIDE_HIDDEN (__fini_array_start = .);
	    KEEP(*(SORT(.fini_array.*)))
	    KEEP(*(.fini_array))
	    PROVIDE_HIDDEN (__fini_array_end = .);

	    . = ALIGN(4);
	    /* All data end */
	    __data_end__ = .;

	} > RAM
        ...  
    }
   
```

> Symbol etext = .;

> Là giá trị LMA, biểu thị điểm cuối của phân đoạn .text, bắt đầu phân đoạn .data nằm trên FLASH.

> Symbol data\_start = .;

> Là giá trị VMA, điểm cần chép dữ liệu đến trước khi sử dụng trong chương trình. Khi biên dịch xong, file .map sẽ cho kết quả như sau:

```
    .data  0x20000000      0x450 load address 0x080009e0
           0x20000000                __data_start__ = .
```

> ` *(vtable) ` là phân đoạn dùng để remap vector table lên RAM, có thể thay đổi các hàm phục vụ ngắt một cách linh hoạt hơn, ví dụ như bootloader...

> Đoạn code sau thực hiện việc chép .data từ FLASH vào RAM

```
    ldr    r1, =__etext         @ Load LMA 
    ldr    r2, =__data_start__  @ Load VMA
    ldr    r3, =__data_end__    @ End of .data in RAM

    .flash_to_ram_loop:
    cmp     r2, r3              @ Compare r2 & r3
    ittt    lt                  @ if (r2 < r3) then then then   
    ldrlt   r0, [r1], #4        @ r0  = *r1, r1 = r1 + 4 
    strlt   r0, [r2], #4        @ *r2 = r0,  r2 = r2 + 4
    blt    .flash_to_ram_loop   @ brand
```

> ## Phân đoạn .bss ##

```
    SECTIONS
    {
        ...  
	.bss :
	{
	    __bss_start__ = .;
	    *(.bss*)
	    *(COMMON)
	    __bss_end__ = .;
	} > RAM
        ... 
    }
```

> Đoạn code sau thực hiện khởi động phân đoạn .bss

```
    ldr     r1, =__bss_start__ 
    ldr     r2, =__bss_end__
    mov     r0, #0x0
  
    .bss_init_loop:
    cmp     r1, r2         @ Compare r1 & r2
    itt     lt             @ if (r1 < r2) then then   
    strlt   r0, [r1], #4   @ *r1 = r0,  r1 = r1 + 4
    blt    .bss_init_loop  @ brand
```

> ## Low level Init ##

> Việc khởi động hardware có thể thực hiện bởi các lệnh ASM hoặc để đơn giản ta có thể gọi đến các hàm viết bằng C, cách thực hiện như sau:

```
    ldr    r0, =SystemInit
    blx    r0
```

> Trong source C, hàm `SystemInit` có prototype như sau :

```
   void SystemInit(void);
```

> ## Nhảy đến hàm main () ##

> Tại điểm này `Reset_Handler` có thể nhảy đến hàm main(), kết thúc quá trình khởi động, cú pháp như sau:

```
    ldr  r0, =main
    bx   r0
```

> Đôi khi hàm main được gọi gián tiếp thông qua hàm _start(), hàm này có trong thư viện chuẩn của C (crt0.o) :_

```
    ldr  r0, =_start
    bx   r0
```

> ## STM32F4 memory map ##

> ![http://armtutorial.googlecode.com/svn/trunk/image/STM32F4_Memory_Map_edited.png](http://armtutorial.googlecode.com/svn/trunk/image/STM32F4_Memory_Map_edited.png)

> ## Chương trình demo ##

> Chương trình demo có ở link sau:

> [STM32F4\_Bare\_Metal.rar](http://armtutorial.googlecode.com/svn/trunk/src/STM32F4_Bare_Metal.rar)

> Dùng emBlocks IDE để biên dịch, phần mềm có thể download miễn phí trên trang web chính.

> http://www.emblocks.org/web/
