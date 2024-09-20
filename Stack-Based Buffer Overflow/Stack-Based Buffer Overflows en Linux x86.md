# Vulnerable Program

Ahora estamos escribiendo un programa C simple llamado ``bow.c`` con una función vulnerable llamada ``strcpy()``.

## Bow.c

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int bowfunc(char *string) {

	char buffer[1024];
	strcpy(buffer, string);
	return 1;
}

int main(int argc, char *argv[]) {

	bowfunc(argv[1]);
	printf("Done.\n");
	return 1;
}
```
## Desactivar ASLR

Los sistemas operativos modernos tienen protecciones integradas contra este tipo de vulnerabilidades, como la aleatorización del diseño del espacio de direcciones (ASLR). Con el fin de aprender los conceptos básicos de la explotación del desbordamiento de búfer, vamos a desactivar estas funciones de protección de memoria:

```shell-session
student@nix-bow:~$ sudo su
root@nix-bow:/home/student# echo 0 > /proc/sys/kernel/randomize_va_space
root@nix-bow:/home/student# cat /proc/sys/kernel/randomize_va_space

0
```
## Compilación

A continuación, compilamos el código C en un binario ELF de 32 bits.

```shell-session
student@nix-bow:~$ sudo apt install gcc-multilib
student@nix-bow:~$ gcc bow.c -o bow32 -fno-stack-protector -z execstack -m32
student@nix-bow:~$ file bow32 | tr "," "\n"

bow: ELF 32-bit LSB shared object
 Intel 80386
 version 1 (SYSV)
 dynamically linked
 interpreter /lib/ld-linux.so.2
 for GNU/Linux 3.2.0
 BuildID[sha1]=93dda6b77131deecaadf9d207fdd2e70f47e1071
 not stripped
```
# Funciones C vulnerables

Existen varias funciones vulnerables en el lenguaje de programación C que no protegen la memoria de forma independiente. Estas son algunas de ellas:

- `strcpy`
- `gets`
- `sprintf`
- `scanf`
- `strcat`
- ...

# Presentacion del GDB

GDB, o GNU Debugger, es el depurador estándar de los sistemas Linux desarrollado por el Proyecto GNU. Se ha adaptado a muchos sistemas y es compatible con los lenguajes de programación C, C++, Objective-C, FORTRAN, Java y muchos más.

GDB nos proporciona las características habituales de trazabilidad como puntos de interrupción o salida de traza de pila y nos permite intervenir en la ejecución de programas. También nos permite, por ejemplo, manipular las variables de la aplicación o llamar a funciones independientemente de la ejecución normal del programa.

Usamos `GNU Debugger`( `GDB`) para visualizar el binario creado en el nivel de ensamblador. Una vez que hayamos ejecutado el binario con `GDB`, podemos desensamblar la función principal del programa.

## GDB - Sintaxis de AT&T

```shell-session
student@nix-bow:~$ gdb -q bow32

Reading symbols from bow...(no debugging symbols found)...done.
(gdb) disassemble main

Dump of assembler code for function main:
   0x00000582 <+0>: 	lea    0x4(%esp),%ecx
   0x00000586 <+4>: 	and    $0xfffffff0,%esp
   0x00000589 <+7>: 	pushl  -0x4(%ecx)
   0x0000058c <+10>:	push   %ebp
   0x0000058d <+11>:	mov    %esp,%ebp
   0x0000058f <+13>:	push   %ebx
   0x00000590 <+14>:	push   %ecx
   0x00000591 <+15>:	call   0x450 <__x86.get_pc_thunk.bx>
   0x00000596 <+20>:	add    $0x1a3e,%ebx
   0x0000059c <+26>:	mov    %ecx,%eax
   0x0000059e <+28>:	mov    0x4(%eax),%eax
   0x000005a1 <+31>:	add    $0x4,%eax
   0x000005a4 <+34>:	mov    (%eax),%eax
   0x000005a6 <+36>:	sub    $0xc,%esp
   0x000005a9 <+39>:	push   %eax
   0x000005aa <+40>:	call   0x54d <bowfunc>
   0x000005af <+45>:	add    $0x10,%esp
   0x000005b2 <+48>:	sub    $0xc,%esp
   0x000005b5 <+51>:	lea    -0x1974(%ebx),%eax
   0x000005bb <+57>:	push   %eax
   0x000005bc <+58>:	call   0x3e0 <puts@plt>
   0x000005c1 <+63>:	add    $0x10,%esp
   0x000005c4 <+66>:	mov    $0x1,%eax
   0x000005c9 <+71>:	lea    -0x8(%ebp),%esp
   0x000005cc <+74>:	pop    %ecx
   0x000005cd <+75>:	pop    %ebx
   0x000005ce <+76>:	pop    %ebp
   0x000005cf <+77>:	lea    -0x4(%ecx),%esp
   0x000005d2 <+80>:	ret    
