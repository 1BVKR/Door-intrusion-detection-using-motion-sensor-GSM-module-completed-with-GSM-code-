/* ============================================================
   DOOR INTRUSION DETECTION SYSTEM
   Files merged: gsm.h + gsm.c + main.c
   Board: LPC1768 | GSM: SIM800L via UART1
   ============================================================ */

/* ==================== gsm.h ==================== */
#ifndef GSM_H
#define GSM_H

#include <lpc17xx.h>
#include <stdint.h>

#define ADMIN_NUMBER  "9908323822"
#define GSM_BAUD      9600

void gsm_init(void);
void gsm_send_char(char c);
void gsm_send_string(const char *str);
void gsm_send_sms(const char *number, const char *message);
void gsm_alert_intruder(void);

#endif


/* ==================== gsm.c ==================== */
#include <lpc17xx.h>
#include "lcd_header.h"

void gsm_init(void) {
    LPC_SC->PCONP        |= (1 << 4);
    LPC_PINCON->PINSEL0  |= (1 << 30);   // P0.15 = TXD1
    LPC_PINCON->PINSEL1  |= (1 << 0);    // P0.16 = RXD1
    LPC_UART1->LCR        = 0x83;        // 8-bit, no parity, 1 stop, DLAB=1
    LPC_UART1->DLM        = 0x00;
    LPC_UART1->DLL        = 162;         // 9600 baud @ 25MHz PCLK
    LPC_UART1->LCR        = 0x03;        // DLAB=0
    LPC_UART1->FCR        = 0x07;        // Enable & clear FIFOs
}

void gsm_send_char(char c) {
    while (!(LPC_UART1->LSR & (1 << 5)));
    LPC_UART1->THR = c;
}

void gsm_send_string(const char *str) {
    while (*str) {
        gsm_send_char(*str++);
    }
}

void gsm_send_sms(const char *number, const char *message) {
    gsm_send_string("AT\r\n");
    delay(5000);
    gsm_send_string("AT+CMGF=1\r\n");
    delay(5000);
    gsm_send_string("AT+CMGS=\"");
    gsm_send_string(number);
    gsm_send_string("\"\r\n");
    delay(5000);
    gsm_send_string(message);
    delay(2000);
    gsm_send_char(0x1A);                 // Ctrl+Z to send SMS
    delay(10000);
}

void gsm_alert_intruder(void) {
    lcd_cmd(0x01);
    lcd_str("Sending Alert..");
    gsm_send_sms(ADMIN_NUMBER, "UNKNOWN PERSON ENTERD");
    lcd_cmd(0x01);
    lcd_str("Alert Sent!");
    delay(3000);
}


/* ==================== main.c ==================== */
#include <lpc17xx.h>
#include <stdint.h>
#include "lcd_header.h"
#include "keypad_header.h"

#define BUZZER_PIN    (1 << 27)          // P1.27
#define IR_SENSOR_PIN (1 << 9)           // P0.9

char correct_password[] = "1212";
char entered_password[5];
uint8_t attempts = 0;
int i;

void servo_pwm_init(void) {
    LPC_PINCON->PINSEL3 |= (1 << 9);    // P1.20 = PWM1.2
    LPC_SC->PCONP       |= (1 << 6);    // Power up PWM1
    LPC_PWM1->PR   = 0;
    LPC_PWM1->MR0  = 20000;             // 20ms period (50Hz)
    LPC_PWM1->MR2  = 1500;             // 90 degrees (neutral)
    LPC_PWM1->MCR  = (1 << 1);         // Reset on MR0 match
    LPC_PWM1->LER  = (1 << 0) | (1 << 2);
    LPC_PWM1->PCR  = (1 << 10);        // Enable PWM1.2 output
    LPC_PWM1->TCR  = (1 << 0) | (1 << 3);
}

void servo_rotate(uint16_t pulse_width_us) {
    int j;
    LPC_PWM1->MR2 = pulse_width_us;
    LPC_PWM1->LER |= (1 << 2);
    for (j = 0; j < 50; j++) { delay(20); }
}

void motor_open(void) {
    lcd_cmd(0x01);
    servo_rotate(500);                  // Close position
    delay(2000);
    lcd_cmd(0x01);
    lcd_str("Door Opening...");
    servo_rotate(1400);                 // Open position
    delay(500);
    lcd_cmd(0x01);
    lcd_str("Door Closed");
    delay(500);
}

void buzzer_alert(uint8_t count) {
    if (count < 4) {
        for (i = 0; i < count; i++) {
            LPC_GPIO1->FIOSET = BUZZER_PIN;
            delay(1000);
            LPC_GPIO1->FIOCLR = BUZZER_PIN;
            delay(500);
        }
    } else {
        LPC_GPIO1->FIOSET = BUZZER_PIN; // Continuous alert
        delay(8000);
        LPC_GPIO1->FIOCLR = BUZZER_PIN;
    }
}

int compare_passwords(char *a, char *b) {
    for (i = 0; i < 4; i++) {
        if (a[i] != b[i]) return 0;
    }
    return 1;
}

int main(void) {
    lcd_init();
    servo_pwm_init();
    gsm_init();                         // GSM UART1 init

    LPC_GPIO1->FIODIR |= BUZZER_PIN;
    LPC_GPIO1->FIOCLR  = BUZZER_PIN;
    LPC_GPIO0->FIODIR &= ~IR_SENSOR_PIN;

    lcd_str("Door Lock System");
    delay(4000);
    lcd_cmd(0x01);
    lcd_str("WELCOME");
    delay(3000);
    lcd_cmd(0x01);

    while (1) {
        lcd_cmd(0x01);
        lcd_str("Waiting for");
        lcd_cmd(0xC0);
        lcd_str("person...");

        while ((LPC_GPIO0->FIOPIN & IR_SENSOR_PIN));  // Wait for IR trigger

        lcd_cmd(0x01);
        lcd_str("Person Detected");
        delay(3000);
        lcd_cmd(0x01);

        while (attempts < 3) {
            if (attempts > 0) {
                lcd_cmd(0x01);
                lcd_str("Attempt ");
                lcd_char('1' + attempts);
                delay(2000);
                lcd_cmd(0x01);
            }

            get_password();

            if (compare_passwords(correct_password, entered_password)) {
                lcd_cmd(0x01);
                lcd_str("Access Granted");
                lcd_cmd(0xC0);
                lcd_str("WELCOME HOME");
                delay(3000);
                motor_open();
                attempts = 0;
                break;
            } else {
                lcd_cmd(0x01);
                lcd_str("Wrong Password");
                delay(2000);

                if (attempts == 2) {
                    buzzer_alert(3);
                    lcd_cmd(0x01);
                    lcd_str("Access Blocked!");
                    lcd_cmd(0xC0);
                    lcd_str("TRY LATER..");
                    delay(4000);
                    gsm_alert_intruder();   // SMS -> "UNKNOWN PERSON ENTERD"
                    buzzer_alert(4);
                    while (1);             // Lock system indefinitely
                } else {
                    buzzer_alert(attempts + 1);
                    attempts++;
                }
            }
        }
    }
}
