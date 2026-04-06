# Laboratorio Post-Contenido 1 — Unidad 3: Manejo del DEBUG
---

## Propósito del Laboratorio

El presente laboratorio tuvo como objetivo configurar el entorno DOSBox, acceder
al depurador DEBUG y utilizar los comandos R, D, F y U para inspeccionar el estado
de los registros del procesador, manipular bloques de memoria y desensamblar
código en modo real x86. Cada etapa fue documentada mediante capturas de pantalla
almacenadas en la carpeta `/capturas/` de este repositorio.

---

## Comandos Utilizados

| Comando | Función |
|---------|---------|
| `R`     | Muestra y modifica los registros del procesador |
| `D`     | Volcado hexadecimal de un bloque de memoria (Dump) |
| `F`     | Rellena un rango de memoria con un patrón de bytes (Fill) |
| `U`     | Desensambla bytes de memoria a instrucciones legibles (Unassemble) |
| `A`     | Ensambla instrucciones directamente en memoria |

---

## Parte A — Configuración del Entorno DOSBox

Se montó la carpeta `C:\DEBUG8086` del sistema anfitrión como unidad C: virtual
mediante el comando `MOUNT C C:\DEBUG8086`. El sistema confirmó el montaje
con el mensaje `Drive C is mounted as local directory C:\DEBUG8086\`.

Al intentar crear la subcarpeta `LAB3POST1` con `MD LAB3POST1`, el sistema
respondió `Unable to make: LAB3POST1`, lo cual indicó que la carpeta ya existía
previamente en el disco. Se navegó hacia ella con `CD LAB3POST1` sin inconveniente.

---

## Checkpoint 1 — Inspección de Registros con el Comando R

Se ejecutó el comando `R` para observar el estado inicial completo del procesador.

```
AX=0000  BX=0000  CX=0000  DX=0000  SP=FFFF  BP=0000  SI=0000  DI=0000
DS=0340  ES=0340  SS=0340  CS=0340  IP=0100   NV UP DI PL NZ NA PO NC
0340:0100  0000        ADD   [BX+SI],AL     DS:0000=CD
```

Se observó que los cuatro registros de propósito general (AX, BX, CX, DX)
iniciaron en 0x0000. El Stack Pointer apuntó a 0xFFFF, correspondiente
al tope inicial de la pila en el segmento asignado. Los cuatro registros de
segmento compartieron el mismo valor 0x0340, que corresponde
al párrafo de memoria asignado por DOS para el PSP del programa. El Instruction
Pointer se ubicó en 0x0100, que es la primera dirección ejecutable
inmediatamente después del PSP. La bandera DI indicó que las interrupciones
se encontraban deshabilitadas en el estado inicial.

A continuación se modificó el registro AX al valor 0x1234 mediante el
comando `R AX`. El DEBUG mostró el valor previo `AX 0000` y esperó la entrada
del nuevo valor. Tras ingresar `1234` y volver a ejecutar `R AX`, se confirmó
que AX pasó a contener el valor 0x1234 sin que ningún otro registro resultara
afectado, demostrando que la modificación es completamente selectiva.
---

## Checkpoint 2 — Volcado de Memoria con D y Relleno con F

La salida del comando `D` se compone de tres columnas diferenciadas.
La primera columna muestra la dirección de memoria en formato segmento:offset, 
indicando exactamente la posición física que ocupa
cada fila en el espacio de memoria del proceso. La segunda columna despliega
16 bytes por fila en representación hexadecimal, separados por un guion central
que divide los dos grupos de 8 bytes para facilitar la lectura. La tercera columna
presenta la representación ASCII de esos mismos 16 bytes. Cuando un byte no
corresponde a un carácter imprimible, el DEBUG
sustituye ese byte por un punto.

El primer intento del comando `F 200 L40 AB CD EF` produjo un error de sintaxis, 
ocasionado por un carácter incorrecto en la entrada. En el segundo
intento el comando se ejecutó correctamente sin mensaje de confirmación. El
prompt indicó que la operación concluyó sin errores. El patrón `AB CD EF`
se repitió de forma cíclica para cubrir los 64 bytes (0x40) solicitados a partir
de la dirección DS:0200.

```
0340:0200  AB CD EF AB CD EF AB CD-EF AB CD EF AB CD EF AB
0340:0210  CD EF AB CD EF AB CD EF-AB CD EF AB CD EF AB CD
0340:0220  EF AB CD EF AB CD EF AB-CD EF AB CD EF AB CD EF
0340:0230  AB CD EF AB CD EF AB CD-EF AB CD EF AB CD EF AB
```

El patrón `AB CD EF` resultó claramente visible y se repitió de forma cíclica
en las cuatro filas del volcado. La columna ASCII mostró únicamente caracteres
no imprimibles representados como puntos, dado que los valores 0xAB, 0xCD y
0xEF se encuentran fuera del rango ASCII visible.

```
0340:0000  CD 20 3F A3 00 EA FF FE-BD DE DB 01 92 01 E0 01
0340:0010  92 01 10 01 18 01 92 01-01 01 01 01 00 02 FF FF
```
---

## Checkpoint 3 — Ensamblado y Desensamblado

Antes de escribir el programa se ejecutó `U 100 L10` para observar el contenido
inicial de la región CS:0100. Se obtuvieron únicamente instrucciones
`ADD [BX+SI],AL`, que corresponden a la codificación de los bytes 0x00 0x00
presentes en memoria no inicializada. Esto confirmó el principio fundamental
del modo real x86. El procesador no distingue entre código y datos, interpreta
como instrucción cualquier byte al que apunte CS:IP.

El primer intento de ensamblar utilizó la sintaxis `A 100 MOV AX,0005`
directamente en la línea de comando, lo cual produjo un error (`^ Error`)
porque el comando `A` no acepta instrucciones en la misma línea. En el segundo
intento se invocó `A 100` sin argumentos adicionales y se ingresaron las
instrucciones línea por línea de forma correcta:

```
0482:0100  MOV AX,0005
0482:0103  MOV BX,0003
0482:0106  ADD AX,BX
0482:010B  INT 20
0482:010A
```

### Verificación con U 100 109

```
0482:0100  BB0500    MOV    AX,0005
0482:0103  BB0300    MOV    BX,0003
0482:0106  01D8      ADD    AX,BX
0482:0108  CD20      INT    20
```

Se verificó la correspondencia entre instrucciones y bytes de código máquina.
La instrucción `ADD AX,BX` se codificó como `01 D8` y la instrucción `INT 20`
como `CD 20`. El programa completo ocupó 10 bytes en memoria, demostrando
la compacidad del conjunto de instrucciones x86 en modo real.

Se observó además que el valor del segmento en esta sesión fue 0x0482, diferente
al 0x0340 de las sesiones anteriores, lo que evidencia que DOS asigna dinámicamente
el párrafo de memoria disponible cada vez que se inicia el DEBUG.

---

## Conclusiones

El laboratorio permitió comprender el funcionamiento del
depurador DEBUG y la arquitectura x86. Se verificó que los
registros de segmento apuntan al mismo párrafo de memoria al iniciar DEBUG,
correspondiente al PSP asignado por DOS. La inspección del PSP evidenció
la instrucción `INT 20` como mecanismo de terminación controlada de programas
COM. El relleno de memoria con el patrón `AB CD EF` y su posterior volcado con
`D` permitió comprender la estructura de tres columnas de la salida del comando. 
Finalmente, el ejercicio de ensamblado y desensamblado demostró que
no existe distinción entre código y datos, ya que el procesador interpreta como 
instrucción cualquier byte al que apunte el registro CS:IP, y que cada error de 
sintaxis cometido durante la sesión constituyó en sí mismo una oportunidad de 
aprendizaje sobre el uso correcto de los comandos del DEBUG.
