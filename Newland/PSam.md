# Psam 说明

### 1.串口要求

```c
Param[0] = 9600;     //波特率
Param[1] = is_block; //是否阻塞
Param[2] = 8;	     //8为数据
Param[3] = 2;        //2为停止位
Param[4] = 'e';      //校验
```
### 2.上电时序

```c
int control_power(int channel, int power)
{
    if(mPowerState == power)
    {
        return 0;
    }
    mPowerState = power;
    if(power)
    {
        psam_set_rst(mFd[channel-1],0);   // 复位拉低
        psam_set_en(mFd[channel-1],0);    // 电源拉低 
    	usleep(1000*20);
    	psam_set_en(mFd[channel-1],1);    // 电源拉搞
    	usleep(1000*50);
        psam_set_rst(mFd[channel-1],1);   // 复位拉高
    }
    else
    {
        psam_set_rst(mFd[channel-1],0);
        usleep(1000*20);
        psam_set_en(mFd[channel-1],0);
    	usleep(1000*50);
    }

    return 0;
}
```

### 3.复位卡片

```c
int ResetCard(unsigned long fd, unsigned char *atr,int *atrLen)
{
    int channel = (int)fd;
    char buf[1024];
    psam_set_rst(mFd[channel-1],0); // 复位拉低
	usleep(1000*10);
	psam_set_rst(mFd[channel-1],1);  // 电源拉搞
    if(NULL != atrLen)
    {
        *atrLen = 0;
    }
    if(NULL != atr)
    {
        atr = NULL;  
    }
}  
```



### 4.检测卡片

```c
int CheckCard(unsigned long fd)
{
    char acRecvBuf[1024];

    int channel = (int)fd;

    Driver_init();
    int ret = serial_open(channel, mBaudrate, 0);
    if(ret < 0)
    {
        return -1;
    }
    control_power(channel, 1);
    ResetCard(fd, NULL, NULL);
    int RecvLen = ReadPacket(channel, acRecvBuf, sizeof(acRecvBuf), 500);
    PrintCustomHex("Check", acRecvBuf, RecvLen);
    serial_clear(channel);
    if (RecvLen > 0)
    {
        if (acRecvBuf[0] == 0x3B || acRecvBuf[0] == 0x3F || (acRecvBuf[0] == 0x00 && acRecvBuf[1] == 0x3B)
            || (acRecvBuf[0] == 0x00 && acRecvBuf[1] == 0x3F))
        {
            
            return 0;
        }
        
    }
    return -1;
}
```

### 5.读写卡

```c
int CardApdu(unsigned long fd, unsigned char* apdu, int apduLength, unsigned char* response,int* respLength)
{
    unsigned long mFd = 0;
    int ret = -1;
    if(NULL == response || NULL == respLength || NULL == apdu)
    {
        return -1;
    }

    int channel = (int)fd;
    if(channel > MAX_PSAMCARD)
    {
        return -1;
    }    
    if(mPowerState == 0)
    {
        OpenCard(&mFd, fd);
        usleep(500*1000);
        serial_clear(channel);
    }
    ret = serial_write(channel, (char*)apdu, (int)apduLength);
    PrintCustomHex("send", apdu, apduLength);
	if (-1 == ret)
	{
		return -1;
	}

	*respLength = ReadPacket(channel, (unsigned char*)response, MAX_PACKET_LENGTH, 500);
    PrintCustomHex("recv", response, *respLength);
    if(*respLength > 0)
    {
        ret = 0;
    }
    return ret;
}

```

