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

<!-- Save this file as README.md in a repo named exactly YOUR_GITHUB_USERNAME -->
<!-- Save banner.svg in the same repo -->

![banner](./banner.svg)

<br/>

## 👋 Hi, I'm Vijay Kumar

> Fresher Embedded Systems Engineer | EEE Graduate | IoT · ARM · Automotive | Bangalore 🇮🇳

I'm a **B.E. (EEE)** graduate passionate about building real-world embedded systems — from bare-metal firmware to RTOS-based applications. I recently completed an internship at **Cranes Varsity Pvt. Ltd.** where I earned the 🏆 **Best Intern Award** for my work in **Embedded Systems with AI**.

Currently open to full-time roles in **Embedded Systems, IoT, and Automotive** domains.

---

## 🛠️ Tech Stack

### Languages
![C](https://img.shields.io/badge/C-00599C?style=flat-square&logo=c&logoColor=white)
![Embedded C](https://img.shields.io/badge/Embedded_C-007ACC?style=flat-square&logo=c&logoColor=white)
![C++](https://img.shields.io/badge/C++-00599C?style=flat-square&logo=cplusplus&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)

### Microcontrollers & Hardware
![ARM](https://img.shields.io/badge/ARM_Cortex--M3-0091BD?style=flat-square&logo=arm&logoColor=white)
![LPC1768](https://img.shields.io/badge/LPC1768-0077C8?style=flat-square&logoColor=white)
![FreeRTOS](https://img.shields.io/badge/FreeRTOS-00A86B?style=flat-square&logoColor=white)

### Protocols
![UART](https://img.shields.io/badge/UART-555555?style=flat-square)
![I2C](https://img.shields.io/badge/I2C-555555?style=flat-square)
![SPI](https://img.shields.io/badge/SPI-555555?style=flat-square)
![CAN](https://img.shields.io/badge/CAN_Bus-FF6600?style=flat-square)

### Tools & OS
![Keil](https://img.shields.io/badge/Keil_µVision-0091BD?style=flat-square&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)
![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white)
![VS Code](https://img.shields.io/badge/VS_Code-007ACC?style=flat-square&logo=visual-studio-code&logoColor=white)

---

## 🚀 Featured Project

### 🔒 Door Intrusion Detection System
> **LPC1768 · ARM Cortex-M3 · GSM SIM800L · Keil µVision · Embedded-C**

Real-time door intrusion detection with **SMS alert over GSM**. A complete single-firmware solution integrating IR sensor, 4×4 keypad, 16×2 LCD, PWM servo lock control, and UART1-based SIM800L communication — all on LPC1768.

```c
/* UART1 Init — SIM800L @ 9600 baud, 25MHz PCLK */
LPC_UART1->LCR = 0x83;   // 8N1, DLAB=1
LPC_UART1->DLL = 162;     // Baud divisor
LPC_UART1->LCR = 0x03;   // DLAB=0
```

**Key Features:**
- 🚨 Interrupt-driven **IR sensor** detection (P0.9)
- 📱 **SMS alert** via `AT+CMGS` → SIM800L on intrusion
- 🔑 Keypad-based **arm / disarm** with LCD feedback
- 🔐 **PWM servo** (500µs–1400µs) for physical lock control
- 🔔 Progressive **buzzer escalation** on wrong password attempts
- 🔒 System **lock-out** after 3 failed attempts + continuous buzzer

```
[ IR Triggered ] → [ 3x Password Attempts ] → [ UART1 → SIM800L → SMS ]
       ↓                       ↓
 [ PWM Servo Open ]    [ Buzzer + LCD Alert ]
```

---

## 🗂️ Other Projects

### ⚡ EV-Vehicle Integration with Solar Energy
> **Power Electronics · Solar MPPT · EV Charging**

Designed a solar energy integration system for EV charging — focused on MPPT and efficient energy transfer to the vehicle battery.

---

### 👨‍💼 Employee Management System
> **C · File Handling · Linked Lists**

Console-based CRUD system in C with file persistence and structured data management using linked lists.

---

### ⚡ Tesla Coil
> **High Voltage Electronics · Resonant Circuits**

Built and tested a functional Tesla coil demonstrating resonant transformer principles and wireless power transmission.

---

## 💼 Experience

### 🏆 Embedded Systems with AI Intern — Cranes Varsity Pvt. Ltd.
**Jul 2025 – Jan 2026 | Bangalore**

- Developed firmware for **ARM-based platforms** using Keil µVision
- Worked on AI inference integration on microcontrollers
- **Awarded Best Intern** for outstanding project delivery

---

## 🎓 Education

| Degree | Institution | Year | Score |
|--------|------------|------|-------|
| B.E. — Electrical & Electronics Engineering | PVKK Institute of Technology | Dec 2021 – Nov 2025 | **71.93%** |

---

## 📜 Certifications

- ⚡ EV Technology — SWAYAM / SKILLDZIRE
- 🌐 IoT Fundamentals — SWAYAM
- 🏭 Industrial Training — AP TRANSCO Substations

---

## 📊 GitHub Stats

<p align="center">
  <img src="https://github-readme-stats.vercel.app/api?username=YOUR_USERNAME&show_icons=true&theme=chartreuse-dark&hide_border=true&bg_color=0d1117&title_color=00ff41&icon_color=00ff41&text_color=8b949e" height="160"/>
  &nbsp;
  <img src="https://github-readme-stats.vercel.app/api/top-langs/?username=YOUR_USERNAME&layout=compact&theme=chartreuse-dark&hide_border=true&bg_color=0d1117&title_color=00ff41&text_color=8b949e" height="160"/>
</p>

<p align="center">
  <img src="https://github-readme-streak-stats.herokuapp.com/?user=YOUR_USERNAME&theme=chartreuse-dark&hide_border=true&background=0d1117&ring=00ff41&fire=00ff41&currStreakLabel=00ff41" height="160"/>
</p>

---

## 🤝 Connect with Me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/YOUR_LINKEDIN)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white)](mailto:YOUR_EMAIL@gmail.com)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/YOUR_USERNAME)

---

## 🎯 Currently Targeting

`KPIT Technologies` · `Bosch India` · `Tata Elxsi` · `L&T Technology Services` · `Sasken Technologies`

---

<p align="center">
  <img src="https://komarev.com/ghpvc/?username=YOUR_USERNAME&style=flat-square&color=00ff41&label=Profile+Views" />
</p>

<p align="center"><i>"Turning ideas into silicon." ⚡</i></p>
