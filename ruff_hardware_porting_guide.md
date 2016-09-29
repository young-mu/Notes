# HW interface porting manual

## PWM / ADC / QEI / I2C / UART

> 这里拿PWM举例，同样适用于ADC，QEI，I2C，UART等接口

### 层次（由高至低）

![RuffHWStack1](https://raw.githubusercontent.com/young-mu/notes/master/res/ruff_hw_stack1.png)

1. **JS层-应用程序**

	直接利用设备驱动提供的API对该驱动型号的设备进行操作，如`$('#gy-30').setRGB(255, 0, 0)`控制三色LED设备显示红色

2. **JS层-设备驱动** 

	[`gy-30`](https://github.com/ruff-drivers/gy-30)

   在`driver.json`中申请若干PWM资源（如`pwmA`/`pwmB`/`pwmC`），配置各个PWM的参数（如`frequency`），在`src/index.js`中定义协同若干PWM完成某一事件的函数（如`setRGB()`)

3. **JS层-接口驱动**

	`nuttx/ruff_modules/mcu-pwm/src/index.js`

   按照ruff-driver框架，封装JS层访问接口，向外暴露PWM接口API，如`set_duty()`/`set_frequency()`等，给PWM设备驱动使用
   
   设备驱动`$('#pwm-N')`返回的是`SysPwmInterface`类型对象，内部含有`handle`资源，`handle`含有`fd`/`duty`/`freqency`资源，因此，一个`SysPwmInterface`对象就代表一个PWM对象

4. **Glue层-JS/C绑定**

	`apss/ruff/pwm_jerry.c`

   用JerryScript封装Native层提供的API，提供JS层访问接口
   
   在接口驱动的实现中，当调用了`ruff.pwm.open()`后，返回`handle`资源对象
  
   > 理论来说，本层应只有逻辑而没有数据

5. **Native层-驱动操作封装**

	`apps/ruff/pwm.c`

   封装驱动上半部的访问接口，使接口更加简单易用
   
   驱动上半部的访问接口就是标准文件的访问接口，通过`open`/`close`/`write`/`read`/`ioctl`等API操作硬件接口的设备文件

6. **Nuttx驱动上半部**

	`nuttx/drivers/pwm.c`

   定义了PWM设备文件的操作方法（如`pwm_open()`，其内部调用PWM驱动下半部实现的`setup()`），本层提供PWM驱动设备标准操作框架，目的示能够用标准IO操作PWM统一设备文件（如`/dev/pwm0`）
   
   > 本层**平台无关**，为不同平台提供了统一操作PWM设备的标准方法，既有驱动下半部必须实现的，也有驱动下半部可独立扩展的

7. **Nuttx驱动下半部**

	`nuttx/arch/arm/src/tiva/tiva_pwm.c`

   参考芯片手册，实现不同芯片上指定实现的方法，以供驱动上半部框架层调用
   
   对于PWM，必须实现这些方法`.setup`/`.shutdown`/`.start`/`.stop`，可在`.ioctl`中实现自己平台上的扩展方法
   
   > 本层**平台相关**
  
## GPIO / EEPROM

> 这里拿GPIO举例，同样适用于内置EEPROM

相比于PWM等接口，在Nuttx中，并未对GPIO和EEPROM提供统一驱动设备文件的支持，也就是说，在Nuttx平台上写的GPIO/EEPROM测试程序是平台相关的，但Ruff会提供一层封装，已保证JS层代码可以跨硬件平台

### 层次（由高至低）

![RuffHWStack2](https://raw.githubusercontent.com/young-mu/notes/master/res/ruff_hw_stack2.png)

1. **JS层-应用程序**

	（同上，略）

2. **JS层-设备驱动**

	（同上，略）

3. **JS层-接口驱动** 

	（同上，略）

4. **Glue层-JS/C绑定**

	（同上，略）

5. **Native层-驱动操作封装**

	`apps/ruff/board/ti/tiva/gpio.c`

    在内部需要完成Ruff指定的GPIO操作API，如`ruff_gpio_write`/`ruff_gpio_read`等，在这些API内部，调用不同芯片的GPIO操作函数完成相应功能，这些GPIO的操作函数由不同芯片提供的GPIO**驱动文件**（平台相关）（如`nuttx/arch/arm/src/tiva/tiva_pwm.c`）或**on-chip ROM API**提供

6. **Nuttx驱动** / **on-chip ROM API**

	`nuttx/arch/arm/src/tiva/tiva_gpio.c`
	
	Nuttx驱动和on-chip ROM API均提供操作硬件接口的方法，但后者并不是所有MCU都支持，但TI/Tiva TM4C129x系列MCU支持，它是指在芯片内部一块只读的ROM上，封装好了硬件接口的操作函数，用户可直接调用这些函数来操作接口 
	
	REF: [Tiva C Series TM4C129x ROM User's Guide](http://www.ti.com/lit/ug/spmu363a/spmu363a.pdf)
