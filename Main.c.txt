#include <msp430.h>
#include <stdint.h>

/*
 * ============================================================
 * MSP430G2553
 * I2C BUS SCANNER + TIMER SOFTWARE UART
 * ============================================================
 *
 * CLOCK
 * ------------------------------------------------------------
 * DCO = 1 MHz
 *
 * SOFTWARE UART
 * ------------------------------------------------------------
 * TX  = P1.2
 * Baud = 9600
 * Format = 8-N-1
 *
 * SOFTWARE I2C
 * ------------------------------------------------------------
 * SCL = P1.6
 * SDA = P1.7
 *
 * DS1307 ADDRESS
 * ------------------------------------------------------------
 * 0x68
 *
 * ============================================================
 */


/* ============================================================
 * PIN DEFINITIONS
 * ============================================================ */

#define UART_TX     BIT2       /* P1.2 */

#define I2C_SCL     BIT6       /* P1.6 */
#define I2C_SDA     BIT7       /* P1.7 */

#define I2C_PINS    (I2C_SCL | I2C_SDA)


/* ============================================================
 * UART TIMING
 * ============================================================ */

/*
 * 1 MHz clock:
 *
 * 1,000,000 / 9600 = 104.1667
 *
 * We use 104 timer ticks per bit.
 */
#define UART_BIT_TICKS    104


/* ============================================================
 * I2C TIMING
 * ============================================================ */

#define I2C_DELAY_TICKS   20


/* ============================================================
 * DEVICE TABLE
 * ============================================================ */

typedef struct
{
    uint8_t address;
    const char *name;
} I2CDevice;


const I2CDevice device_table[] =
{
    {0x27, "PCF8574 LCD Adapter"},
    {0x3C, "SSD1306 OLED Display"},
    {0x50, "AT24 EEPROM"},
    {0x68, "DS1307 / DS3231 / MPU6050"},
    {0x76, "BMP280 / BME280"}
};


#define TABLE_SIZE \
    (sizeof(device_table) / sizeof(device_table[0]))


/* ============================================================
 * FUNCTION PROTOTYPES
 * ============================================================ */

/* Clock */
void init_clock(void);

/* GPIO */
void init_gpio(void);

/* Timer */
void init_timer(void);

/* Software UART */
void uart_init(void);
void uart_wait_until(uint16_t target);
void uart_send_char(char c);
void uart_send_string(const char *str);
void uart_send_hex8(uint8_t value);
void uart_send_dec8(uint8_t value);

/* Software I2C */
void i2c_init(void);

void i2c_scl_low(void);
void i2c_scl_release(void);

void i2c_sda_low(void);
void i2c_sda_release(void);

uint8_t i2c_scl_read(void);
uint8_t i2c_sda_read(void);

void i2c_delay(void);

uint8_t i2c_wait_scl_high(void);

uint8_t i2c_start(void);
void i2c_stop(void);

uint8_t i2c_write_byte(uint8_t data);
uint8_t i2c_read_ack(void);

uint8_t scan_i2c_address(uint8_t address);

/* Lookup */
const char *lookup_device(uint8_t address);


/* ============================================================
 * MAIN
 * ============================================================ */

