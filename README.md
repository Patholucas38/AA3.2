
PROCESSOR 16F877A
#include <xc.inc>

CONFIG FOSC = HS     ; Oscilador de alta velocidad (4MHz)
CONFIG WDTE = OFF
CONFIG PWRTE = ON
CONFIG BOREN = ON
CONFIG LVP = OFF
CONFIG CPD = OFF
CONFIG WRT = OFF
CONFIG CP = OFF

#define LCD_RS PORTC, 0
#define LCD_RW PORTC, 1
#define LCD_EN PORTC, 2

PSECT udata_bank0
TIMER_1: DS 1
TIMER_2: DS 1
VALUE_TEMP: DS 1
CALCULATED: DS 1
DIV_RESULT: DS 1
DIV_REMAINDER: DS 1

PSECT resetVec, class=CODE, delta=2
reset_vector:
    GOTO main

PSECT code, delta=2
main:
    BANKSEL TRISA
    MOVLW 0x01
    MOVWF TRISA

    BANKSEL ADCON1
    MOVLW 0x0E
    MOVWF ADCON1

    BANKSEL ADCON0
    MOVLW 0x41
    MOVWF ADCON0

    MOVLW 50
    CALL Delay_ms

    BANKSEL TRISB
    CLRF TRISB
    BANKSEL TRISC
    CLRF TRISC
    BANKSEL TRISD
    CLRF TRISD
    BANKSEL PORTA

    CALL LCD_Setup

    ; **Mostrar mensaje de bienvenida en la primera pantalla**
    MOVLW 0x80   ; Primera línea LCD
    CALL LCD_Command

    MOVLW 'B'
    CALL LCD_PrintChar
    MOVLW 'I'
    CALL LCD_PrintChar
    MOVLW 'E'
    CALL LCD_PrintChar
    MOVLW 'N'
    CALL LCD_PrintChar
    MOVLW 'V'
    CALL LCD_PrintChar
    MOVLW 'E'
    CALL LCD_PrintChar
    MOVLW 'N'
    CALL LCD_PrintChar
    MOVLW 'I'
    CALL LCD_PrintChar
    MOVLW 'D'
    CALL LCD_PrintChar
    MOVLW 'O'
    CALL LCD_PrintChar

    MOVLW 0xC0   ; Segunda línea LCD
    CALL LCD_Command
    MOVLW 'I'
    CALL LCD_PrintChar
    MOVLW 'S'
    CALL LCD_PrintChar
    MOVLW 'C'
    CALL LCD_PrintChar
    MOVLW ' '
    CALL LCD_PrintChar
    MOVLW '6'
    CALL LCD_PrintChar
    MOVLW 'A'
    CALL LCD_PrintChar

    ; **Mantener el mensaje por 60 segundos**
    MOVLW 60000
    CALL Delay_ms

    ; **Esperar 5 segundos antes de limpiar pantalla**
    MOVLW 5000
    CALL Delay_ms

    ; **Limpiar pantalla antes de iniciar la lectura del sensor**
    MOVLW 0x01
    CALL LCD_Command

loop:
    MOVLW 30
    CALL Delay_ms

    BSF ADCON0, 2

adc_wait:
    BTFSC ADCON0, 2
    GOTO adc_wait

    BANKSEL ADRESH
    MOVF ADRESH, W
    MOVWF VALUE_TEMP
    MOVF VALUE_TEMP, W
    ADDWF VALUE_TEMP, W
    MOVWF CALCULATED

    MOVLW 0x80
    CALL LCD_Command

    MOVLW 'T'
    CALL LCD_PrintChar
    MOVLW 'e'
    CALL LCD_PrintChar
    MOVLW 'm'
    CALL LCD_PrintChar
    MOVLW 'p'
    CALL LCD_PrintChar
    MOVLW ':'
    CALL LCD_PrintChar
    MOVLW ' '
    CALL LCD_PrintChar

    CALL LCD_ShowValue

    ; **Mantener "ISC 6A" debajo de la temperatura**
    MOVLW 0xC0
    CALL LCD_Command
    MOVLW 'I'
    CALL LCD_PrintChar
    MOVLW 'S'
    CALL LCD_PrintChar
    MOVLW 'C'
    CALL LCD_PrintChar
    MOVLW ' '
    CALL LCD_PrintChar
    MOVLW '6'
    CALL LCD_PrintChar
    MOVLW 'A'
    CALL LCD_PrintChar

    MOVLW 200
    CALL Delay_ms

    BSF ADCON0, 0
    GOTO loop

Delay_ms:
    MOVWF TIMER_1
Delay_Outer:
    MOVLW 250
    MOVWF TIMER_2
Delay_Inner:
    NOP
    NOP
    DECFSZ TIMER_2, F
    GOTO Delay_Inner
    DECFSZ TIMER_1, F
    GOTO Delay_Outer
    RETURN

LCD_Setup:
    MOVLW 50
    CALL Delay_ms

    MOVLW 0x30
    CALL LCD_Command_Init
    MOVLW 10
    CALL Delay_ms

    MOVLW 0x30
    CALL LCD_Command_Init
    MOVLW 5
    CALL Delay_ms

    MOVLW 0x30
    CALL LCD_Command_Init
    MOVLW 5
    CALL Delay_ms

    MOVLW 0x38
    CALL LCD_Command
    MOVLW 5
    CALL Delay_ms

    MOVLW 0x08
    CALL LCD_Command
    MOVLW 5
    CALL Delay_ms

    MOVLW 0x01
    CALL LCD_Command
    MOVLW 20
    CALL Delay_ms

    MOVLW 0x06
    CALL LCD_Command
    MOVLW 5
    CALL Delay_ms

    MOVLW 0x0C
    CALL LCD_Command
    MOVLW 5
    CALL Delay_ms

    RETURN

LCD_Command_Init:
    BCF LCD_RS
    BCF LCD_RW
    MOVWF PORTD
    BSF LCD_EN
    NOP
    NOP
    NOP
    NOP
    BCF LCD_EN
    RETURN

LCD_Command:
    BCF LCD_RS
    BCF LCD_RW
    MOVWF PORTD
    BSF LCD_EN
    NOP
    NOP
    NOP
    NOP
    BCF LCD_EN
    MOVLW 2
    CALL Delay_ms
    RETURN

LCD_PrintChar:
    BSF LCD_RS
    BCF LCD_RW
    MOVWF PORTD
    BSF LCD_EN
    NOP
    NOP
    NOP
    NOP
    BCF LCD_EN
    MOVLW 1
    CALL Delay_ms
    RETURN

LCD_ShowValue:
    MOVF CALCULATED, W
    MOVWF DIV_REMAINDER
    CLRF DIV_RESULT

    MOVLW 10
    SUBWF DIV_REMAINDER, W
    BTFSS STATUS, 0
    GOTO ShowOutput

DivideLoop:
    MOVLW 10
    SUBWF DIV_REMAINDER, W
    BTFSS STATUS, 0
    GOTO ShowOutput
    INCF DIV_RESULT, F
    MOVLW 10
    SUBWF DIV_REMAINDER, F
    GOTO DivideLoop

ShowOutput:
    MOVF DIV_RESULT, W
    ADDLW '0'
    CALL LCD_PrintChar

    MOVF DIV_REMAINDER, W
    ADDLW '0'
    CALL LCD_PrintChar

    MOVLW 0xDF
    CALL LCD_PrintChar
    MOVLW 'C'
    CALL LCD_PrintChar
    RETURN

END


