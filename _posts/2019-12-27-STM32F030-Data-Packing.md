---
layout: post_layout
title: 被STM32F030的SPI坑了一遭
time: 2019年12月27日 星期五
location: 成都
pulished: true
excerpt_separator: "这导"
---

Reference Manual (RM0360) 654页提到了Data packing功能。在使用16位操作访问DR寄存器时，如果SPI是8位的，会自动转换为两次数据传输。

原文如下：
> When the data frame size fits into one byte (less than or equal to 8 bits), data packing is used automatically when any read or write 16-bit access is performed on the SPIx_DR register. The double data frame pattern is handled in parallel in this case. At first, the SPI operates using the pattern stored in the LSB of the accessed word, then with the other half stored in the MSB. Figure 254 provides an example of data packing mode sequence handling. Two data frames are sent after the single 16-bit access the SPIx_DR register of the transmitter. This sequence can generate just one RXNE event in the receiver if the RXFIFO threshold is set to 16 bits (FRXTH=0). The receiver then has to access both data frames by a single 16-bit read of SPIx_DR as a response to this single RXNE event. The RxFIFO threshold setting and the following read access must be always kept aligned at the receiver side, as data can be lost if it is not in line.

这导致了我的OLED驱动移植后出现问题，因为原来的代码是这个样的：
```c
hSPI.Instance->DR = OLED_data;
```
其实这里用了32位操作，实际STM32按照跟16位一样处理，导致每次传输的数据多了个00.

修改成以下之后工作正常。
```c
*((__IO uint8_t*) & hSPI.Instance->DR) = OLED_data;
```

PS: 破事水
