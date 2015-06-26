Si5351 Library for avr-gcc on ATtiny Microcontrollers
=====================================================
This is a basic library for the Si5351 series of clock generator ICs from [Silicon Labs][1] for the avr-gcc development environment. It will allow you to control the Si5351 with an AVR ATtiny microcontroller with a USI peripheral module and preferably at least 8 kB of flash memory, without depending on the proprietary ClockBuilder software from Silicon Labs.

This library is focused towards usage in RF/amateur radio applications, but it may be useful in other cases. However, keep in mind that coding decisions are and will be made with those applications in mind first, so if you need something a bit different, please do fork this repository. Also, since the Si5351A3 version is the one which seems most useful in amateur radio applications, this is where the current development will be focused. Once the Si5351A3 has a decent and mature feature set, hopefully we will be able to turn to the 8-output version, and perhaps even the B and C variants.

Hardware Requirements and Setup
-------------------------------
An 8-bit AVR ATtiny microcontroller with the USI peripheral is required for this library. The code size of the library, compiled with a simple program to set the output frequency on all three clock outputs of a Si5351A3 is 3826 bytes (using avr-gcc version 4.8.2), therefore it is best used with a microcontroller with at least 8 kB of flash memory.

The Si5351 is a +3.3 V only part, so if you are not using a +3.3 V microcontroller, be sure you have some kind of level conversion strategy.

Wire the SDA and SCL pins of the Si5351 to the corresponding pins on the AVR. Since the I2C interface is set to 400 kHz, use 1 k&Omega; pullup resistors from +3.3 V to the SDA and SCL lines.

Connect a 25 MHz or 27 MHz crystal with a load capacitance of 6, 8, or 10 pF to the Si5351 XA and XB pins. Locate the crystal as close to the Si5351 as possible and keep the traces as short as possible. Please use a SMT crystal. A crystal with leads will have too much stray capacitance.

Usage
-----
Include the si5351-avr-tiny-minimal.c, si5351-avr-tiny-minimal.h, USI_TWI_Master.c, and USI_TWI_Master.h files into your avr-gcc project as you would with any other standard project.

The functions of the library are documented in the code. It should be fairly self-explanatory, but here's a very short introduction.

Before you do anything with the Si5351, you will need to initialize the communications and the IC. Let's initialize communications with the Si5351, specify the load capacitance of the reference crystal, and to use the default reference oscillator frequency of 25 MHz:

    si5351_init(SI5351_CRYSTAL_LOAD_8PF, 0)

Next, let's set the CLK0 output to 14 MHz):

    si5351_set_freq(14000000ULL, SI5351_CLK0);

If we like we can adjust the output drive power:

    si5351_drive_strength(SI5351_CLK0, SI5351_DRIVE_4MA);

Also, there will be some inherent error in the reference crystal's actual frequency, so we can measure the difference between the actual and nominal output frequency in Hz, multiply by 10, make it an integer, and enter this correction factor into the library. With an accurate measurement at one frequency, this calibration should be good across the entire tuning range:

    si5351_set_correction(-900);

One thing to note: the library is set for a 25 MHz reference crystal. If you are using a 27 MHz crystal, please enter the reference oscillator frequency as the 2nd parameter in the si5351_init() function.

Also, the si5351_init() function sets the crystal load capacitance for 8 pF. Change this value if you are using a crystal with a different load capacitance.

Setting the Output Frequency
----------------------------
As indicated above, the library accepts and indicates clock frequencies in units of 1 Hz, as an _unsigned long_ variable type (or _uint32_t_). When entering literal values, append "UL" to make an explicit _unsigned long_ number to ensure proper tuning.

The most simple way to set the output frequency is to let the library pick a PLL assignment for you. You do this by using the si5351_set_freq() function, which will use a PLL frequency of 900 MHz and assign all multisynths (clock outputs) to PLLA. This function will also calculate the PLL parameters based on the correction factor set with the si5351_set_correction() function:

    si5351_set_freq(1014000000ULL, SI5351_CLK0);

