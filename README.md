# dac-cosine


## Introduction

ESP32 is described by Espressif as [Wi-Fi + Bluetooth Combo Chip](https://espressif.com/en/products/hardware/esp32/overview). Besides the core wireless functionality it combines numerous other features like wired communication interfaces ranging from UART or I2S to CAN and Ethernet, several controllers like LED or MCPWM, as well as couple other building blocks including low noise amplifier, hall sensor or camera interface. Lurking in the shadows of these features is cosine waveform generator.

The purpose of this repository is to review functionality of ESP32's cosine waveform generator and develop a simple API to operate it. 


## Documentation is there

The cosine waveform (CW) generator is described in details in the [ESP32 Technical Reference Manual](http://espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf), but if we check API of [ESP-IDF](https://github.com/espressif/esp-idf), it has no any support implemented (as of August 2017). Fortunately, with commonly available information we can fairly easy put it into action.
 
Starting with ESP32 Technical Reference Manual, in chapter 24.7, we learn that the cosine waveform (CW) generator it is part of 8-bit DAC functional block and find the following functional diagram:

![alt text](pictures/dac.png "ESP32 DAC functional diagram (figure from ESP32 Technical Reference Manual)")
 
Diagram is showing "CW generator" box in upper left corner of diagram and two mux transfer blocks that are routed to DAC taking 8 bits on input on one side, and providing `dacn_out` analog output on the other.
 
Besides CW, DAC is also accepting input from DMA, to generate arbitrary waveform, or from `RTCIO_PAD_DACn_REG` register to provide ordinary conversion of 8-bit numeric value. Several “n” letters on this diagram indicate either 1 or 2 because DAC has two channels.

 
## Check basic DAC functionality

To start with we should explore how ordinary conversion of 8-bit numeric values is performed. Checking [DAC API](http://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/dac.html) in ESP-IDF, we will find couple of functions to enable DAC and output an analog value. Unfortunately these functions are not provided in source code. But checking further in [arduino-esp32](https://github.com/espressif/arduino-esp32) repository, we will find [esp32-hal-dac.c](https://github.com/espressif/arduino-esp32/blob/758553a786520b138307691e842e53b25606960b/cores/esp32/esp32-hal-dac.c) file that contains couple lines of code with comments that are behind Arduino `dacWrite` function. This should give us initial understanding which registers are used to provide digital value to be converted to analog output, and how to enable or disable the output. Checking further the [ESP32 Technical Reference Manual](http://espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf) and searching for `SENS_SAR_DAC_CTRL1_REG` we will easy find a “Register Summary” section 28.4, that provides this and other register names used in esp32-hal-dac.c with explanation of particular bits and functions.


## Use CW instead of ordinary DAC conversion

In next step, looking on ESP32 DAC Functional Diagram, we should figure out how to use CW generator block instead of `RTCIO_PAD_DACn_REG` register. Switching between these two is done with `SENS_DACn_CW_EN` bit. Searching for `SENS_DAC_CW` name, we will find it in ESP32 Technical Reference Manual under `SENS_SAR_DAC_CTRL2_REG` register together with explanation:

![alt text](pictures/explanation_SENS_DAC_CW_EN.png "Explanation of SENS_DAC_CW_ENn bits (from ESP32 Technical Reference Manual)")

In other words setting `SENS_DAC_CW_EN1` bit will connect CW generator, clearing it will connect `PDAC1_DAC[7:0]` as the input source of DAC on channel 1. One extra crosschecking of this description against ESP32 DAC Functional Diagram and [esp32-hal-dac.c](https://github.com/espressif/arduino-esp32/blob/758553a786520b138307691e842e53b25606960b/cores/esp32/esp32-hal-dac.c) will confirm that this is what we are looking for.

After additional studying of fields of `SENS_SAR_DAC_CTRL1_REG` register we will also discover `SENS_SW_TONE_EN` bit that is enabling the CW generator itself.

The last component to complete this exercise is setting of generator's frequency. Applicable information is provided in first paragraph of “Cosine Waveform Generator” section in ESP32 Technical Reference Manual, that “The frequency of CW can be adjusted by register SENS_SAR_SW_FSTEP[15:0]”. Looking further we will find a formula how to calculate it:

```
freq = dig_clk_rtc_freq x SENS_SAR_SW_FSTEP / 65536
```

Where the frequency of `dig_clk_rtc_freq` is typically 8 MHz.

Basing on this formula the theoretical range of generator's output `freq` should be from 122 Hz to 8 MHz. In later paragraphs will will find out if this is true or not.


## Enable cosine generator on DAC channel 1

Now we are ready to write the code to use CW generator. To connect it to channel 1 (GPIO25), the code will consists of the following steps:


1.	Enable CW generator by setting `SENS_SW_TONE_EN` bit in `SENS_SAR_DAC_CTRL1_REG` register.

	```c
	SET_PERI_REG_MASK(SENS_SAR_DAC_CTRL1_REG, SENS_SW_TONE_EN);
	```

2.	Connect generator to DAC channel 1 by setting `SENS_DAC_CW_EN1` bit in `SENS_SAR_DAC_CTRL2_REG` register.

	```c
	SET_PERI_REG_MASK(SENS_SAR_DAC_CTRL2_REG, SENS_DAC_CW_EN1_M);
	```

3.	Set generator's frequecy by writing a value to `SENS_SAR_SW_FSTEP[15:0]` field in `SENS_SAR_DAC_CTRL1_REG` register.

	```c
	SET_PERI_REG_BITS(SENS_SAR_DAC_CTRL1_REG, SENS_SW_FSTEP, frequency_step, SENS_SW_FSTEP_S);
	```

4. Enable output on DAC channel 1 by calling API function [dac_output_enable()](http://esp-idf.readthedocs.io/en/latest/api-reference/peripherals/dac.html#_CPPv217dac_output_enable13dac_channel_t).

	```c
	dac_output_enable(DAC_CHANNEL_1);
	```

After uploading code to ESP32 and connecting a scope to DAC channel 1 (GPIO25) we should see a similar waveform:

![alt text](pictures/initial-sine-waveform.png "Initial waveform from cosine generator")

The waveform looks promising but requires some extra bit tweaking.


## Fix the waveform

In initial waveform shown above the sinusoid halves are inverted. We should look for some additional configuration bits that let us revert the halves back. Further searching through DAC registers will reveal that `SENS_SAR_DAC_CTRL2_REG` register has two bit `SENS_DAC_INV` field with promising explanation:

- 00: does not invert any bits,
- 01: inverts all bits,
- 10: inverts MSB,
- 11: inverts all bits except for MSB.

This field currently contains default value of both bits cleared `00`, so we should try remaining three options by adding the following code:

```c
SET_PERI_REG_BITS(SENS_SAR_DAC_CTRL2_REG, SENS_DAC_INV1, invert, SENS_DAC_INV1_S);
```

Indeed, after setting `invert` bits to `10` (decimal 2), we will finally get expected sinusoidal waveform:

![alt text](pictures/expected-sine-waveform.png "Sinusoid waveform by ESP32's internal CW generator")

The blue waveform below is [FFT (Fast Fourier transform)](https://en.wikipedia.org/wiki/Fast_Fourier_transform), with strong initial peak we have been looking for.

The sinusoid itself has some random artifacts around zero and 180 phase, but I am not sure we can do anything about it. Later testing revealed no such artifacts at channel 2.


## What is the other functionality we can get?

[ESP32 Technical Reference Manual](http://espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf) specifies several parameters of CW we can adjust. It also provides applicable figure:

![alt text](pictures/dac2.png "Cosine generator parameter setting (figure from ESP32 Technical Reference Manual)")

This indicates we should be able to scale and add bias to the output with `SENS_DAC_SCALE` and `SENS_DAC_DC` registers. 

Let's also examine if we can widen the range of CW's output frequencies by changing the frequency of input clock signal `dig_clk_rtc`. If you are tracking development of [ESP-IDF](https://github.com/espressif/esp-idf), you might have noticed a [post by Ivan Grokhotkov about opening the source of #ESP32's clock functions](https://twitter.com/i_grr/status/852177740581052416). Checking provided link we will find several promising [RTC clock dividers](https://github.com/espressif/esp-idf/blob/6353bc40d7adaa64a414e36276e31912bc450bfd/components/soc/esp32/include/soc/rtc.h#L110-L118) and [their default settings](https://github.com/espressif/esp-idf/blob/6353bc40d7adaa64a414e36276e31912bc450bfd/components/soc/esp32/include/soc/rtc.h#L123-L131). After analyzing provided information and crosschecking it against section "3.2 System Clock" of [ESP32 Technical Reference Manual](http://espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf), we should be able to identify field `RTC_CNTL_CK8M_DIV_SEL` in register `RTC_CNTL_CLK_CONF_REG` that let us to step change the frequency of `dig_clk_rtc` signal. 


With all collected information we should be able to compile complete list of fields to tweak:

- Divider of RTC 8 MHz Clock - `RTC_CNTL_CK8M_DIV_SEL`
- Frequency Step - `SENS_SW_FSTEP`
- Scale - `SENS_DAC_SCALE`
- Offset - `SENS_DAC_DC`
- Inversion -`SENS_DAC_INV`

**Note:** while this is fine to change the frequency of RTC 8 MHz clock to tune operation of CW, it may affect operation of other RTC peripheries, that are using this clock signal. Check it, if you are planning to change default value of `RTC_CNTL_CK8M_DIV_SEL` as provided in ESP-IDF, in your application.

Having five parameters to manage and check, it would be reasonable to write an user interface to make it easier to adjust individual values.

Before you rush to write the UI, I propose an easier and quicker way to examine all parameters in action.


## JTAG makes testing easier

ESP32 is equipped with [JTAG functionality](http://esp-idf.readthedocs.io/en/latest/api-guides/jtag-debugging/index.html) that together with [OpenOCD](https://github.com/espressif/openocd-esp32) software make it much easier to test and troubleshoot the ESP32 hardware and application.

In this particular case we can use JTAG + OpenOCD to stop application execution, apply CW parameter changes and resume execution to see how CW responds.

Clone [this repository](https://github.com/krzychb/dac-cosine), build the code and load it to ESP32. Connect a scope to DAC channels 1 and 2 (GPIO25 and GPIO26).

The waveform on channel 1 will be set to default values of 1kHz frequecy without any scaling or offset. We will then tweak CW parameters on channel 2 to see how the output on this channel differs from the reference channel 1.

After running debugger establish a breakpoint inside main loop of `dactask()`. On 
"Expressions" tab add all CW parameters to tweak, see upper right corner of debugger window below.

![alt text](pictures/debugger-setup.png "Debugger setup")

Once ready you can change individual CW parameters. Then click "Resume" or press F8 to run the application. The application will use new parameter values and halt waiting for another change.

Application's output looks as follows:

```
clk_8m_div: 0, frequency step: 8, frequency: 1038 Hz
DAC2 scale: 1, offset 0, invert: 2
clk_8m_div: 0, frequency step: 1000, frequency: 129700 Hz
DAC2 scale: 1, offset 0, invert: 2
clk_8m_div: 7, frequency step: 1, frequency: 16 Hz
DAC2 scale: 1, offset 0, invert: 2
clk_8m_div: 0, frequency step: 1, frequency: 130 Hz
DAC2 scale: 1, offset 0, invert: 2
clk_8m_div: 3, frequency step: 8, frequency: 259 Hz
DAC2 scale: 1, offset 0, invert: 2
clk_8m_div: 1, frequency step: 4, frequency: 259 Hz
DAC2 scale: 1, offset 0, invert: 2
```

Below is an example of output of channel 1 (red trace = reference) and channel 2 (yellow trace) with scale set to 1/2 and inversion of all bits except MSB. 

![alt text](pictures/comparing-dac-output.png "Comparing output on DAC channels")

JTAG interface operated from [Eclipse IDE](https://eclipse.org/) makes is easy and intuitive to interact with the application that has no user interface provided. Within couple of minutes it was possible to probe critical parameter and e.g. discover that at frequencies of around 100 kHz the waveform it is getting visible saw noise.

![alt text](pictures/saw-noise-100kHz.png "Saw noise is getting visible at 100kHz")

The lowest frequency that can be set is around 130 Hz without changing default value (0b000) of RTC 8 MHz clock divider, or 16 Hz if divider is set to maximum 0b111.


## Conclusion

ESP32 has on board cosine waveform generator with adjustable frequency, scale and offset. The waveform has 8-bit resolution and can be output to GPIO25 (channel 1) and / or GPIO26 (channel 2) pins. Scale and offset can be set individually per channel. Frequency setting is common to both channels and ranges from 130 Hz to about 100 kHz, where conversion artifacts became visible. It is possible to lower bottom frequency to 16 Hz with setting non default `clk_8m_div` divider. The waveforms are generated by ESP32's hardware without any overhead on CPUs. 

![alt text](pictures/test-setup.jpg "Setup used for testing of cosing waveform generator of ESP32")


## Contribute

Feel free to contribute to the project in any way you like!

If you find any issues with code or description please report them using *Issues* tab above.


## Credits

This small repository has been prepared thanks to great work of the following teams and individuals:

* Espressif team that develops and maintains [esp-idf](https://github.com/espressif/esp-idf) and [openocd-esp32](https://github.com/espressif/openocd-esp32) repositories.
* [Ivan Grokhotkov](https://twitter.com/i_grr) who provided tips how to operate CW generator.
* [robert-hh](https://github.com/robert-hh), developer of excellent tool [owonread](https://github.com/robert-hh/owonread) to get screen shots and track data data from Owon TDS series Oscilloscopes with Linux.


## License

[Apache License Version 2.0, January 2004](LICENSE)