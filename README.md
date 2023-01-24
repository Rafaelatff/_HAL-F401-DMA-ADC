# _HAL-F401-DMA-ADC
This repository was create to follow the content of the course 'ARM Cortex M Microcontroller DMA Programming Demystified'.  

I am using:  
* NUCLEO-F401RE Board 
* STM32Cube FW_F4 V1.27.1 
* STM32CubeIDE version 1.10.1

# STM32CubeMX

We won't change the default clock configuration. APB2 will be running at 84 MHz.

On STM32CubeMX, at 'Analog' we go to 'ADC1', then we set the 'Temperature Sensor Channel'. We set the resolution to '12 bits' (maximum option) and we use 'Right alignment' (data to the right side of the byte, on LSB side). We enable the 'Continuous Conversion Mode' and we set the 'Number Of Conversion' to 16, this way it will continuous do the conversion 16 times. We also enable the 'DMA Continuous Requests' since we will be using the DMA to take the data from ADC1 and move to the SRAM1 memory. The 'External Trigger Conversion Source' is set to 'Regular Conversion launched by software' because our code will trigger the request for the ADC start the conversion. A differente source could be use, such as the available timers of that microcontroller.

![image](https://user-images.githubusercontent.com/58916022/214272413-f6a47be6-7d53-4be3-ab6e-d40f4a94759c.png)

We can see, by following the 'Figure 3. STM32F401xD/xE block diagram' on datasheet that the DMA2 must be used since it is the only one connected to ADC/

![image](https://user-images.githubusercontent.com/58916022/214271252-074f2f7d-d668-4691-bda0-e3697663d51e.png)

We have the 'Table 29. DMA2 request mapping (STM32F401xB/C and STM32F401xD/E)' on Reference Manual that shows the possible channel and stream to connect ADC1 to DMA2.

![image](https://user-images.githubusercontent.com/58916022/214270719-bca9f38e-4eaa-47f7-acf0-466cf4bba74c.png)

On DMA configuration we use one of the available streams. We configure to Increment the memory address (to allocate all the 16 conversions of our ADC) and set the 'Data Width' to 'Half Word' (16 bits, once we only use 12 bits).

![image](https://user-images.githubusercontent.com/58916022/214275133-31ca1890-442e-45bd-a79c-9a33033232d3.png)

In the 'System Core' -> 'NVIC' option, we make sure that the 'ADC1 global interrupts' is unchecked, we don't want that ADC calls an interrupt for the ARM controller (we are using DMA, so it won't be needed). Then we make sure that the 'DMA2 streamx global interrupt' option is ticked and we also tick the 'EXTI line [15:10] interrupts' to generate an interrupt every time buttom 'B1' is pressed. The Temperature read will start always that the 'B1' is pressed.

![image](https://user-images.githubusercontent.com/58916022/214276809-edd4992f-b85b-4c90-9c12-bf989d9bb328.png)

And before we generate the code, we configure the 'Code generation' to generate the IRQ Handler for the following devices:

![image](https://user-images.githubusercontent.com/58916022/214277536-60123e44-7bc4-4ac5-b472-7dfff7a497d4.png)

# STM32CubeIDE
