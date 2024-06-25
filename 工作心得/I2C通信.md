# I2C通信

## I2C写

### i2c_page_write

```c
#define DEVICE_LOCAL_ADDR_LEN_1BYTE  0
#define DEVICE_LOCAL_ADDR_LEN_2BYTE  1

static void i2c_page_write(uint32_t i2c_address, uint8_t* p_buffer, uint32_t write_address, uint32_t write_address_len, uint8_t number_of_byte)
{
	/* 等待i2c空闲 */
	while (i2c_flag_get(I2C0, I2C_FLAG_I2CBSY)) {
		delay_100us(1);
	}

	i2c_start_on_bus(I2C0);
	
	/* 等待i2c设置启动条件,主设备开启一个新的通话 */
	while (!i2c_flag_get(I2C0, I2C_FLAG_SBSEND)) {
		delay_100us(1);
	}

	/* 指定要通信的从设备地址以及主设备通信方式 */
	i2c_master_addressing(I2C0, i2c_address, I2C_TRANSMITTER);

	/* 等待地址发送完成 */
	while (!i2c_flag_get(I2C0, I2C_FLAG_ADDSEND)) {
		delay_100us(1);
	}

	/* 清除发送标志位 */
	i2c_flag_clear(I2C0, I2C_FLAG_ADDSEND);

	/* 等待发送数据为空 */
	while (!i2c_flag_get(I2C0, I2C_FLAG_TBE)) {
		delay_100us(1);
	}

	if (DEVICE_LOCAL_ADDR_LEN_2BYTE == write_address_len) {
		i2c_data_transmit(I2C0, (unsigned char)(write_address >> 8));

		/* 等待数据发送完毕 */
		while (!i2c_flag_get(I2C0, I2C_FLAG_BTC)) {
			delay_100us(1);
		}
	}

	/* 发送实际写地址 */
	i2c_data_transmit(I2C0, (write_address & 0xFF));

	while (!i2c_flag_get(I2C0, I2C_FLAG_BTC)) {
		delay_100us(1);
	}

	/* 等待发送数据发送完毕 */
	while (number_of_byte--) {
		i2c_data_transmit(I2C0, *p_buffer);
		p_buffer++;
		while (!i2c_flag_get(I2C0, I2C_FLAG_BTC)) {
			delay_100us(1);
		}
	}
	/* 停止占用总线 */
	i2c_stop_on_bus(I2C0);

	/* 关闭控制信号 */
	while (I2C_CTL0(I2C0) & 0x0200) {
		delay_100us(1);
	}
}
```



### i2c_wait_standby_state

```c
static void i2c_wait_standby_state(uint32_t i2c_address) 
{
	__IO uint32_t val = 0;

	while (1) {
		while (i2c_flag_get(I2C0, I2C_FLAG_I2CBSY));

		i2c_start_on_bus(I2C0);

		while (!i2c_flag_get(I2C0, I2C_FLAG_SBSEND));

		i2c_master_addressing(I2C0, i2c_address, I2C_TRANSMITTER);

		/* 当寄存器地址发送成功或发送失败 */
		do {
			/* 获取寄存器状态 */
			val = I2C_STAT0(I2C0);
		} while (0 == (val & (I2C_STAT0_ADDSEND | I2C_STAT0_AERR)));

		if (val & I2C_STAT0_ADDSEND) {
			i2c_flag_clear(I2C0, I2C_FLAG_ADDSEND);
			i2c_stop_on_bus(I2C0);
			while (I2C_CTL0(I2C0) & 0x0200);
			return;
		} else {
			i2c_flag_clear(I2C0, I2C_FLAG_AERR);
		}

		i2c_stop_on_bus(I2C0);
		while (I2C_CTL0(I2C0) & 0x0200);
	}
}
```



### i2c_buffer_write

