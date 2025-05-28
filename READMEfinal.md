section .data
    msgA0      db "Enter A[0]: ", 0
    msgA1      db "Enter A[1]: ", 0
    msgA2      db "Enter A[2]: ", 0
    msgB0      db "Enter B[0]: ", 0
    msgB1      db "Enter B[1]: ", 0
    msgB2      db "Enter B[2]: ", 0
    result_msg db "Dot product is: ", 0

section .bss
    A      resd 3 ; input for A 3 bytes
    B      resd 3 ; input for B 3 bytes
    input  resb 10 ; 10 total bytes reserved for user input 

section .text
    global _start

_start:
    ; --- read vector A ---
    mov ecx, A
    mov edx, msgA0
    call read_store    ; saving input after A0-2 to edx
    mov edx, msgA1
    call read_store
    mov edx, msgA2
    call read_store

    ; --- read vector B ---
    mov ecx, B
    mov edx, msgB0   ; same for vector B
    call read_store
    mov edx, msgB1
    call read_store
    mov edx, msgB2
    call read_store

    ; --- compute dot product into EAX ---
    xor eax, eax        ; accumulator = 0 cuz we gonna store sum here
    xor ebx, ebx        ; index = 0 cuz we start at 0
.dot_loop:
    mov esi, [A + ebx*4] ; separating input values into separate registers
    mov edi, [B + ebx*4] ; for B
    imul esi, edi ; multiplying the arrays
    add eax, esi ; putting sum into eax
    inc ebx ; setting ebx to go +!+!+!+
    cmp ebx, 3 ; loop before 3
    jl .dot_loop ; loops 0-2

    ; --- save dot in EAX across the upcoming syscall ---
    push eax ; saving sum value in eax cuz we gonna use it for something

    ; --- print "Dot product is: " ---
    mov eax, 4 ; syscall to write
    mov ebx, 1 ; 1 = screen
    mov ecx, result_msg ; where we have our string
    mov edx, 15             ; length of the string
    int 0x80 ; go shawty

    ; --- restore dot product into EAX ---
    pop eax ; putting sum back into eax with our string

    ; --- print the numeric result plus newline ---
    call print_eax 

    ; --- exit ---
    mov eax, 1 ; syscall number for exit 
    xor ebx, ebx ; 0 = xit code
    int 0x80 ; go


; read_store: print prompt in EDX, then read integer into [ECX], advance ECX by 4 ; saving arrays a and b one element at a time 
read_store:
    push ecx ; saving ecx
    ; print prompt
    mov eax, 4 ; write 
    mov ebx, 1 ; to screen 
    mov ecx, edx ; moving prompt string into ecx
    mov edx, 13     ; all prompts are 13 bytes long
    int 0x80 ; go

    ; read integer
    call read_int

    pop ecx ; bringing back to ecx
    mov [ecx], eax  ; store parsed int into array
    add ecx, 4 ; move forward 4 bytes each int is 4 bytes
    ret ; time for next thing


; read_int: read up to newline from STDIN into 'input', parse decimal, return value in EAX
read_int:
    ; read up to 10 bytes into 'input'
    mov eax, 3 ; syscall for read
    mov ebx, 0 ; reading from keyboard
    mov ecx, input ; putting inout into ecx
    mov edx, 10 ; read up to 10 bytes
    int 0x80 ; go

    ; parse ASCII digits (supports negative numbers)
    xor eax, eax            ; result = 0
    xor ebx, ebx            ; index = 0
    xor esi, esi            ; sign flag = 0

    mov dl, [input + ebx] ; load first char from inout buffer into dl
    cmp dl, '-' ; check if theres a minus sign
    jne .parse_loop ; if not a minus sign, skip to parsing digits 
    mov esi, 1              ; set sign flag
    inc ebx                 ; skip '-'

.parse_loop:
    mov dl, [input + ebx] ; get next character from inout buffer 
    cmp dl, 10              ; newline?
    je .done                ; if yes, we done parsing
    sub dl, '0'             ; convert ASCII to digit
    movzx edx, dl           ; move digit from dl to edx
    imul eax, eax, 10       ; multiply result by 10
    add eax, edx            ; add digit to result 
    inc ebx                 ; move to next character in input 
    jmp .parse_loop         ; go back and process next digit 

.done:
    cmp esi, 1 ;  check if number was marked as negative earlier 
    jne .ret ; if not negative, skip negation 
    neg eax ; if it was negative, negate final result 
.ret:
    ret ; return to caller


; print_eax: print signed integer in EAX, followed by newline
print_eax:
    mov ebx, 10             ; divisor set to 10 
    mov ecx, input+9        ; start from the end of the input buffer
    mov byte [ecx], 10      ; append '\n'
    cmp eax, 0              ; check if number is negative 
    jge .positive           ; if number is greater or = 0 skip to .pos
    neg eax                 ; if it was negative flip to positive 
    mov byte [ecx-1], '-'   ; put a minus sign in the buffer 
    dec ecx                 ; move back one position to prep for digits
.positive:
.convert:
    dec ecx                 ; move backward in buffer to store next digit
    xor edx, edx            ; clear edx before division
    div ebx                 ; eax = eax/10, edx = remainder
    add dl, '0'             ; convert the digit in edx to ascii
    mov [ecx], dl           ; store that ascii digit in buffer
    test eax, eax           ; check if theres more digits to convert
    jnz .convert            ; if eax is not 0 loop again

    mov eax, 4              ; sys_write
    mov ebx, 1              ; stdout
    ; ecx already points to start of string
    mov edx, input+10
    sub edx, ecx            ; length = (input+10)-ecx
    int 0x80
    ret
