# Lab-3
	EE 319K Lab 3
 	GPIO_PORTE_DATA_R  EQU 0x400243FC
 	GPIO_PORTE_DIR_R   EQU 0x40024400
	GPIO_PORTE_AFSEL_R EQU 0x40024420
	GPIO_PORTE_DEN_R   EQU 0x4002451C
								; PortF device registers
	GPIO_PORTF_DATA_R  EQU 0x400253FC
	GPIO_PORTF_DIR_R   EQU 0x40025400
	GPIO_PORTF_AFSEL_R EQU 0x40025420
	GPIO_PORTF_PUR_R   EQU 0x40025510
	GPIO_PORTF_DEN_R   EQU 0x4002551C
	GPIO_PORTF_LOCK_R  EQU 0x40025520
	GPIO_PORTF_CR_R    EQU 0x40025524
	GPIO_LOCK_KEY      EQU 0x4C4F434B  			; Unlocks the GPIO_CR register
	SYSCTL_RCGCGPIO_R  EQU 0x400FE608

       IMPORT  TExaS_Init
       THUMB
       AREA    DATA, ALIGN=2
		  

								;global variables go here


       AREA    |.text|, CODE, READONLY, ALIGN=2
       THUMB
       EXPORT  Start
		   

	tonn 	DCD 2412000					; initial time on  30%
	tof  	DCD 5628000					; initial time off 70% TOTAL RUN:8,040,000	
	incr 	DCD 1608000					; actual calculated is 160800

	min		DCD 804000				; 10%
	max		DCD 7236000				; 90%

								;breathing
	bron	DCD 8040					
	broff	DCD 152760
	bncr	DCD 8040
	total 	DCD 160800
	
	Start
	
	
                              					; TExaS_Init sets bus clock at 80 MHz
     BL  TExaS_Init          					; voltmeter, scope on PD3 ; Initialization goes here
	LDR R1,=SYSCTL_RCGCGPIO_R				; R1 <- Address of SYSCTL	
	LDR R0, [R1]
	ORR R0, #0x30						; Friendly mask that will turn on Ports E and F
	STR R0,[R1]					      	; R0 -> Mem[SYSCTL] 
	
	
	NOP		                    			; wait for clock to stabilize
	NOP			                    
	
	MOV R0,#0x08						; For Port E, PE2 = 0 and PE3 = 1 , so 1000 = x08
	MOV R1,#0x00						; For Port F, PF4 = 0, so 0000 = x00
	LDR R2,=GPIO_PORTE_DIR_R				; R2 <- address of DIR PORT E
	LDR R3,=GPIO_PORTF_DIR_R				; R3 <- address of DIR Port F 
	STR R0,[R2]						; storing PORT E Input/Output Data at PORT E DIR
	STR R1,[R3]						; storing PORT F Input/Output Data at PORT F DIR
	
	MOV R0,#0x0C						;Port E is using PE2 and PE3, so 1100 - x0C
	MOV R1,#0x10						;Port F is using PF4, so 0001 0000 = x10 
	LDR R2,=GPIO_PORTE_DEN_R				;R2 <- address of DEN PORT E
	LDR R3,=GPIO_PORTF_DEN_R				;R3 <- address of DEN PORT F
	STR R0,[R2]						; storing PORT E PINS Data at PORT E DEN
	STR R1,[R3]						; storing PORT F PINS Data at PORT F DEN
	
	LDR R0, =GPIO_PORTF_PUR_R
	LDR R1, [R0]
	ORR R1, #0X10
	STR R1, [R0]
	
     CPSIE  I   				 		; TExaS voltmeter, scope runs on interrupts
	 
	 
	 
	 
	LDR R8,tonn
	LDR R9,tof 
	LDR R10,incr
	LDR R11,min
	LDR R12,max
	
	
	loop  
							
	
	LDR R0,=GPIO_PORTE_DATA_R  				;get Port E Data 
	LDRB R1,[R0]					   	;get contents of the PORT E
	MOV R2,#0x08				           	; make a mask for PE3
	ORR R1,R2					        ;Isolate PE3 in PORT E DATA
	STR R1,[R0]					        ;SET PE3 Bit
	BL hold		                  			;stall
	EOR R1,#0x08					        ;invert bit
	STR R1,[R0]						;flip LED on or off
	BL hw				
	BL check
								;is breathing	
     	B    loop
	 
	check 		LDR R6,=GPIO_PORTE_DATA_R 		;load address of POrt E data
	  		LDR R7,[R6]				; load actusl PORT E data 
	  		MOV R6,#0x04				;create a mask to isolate PE2
	  		AND R7,R6				;Check if PE2 Switch was pressed
	  		CMP R7,#0
	  		BNE keepon				;branch if switch is pressed
	checkf	  
	  		LDR R6, =GPIO_PORTF_DATA_R
	  		LDR R7,[R6]				; load actual PORT f data 
	  		MOV R6,#0x10				;create a mask to isolate PF4
	  		AND R7,R6				;Check if PF4 Switch was pressed
	  		CMP R7,#0					
	  		BEQ breathe
     			B loop					; check status of PE2
					
	keepon
			LDR R6,=GPIO_PORTE_DATA_R	 	;load address of POrt E data
			MOV R7,#0x08				;get ready to turn on PE3
			STR R7,[R6]				;turning light on
			LDR R6,=GPIO_PORTE_DATA_R 		;load address of POrt E data
			LDR R7,[R6]				; load actusl PORT E data 
			MOV R6,#0x04				;create a mask to isolate PE2
			AND R7,R6				;Check if PE2 Switch was pressed
			CMP R7,#0
			BNE keepon
			B uptwenty

	uptwenty	
			CMP R9,R11				; compare to see if at max value
			BEQ reset 				; if at max value, go to 
			ADD R8,R10 
			SUBS R9,R10
			B checkf

	reset 		MOV R8,R11
			MOV R9,R12 
			B  loop 
			
	hold  		MOV R4, R8  				;hold is on
	ba		SUB R4,#1
	 		CMP R4,#0
	  		BNE ba
	  		BX  LR
	  
	hw    	 	MOV R5,R9 ;off
	by	 	SUB R5,#1
		  	CMP R5,#0
	  	 	BNE by
	  	  	BX  LR
			
			
	breathe 	
			LDR R0,=GPIO_PORTF_DATA_R		
			LDR R1,[R0]
			MOV R2,#0x10
			AND R1,R2				;CHECK JUST TO MAKE SURE
			CMP R1,#0
			BNE turnofftwo
			LDR R6,=8040				;bron is a counter
			LDR R7,=152760				;broff is a counter
	here 		
			LDR R1,=152760				;R1 holds the max for the bron counter
			CMP R6,R1
			BGT	renew
			BL turnon				;function to turn on light
			MOV R4, R6
	go
			SUB R4,#1
			CMP R4,#0
			BNE go
			BL turnoff 				;function to turn off light
			MOV R4, R7
	low			
			SUB R4,#1
			CMP R4,#0
			BNE low
			LDR R0,=8040				;bncr holds the 5% increment
			ADD R6,R0
			SUB R7,R0
			B here
			
	renew 		
			LDR R6,=152760
			LDR R7,=8040
	her
			LDR R1,=8040				;R1 holds the max for the bron counter
			CMP R6,R1
			BLT	breathe
			BL turnon				;function to turn on light
			MOV R4, R6
	yo
			SUB R4,#1
			CMP R4,#0
			BNE yo
			BL turnoff 				;function to turn off light
			MOV R4, R7
	dow		
			SUB R4,#1
			CMP R4,#0
			BNE dow
			LDR R0,=8040				;bncr holds the 5% increment
			SUB R6,R0
			ADD R7,R0
			B her
			
			B breathe

	turnon
		LDR R0,=GPIO_PORTE_DATA_R  			;get Port E Data 
		LDR R1,[R0]					;get contents of the PORT E
		MOV R2,#0x08					; make a mask for PE3
		ORR R1,R2					;Isolate PE3 in PORT E DATA
		STR R1,[R0]					;SET PE3 Bit
		BX LR

	turnoff 
		LDR R0,=GPIO_PORTE_DATA_R  			;get Port E Data 
		LDR R1,[R0]					;get contents of the PORT E
		MOV R2,#0x08					; make a mask for PE3
		EOR R1,R2					;Isolate PE3 in PORT E DATA
		STR R1,[R0]					;SET PE3 Bit
		BX LR

	datshit	LDR R0,=GPIO_PORTF_DATA_R		 
		LDR R1,[R0]
		MOV R2,#0x10
		AND R1,R2					;CHECK JUST TO MAKE SURE
		CMP R1,#0
		BEQ loop
		BX LR
		
	turnofftwo 
		LDR R0,=GPIO_PORTE_DATA_R  			;get Port E Data 
		LDR R1,[R0]					;get contents of the PORT E
		MOV R2,#0x08					; make a mask for PE3
		EOR R1,R2					;Isolate PE3 in PORT E DATA
		STR R1,[R0]					;SET PE3 Bit
		B loop
		
     ALIGN      ; make sure the end of this section is aligned
     END        ; end of file