```c
void i2c_buffer_write(uint32_t i2c_address, uint8_t* p_buffer, uint32_t write_address, uint32_t write_address_len, uint8_t number_of_byte)
{
	uint32_t number_of_page, number_of_single, needNum;
	uint32_t leftNum = I2C_PAGE_SIZE - (write_address & (I2C_PAGE_SIZE - 1));//计算当前页面剩余可写字节数

	i2c_page_write(i2c_address, p_buffer, write_address, write_address_len, 
		(uint8_t)(leftNum > number_of_byte ? number_of_byte : leftNum));	//将当前页写满
	i2c_wait_standby_state(i2c_address);	//确保i2c通信可靠性

	p_buffer += leftNum;	//写入数据偏移
	write_address += leftNum;	//写入地址加上偏移量
	needNum = number_of_byte > leftNum ? number_of_byte - leftNum : 0;	//还需要写入的字节数
	number_of_page = needNum >> 7;	//计算还需要的整页数
	number_of_single = needNum & (I2C_PAGE_SIZE - 1);	//计算还需要的单页字节数

	while (number_of_page) {
		i2c_page_write(i2c_address, p_buffer, write_address, write_address_len, I2C_PAGE_SIZE);
		i2c_wait_standby_state(i2c_address);

		write_address += I2C_PAGE_SIZE;
		p_buffer += I2C_PAGE_SIZE;
		number_of_page--;
	}

	if (number_of_single) {
		i2c_page_write(i2c_address, p_buffer, write_address, write_address_len, (uint8_t)number_of_single);
		i2c_wait_standby_state(i2c_address);
	}
}
```



## I2C读

### i2c_bus_reset

```c
static void i2c_bus_reset(void)
{
    i2c_deinit(I2C0);
    /* configure SDA/SCL for GPIO */
    GPIO_BC(I2C_SCL_PORT) |= I2C_SCL_PIN;
    GPIO_BC(I2C_SDA_PORT) |= I2C_SDA_PIN;
    gpio_output_options_set(I2C_SCL_PORT, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, I2C_SCL_PIN);
    gpio_output_options_set(I2C_SDA_PORT, GPIO_OTYPE_PP, GPIO_OSPEED_50MHZ, I2C_SDA_PIN);
    __NOP();
    __NOP();
    __NOP();
    __NOP();
    __NOP();
    GPIO_BOP(I2C_SCL_PORT) |= I2C_SCL_PIN;
    __NOP();
    __NOP();
    __NOP();
    __NOP();
    __NOP();
    GPIO_BOP(I2C_SDA_PORT) |= I2C_SDA_PIN;
    /* connect I2C_SCL_PIN to I2C_SCL */
    /* connect I2C_SDA_PIN to I2C_SDA */
    gpio_output_options_set(I2C_SCL_PORT, GPIO_OTYPE_OD, GPIO_OSPEED_50MHZ, I2C_SCL_PIN);
    gpio_output_options_set(I2C_SDA_PORT, GPIO_OTYPE_OD, GPIO_OSPEED_50MHZ, I2C_SDA_PIN);
    /* configure the I2CX interface */
    i2c_config();
}
```



### i2c_buffer_read_dma_timeout