You may also manually set the output frequency by specifiying the A, B, and C divider values for both the PLLs and multisynths (each fractional synth divides by `a * b/c`):

    struct Si5351Frac pll_frac, ms_frac;

    // Set PLLA to 600 MHz (600 MHz / 25 MHz = 24)
    pll_frac.a = 24;
    pll_frac.b = 0;
    pll_frac.c = 1;
    si5351_set_pll(pll_frac, SI5351_PLLA);

    // Set MS0 (CLK0) to 10 MHz (600 MHz / 10 MHz = 60)
    ms_frac.a = 60;
    ms_frac.b = 0;
    ms_frac.c = 1;
    si5351_set_ms(SI5351_CLK0, ms_frac, 0, SI5351_OUTPUT_CLK_DIV_1, 0);

Note that this does not use si5351_set_correction() to integrate the correction factor into the PLL tuning. Consult the si5351_set_freq() code to see how that is accomplished.

Further Details
---------------
If we like we can adjust the output drive power:

    si5351_drive_strength(SI5351_CLK0, SI5351_DRIVE_4MA);

The drive strength is the amount of current into a 50&Omega; load. 2 mA roughly corresponds to 7 dBm output and 8 mA is approximately 14 dBm output.

Individual outputs can be turned on and off. In the second argument, use a 0 to disable and 1 to enable:

    si5351_output_enable(SI5351_CLK0, 0);

You may invert a clock output signal by using this command:

    si5351_set_clock_invert(SI5351_CLK0, 1);

Calibration
-----------
There will be some inherent error in the reference oscillator's actual frequency, so we can account for this by measuring the difference between the uncalibrated actual and nominal output frequencies, then using that difference as a correction factor in the library. The set_correction() method uses a signed integer calibration constant measured in parts-per-ten million. The easist way to determine this correction factor is to measure the actual frequency of a 10 MHz signal from one of the clock outputs, calculate how many Hz it differs from the nominal frequency of 10000000 Hz, then use that number in the set_correction() method in future use of this particular reference oscillator. Once this correction factor is determined, it should not need to be measured again for the same reference oscillator/Si5351 pair unless you want to redo the calibration. With an accurate measurement at one frequency, this calibration should be good across the entire tuning range.

The calibration method is called like this:

    si5351_set_correction(-61);

One thing to note: the library is set for a 25 MHz reference crystal. If you are using a 27 MHz crystal, use the second parameter in the si5351_init() function to specify that as the reference oscillator frequency.

Phase
------
The phase of the output clock signal can be changed by using the set_phase() method. Phase is in relation to (and measured against the period of) the PLL that the output multisynth is referencing. When you change the phase register from its default of 0, you will need to keep a few considerations in mind.

Setting the phase of a clock requires that you manually set the PLL and take the PLL frequency into account when calculation the value to place in the phase register. As shown on page 10 of Silicon Labs Application Note 619 (AN619), the phase register is a 7-bit register, where a bit represents a phase difference of 1/4 the PLL period. Therefore, the best way to get an accurate phase setting is to make the PLL an even multiple of the clock frequency, depending on what phase you need.

If you need a 90 degree phase shift (as in many RF applications), then it is quite easy to determine your parameters. Pick a PLL frequency that is an even multiple of your clock frequency (remember that the PLL needs to be in the range of 600 to 900 MHz). Then to set a 90 degree phase shift, you simply enter that multiple into the phase register. Remember when setting multiple outputs to be phase-related to each other, they each need to be referenced to the same PLL.

