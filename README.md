# 电控组第二次培训作业

- 姓名：**王国阳**
- 学号：***20253008726***


## 定时器中断实现原理

用的TIM2定时器，内部时钟72 MHz，7200分频，计数器向上计数 4999，计数上限即 500 ms 触发一次更新中断。在中断函数里配置gpio口低电平，程序启动时先点亮 LED0，每次中断将灯号依次切换为 LED1、LED2、LED0。

## 核心代码

### 配置定时器

`Core/Src/main.c`：

```c
static void MX_TIM2_Init(void)
{
  TIM_ClockConfigTypeDef sClockSourceConfig = {0};
  TIM_MasterConfigTypeDef sMasterConfig = {0};

  htim2.Instance = TIM2;
  htim2.Init.Prescaler = 7199;
  htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim2.Init.Period = 4999;
  htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
  htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;
  if (HAL_TIM_Base_Init(&htim2) != HAL_OK)
  {
    Error_Handler();
  }

  sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
  if (HAL_TIM_ConfigClockSource(&htim2, &sClockSourceConfig) != HAL_OK)
  {
    Error_Handler();
  }

  sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
  if (HAL_TIMEx_MasterConfigSynchronization(&htim2, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
}
```

`Core/Src/stm32f1xx_hal_msp.c`：

```c
void HAL_TIM_Base_MspInit(TIM_HandleTypeDef *htim_base)
{
  if (htim_base->Instance == TIM2)
  {
    __HAL_RCC_TIM2_CLK_ENABLE();
    HAL_NVIC_SetPriority(TIM2_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(TIM2_IRQn);
  }
}
```

### 初始化和启动调用

`Core/Src/main.c` 的 `main()` 中：

```c
MX_GPIO_Init();
MX_TIM2_Init();
current_led = 0U;
LED_Show(current_led);

__HAL_TIM_SET_COUNTER(&htim2, 0U);
__HAL_TIM_CLEAR_FLAG(&htim2, TIM_FLAG_UPDATE);
if (HAL_TIM_Base_Start_IT(&htim2) != HAL_OK)
{
  Error_Handler();
}
```

### 中断函数

`Core/Src/stm32f1xx_it.c`：

```c
void TIM2_IRQHandler(void)
{
  HAL_TIM_IRQHandler(&htim2);
}
```

`Core/Src/main.c`：

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  if (htim->Instance == TIM2)
  {
    current_led++;
    if (current_led >= 3U)
    {
      current_led = 0U;
    }
    LED_Show(current_led);
  }
}
```