```c
#define I2C_TIME_OUT            (uint16_t)(100000)

static void i2c_buffer_read_dma_timeout(uint32_t i2c_address, uint8_t* p_buffer, uint32_t read_address, uint32_t read_address_len, uint8_t number_of_byte)
{
	dma_single_data_parameter_struct dma_init_parameter;

	uint8_t state = I2C_START;
	uint8_t read_cycle = RESET;
	uint8_t timeout = 0;
	uint8_t i2c_timeout_flag = 0;

	i2c_ack_config(I2C0, I2C_ACK_ENABLE);
	while (!(i2c_timeout_flag)) {
		switch(state) {
			case I2C_START:
				if (RESET == read_cycle) {
					i2c_disable(I2C0);
					i2c_enable(I2C0);

					i2c_ack_config(I2C0, I2C_ACK_ENABLE);
					while (i2c_flag_get(I2C0, I2C_FLAG_I2CBSY) && (timeout < I2C_TIME_OUT)) {
						delay_100us(1);
						timeout++;
					}
					if (timeout < I2C_TIME_OUT) {
						// 发送开始信号
						i2c_start_on_bus(I2C0);
						timeout = 0;
						state = I2C_SEND_ADDRESS;
					} else {
						// 复位i2c总线
						i2c_bus_reset();
						timeout = 0;
						state = I2C_START;
						printf("i2c bus is busy\n");
					}
				} 
				else {
					i2c_start_on_bus(I2C0);
					timeout = 0;
					state = I2C_SEND_ADDRESS;
				}
				break;
                case I2C_SEND_ADDRESS:
				while ((!i2c_flag_get(I2C0, I2C_FLAG_SBSEND)) && (timeout < I2C_TIME_OUT)) {
					delay_100us(1);
					timeout++;
				}
				if (timeout < I2C_TIME_OUT) {
					if (RESET == read_cycle) {
						// 发送从设备地址
						i2c_master_addressing(I2C0, i2c_address, I2C_TRANSMITTER);
						state = I2C_CLEAR_ADDRESS_FLAG;
					} else {
						// 设置读取数据模式
						i2c_master_addressing(I2C0, i2c_address, I2C_RECEIVER);
						state = I2C_CLEAR_ADDRESS_FLAG;
					}
					timeout = 0;
				} else {
					timeout = 0;
					state = I2C_START;
					read_cycle = RESET;
				}
				break;
			case I2C_CLEAR_ADDRESS_FLAG:
				while (!i2c_flag_get(I2C0, I2C_FLAG_ADDSEND) && (timeout < I2C_TIME_OUT)) {
					delay_100us(1);
					timeout++;
				}
				if (timeout < I2C_TIME_OUT) }{
					i2c_flag_clear(I2C0, I2C_FLAG_ADDSEND);
					timeout = 0;
					state = I2C_TRANSMIT_DATA;
				} else {
					timeout = 0;
					state = I2C_START;
					read_cycle = 0;
				}
				break;
        	case I2C_TRANSMIT_DATA:
				if (RESET == read_cycle) {
					while (!i2c_flag_get(I2C0, I2C_FLAG_TBE) && (timeout < I2C_TIME_OUT)) {
						delay_100us(1);
						timeout++;
					}
					if (timeout < I2C_TIME_OUT) {
						if (DEVICE_LOCAL_ADDR_LEN_2BYTE == read_address_len) {
							// 发送读取数据地址(若为两个字节则先发送高位,在发送低位)
							i2c_data_transmit(I2C0, (unsigned char)(read_address >> 8));
							while (!i2c_flag_get(I2C0, I2C_FLAG_BTC)) {
								delay_100us(1);
							}
						}
						i2c_data_transmit(I2C0, (read_address & 0xFF));
						timeout = 0;
					} else {
						timeout = 0;
						state = I2C_START;
						read_cycle = 0;
					}
                    					
					while (!i2c_flag_get(I2C0, I2C_FLAG_BTC) && (timeout < I2C_TIME_OUT)) {
						delay_100us(1);
						timeout++;
					}
					if (timeout < I2C_TIME_OUT) {
						timeout = 0;
						// 重新开始发送信号
						state = I2C_START;
						read_cycle++;
					} else {
						timeout = 0;
						state = I2C_START;
						read_cycle = 0;
					}
				}
        		else {
					if (number_of_byte < 2) {
						// 只读取一个字节MCU就发送非应答
						i2c_ack_config(I2C0, I2C_ACK_DISABLE);
						i2c_flag_get(I2C0, I2C_FLAG_ADDSEND);
						i2c_stop_on_bus(I2C0);
						while (!i2c_flag_get(I2C0, I2C_FLAG_TBE));
						*p_buffer = i2c_data_receive(I2C0);
						number_of_byte--;
						timeout = 0;
						state = I2C_STOP;
					} else {
                        // 超过一个字节使用DMA读取全部数据
						dma_deinit(I2C_DMA, DMA_RX_CH);
						dma_single_data_para_struct_init(&dma_init_parameter);
						dma_init_parameter.direction = DMA_PERIPH_TO_MEMORY;
						dma_init_parameter.memory0_addr = (uint32_t)p_buffer;
						dma_init_parameter.memory_inc = DMA_MEMORY_INCREASE_ENABLE;
						dma_init_parameter.periph_memory_width = DMA_MEMORY_WIDTH_8BIT;
						dma_init_parameter.number = number_of_byte;
						dma_init_parameter.periph_addr = (uint32_t)&I2C_DATA(I2C0);
						dma_init_parameter.periph_inc = DMA_PERIPH_INCREASE_DISABLE;
						dma_init_parameter.priority = DMA_PRIORITY_ULTRA_HIGH;
						dma_single_data_mode_init(I2C_DMA, DMA_RX_CH, &dma_init_parameter);
						dma_circulation_disable(I2C_DMA, DMA_RX_CH);
						dma_channel_subperipheral_select(I2C_DMA, DMA_RX_CH, DMA_SUBPERI1);
						i2c_dma_last_transfer_config(I2C0, I2C_DMALST_ON);
						i2c_dma_config(I2C0, I2C_DMA_ON);
						dma_channel_enable(I2C_DMA, DMA_RX_CH);
                        while (!dma_flag_get(I2C_DMA, DMA_RX_CH, DMA_FLAG_FTF)) {
							delay_100us(1);
						}
						state = I2C_STOP;
					}
				}
				break;
        	case I2C_STOP:
				i2c_stop_on_bus(I2C0);
				while ((I2C_CTL0(I2C0) & I2C_CTL0_STOP) && (timeout < I2C_TIME_OUT)) {
					delay_100us(1);
					timeout++;
				}
				if (timeout < I2C_TIME_OUT) {
					timeout = 0;
					state = I2C_END;
					i2c_timeout_flag = I2C_OK;
					dma_flag_clear(I2C_DMA, DMA_RX_CH, DMA_FLAG_FTF);
				} else {
					timeout = 0;
					state = I2C_START;
					read_cycle = 0;
				}
				break;
        	default:
				state = I2C_START;
				read_cycle = 0;
				i2c_timeout_flag = I2C_OK;
				timeout = 0;
				break;
		}
	}

	return I2C_END;
}
```



