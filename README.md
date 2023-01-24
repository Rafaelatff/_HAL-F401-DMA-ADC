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

All initializations are made according to our configuration on MX software. Lesson 51 shows and explain the generataded codes. It is importante to hightlit that the LINK between ADC and DMA is made on source file 'stm32f4xx_hal_msp.c' by passing both handle types (hadc and hdma_adc1) to the API __HAL_LINKDMA.

```c
  __HAL_LINKDMA(hadc,DMA_Handle,hdma_adc1);
```

Inside the *EXTI15_10_IRQHandler* the interrupt is already configurated to be generated onde 'B1' buttom is pressed. Then we just need to add to start the reading of the ADC1 by using the API *HAL_ADC_Start_DMA*. When we add the address for the adc1 handle, a syntax erros is generated. We need to call inside the 'USER CODE BEGIN EV' the *extern ADC_HandleTypeDef hadc1;*.

![image](https://user-images.githubusercontent.com/58916022/214282128-ba657f43-54e0-4e1a-8182-ac2017f04726.png)

Then we need, inside the 'USER CODE BEGIN PV' to add the array for the 16 readings of temperature data *uint16_t temp_data[16];*. We use type uint16_t since ADC is only 12 bits. We pass this address to the API *HAL_ADC_Start_DMA*, but typecasted to uint32_t type. Then, as a last parameter, we send the 'Length' parameter, that is the length of data to be transferred from ADC peripheral to memory.

```c
/**
  * @brief This function handles EXTI line[15:10] interrupts.
  */
void EXTI15_10_IRQHandler(void)
{
  /* USER CODE BEGIN EXTI15_10_IRQn 0 */
	HAL_ADC_Start_DMA(&hadc1, (uint32_t *) temp_data, 16);
  /* USER CODE END EXTI15_10_IRQn 0 */
  HAL_GPIO_EXTI_IRQHandler(B1_Pin);
  /* USER CODE BEGIN EXTI15_10_IRQn 1 */
  /* USER CODE END EXTI15_10_IRQn 1 */
}
```

In 'USER CODE BEGIN 4' we call the Convertion Complete Callback for the ADC. Just copy and removed the __weak at the begin of the function.

# Debug 1

We add a breakpoing at the end of the callback function (and end bracket '}'). Then we added the 'temp_data' variable to the the 'Expressions' window and we also were able to see its address.

![image](https://user-images.githubusercontent.com/58916022/214288940-7d030427-4a0c-4155-b691-21acd1f47b0d.png)

We also added this address to the Memory Monitor:

![image](https://user-images.githubusercontent.com/58916022/214289123-224b9918-b045-4d26-beff-1f68d89d51e8.png)

Then we also added the 'temp_data' to the 'Live Expressions' windows, we started to run the application and pressed B1 on the board. The code run up to the breakpoint and we were able to see the data of the ADC reading going to the 'temp_data' address.

![image](https://user-images.githubusercontent.com/58916022/214290469-caaa9071-01e4-4d3f-a46e-4e4a610c7333.png)

# Reading the values in °C

In the reference manual we have the formula for the temperature reading. The values inside the temp_data are 16 bits and correspond to the VSENSE.

![image](https://user-images.githubusercontent.com/58916022/214291335-9ff1a5cb-80e8-4ac6-a00c-ebdaa89b3fa6.png)

Our ADC resolution is = 'Vref' / (2^bits_resolution -1) = 3.3V / (2^12 -1) = 3.3/4095 [V]

The V25 and Avg_Slope can be found in the datasheet, and in our case is: 

![image](https://user-images.githubusercontent.com/58916022/214292370-1261b265-484e-4c46-80c9-08bfc87830ed.png)

By doing the math (for the first reading) we have: 

* 0xB103 = 0b1011 0001 0000 0011 -> taking the 12 bits to the right 0x0103 = 0b0001 0000 0011.
* Decimal value of reading = 259.
* VSENSE = 259 * 3.3/4095 = 0,20871794871794871794871794871795.
* Temperature (in °C) = {(VSENSE – V25) / Avg_Slope} + 25.
* Temperature (in °C) = {(0,208717.. - 0.76) / 2.5} + 25 = 24,779487179487179487179487179487 °C.




