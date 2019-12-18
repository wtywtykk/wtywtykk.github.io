---
layout: post_layout
title: 一次STM32 Flash写入异常的分析
time: 2019年12月15日 星期日
location: 成都
pulished: true
excerpt_separator: "在我"
---

一个LED引发的血案

在我的一个STM32程序里面，用内部Flash保存配置设置。但是出现了一定几率把自己刷死的情况。

用STLINK读取被刷死的Flash内容，可以看到Flash的前4个字节被写成了0。Flash的前4字节保存的是初始化时候的堆栈地址，写成0之后无法操作堆栈，自然不能启动。
<img src="/postimg/2019-12-14-STM32-Flash-Error-Readback.png" width="100%" />

显然问题跟操作Flash的代码有关，相关代码差不多下面这样：

```c
bool FlashStorage_FlashProgramSafe(uint32_t TypeProgram, uint32_t Address, uint64_t Data)
{
	uint32_t ProgramSize = 8;
	......
	if ((Address >= StorageAddr) && (Address + ProgramSize <= StorageAddr + StorageSize))
	{
		......
	}
	else
	{
		assert(false);
		return false;
	}
}

bool FlashStorage_WriteNewBlockAndEraseOldBlock(uint8_t* NewBlock, uint8_t* NewData, uint32_t DataSize, uint8_t* OldBlock)
{
	uint32_t i = 0;
	uint32_t BlockDataBase = (uint32_t)NewBlock + sizeof(uint32_t);

	assert(DataSize <= STORAGE_RECORD_SIZE - sizeof(uint32_t));
	......
	for (; i < DataSize; i++)
	{
		FlashStorage_FlashProgramSafe(FLASH_TYPEPROGRAM_BYTE, BlockDataBase + i, NewData[i]);
	}
	......
	if (OldBlock)
	{
		FlashStorage_FlashProgramSafe(FLASH_TYPEPROGRAM_WORD, (uint32_t)OldBlock, 0);
	}
	......
}
```

但是看代码这个问题无法解释，在写这段代码的时候就感觉Flash操作比较危险，所以加了很多的校验，不仅检查了0指针，还查了在不在指定范围内，不应该是地址写错导致的。

不过，看网上别人写的Flash操作，大部分关了中断，而官方手册并没有说这一点。

加上关中断之后，一切居然都正常了，证明确实是有中断在搞鬼。那么下一步要判断的是因为某个中断里面代码导致的问题，还是单纯“进中断”这个操作就会导致问题。

同样思路，往下删减代码，当把Systick里面的东西删掉后，问题消失。这时SysTick里面有这堆东西：

```c
void HAL_SYSTICK_Callback(void)
{
	LED_Set(true);
	if (InitDone)
	{
		static bool b = false;

		xxxxxx();
		xxxxxxxxxxx();
		b = !b;
		if (b)
		{
			xxxxxxxxxxxxxxxxxxxxxxx();
			xxxxxxxxxxxxxxxxx();
		}
	}
	LED_Set(false);
}
```
删到最后，发现是LED_Set的问题，这个函数大概是这样的：
```c
void LED_Set(bool Val)
{
	GPIO_SetVal(&LEDHandle, !Val);
}
```
这么简单的一个东西，怎么可能出错，跟进去

是一个用bitband实现的io操作函数
```c
void GPIO_SetVal(GPIO_Handle* Handle, bool Val)
{
	assert(Handle);
	if (Handle)
	{
		*Handle->ODRAddr = Val ? 1 : 0;
	}
}
```
调试到这里，发现问题了，Handle->ODRAddr居然是0！

而在HAL库里面：
```c
static void FLASH_Program_Word(uint32_t Address, uint32_t Data)
{
  /* Check the parameters */
  assert_param(IS_FLASH_ADDRESS(Address));
  
  /* If the previous operation is completed, proceed to program the new data */
  CLEAR_BIT(FLASH->CR, FLASH_CR_PSIZE);
  FLASH->CR |= FLASH_PSIZE_WORD;
  FLASH->CR |= FLASH_CR_PG;

  *(__IO uint32_t*)Address = Data;
}


HAL_StatusTypeDef HAL_FLASH_Program(uint32_t TypeProgram, uint32_t Address, uint64_t Data)
{
  HAL_StatusTypeDef status = HAL_ERROR;
  
  /* Process Locked */
  __HAL_LOCK(&pFlash);
  
  /* Check the parameters */
  assert_param(IS_FLASH_TYPEPROGRAM(TypeProgram));
  
  /* Wait for last operation to be completed */
  status = FLASH_WaitForLastOperation((uint32_t)FLASH_TIMEOUT_VALUE);
  
  if(status == HAL_OK)
  {
    if(TypeProgram == FLASH_TYPEPROGRAM_BYTE)
    {
      /*Program byte (8-bit) at a specified address.*/
      FLASH_Program_Byte(Address, (uint8_t) Data);
    }
    else if(TypeProgram == FLASH_TYPEPROGRAM_HALFWORD)
    {
      /*Program halfword (16-bit) at a specified address.*/
      FLASH_Program_HalfWord(Address, (uint16_t) Data);
    }
    else if(TypeProgram == FLASH_TYPEPROGRAM_WORD)
    {
      /*Program word (32-bit) at a specified address.*/
      FLASH_Program_Word(Address, (uint32_t) Data);
    }
    else
    {
      /*Program double word (64-bit) at a specified address.*/
      FLASH_Program_DoubleWord(Address, Data);
    }
    
    /* Wait for last operation to be completed */
    status = FLASH_WaitForLastOperation((uint32_t)FLASH_TIMEOUT_VALUE);
    
    /* If the program operation is completed, disable the PG Bit */
    FLASH->CR &= (~FLASH_CR_PG);  
  }
  
  /* Process Unlocked */
  __HAL_UNLOCK(&pFlash);
  
  return status;
}
```

在FLASH_Program_Word函数中，FLASH->CR \|= FLASH_CR_PG;打开了Flash的写入功能，一直到HAL_FLASH_Program里的FLASH->CR &= (~FLASH_CR_PG); 才关闭。

如果在这几句之间，产生了一个SysTick中断，0地址就被刷掉了！

下面这张图大概表示了整个执行过程：
<img src="/postimg/2019-12-14-STM32-Flash-Error-Overview.png" width="100%" />

回到Systick函数，问题在于，LED_Set在InitDone外面！也就是说，可能在GPIO没有初始化的时候，去设置了一下LED，向0地址写了个0进去，刷掉了堆栈指针。

但是大多数情况下Flash不是只读的吗？这里我一直以为跟电脑一样，向只读区域写数据会产生一个HardFault之类的错误。但实际并非如此，程序还会继续执行，只有Flash操作时候才会表现出问题。

把LED操作放到if内部之后，问题修复。就算不关中断也可以正常工作了。当然，最后关中断这个还是保留了下来，避免之后出其他的问题，毕竟Flash操作时候本来就不能响应中断，实际性能损失也不大。

PS：这个问题并不是我一人解决的，实际调试的时候是个概率发生的问题，并且对代码也不熟悉，花了两个周时间才搞定的。LED操作也很不起眼，很难被怀疑，甚至一度认为是芯片bug。
