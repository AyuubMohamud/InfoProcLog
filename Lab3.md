# Lab 3

## Table of Contents

- [Task 1](#task-1)
- [Task 2](#task-2)
- [Task 3](#task-3)
- [Challenge](#challenge-optimise-fir-filter)
- [Code Appendix](#code-appendix)

## Task 1

![PDNonsense](images/Lab3/Task1/task1_1.png)
![Eclipse](images/Lab3/Task1/task1_2.png)

## Task 2

```C
void convert_read(alt_32 acc_read, int * level, alt_u8 * led) {
    acc_read += OFFSET;
    alt_u8 val = (acc_read >> 6) & 0x07; // extracts the top three bits of the x_axis reading: when its positive
    // its 0-3, when its negative its 4-7, with 4 being the most negative
    * led = (8 >> val) | (8 << (8 - val)); // this sets the led with the reflected values, led7-led4 get lit up when the direction
    // is negative, and led3-led0 get set when the direction is positive
    // (8>>val) responsible for positive values, (8<<(8-val)) responsible for negative values
    * level = (acc_read >> 1) & 0x1f; // gives absolute reading of MSB of X axis reading from bit 0 to 4, aka giving the other 5 bits not read by val
}
/*
The writing of the value on the LEDs is performed at a specific rate dictated by the timer. The sys_timer_isr() is an interrupt service routine that is executed when a specific interrupt is received.

As such, the processor will only execute this code at a specific intervals, letting the processor to focus on the execution of the while-loop code.

You can also notice that the code uses pulse width modulation (PWM), which utilizes the convert_read() function, to create a smooth effect on the LEDS, creating a more “pleasant” indicator of the tilted angle of the board.

Using the alt_print() function, you can print on the host terminal the actual values of the x_read.

*/
```

## Task 3

N-tap moving average filter (can be used for 5 or more)

```C
float buf[NO_OF_TAPS];
float fir_filter(alt_32 sampl_value) {
    static int x = 0; // Note that this is a STATIC int
    buf[x%NO_OF_TAPS] = (float)sampl_value;
    x++; // Buffer rolls over with %NO_OF_TAPS term, causing it to store NO_OF_TAPS most recent samples
    float output = 0;
    for (int y = 0; y < NO_OF_TAPS; y++) {
        output += (1.0f/NO_OF_TAPS)*buf[y];
    }
    return output;
}
```

For an N-tap moving average filter of 5:


https://github.com/user-attachments/assets/854b706f-c43b-497c-9b44-fb3cecd48a61


For an N-tap moving average filter of 50:


https://github.com/user-attachments/assets/a63b6404-4af0-4473-bfbe-6f862352e9f3


For an N-tap Low Pass Filter in the example:

```C
float array_of_coeffs[NO_OF_TAPS] = {
        0.0046  ,  0.0074 ,  -0.0024 ,  -0.0071  ,  0.0033 ,   0.0001 ,  -0.0094 ,   0.0040 ,   0.0044 ,  -0.0133 ,   0.0030,
        0.0114  , -0.0179 ,  -0.0011 ,   0.0223  , -0.0225 ,  -0.0109 ,   0.0396 ,  -0.0263 ,  -0.0338 ,   0.0752 ,  -0.0289,
       -0.1204  ,  0.2879 ,   0.6369 ,   0.2879  , -0.1204 ,  -0.0289 ,   0.0752 ,  -0.0338 ,  -0.0263 ,   0.0396 ,  -0.0109,
       -0.0225  ,  0.0223 ,  -0.0011 ,  -0.0179  ,  0.0114 ,   0.0030 ,  -0.0133 ,   0.0044 ,   0.0040 ,  -0.0094 ,   0.0001,
        0.0033  , -0.0071 ,  -0.0024 ,   0.0074  ,  0.0046};
float lpf_fir_filter(alt_32 sampl_value) {

    static int x = 0;
    buf[x%NO_OF_TAPS] = (float)sampl_value;
    x++;
    float output = 0;
    for (int y = 0; y < NO_OF_TAPS; y++) {
        output += array_of_coeffs[y]*buf[(x+y)%NO_OF_TAPS];
    }
    return output;
}
```

https://github.com/user-attachments/assets/daa131d3-2252-4ace-9928-2fa2d02b0a40


## Challenge: Optimise FIR filter

## Code appendix

```C
#include "system.h"
#include "altera_up_avalon_accelerometer_spi.h"
#include "altera_avalon_timer_regs.h"
#include "altera_avalon_timer.h"
#include "altera_avalon_pio_regs.h"
#include "sys/alt_irq.h"
#include <stdlib.h>

#define OFFSET -32
#define PWM_PERIOD 16
#define NO_OF_TAPS 49
alt_8 pwm = 0;
alt_u8 led;
int level;
float array_of_coeffs[NO_OF_TAPS] = {
        0.0046  ,  0.0074 ,  -0.0024 ,  -0.0071  ,  0.0033 ,   0.0001 ,  -0.0094 ,   0.0040 ,   0.0044 ,  -0.0133 ,   0.0030,
        0.0114  , -0.0179 ,  -0.0011 ,   0.0223  , -0.0225 ,  -0.0109  ,  0.0396 ,  -0.0263 ,  -0.0338 ,   0.0752 ,  -0.0289,
       -0.1204  ,  0.2879 ,   0.6369 ,   0.2879  , -0.1204 ,  -0.0289  ,  0.0752 ,  -0.0338 ,  -0.0263 ,   0.0396  , -0.0109,
       -0.0225  ,  0.0223  , -0.0011 ,  -0.0179  ,  0.0114  ,  0.0030 ,  -0.0133 ,   0.0044 ,   0.0040 ,  -0.0094 ,   0.0001,
        0.0033  , -0.0071 ,  -0.0024 ,  0.0074  ,  0.0046};
void led_write(alt_u8 led_pattern) {
    IOWR(LED_BASE, 0, led_pattern);
}

void convert_read(alt_32 acc_read, int * level, alt_u8 * led) {
    acc_read += OFFSET;
    alt_u8 val = (acc_read >> 6) & 0x07; // extracts the top three bits of the x_axis reading: when its positive
    // its 0-3, when its negative its 4-7, with 4 being the most negative
    * led = (8 >> val) | (8 << (8 - val)); // this sets the led with the reflected values, led7-led4 get lit up when the direction
    // is negative, and led3-led0 get set when the direction is positive
    // (8>>val) responsible for positive values, (8<<(8-val)) responsible for negative values
    * level = (acc_read >> 1) & 0x1f; // gives absolute reading of MSB of X axis reading from bit 0 to 4, aka giving the other 5 bits not read by val
}
/*
The writing of the value on the LEDs is performed at a specific rate dictated by the timer. The sys_timer_isr() is an interrupt service routine that is executed when a specific interrupt is received.

As such, the processor will only execute this code at a specific intervals, letting the processor to focus on the execution of the while-loop code.

You can also notice that the code uses pulse width modulation (PWM), which utilizes the convert_read() function, to create a smooth effect on the LEDS, creating a more “pleasant” indicator of the tilted angle of the board.

Using the alt_print() function, you can print on the host terminal the actual values of the x_read.

*/
void sys_timer_isr() {
    IOWR_ALTERA_AVALON_TIMER_STATUS(TIMER_BASE, 0);
    if (pwm < abs(level)) {

        if (level < 0) {
            led_write(led << 1);
        } else {
            led_write(led >> 1);
        }

    } else {
        led_write(led);
    }

    if (pwm > PWM_PERIOD) {
        pwm = 0;
    } else {
        pwm++;
    }

}

void timer_init(void * isr) {

    IOWR_ALTERA_AVALON_TIMER_CONTROL(TIMER_BASE, 0x0003);
    IOWR_ALTERA_AVALON_TIMER_STATUS(TIMER_BASE, 0);
    IOWR_ALTERA_AVALON_TIMER_PERIODL(TIMER_BASE, 0x0900);
    IOWR_ALTERA_AVALON_TIMER_PERIODH(TIMER_BASE, 0x0000);
    alt_irq_register(TIMER_IRQ, 0, isr);
    IOWR_ALTERA_AVALON_TIMER_CONTROL(TIMER_BASE, 0x0007);

}
float buf[NO_OF_TAPS];
float fir_filter(alt_32 sampl_value) {

    static int x = 0;
    buf[x%NO_OF_TAPS] = (float)sampl_value;
    x++;
    float output = 0;
    for (int y = 0; y < NO_OF_TAPS; y++) {
        output += (1.0f/NO_OF_TAPS)*buf[y];
    }
    return output;
}
float lpf_fir_filter(alt_32 sampl_value) {

    static int x = 0;
    buf[x%NO_OF_TAPS] = (float)sampl_value;
    x++;
    float output = 0;
    for (int y = 0; y < NO_OF_TAPS; y++) {
        output += array_of_coeffs[y]*buf[(x+y)%NO_OF_TAPS];
    }
    return output;
}
int main() {

    alt_32 x_read;
    alt_up_accelerometer_spi_dev * acc_dev;
    acc_dev = alt_up_accelerometer_spi_open_dev("/dev/accelerometer_spi");
    if (acc_dev == NULL) { // if return 1, check if the spi ip name is "accelerometer_spi"
        return 1;
    }

    timer_init(sys_timer_isr);
    while (1) {

        alt_up_accelerometer_spi_read_x_axis(acc_dev, & x_read);
        float x_read_f = fir_filter(x_read);
        x_read = (int)x_read_f;
        //alt_printf("raw data: %x\n", x_read);
        convert_read(x_read, & level, & led);

    }

    return 0;
}
```