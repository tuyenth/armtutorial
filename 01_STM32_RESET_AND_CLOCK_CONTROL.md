# Reset And Clock Control (RCC) #

## Reset circuit ##

![https://armtutorial.googlecode.com/svn/trunk/image/Reset_Circuit.png](https://armtutorial.googlecode.com/svn/trunk/image/Reset_Circuit.png)

## Clock tree ##

![https://armtutorial.googlecode.com/svn/trunk/image/Clock_Tree.png](https://armtutorial.googlecode.com/svn/trunk/image/Clock_Tree.png)

Three different clock sources can be used to drive the system clock (SYSCLK):

> ● HSI oscillator clock

> ● HSE oscillator clock

> ● PLL clock

The devices have the following two secondary clock sources:

> ● 40 kHz low speed internal RC (LSI RC) which drives the independent watchdog and optionally the RTC used for Auto-wakeup from Stop/Standby mode.

> ● 32.768 kHz low speed external crystal (LSE crystal) which optionally drives the realtime clock (RTCCLK)

Each clock source can be switched on or off independently when it is not used, to optimize power consumption.

# RCC Register Map #

![https://armtutorial.googlecode.com/svn/trunk/image/RCC_MAP.png](https://armtutorial.googlecode.com/svn/trunk/image/RCC_MAP.png)

# Example #

```

/*****************************************************************************
 * Function Name  : RCC_Configuration
 * Description    : Reset and Clock Control configuration
 * Input          : None
 * Output         : None
 * Return         : None
 ******************************************************************************/
void RCC_Configuration(void)
{
    ErrorStatus HSEStartUpStatus;
    
    /* Reset the RCC clock configuration to default reset state */
    RCC_DeInit();
    
    /* Configure the High Speed External oscillator */
    RCC_HSEConfig(RCC_HSE_ON);
    
    /* Wait for HSE start-up */
    HSEStartUpStatus = RCC_WaitForHSEStartUp();
    
    if(HSEStartUpStatus == SUCCESS)
    {
        /* Enable Prefetch Buffer */
        FLASH_PrefetchBufferCmd(FLASH_PrefetchBuffer_Enable);
        
        /* Set the code latency value: FLASH Two Latency cycles */
        FLASH_SetLatency(FLASH_Latency_2);
        
        /* Configure the AHB clock(HCLK): HCLK = SYSCLK */
        RCC_HCLKConfig(RCC_SYSCLK_Div1);
        
        /* Configure the High Speed APB2 clcok(PCLK2): PCLK2 = HCLK */
        RCC_PCLK2Config(RCC_HCLK_Div1);
        
        /* Configure the Low Speed APB1 clock(PCLK1): PCLK1 = HCLK/2 */
        RCC_PCLK1Config(RCC_HCLK_Div2);
        
        /* Configure the PLL clock source and multiplication factor     */
        /* PLLCLK = HSE*PLLMul = 8*9 = 72MHz */
        RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9);
        
        /* Enable PLL   */
        RCC_PLLCmd(ENABLE);
        
        /* Check whether the specified RCC flag is set or not */
        /* Wait till PLL is ready       */
        while(RCC_GetFlagStatus(RCC_FLAG_PLLRDY) == RESET);
        
        /* Select PLL as system clock source */
        RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK);
        
        /* Get System Clock Source */
        /* Wait till PLL is used as system clock source */
        while(RCC_GetSYSCLKSource() != 0x08);
    } 
}

```