End of assembler dump.
```

## GDB - Change the Syntax to Intel

```shell-session
(gdb) set disassembly-flavor intel
(gdb) disassemble main

Dump of assembler code for function main:
   0x00000582 <+0>:	    lea    ecx,[esp+0x4]
   0x00000586 <+4>:	    and    esp,0xfffffff0
   0x00000589 <+7>:	    push   DWORD PTR [ecx-0x4]
   0x0000058c <+10>:	push   ebp
   0x0000058d <+11>:	mov    ebp,esp
   0x0000058f <+13>:	push   ebx
   0x00000590 <+14>:	push   ecx
   0x00000591 <+15>:	call   0x450 <__x86.get_pc_thunk.bx>
   0x00000596 <+20>:	add    ebx,0x1a3e
   0x0000059c <+26>:	mov    eax,ecx
   0x0000059e <+28>:	mov    eax,DWORD PTR [eax+0x4]
<SNIP>
```

## Change GDB Syntax permanently

```shell-session
student@nix-bow:~$ echo 'set disassembly-flavor intel' > ~/.gdbinit
```

# Exploitation
## Take Control of EIP

#### Falla de segmentación

```shell-session
student@nix-bow:~$ gdb -q bow32

(gdb) run $(python -c "print '\x55' * 1200")
Starting program: /home/student/bow/bow32 $(python -c "print '\x55' * 1200")

Program received signal SIGSEGV, Segmentation fault.
0x55555555 in ?? ()
```

Si insertamos 1200 " `U`"s (hex " `55`") como entrada, podemos ver a partir de la información del registro que hemos sobrescrito el `EIP`. Hasta donde sabemos, el `EIP`apunta a la siguiente instrucción que se ejecutará.

```shell-session
(gdb) info registers 

eax            0x1	1
ecx            0xffffd6c0	-10560
edx            0xffffd06f	-12177
ebx            0x55555555	1431655765
esp            0xffffcfd0	0xffffcfd0
ebp            0x55555555	0x55555555		# <---- EBP overwritten
esi            0xf7fb5000	-134524928
edi            0x0	0
eip            0x55555555	0x55555555		# <---- EIP overwritten
eflags         0x10286	[ PF SF IF RF ]
cs             0x23	35
ss             0x2b	43
ds             0x2b	43
es             0x2b	43
fs             0x0	0
gs             0x63	99
```

#### Determinar el desplazamiento

El desplazamiento se utiliza para determinar cuántos bytes se necesitan para sobrescribir el búfer y cuánto espacio tenemos alrededor de nuestro shellcode.

#### Crear patrón

```shell-session
0xRh4ps00dy@htb[/htb]$ /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 1200 > pattern.txt
0xRh4ps00dy@htb[/htb]$ cat pattern.txt

Aa0Aa1Aa2Aa3Aa4Aa5...<SNIP>...Bn6Bn7Bn8Bn9
```

#### GDB - Uso de patrones generados

```shell-session
(gdb) run $(python -c "print 'Aa0Aa1Aa2Aa3Aa4Aa5...<SNIP>...Bn6Bn7Bn8Bn9'") 

The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /home/student/bow/bow32 $(python -c "print 'Aa0Aa1Aa2Aa3Aa4Aa5...<SNIP>...Bn6Bn7Bn8Bn9'")
Program received signal SIGSEGV, Segmentation fault.
0x69423569 in ?? ()
```

#### BGF - EIP

```shell-session
(gdb) info registers eip

eip            0x69423569	0x69423569
```

#### GDB - Desplazamiento

```shell-session
0xRh4ps00dy@htb[/htb]$ /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 0x69423569

[*] Exact match at offset 1036
```




## Determine the Length for Shellcode



## Identifications of Bad Characters



## Generating Shellcode



## Generating Shell



## Identification of the Return Address
