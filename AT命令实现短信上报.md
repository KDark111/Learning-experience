# AT命令实现短信上报

## Text格式发送

Text格式发送短信只能实现发送英文或数字

发送短信前应使用AT+CSCA?查看当前服务中心地址是否被正确配置,否则将无法发送短信

> 1. AT+CMEE=1	(开启错误报告)
> 2. AT+CMGF=1	(设置为Text模式)
> 3. AT+CMGS=接收方电话	(发送短信)
> 4. 等待出现>符号, 表示可以发送信息
> 5. 发送完信息后需要接上结束字符(ctrl+z),可以通过输入16进制的0x1A实现
> 6. 如果返回+CMGS:XX表示发送成功

## PDU格式发送

PDU格式能发送中文,但是需要将发送短信转换为指定格式

> 1. AT+CMEE=1
> 2. AT+CMGF=0  (设置为PDU格式)
> 3. AT+CMGS=发送短信长度/2
> 4. 等待出现>符号,发送转换格式后的信息
> 5. 发送完信息后需要接上结束字符(ctrl+z),可以通过输入16进制的0x1A实现
> 6. 如果返回+CMGS:XX表示发送成功



### 格式转换要求

#### 短信服务中心部分处理

> 1. 号码若以+86开头，则移除+号
> 2. 判断号码长度是否为偶数，不为偶数则在末尾添加F字符
> 3. 将号码奇偶位互换
> 4. 在得到的字符串最前方加上“91”
> 5. 在字符串的前方加上字符串长度/2的16进制数（二位）

```c
if (strstr(addr, "+") != NULL) {
    for (unsigned int i = 0; i < sizeof(addr); i++) {
        addr[i] = addr[i + 1];
    }
}

//不为偶数则补上F
addrLen = strlen(addr);
if ((addrLen % 2) != 0) {
    addr[addrLen] = 'F';
}
//奇偶位互换
for (unsigned int i = 0; i < sizeof(addr); i += 2) {
    temp = addr[i];
    addr[i] = addr[i + 1];
    addr[i + 1] = temp;
}
sprintf(strTemp, "91%s", addr);
//前方加上字符串长度/2的16进制(二位)
addrLen = strlen(strTemp) / 2;
sprintf(addr, "%02X%s", addrLen, strTemp);
```

#### 手机号码部分处理

> 1. 号码若以+86开头，则移除+号
> 2. 判断号码长度是否为偶数，不为偶数则在末尾添加F字符
> 3. 将号码奇偶位互换

#### 短信息部分处理

> 1. 将字符串转换为UCS2格式
> 2. 将UCS2格式转换为字符串
> 3. 在字符串的前方加上字符串长度/2的16进制数（二位）

```c
// utf8格式转换为ucs2格式
void utf8_to_unicode(const unsigned char *utf8_buf, uint16_t *unic_buf) {
    const unsigned char *utf8_ptr = utf8_buf;
    uint16_t *unic_ptr = unic_buf;

    while (*utf8_ptr) {
        if ((*utf8_ptr & 0x80) == 0x00) { // 单字节(ASCII)
            *unic_ptr++ = *utf8_ptr++;
        } else if ((*utf8_ptr & 0xE0) == 0xC0) { // 双字节
            uint8_t b1 = *utf8_ptr;
            uint8_t b2 = *(utf8_ptr + 1);

            uint16_t unicode = (uint16_t)((b1 & 0x1F) << 6) |
                               (b2 & 0x3F);

            *unic_ptr++ = unicode;
            utf8_ptr += 2;
        } else if ((*utf8_ptr & 0xF0) == 0xE0) { // 三字节
            uint8_t b1 = *utf8_ptr;
            uint8_t b2 = *(utf8_ptr + 1);
            uint8_t b3 = *(utf8_ptr + 2);

            uint16_t unicode = (uint16_t)((b1 & 0x0F) << 12) |
                               (uint16_t)((b2 & 0x3F) << 6) |
                               (b3 & 0x3F);

            *unic_ptr++ = unicode;
            utf8_ptr += 3;
        } else {
            // 非法字符
            utf8_ptr++;
        }
    }
    *unic_ptr = 0;
}

//usc2格式转化为字符串格式
void uint16t_to_char(uint16_t* unic_buf, char* char_buf) {
	while (*unic_buf) {
		char temp[6] = {0};
		sprintf(temp, "%04X", *unic_buf++);
		strcat(char_buf, temp);
	}
}

//短信息部分处理(将UTF-8转换为UCS2)
utf8_to_unicode((const unsigned char*)sendInfo, msgTemp);
uint16t_to_char(msgTemp, msg);	
addrLen = strlen(msg) / 2;
sprintf(sendTemp, "%02X%s", addrLen, msg);
```

#### 编码组合

> 1. “01000D91“+手机部分+”0008“+短信部分
> 2. AT+CMGS发送的长度为此时字符串长度/2
> 3. 服务中心部分+字符串