You can see this in action in a sketch in the examples folder called _si5351phase_. It shows how one would set up an I/Q pair of signals at 14.1 MHz.

    // We will output 14.1 MHz on CLK0 and CLK1.
    // A PLLA frequency of 705 MHz was chosen to give an even
    // divisor by 14.1 MHz.
    struct Si5351Frac pll_frac, ms_frac;

    // Set PLLA to 705 MHz (600 MHz / 25 MHz = 28.2)
    pll_frac.a = 28;
    pll_frac.b = 2;
    pll_frac.c = 10;
    si5351_set_pll(pll_frac, SI5351_PLLA);

    // Set CLK0 and CLK1 to use PLLA as the MS source
    si5351.set_ms_source(SI5351_CLK0, SI5351_PLLA);
    si5351.set_ms_source(SI5351_CLK1, SI5351_PLLA);

    // Set CLK0 and CLK1 to output 14.1 MHz (705 MHz / 14.1 MHz = 50)
    ms_frac.a = 50;
    ms_frac.b = 0;
    ms_frac.c = 1;
    si5351_set_ms(SI5351_CLK0, ms_frac, 0, SI5351_OUTPUT_CLK_DIV_1, 0);
    si5351_set_ms(SI5351_CLK1, ms_frac, 0, SI5351_OUTPUT_CLK_DIV_1, 0);

    // Now we can set CLK1 to have a 90 deg phase shift by entering
    // 50 in the CLK1 phase register, since the ratio of the PLL to
    // the clock frequency is 50.
    si5351_set_phase(SI5351_CLK0, 0);
    si5351_set_phase(SI5351_CLK1, 50);

    // We need to reset the PLL before they will be in phase alignment
    si5351_pll_reset(SI5351_PLLA);


Constraints
-----------
* Two multisynths cannot share a PLL with when both outputs are < 1.024 MHz or >= 112.5 MHz.

* Setting phase will be limited in the extreme edges of the output tuning ranges. Because the phase register is 7-bits in size and is denominated in units representing 1/4 the PLL period, not all phases can be set for all output frequencies. For example, if you need a 90&deg; phase shift, the lowest frequency you can set it at is 4.6875 MHz (600 MHz PLL/128).