int main(void)
{
    uint8_t address;
    uint8_t result;
    uint8_t devices_found = 0;


    /*
     * Stop watchdog timer.
     */
    WDTCTL = WDTPW | WDTHOLD;


    /*
     * 1 MHz CPU clock.
     */
    init_clock();


    /*
     * GPIO.
     */
    init_gpio();


    /*
     * Timer used as precise time base for UART.
     */
    init_timer();


    /*
     * Software UART.
     */
    uart_init();


    /*
     * Software I2C.
     */
    i2c_init();


    /*
     * Startup delay.
     */
    __delay_cycles(50000);


    /* ========================================================
     * STARTUP MESSAGE
     * ======================================================== */

    uart_send_string("\r\n");
    uart_send_string("\r\n");
    uart_send_string("========================================\r\n");
    uart_send_string("      MSP430 I2C BUS SCANNER\r\n");
    uart_send_string("========================================\r\n");

    uart_send_string("MCU       : MSP430G2553\r\n");
    uart_send_string("Clock     : 1 MHz\r\n");
    uart_send_string("UART TX   : P1.2\r\n");
    uart_send_string("UART      : Software / Timer_A\r\n");
    uart_send_string("Baud      : 9600\r\n");
    uart_send_string("I2C SCL   : P1.6\r\n");
    uart_send_string("I2C SDA   : P1.7\r\n");

    uart_send_string("----------------------------------------\r\n");


    /* ========================================================
     * CHECK I2C BUS
     * ======================================================== */

    /*
     * Release both lines.
     */
    i2c_scl_release();
    i2c_sda_release();

    i2c_delay();


    /*
     * SCL must be HIGH.
     */
    if (!i2c_scl_read())
    {
        uart_send_string("ERROR: SCL IS LOW\r\n");
        uart_send_string(
            "Check SCL pull-up and wiring.\r\n"
        );

        uart_send_string("----------------------------------------\r\n");

        while (1)
        {
            /* Stop */
        }
    }


    /*
     * SDA must be HIGH.
     */
    if (!i2c_sda_read())
    {
        uart_send_string("ERROR: SDA IS LOW\r\n");
        uart_send_string(
            "Check SDA pull-up and wiring.\r\n"
        );

        uart_send_string("----------------------------------------\r\n");

        while (1)
        {
            /* Stop */
        }
    }


    uart_send_string("I2C BUS STATUS: OK\r\n");
    uart_send_string("SCL = HIGH\r\n");
    uart_send_string("SDA = HIGH\r\n");

    uart_send_string("----------------------------------------\r\n");

    uart_send_string(
        "Scanning addresses 0x08 to 0x77...\r\n"
    );

    uart_send_string("\r\n");


    /* ========================================================
     * SCAN I2C ADDRESSES
     * ======================================================== */

    for (address = 0x08; address <= 0x77; address++)
    {
        result = scan_i2c_address(address);


        /*
         * Device ACK.
         */
        if (result == 1)
        {
            devices_found++;


            uart_send_string(
                "[ACK] Device found at 0x"
            );


            uart_send_hex8(address);


            uart_send_string(
                " -> "
            );


            uart_send_string(
                lookup_device(address)
            );


            uart_send_string(
                "\r\n"
            );
        }


        /*
         * I2C bus error.
         */
        else if (result == 2)
        {
            uart_send_string(
                "\r\nERROR: I2C BUS ERROR\r\n"
            );

            uart_send_string(
                "Check SCL/SDA/pull-ups.\r\n"
            );

            break;
        }
    }


    /* ========================================================
     * FINAL RESULT
     * ======================================================== */

    uart_send_string("\r\n");

    uart_send_string(
        "----------------------------------------\r\n"
    );

    uart_send_string(
        "SCAN COMPLETE\r\n"
    );

    uart_send_string(
        "Devices found: "
    );

    uart_send_dec8(devices_found);

    uart_send_string("\r\n");

    uart_send_string(
        "----------------------------------------\r\n"
    );


    /* ========================================================
     * ENDLESS LOOP
     * ======================================================== */

    while (1)
    {
        __delay_cycles(100000);
    }
}


/* ============================================================
 * CLOCK INITIALIZATION
 * ============================================================ */

void init_clock(void)
{
    /*
     * Factory calibrated 1 MHz DCO.
     */
    DCOCTL = 0;

    BCSCTL1 = CALBC1_1MHZ;

    DCOCTL = CALDCO_1MHZ;


    /*
     * SMCLK divider = 1.
     */
    BCSCTL2 &= ~DIVS_3;
}


/* ============================================================
 * GPIO INITIALIZATION
 * ============================================================ */

void init_gpio(void)
{
    /*
     * UART TX is GPIO because we're using software UART.
     */
    P1SEL &= ~UART_TX;
    P1SEL2 &= ~UART_TX;


    /*
     * UART idle state = HIGH.
     */
    P1OUT |= UART_TX;


    /*
     * UART TX output.
     */
    P1DIR |= UART_TX;


    /*
     * I2C pins are GPIO.
     */
    P1SEL &= ~I2C_PINS;
    P1SEL2 &= ~I2C_PINS;


    /*
     * Initially release I2C pins.
     */
    P1DIR &= ~I2C_PINS;


    /*
     * Enable internal pull-ups.
     *
     * External 4.7k pull-ups should still be present.
     */
    P1REN |= I2C_PINS;

    P1OUT |= I2C_PINS;
}


/* ============================================================
 * TIMER INITIALIZATION
 * ============================================================ */

void init_timer(void)
{
    /*
     * Timer_A0:
     *
     * SMCLK = 1 MHz
     *
     * Continuous mode.
     */
    TA0CTL =
        TASSEL_2 |
        MC_2 |
        TACLR;
}


/* ============================================================
 * SOFTWARE UART INIT
 * ============================================================ */

