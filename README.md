# Atmega328p-bromograph
/* The candidate should design an application capable of simulating the control of a
bromograph for photolithography. In the bromograph, it is possible to adjust the intensity of the
UV lamp and the exposure time of the mask.
The lamp intensity level can be selected from 4 possible values [20%,
40%, 60%, 80%] using the B6 key to increase the intensity and the B7 key to
decrease the intensity; the selected value must be displayed in binary code
using LEDs L0-L1. The exposure duration can also be selected from 4 possible
values [2, 4, 8, 16 seconds] using button B4 to increase the duration and button B5
to decrease the duration. The duration value must be displayed in binary
code using LEDs L2-L3.
Also use LEDs L4 and L5 to control the lamp and
display status information about the exposure process. In
particular, pressing button B2 starts the exposure process. Initially,
the lamp must warm up for 3 seconds, during which LED
L5 flashes with an on/off interval of 500 ms, while LED L4 flashes with a duty cycle
of 5% and a period of 100 ms. After the warm-up, LED L4 flashes for the duration of the
exposure with a duty cycle equal to those previously set and read
at the moment button B2 is pressed; meanwhile, LED L5 remains on. At the end of the
display time, L4 and L5 turn off.
If button B3 is pressed after the start (B2), the display will stop immediately,
L4 will turn off, and LED L5 will flash for 2 seconds (with
an on/off interval of 250 ms) before turning off. */

#include <avr/io.h>
#include <avr/interrupt.h>

#define  B2 (1<<PIND2)
#define  B3 (1<<PIND3)
#define  B4 (1<<PIND4)
#define  B5 (1<<PIND5)
#define  B6 (1<<PIND6)
#define  B7 (1<<PIND7)

typedef enum {MODALITA, RISCALDAMENTO, ESPOSIZIONE, STOP}states_t;
	states_t statocorrente = MODALITA;
	
volatile uint16_t pressed;
volatile uint16_t old =0xff;

volatile uint16_t tick;
volatile uint16_t timer;
volatile uint16_t ciclo;

volatile uint16_t intensita=0;
volatile uint16_t durata =0;

volatile uint16_t secondi=0;


int main(void)
{
	DDRC |= 0x3f;
	PORTC &=~ 0x3f;
	
	DDRD &=~ 0xfc;
	PORTD |= 0xfc;
	
	TCCR0A |= (1<<WGM01);
	TCCR0B |= (1<<CS00)|(1<<CS02);
	TIMSK0 |= (1<<OCIE0A);
	OCR0A = 77; //----->5ms
	
	TCCR1B |= (1<<WGM12);
	TCCR1B |= (1<<CS10)|(1<<CS12);
	TIMSK1 |= (1<<OCIE1A);
	OCR1A = 7812; //----->0,5s
	
	/*attivazione timer
	TCCR1B |= (1<<CS10)|(1<<CS12);
	TCCR0B |= (1<<CS00)|(1<<CS02);	
	*/
	
	PCICR |= (1<<PCIE2);
	PCMSK2 |= 0xfc;
	
	sei();
    while(1)
    {
        switch(statocorrente)
		{
			case MODALITA:
			if(pressed==B6)
			{
				intensita++;
				pressed=0;
				if(intensita>=3)
				{
					intensita=3;
				}
			}
			if(pressed==B7)
			{
				intensita--;
				pressed=0;
				if(intensita>4)
				{
					intensita=0;
				}
			}
			if(pressed==B4)
			{
				durata++;
				pressed=0;
				if(durata>=3)
				{
					durata=3;
				}
			}
			if(pressed==B5)
			{
				durata--;
				pressed=0;
				if(durata>4)
				{
					durata=0;
				}
			}
			if(intensita==0)
			{
				PORTC &=~ (1<<PORTC0);
				PORTC &=~ (1<<PORTC1);
				
			}
			if(intensita==1)
			{
				PORTC |= (1<<PORTC0);
				PORTC &=~ (1<<PORTC1);
				
			}
			if(intensita==2)
			{
				PORTC &=~  (1<<PORTC0);
				PORTC |= (1<<PORTC1);
				
			}
			if(intensita==3)
			{
				PORTC |= (1<<PORTC0);
				PORTC |= (1<<PORTC1);
				
			}
			if(durata==0)
			{
				PORTC &=~ (1<<PORTC2);
				PORTC &=~ (1<<PORTC3);
				secondi=4;
			}
			if(durata==1)
			{
				PORTC |= (1<<PORTC2);
				PORTC &=~ (1<<PORTC3);
				secondi=8;
			}
			if(durata==2)
			{
				PORTC &=~  (1<<PORTC2);
				PORTC |= (1<<PORTC3);
				secondi=16;
			}
			if(durata==3)
			{
				PORTC |= (1<<PORTC2);
				PORTC |= (1<<PORTC3);
				secondi=32;
			}
			if(pressed==B2)
			{
				statocorrente= RISCALDAMENTO;
				pressed=0;
				TCCR1B |= (1<<CS10)|(1<<CS12);
				TCCR0B |= (1<<CS00)|(1<<CS02);
				tick=0;
				ciclo=0;
				timer=0;				
				PORTC |= (1<<PORTC4);
				PORTC |= (1<<PORTC5);
			}
			
			break;	
			
			case RISCALDAMENTO:
			if(ciclo==6)
			{
				statocorrente= ESPOSIZIONE;
				tick=0;
				ciclo=0;
				timer=0;	
				PORTC |= (1<<PORTC5);			
			}
			break;
			
			case ESPOSIZIONE:
			if(ciclo==secondi)
			{
				statocorrente=MODALITA;
				PORTC &=~ (1<<PORTC0);
				PORTC &=~ (1<<PORTC1);
				PORTC &=~ (1<<PORTC2);
				PORTC &=~ (1<<PORTC3);
				PORTC &=~ (1<<PORTC4);
				PORTC &=~ (1<<PORTC5);
				secondi=0;
				intensita=0;
				durata=0;
				tick=0;
				ciclo=0;
				timer=0;
				pressed=0;
				TCCR1B &=~ (1<<CS10)|(1<<CS12);
				TCCR0B &=~ (1<<CS00)|(1<<CS02);
				
				
			}
			if(pressed== B3)
			{
				secondi=0;
				intensita=0;
				durata=0;
				tick=0;
				ciclo=0;
				pressed=0;
				timer=0;
				PORTC &=~ (1<<PORTC4);	
				statocorrente= STOP;			
				
			}
			break;
			
			case STOP:
			if(ciclo==4)
			{
				statocorrente=MODALITA;
				PORTC &=~ (1<<PORTC0);
				PORTC &=~ (1<<PORTC1);
				PORTC &=~ (1<<PORTC2);
				PORTC &=~ (1<<PORTC3);
				PORTC &=~ (1<<PORTC4);
				PORTC &=~ (1<<PORTC5);
				secondi=0;
				intensita=0;
				durata=0;
				tick=0;
				timer=0;
				ciclo=0;
				pressed=0;
				TCCR1B &=~ (1<<CS10)|(1<<CS12);
				TCCR0B &=~ (1<<CS00)|(1<<CS02);
			}
			
			break;
		}
    }
}
ISR (TIMER1_COMPA_vect)
{
	ciclo++;
}