### i2c_buffer_read

```c
void i2c_buffer_read(uint32_t i2c_address, uint8_t* p_buffer, uint32_t read_address, uint32_t read_address_len, uint8_t number_of_byte)
{
	uint32_t number_of_page, number_of_single, needNum;
	uint32_t leftNum = I2C_PAGE_SIZE - (read_address & (I2C_PAGE_SIZE - 1));//计算当前页面剩余可读字节数

	i2c_buffer_read_dma_timeout(i2c_address, p_buffer, read_address, read_address_len, 
		(uint8_t)(leftNum > number_of_byte ? number_of_byte : leftNum));
	i2c_wait_standby_state(i2c_address);

	p_buffer += leftNum;
	read_address += leftNum;
	needNum = number_of_byte > leftNum ? number_of_byte - leftNum : 0;	//还需要读的字节数
	number_of_page = needNum >> 7;	//计算还需要的整页数
	number_of_single = needNum & (I2C_PAGE_SIZE - 1);	//计算还需要的单页字节数
	
	while (number_of_page) {
		delay_100us(10);
		i2c_buffer_read_dma_timeout(i2c_address, p_buffer, read_address, read_address_len, I2C_PAGE_SIZE);
		i2c_wait_standby_state(i2c_address);

		write_address += I2C_PAGE_SIZE;
		p_buffer += I2C_PAGE_SIZE;
		number_of_page--;
	}

	if (number_of_single) {
		i2c_buffer_read_dma_timeout(i2c_address, p_buffer, read_address, read_address_len, (uint8_t)number_of_single);
		i2c_wait_standby_state(i2c_address);
	}
}
```

