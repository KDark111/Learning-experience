# ADC定数采样

## 定时器

```c
#define HX_ADC_TIMER  TIMER6
#define HX_ADC_RCU_TIMER RCU_TIMER6
#define HX_ADC_TIMER_PRESCALER		(4000-1)
#define HX_ADC_TIMER_PERIOD			(5-1)   //(50000-1)   单位s; (5-1), 单位100us

void TIMER6_IRQHandler(void)
{
  if(timer_interrupt_flag_get(HX_ADC_TIMER, TIMER_INT_FLAG_UP)==SET)//获取定时器更新中断标志
  {
    timer_interrupt_flag_clear(HX_ADC_TIMER, TIMER_INT_FLAG_UP);//清除定时器更新中断标志
	
	adc_updateSample();
  }
}

void xtimer6_start(void) //用于ADC采样
{
    timer_parameter_struct timer_initpara_test;

    rcu_periph_clock_enable(HX_ADC_RCU_TIMER);
	timer_deinit(HX_ADC_TIMER);
    
    /* TIMER configuration */
    timer_initpara_test.prescaler         = HX_ADC_TIMER_PRESCALER;
    timer_initpara_test.alignedmode       = TIMER_COUNTER_EDGE;
    timer_initpara_test.counterdirection  = TIMER_COUNTER_UP;
    timer_initpara_test.period            = HX_ADC_TIMER_PERIOD;
    timer_initpara_test.clockdivision     = TIMER_CKDIV_DIV1;
    timer_initpara_test.repetitioncounter = 0;
	rcu_timer_clock_prescaler_config(RCU_TIMER_PSC_MUL4);
    timer_init(HX_ADC_TIMER,&timer_initpara_test);
	timer_interrupt_enable(HX_ADC_TIMER, TIMER_INT_UP);
	nvic_irq_enable(TIMER6_IRQn, 0, 5);
	timer_enable(HX_ADC_TIMER);
}

void xtimer6_stop(void)
{
	rcu_periph_clock_disable(HX_ADC_RCU_TIMER);//关闭定时器时钟
	timer_interrupt_flag_clear(HX_ADC_TIMER, TIMER_INT_FLAG_UP);
	timer_deinit(HX_ADC_TIMER);
	timer_interrupt_disable(HX_ADC_TIMER,TIMER_INT_FLAG_UP);
	nvic_irq_disable(TIMER6_IRQn);
	timer_disable(HX_ADC_TIMER);
}
```



## ADC采样

### 定时器中断采样处理函数

```c
static uint8_t g_adc_sample_index = 0;
static uint8_t g_adc_sample_status = HX_OFF;
static uint32_t g_adc_lastSampleTime = 0;
static uint32_t g_adc_startSampleTime = 0;
static uint32_t g_adc_endSampleTime = 0;

static uint16_t g_adc_out_sample_v[rfA][ADC_ALC_SAMPLE_NUM] = {0};
#define ADC_ALC_SAMPLE_NUM	64

//定时器中断处理函数，不能添加有长时间处理
void adc_updateSample()
{
	if (HX_OFF == g_adc_sample_status) {
		return;
	}

	if (g_adc_sample_index  >= ADC_ALC_SAMPLE_NUM ) {
		return;
	}
	
	g_adc_lastSampleTime = g_systimeout;

	adc_updatePoutSample();
	
	g_adc_sample_index++;
	
	if (1 == g_adc_sample_index) {
		g_adc_startSampleTime = g_adc_lastSampleTime;
	}
	
	if (ADC_ALC_SAMPLE_NUM == g_adc_sample_index) {
		g_adc_endSampleTime = g_adc_lastSampleTime;
	}
}
```



### 状态正常进行ADC采样

```c
//定时器中断处理函数，不能添加有长时间处理
static void adc_updatePoutSample()
{
	if (HX_OFF == g_adc_sample_status) {
		return;
	}
	
	if (g_adc_sample_index < ADC_ALC_SAMPLE_NUM) {
		adc_poutSample(g_adc_sample_index);
	}
}
```



### 选择采样ADC通道

```c
//定时器中断处理函数，不能添加有长时间处理
static void adc_poutSample(int i)
{
	uint8_t adc_chl;
	uint8_t chl;
	
	if (i >= ADC_ALC_SAMPLE_NUM) {
		return;
	}

	for (chl = b8dl; chl < rfA; ++chl) {
		if (!adc_isChannelOn(chl))
			continue;
		
        //获取采样ADC通道
		adc_chl = adc_getSampleChannel(chl, !ADC_IN);
        //选择上下行通道
		adc_poutChannelSelect(chl);
		
		g_adc_out_sample_v[chl][i] = adc0_channel_sample(adc_chl);
	}
}
```



### ADC实际采样函数

```c
static uint16_t adc_channel_sample(uint32_t adc_periph, uint8_t adcChl)
{
	/* ADC routine channel config */
	adc_routine_channel_config(adc_periph, 0U, adcChl, ADC_SAMPLETIME_15);
	/* ADC software trigger enable */
	adc_software_trigger_enable(adc_periph, ADC_ROUTINE_CHANNEL);

	/* wait the end of conversion flag */
	while(!adc_flag_get(adc_periph, ADC_FLAG_EOC));
	/* clear the end of conversion flag */
	adc_flag_clear(adc_periph, ADC_FLAG_EOC);
	/* return regular channel sample value */
	return (adc_routine_data_read(adc_periph));
}

static uint16_t adc0_channel_sample(uint8_t adcChl)
{
	return adc_channel_sample(ADC0, adcChl);
}
```



### 更新ADC采样状态(结果)

```c
static void adc_refreshAdcResult()
{
	static uint32_t refreshTmo = 0;
	static uint32_t ulSilentSw = HX_OFF;
	static uint32_t hwmodulepwrstat = 0;
	
	if (HX_ON == g_adc_sample_status && ADC_ALC_SAMPLE_NUM == g_adc_sample_index) {
		if (ulSilentSw == hx_eeprom_sys_cfg.ulSilentSw && hwmodulepwrstat == hx_eeprom_sys_cfg.hwmodulepwrstat) {
			//采样完成，开关状态无变化，计算并更新
			adc_update();
			
			//adc_autoDsaAgc();
			adc_agc();
			adc_openLoopPwrCtrl();
		}else{
			//忽略本次采样结果
			HX_LOG_ALWAYS("ignore the sample result as sw changed.");
		}
	
		g_adc_sample_status = HX_OFF;
		g_adc_sample_index = 0;
		g_adc_startSampleTime = g_adc_endSampleTime = g_adc_lastSampleTime = 0;
	}
    
    if (g_systimeout > refreshTmo ) {
        //100ms采集64次
		refreshTmo = g_systimeout + TICKS_PER_MILLI_SECOND * 100; //100ms
		if (g_adc_sample_status == HX_ON ) {//上次周期未完成采样
			g_adc_sample_status = HX_OFF;
			HX_LOG_WARNING("Failed to finish Sample in 100ms. just finish %d times sample.", g_adc_sample_index);
		}

		//记录开关状态，用以检查采样过程中开关状态是否有变化
		ulSilentSw = hx_eeprom_sys_cfg.ulSilentSw;
		hwmodulepwrstat = hx_eeprom_sys_cfg.hwmodulepwrstat;
		
		//使能定时器中断重新开始采样动作
		g_adc_sample_index = 0;
		g_adc_sample_status = HX_ON;
	}

}
```