ISR (TIMER0_COMPA_vect)
{
	tick++;
	timer++;
	if(statocorrente== RISCALDAMENTO)
	{
		if(tick>=100)
		{
			PORTC &=~ (1<<PORTC5);
			
		}
		if(tick>=200)
		{
			PORTC |= (1<<PORTC5);
			tick=0;
		}
		if(timer>=1)
		{
			PORTC &=~ (1<<PORTC4);
			
		}
		if(timer>=20)
		{
			PORTC |= (1<<PORTC4);
			timer=0;
		}
		
	}
	if(statocorrente== ESPOSIZIONE)
	{
		if(intensita==0)
			{
				if(timer>=20)
				{
					PORTC &=~ (1<<PORTC4);
					
				}
				if(timer==100)
				{
					PORTC |= (1<<PORTC4);
					timer=0;
				}
				
			}
		if(intensita==1)
		{
			if(timer>=40)
			{
				PORTC &=~ (1<<PORTC4);
				
			}
			if(timer==100)
			{
				PORTC |= (1<<PORTC4);
				timer=0;
			}
			
		}
		if(intensita==2)
		{
			if(timer>=60)
			{
				PORTC &=~ (1<<PORTC4);
				
			}
			if(timer==100)
			{
				PORTC |= (1<<PORTC4);
				timer=0;
			}
			
		}
		if(intensita==3)
		{
			if(timer>=80)
			{
				PORTC &=~ (1<<PORTC4);
				
			}
			if(timer==100)
			{
				PORTC |= (1<<PORTC4);
				timer=0;
			}
			
		}
	}
	
	if(statocorrente==STOP)
	{
		if(tick>=50)
		{
			PORTC &=~ (1<<PORTC5);
			
		}
		if(tick==100)
		{
			PORTC |= (1<<PORTC5);
			tick=0;
		}
	}
	
	
	
	
}
ISR (PCINT2_vect)
{
	uint16_t cambiamento = PIND ^ old;
	old =PIND;
	
	if(cambiamento&(1<<PIND2))
	{
		if((old&(1<<PIND2))==0)
		{
			pressed = (1<<PIND2);
		}
	}
	
	if(cambiamento&(1<<PIND3))
	{
		if((old&(1<<PIND3))==0)
		{
			pressed = (1<<PIND3);
		}
	}
	
	if(cambiamento&(1<<PIND4))
	{
		if((old&(1<<PIND4))==0)
		{
			pressed = (1<<PIND4);
		}
	}
	
	if(cambiamento&(1<<PIND5))
	{
		if((old&(1<<PIND5))==0)
		{
			pressed = (1<<PIND5);
		}
	}
	if(cambiamento&(1<<PIND6))
	{
		if((old&(1<<PIND6))==0)
		{
			pressed = (1<<PIND6);
		}
	}
	
	if(cambiamento&(1<<PIND7))
	{
		if((old&(1<<PIND7))==0)
		{
			pressed = (1<<PIND7);
		}
	}
	
	
}