Functions
--------------
###si5351_init()
```
/*
 * si5351_init(uint8_t xtal_load_c, uint32_t ref_osc_freq)
 *
 * Setup communications to the Si5351 and set the crystal
 * load capacitance.
 *
 * xtal_load_c - Crystal load capacitance. Use the SI5351_CRYSTAL_LOAD_*PF
 * defines in the header file
 * ref_osc_freq - Crystal/reference oscillator frequency in 1 Hz increments.
 * Defaults to 25000000 if a 0 is used here.
 *
 */
void si5351_init(uint8_t xtal_load_c, uint32_t ref_osc_freq)
```
###si5351_set_freq()
```
/*
 * si5351_set_freq(uint64_t freq, enum si5351_clock clk)
 *
 * Uses SI5351_PLL_FIXED (900 MHz) for PLLA.
 * All multisynths are assigned to PLLA using this function.
 * PLLA is set to 900 MHz.
 * Restricted to outputs from 1 to 150 MHz.
 * If you need frequencies outside that range, use set_pll()
 * and set_ms() to set the synth dividers manually.
 *
 * freq - Output frequency in Hz
 * clk - Clock output
 *   (use the si5351_clock enum)
 */
uint8_t si5351_set_freq(uint32_t freq, enum si5351_clock clk)
```
###si5351_set_pll()
```
/*
 * si5351_set_pll(struct Si5351Frac frac, enum si5351_pll target_pll)
 *
 * Set the specified PLL to a specific oscillation frequency by
 * using the Si5351Frac struct to specify the synth divider ratio.
 *
 * frac - PLL fractional divider values
 * target_pll - Which PLL to set
 *     (use the si5351_pll enum)
 */
void si5351_set_pll(struct Si5351Frac frac, enum si5351_pll target_pll)
```
###si5351_set_ms()
```
/*
 * si5351_set_ms(enum si5351_clock clk, struct Si5351Frac frac, uint8_t int_mode, uint8_t r_div, uint8_t div_by_4)
 *
 * Set the specified multisynth parameters.
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * frac - Synth fractional divider values
 * int_mode - Set integer mode
 *  Set to 1 to enable, 0 to disable
 * r_div - Desired r_div ratio
 * div_by_4 - Set Divide By 4 mode
 *   Set to 1 to enable, 0 to disable
 */
void si5351_set_ms(enum si5351_clock clk, struct Si5351Frac frac, uint8_t int_mode, uint8_t r_div, uint8_t div_by_4)
```
###si5351_output_enable()
```
/*
 * si5351_output_enable(enum si5351_clock clk, uint8_t enable)
 *
 * Enable or disable a chosen output
 * clk - Clock output
 *   (use the si5351_clock enum)
 * enable - Set to 1 to enable, 0 to disable
 */
void si5351_output_enable(enum si5351_clock clk, uint8_t enable)
```
###si5351_drive_strength()
```
/*
 * si5351_drive_strength(enum si5351_clock clk, enum si5351_drive drive)
 *
 * Sets the drive strength of the specified clock output
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * drive - Desired drive level
 *   (use the si5351_drive enum)
 */
void si5351_drive_strength(enum si5351_clock clk, enum si5351_drive drive)
```
###si5351_update_status()
```
/*
 * si5351_update_status(void)
 *
 * Call this to update the status structs, then access them
 * via the dev_status and dev_int_status global variables.
 *
 * See the header file for the struct definitions. These
 * correspond to the flag names for registers 0 and 1 in
 * the Si5351 datasheet.
 */
void si5351_update_status(void)
```
###si5351_set_correction()
```
/*
 * si5351_set_correction(int32_t corr)
 *
 * Use this to set the oscillator correction factor to
 * EEPROM. This value is a signed 32-bit integer of the
 * parts-per-10 million value that the actual oscillation
 * frequency deviates from the specified frequency.
 *
 * The frequency calibration is done as a one-time procedure.
 * Any desired test frequency within the normal range of the
 * Si5351 should be set, then the actual output frequency
 * should be measured as accurately as possible. The
 * difference between the measured and specified frequencies
 * should be calculated in Hertz, then multiplied by 10 in
 * order to get the parts-per-10 million value.
 *
 * Since the Si5351 itself has an intrinsic 0 PPM error, this
 * correction factor is good across the entire tuning range of
 * the Si5351. Once this calibration is done accurately, it
 * should not have to be done again for the same Si5351 and
 * crystal.
 */
void si5351_set_correction(int32_t corr)
```
###si5351_set_phase()
```
/*
 * si5351_set_phase(enum si5351_clock clk, uint8_t phase)
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * phase - 7-bit phase word
 *   (in units of VCO/4 period)
 *
 * Write the 7-bit phase register. This must be used
 * with a user-set PLL frequency so that the user can
 * calculate the proper tuning word based on the PLL period.
 */
void si5351_set_phase(enum si5351_clock clk, uint8_t phase)
```
###si5351_pll_reset()
```
/*
 * si5351_pll_reset(enum si5351_pll target_pll)
 *
 * target_pll - Which PLL to reset
 *     (use the si5351_pll enum)
 *
 * Apply a reset to the indicated PLL.
 */
void si5351_pll_reset(enum si5351_pll target_pll)
```
###si5351_set_ms_source()
```
/*
 * si5351_set_ms_source(enum si5351_clock clk, enum si5351_pll pll)
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * pll - Which PLL to use as the source
 *     (use the si5351_pll enum)
 *
 * Set the desired PLL source for a multisynth.
 */
void si5351_set_ms_source(enum si5351_clock clk, enum si5351_pll pll)
```
###si5351_set_int()
```
/*
 * si5351_set_int(enum si5351_clock clk, uint8_t int_mode)
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * enable - Set to 1 to enable, 0 to disable
 *
 * Set the indicated multisynth into integer mode.
 */
void si5351_set_int(enum si5351_clock clk, uint8_t enable)
```
###si5351_set_clock_pwr()
```
/*
 * si5351_set_clock_pwr(enum si5351_clock clk, uint8_t pwr)
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * pwr - Set to 1 to enable, 0 to disable
 *
 * Enable or disable power to a clock output (a power
 * saving feature).
 */
void si5351_set_clock_pwr(enum si5351_clock clk, uint8_t pwr)
```
###si5351_set_clock_invert()
```
/*
 * si5351_set_clock_invert(enum si5351_clock clk, uint8_t inv)
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * inv - Set to 1 to enable, 0 to disable
 *
 * Enable to invert the clock output waveform.
 */
void si5351_set_clock_invert(enum si5351_clock clk, uint8_t inv)
```
###si5351_set_clock_source()
```
/*
 * si5351_set_clock_source(enum si5351_clock clk, enum si5351_clock_source src)
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * src - Which clock source to use for the multisynth
 *   (use the si5351_clock_source enum)
 *
 * Set the clock source for a multisynth (based on the options
 * presented for Registers 16-23 in the Silicon Labs AN619 document).
 * Choices are XTAL, CLKIN, MS0, or the multisynth associated with
 * the clock output.
 */
void si5351_set_clock_source(enum si5351_clock clk, enum si5351_clock_source src)
```
###si5351_set_clock_disable()
```
/*
 * si5351_set_clock_disable(enum si5351_clock clk, enum si5351_clock_disable dis_state)
 *
 * clk - Clock output
 *   (use the si5351_clock enum)
 * dis_state - Desired state of the output upon disable
 *   (use the si5351_clock_disable enum)
 *
 * Set the state of the clock output when it is disabled. Per page 27
 * of AN619 (Registers 24 and 25), there are four possible values: low,
 * high, high impedance, and never disabled.
 */
void si5351_set_clock_disable(enum si5351_clock clk, enum si5351_clock_disable dis_state)
```
###si5351_set_clock_fanout()
```
/*
 * si5351_set_clock_fanout(enum si5351_clock_fanout fanout, uint8_t enable)
 *
 * fanout - Desired clock fanout
 *   (use the si5351_clock_fanout enum)
 * enable - Set to 1 to enable, 0 to disable
 *
 * Use this function to enable or disable the clock fanout options
 * for individual clock outputs. If you intend to output the XO or
 * CLKIN on the clock outputs, enable this first.
 *
 * By default, only the Multisynth fanout is enabled at startup.
 */
void si5351_set_clock_fanout(enum si5351_clock_fanout fanout, uint8_t enable)
```
###si5351_write_bulk()
```
uint8_t si5351_write_bulk(uint8_t addr, uint8_t bytes, uint8_t *data)
```
###si5351_write()
```
uint8_t si5351_write(uint8_t addr, uint8_t data)
```
###si5351_read()
```
uint8_t si5351_read(uint8_t addr)
```