void uart_init(void)
{
    /*
     * UART idle HIGH.
     */
    P1OUT |= UART_TX;


    /*
     * TX output.
     */
    P1DIR |= UART_TX;
}


/* ============================================================
 * TIMER WAIT
 * ============================================================ */

void uart_wait_until(uint16_t target)
{
    /*
     * Wait until Timer_A reaches target.
     *
     * Unsigned subtraction handles timer overflow.
     */
    while ((int16_t)(TA0R - target) < 0)
    {
        /* Wait */
    }
}


/* ============================================================
 * SOFTWARE UART SEND CHARACTER
 * ============================================================ */

void uart_send_char(char c)
{
    uint8_t i;

    uint16_t target;


    /*
     * Start timing from current timer value.
     */
    target = TA0R;


    /*
     * --------------------------------------------------------
     * START BIT
     * --------------------------------------------------------
     */

    P1OUT &= ~UART_TX;


    target += UART_BIT_TICKS;

    uart_wait_until(target);


    /*
     * --------------------------------------------------------
     * 8 DATA BITS
     * --------------------------------------------------------
     */

    for (i = 0; i < 8; i++)
    {
        /*
         * Send LSB first.
         */
        if (c & 0x01)
        {
            P1OUT |= UART_TX;
        }
        else
        {
            P1OUT &= ~UART_TX;
        }


        /*
         * Schedule next bit boundary.
         */
        target += UART_BIT_TICKS;


        uart_wait_until(target);


        /*
         * Next bit.
         */
        c >>= 1;
    }


    /*
     * --------------------------------------------------------
     * STOP BIT
     * --------------------------------------------------------
     */

    P1OUT |= UART_TX;


    target += UART_BIT_TICKS;

    uart_wait_until(target);
}


/* ============================================================
 * UART SEND STRING
 * ============================================================ */

void uart_send_string(const char *str)
{
    while (*str)
    {
        uart_send_char(*str);

        str++;
    }
}


/* ============================================================
 * UART SEND HEX
 * ============================================================ */

void uart_send_hex8(uint8_t value)
{
    const char hex[] =
        "0123456789ABCDEF";


    uart_send_char(
        hex[(value >> 4) & 0x0F]
    );


    uart_send_char(
        hex[value & 0x0F]
    );
}


/* ============================================================
 * UART SEND DECIMAL
 * ============================================================ */

void uart_send_dec8(uint8_t value)
{
    uint8_t hundreds;
    uint8_t tens;
    uint8_t ones;


    hundreds = value / 100;

    tens = (value % 100) / 10;

    ones = value % 10;


    if (hundreds != 0)
    {
        uart_send_char('0' + hundreds);

        uart_send_char('0' + tens);

        uart_send_char('0' + ones);
    }
    else if (tens != 0)
    {
        uart_send_char('0' + tens);

        uart_send_char('0' + ones);
    }
    else
    {
        uart_send_char('0' + ones);
    }
}


/* ============================================================
 * SOFTWARE I2C INIT
 * ============================================================ */

void i2c_init(void)
{
    /*
     * Release both bus lines.
     */
    i2c_scl_release();

    i2c_sda_release();
}


/* ============================================================
 * SCL LOW
 * ============================================================ */

void i2c_scl_low(void)
{
    P1OUT &= ~I2C_SCL;

    P1DIR |= I2C_SCL;
}


/* ============================================================
 * RELEASE SCL
 * ============================================================ */

void i2c_scl_release(void)
{
    /*
     * Set output latch HIGH first.
     */
    P1OUT |= I2C_SCL;


    /*
     * Then release pin.
     */
    P1DIR &= ~I2C_SCL;
}


/* ============================================================
 * SDA LOW
 * ============================================================ */

void i2c_sda_low(void)
{
    P1OUT &= ~I2C_SDA;

    P1DIR |= I2C_SDA;
}


/* ============================================================
 * RELEASE SDA
 * ============================================================ */

void i2c_sda_release(void)
{
    /*
     * Set output latch HIGH first.
     */
    P1OUT |= I2C_SDA;


    /*
     * Release pin.
     */
    P1DIR &= ~I2C_SDA;
}


/* ============================================================
 * READ SCL
 * ============================================================ */

uint8_t i2c_scl_read(void)
{
    return (P1IN & I2C_SCL) ? 1 : 0;
}


/* ============================================================
 * READ SDA
 * ============================================================ */

uint8_t i2c_sda_read(void)
{
    return (P1IN & I2C_SDA) ? 1 : 0;
}


/* ============================================================
 * I2C DELAY
 * ============================================================ */

