## Diverting surplus PV Power, by Robin Emley

[<< 5: Mk2 Controller Operating Modes](modes)

[7: Calibration of power and voltage, plus phase-alignment >>](calibration)

### 6: Different ways of Measuring Voltage and Current

The hardware for measuring power using non-intrusive sensors is fully described in the Building Blocks section at [https://openenergymonitor.org/emon/buildingblocks](https://openenergymonitor.org/emon/buildingblocks). Behind the hardware lies some associated software in the OEM’s library files. The standard code for measuring and calculating various parametric values including Power may be found in the `calcVI()`function in the source file `emonLib.cpp`. Instructions for downloading the EmonLib library are given at [Arduino sketch - voltage and current](https://openenergymonitor.org/emon/buildingblocks/arduino-sketch-voltage-and-current).

Essentially, _pairs of voltage and current measurements are multiplied together and accumulated over a period of time to reveal the average power that is present_.

Samples of voltage and current are alternately produced by the Analogue-to-Digital Converter (ADC) which is part of the micro-controller. For most constructors, this is the ATmega328 which is the main component in an [Arduino Uno](http://arduino.cc/en/Main/ArduinoBoardUno) or equivalent system. The input to the ADC is single-ended, and operates from 0V to some positive voltage, usually 3.3V or 5V. Each sensor must ensure that its output signal is correctly biassed near the mid-point of the ADC’s input range.

If an input signal is correctly biassed, the ADC’s output will be centred close to the mid-point of the output range. The ADC is normally run in its 10-bit mode which gives integer values from 0 to 1023\. Each output stream of voltage and current samples would therefore normally be centred at around 512.

The standard means of processing voltage and current samples is to pass them through a pair of independent high-pass filters (HPF). This removes the DC content so that each waveform is then centred around zero. Pairs of samples can then be multiplied together to find the instantaneous power that they represent. These values are accumulated over the period of interest to reveal the average power that is present.

_So far, I have described just the conventional approach that is used in the standard OEM sketch for measuring power. The remainder of this section is a summary of the further development that I and others have undertaken for the specific purpose of diverting surplus power to a dump load more efficiently._

Any conventional filter, whether high-pass or low-pass, introduces some distortion. While developing the Mk2 PV Router, I devised a variant of low-pass filter which had minimal effect on the underlying waveform. This improvement was achieved by updating the LPF only once per mains cycle rather than after each individual sample. Having ascertained the DC-offset using the LP filter, this value could then be subtracted from each raw sample to remove the DC offset.

The above variant of LPF gives excellent performance, but suffers from the minor problem that it can sometimes fail to start up correctly. Two solutions to this problem have been devised and implemented. One approach is to retain the HPF on the voltage stream, which allows voltage samples to be reliably grouped together into mains cycles. This is how the original Mk2 build was constructed. The second approach is to simply prevent the output from the LPF from ever straying too far away from the expected value of 512\. The Mk2a_rev3 and Mk2i versions use this method. Both techniques have been found to work reliably, the second one being very much simpler and less demanding on processor time than the first.

Each operation of the ADC takes a finite amount of time, around 104µs. Having taken a pair of samples, subsequent processing needs to be performed. When using the standard maths, processing of the sample values takes a similar amount of time to the pair of ADC conversions.

Early Mk2 builds were limited to this rate of operation, with only around 50 sample pairs (i.e. "loops" of the code) per mains cycle. It then became apparent that the time while ADC conversions were in progress could be beneficially used by the main processor. By rearranging the code, I found that all of the general processing could be neatly hidden away within these periods, and the Arduino still had plenty of time to spare. The number of loops per mains cycle correspondingly rose to around 89\. This version was released as Mk2a_rev2.

A further version (Mk2i) was released after it became apparent that the ADC could be controlled in a better way by using interrupts. For absolute speed, the “free-running” mode is best, this giving around 96 loops per mains cycle. This corresponds to the flat-out rate of the ADC, so no further improvement in speed seems likely. A second option is to use a fixed timer within the Arduino’s hardware. One advantage of this approach is that the operation can be slowed down as required to allow additional tasks to be performed such as communication with external equipment. When running in this mode at 50 loops per mains cycle, the Arduino’s main processor is only around 25% occupied.

The above improvements in speed have also been made possible by the use of integer maths, this being very much faster than the standard floating-point maths which was used in the original OEM code. One consequence of this change is that the energy bucket on later Mk2 builds has had to be re-scaled. Although still corresponding to an energy range of 3600 J (1 Wh), its numerical capacity is instead some large number in excess of a million. When developing software tools, where the ability to understand the code’s operation is of prime importance, the original floating point maths and scaling have been retained. But for sheer performance, integer maths is in a league of its own.

When the full range of parameters are required, such as Power Factor, it is necessary to remove the DC content from both the voltage and current streams before further calculations can proceed. This is how the standard OEM calculations are undertaken. However, when only “real power” is required, it turns out that there is no need to remove the DC from one of these sets of values. This is because the DC content gets cancelled out within the maths. Not removing the DC from the current stream saves on processing time. In practice, to avoid the system becoming over-sensitive to random effects, it is prudent to remove a nominal amount of DC from each raw current sample. The actual value, however, is not important, and this approach has been satisfactorily used in Mk2a_rev3 and all versions of Mk2i.

[<< 5: Mk2 Controller Operating Modes](modes)

[7: Calibration of power and voltage, plus phase-alignment >>](calibration)