Tokens
------
Here are the defines, structs, and enumerations you will find handy to use with the library.

Crystal load capacitance:

    SI5351_CRYSTAL_LOAD_6PF
    SI5351_CRYSTAL_LOAD_8PF
    SI5351_CRYSTAL_LOAD_10PF

Clock outputs:

    enum si5351_clock {SI5351_CLK0, SI5351_CLK1, SI5351_CLK2, SI5351_CLK3,
      SI5351_CLK4, SI5351_CLK5, SI5351_CLK6, SI5351_CLK7};

PLL sources:

    enum si5351_pll {SI5351_PLLA, SI5351_PLLB};

Drive levels:

    enum si5351_drive {SI5351_DRIVE_2MA, SI5351_DRIVE_4MA, SI5351_DRIVE_6MA, SI5351_DRIVE_8MA};

Clock sources:

    enum si5351_clock_source {SI5351_CLK_SRC_XTAL, SI5351_CLK_SRC_CLKIN, SI5351_CLK_SRC_MS0, SI5351_CLK_SRC_MS};

Clock disable states:

    enum si5351_clock_disable {SI5351_CLK_DISABLE_LOW, SI5351_CLK_DISABLE_HIGH, SI5351_CLK_DISABLE_HI_Z, SI5351_CLK_DISABLE_NEVER};

Clock fanout:

    enum si5351_clock_fanout {SI5351_FANOUT_CLKIN, SI5351_FANOUT_XO, SI5351_FANOUT_MS};

Synth fractional divider:

    struct Si5351Frac
    {
      uint16_t a;
      uint32_t b;
      uint32_t c;
    };

Status register:

    struct Si5351Status
    {
      uint8_t SYS_INIT;
      uint8_t LOL_B;
      uint8_t LOL_A;
      uint8_t LOS;
      uint8_t REVID;
    };

Interrupt register:

    struct Si5351IntStatus
    {
      uint8_t SYS_INIT_STKY;
      uint8_t LOL_B_STKY;
      uint8_t LOL_A_STKY;
      uint8_t LOS_STKY;
    };

Raw Commands
------------
If you need to read and write raw data to the Si5351, there is public access to the library's si5351_read(), si5351_write(), and si5351_write_bulk() methods.