void i2c_delay(void)
{
    __delay_cycles(I2C_DELAY_TICKS);
}


/* ============================================================
 * WAIT FOR SCL HIGH
 * ============================================================ */

uint8_t i2c_wait_scl_high(void)
{
    uint16_t timeout = 5000;


    while (!i2c_scl_read())
    {
        timeout--;


        if (timeout == 0)
        {
            return 0;
        }
    }


    return 1;
}


/* ============================================================
 * I2C START
 * ============================================================ */

uint8_t i2c_start(void)
{
    /*
     * Release bus.
     */
    i2c_sda_release();

    i2c_scl_release();

    i2c_delay();


    /*
     * Bus must be idle HIGH.
     */
    if (!i2c_scl_read())
    {
        return 0;
    }


    if (!i2c_sda_read())
    {
        return 0;
    }


    /*
     * START:
     *
     * SDA HIGH -> LOW
     * while SCL HIGH
     */
    i2c_sda_low();

    i2c_delay();


    /*
     * Pull SCL LOW.
     */
    i2c_scl_low();

    i2c_delay();


    return 1;
}


/* ============================================================
 * I2C STOP
 * ============================================================ */

void i2c_stop(void)
{
    /*
     * SDA LOW.
     */
    i2c_sda_low();

    i2c_delay();


    /*
     * Release SCL.
     */
    i2c_scl_release();


    /*
     * Wait for SCL HIGH.
     */
    i2c_wait_scl_high();

    i2c_delay();


    /*
     * STOP:
     *
     * SDA LOW -> HIGH
     * while SCL HIGH.
     */
    i2c_sda_release();

    i2c_delay();
}


/* ============================================================
 * I2C WRITE BYTE
 * ============================================================ */

uint8_t i2c_write_byte(uint8_t data)
{
    uint8_t i;


    for (i = 0; i < 8; i++)
    {
        /*
         * SCL LOW.
         */
        i2c_scl_low();


        /*
         * Send MSB.
         */
        if (data & 0x80)
        {
            /*
             * Logic HIGH:
             * release SDA.
             */
            i2c_sda_release();
        }
        else
        {
            /*
             * Logic LOW.
             */
            i2c_sda_low();
        }


        i2c_delay();


        /*
         * SCL HIGH.
         */
        i2c_scl_release();


        /*
         * Check clock actually went HIGH.
         */
        if (!i2c_wait_scl_high())
        {
            return 0;
        }


        i2c_delay();


        /*
         * Next bit.
         */
        data <<= 1;
    }


    /*
     * Return SCL LOW for ACK cycle.
     */
    i2c_scl_low();

    i2c_delay();


    return 1;
}


/* ============================================================
 * I2C READ ACK
 * ============================================================ */

uint8_t i2c_read_ack(void)
{
    uint8_t ack;


    /*
     * Slave controls SDA.
     */
    i2c_sda_release();


    i2c_delay();


    /*
     * Clock ACK.
     */
    i2c_scl_release();


    /*
     * Wait for SCL HIGH.
     */
    if (!i2c_wait_scl_high())
    {
        return 0;
    }


    i2c_delay();


    /*
     * ACK = SDA LOW
     * NACK = SDA HIGH
     */
    if (i2c_sda_read())
    {
        ack = 0;
    }
    else
    {
        ack = 1;
    }


    /*
     * Finish ACK clock.
     */
    i2c_scl_low();

    i2c_delay();


    return ack;
}


/* ============================================================
 * SCAN ONE ADDRESS
 * ============================================================ */

uint8_t scan_i2c_address(uint8_t address)
{
    uint8_t success;


    /*
     * START.
     */
    if (!i2c_start())
    {
        i2c_stop();

        return 2;
    }


    /*
     * Address + WRITE.
     *
     * Example:
     *
     * 0x68 << 1 = 0xD0
     */
    success =
        i2c_write_byte(address << 1);


    if (!success)
    {
        i2c_stop();

        return 2;
    }


    /*
     * Read ACK from slave.
     */
    success =
        i2c_read_ack();


    /*
     * STOP.
     */
    i2c_stop();


    /*
     * ACK = device found.
     */
    if (success)
    {
        return 1;
    }


    /*
     * NACK = no device.
     */
    return 0;
}


/* ============================================================
 * DEVICE LOOKUP
 * ============================================================ */

const char *lookup_device(uint8_t address)
{
    uint8_t i;


    for (i = 0; i < TABLE_SIZE; i++)
    {
        if (device_table[i].address == address)
        {
            return device_table[i].name;
        }
    }


    return "Unknown I2C Device";
}