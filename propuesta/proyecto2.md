% Proyecto 2
% CI3825 (Sistemas de operación I)
% Enero–Marzo 2012

- - -

# Introducción

*Esta sección es informativa.*



## Objetivos

1.  Los objetivos de esta asignación incluyen:

    1.  Familiarizarse con las llamadas al sistema para la creación y acceso a las estructuras del sistema de archivos.

    2.  Desarrollar un mecanismo de comunicación entre un proceso y múltiples hijos concurrentes utilizando pipes no nominales y la llamada al sistema `select`.

    3.  Fortalecer el conocimiento de herramientas de automatización del proceso de desarrollo de software como `make`.



## *Build system*s

1.  El proceso de desarrollo de software con cualquier conjunto de herramientas implica realizar ciertas tareas repetitivas para transformar los datos crudos y el código fuente desarrollados por los programadores en un producto terminado que se pueda distribuir y utilizar.  Si se utilizan lenguajes de programación compilados, este proceso incluye la ejecución de compiladores para transformar código fuente en ejecutables.  Esta tarea puede resultar tediosa en el contexto de un proyecto de complejidad considerable, y como el tiempo de un desarrollador de software es mejor aprovechado cuando se utiliza para desarrollar software, sería deseable eliminar esa carga de trabajo con procesos automatizados.

2.  Un ***build system*** es un sistema de automatización para las tareas de producción de paquetes de software a partir de código y datos fuente.  Esta definición general incluye sistemas de alcances diversos:

    1.  Los `Makefile`s simples para compilar alguno de sus proyectos pueden ser considerados *build system*s: son programas que describen el proceso de construcción de los ejecutables de sus proyectos.

    2.  El programa que interpreta sus `Makefile`s y ejecuta acciones descritas en ellos, [make][], también puede considerarse un *build system*.

    3.  Existen *build system*s que trabajan a un nivel más alto, como [GNU Automake][] y [GNU Autoconf][] que son dos de los componentes del [GNU build system], también conocido como *Autotools*.  Este sistema puede generar `Makefile`s automáticamente ajustados a las peculiaridades de cada tipo de sistema UNIX, lo que lo hace adecuado para proyectos de gran escala que deban funcionar en múltiples plataformas.

[make]: <https://www.gnu.org/software/make>
        GNU Make: la implementación del proyecto GNU del build system básico tradicionalmente encontrado en UNIX.

[GNU Automake]: <https://www.gnu.org/software/automake>
        GNU Automake: un componente del GNU build system.

[GNU Autoconf]: <https://www.gnu.org/software/autoconf>
        GNU Autoconf: un componente del GNU build system.

[GNU build system]: <https://www.gnu.org/software/automake/manual/html_node/GNU-Build-System>
        Una breve descripción del GNU build system y su motivación.



## Compilación separada por directorios

1.  En un proyecto de software de escala no trivial resulta conveniente dividir la fuente del proyecto en categorías y agrupar los archivos de cada una en subdirectorios del directorio del proyecto.  Por ejemplo, un sistema de documentación de código escrito en C con soporte para múltiples lenguajes podría tener una estructura de directorios en su código fuente como esta:

        multidoc/             # Directorio principal del proyecto
        multidoc/src          # Código fuente del proyecto
        multidoc/src/c        # Uso de documentación en código C
        multidoc/src/c++      # Uso de documentación en código C++
        multidoc/src/asm      # Uso de documentación en assembly
        multidoc/src/asm/x86  # Lenguaje de máquina x86
        multidoc/src/asm/mips # Lenguaje de máquina MIPS
        multidoc/src/java     # Uso de documentación en código Java
        multidoc/lib          # Código de una biblioteca incluida
        multidoc/doc          # Documentación del proyecto
        multidoc/test         # Pruebas de correctitud del proyecto

2.  El directorio `src` y cada uno de sus subdirectorios podrían contener varios archivos de código que se compilen de la misma manera a archivos de objeto y se combinen finalmente en un ejecutable de todo el proyecto.  En ese caso, sería conveniente tener en cada subdirectorio un `Makefile` que describa cómo compilar los archivos de código de ese subdirectorio, y si contiene a otro subdirectorio, que llame a los `Makefiles` contenidos en ellos.  Por ejemplo, el `Makefile` del directorio `multidoc/src` llamará al `Makefile` del directorio `multidoc/src/asm`, que a su vez llamará al `Makefile` del directorio `x86`.  Claro que cada `Makefile` no debe llamar a solo *uno* de los `Makefiles` de sus subdirectorios, sino a *todos*.

3.  Los archivos de código (los `.c`) de cada subdirectorio generarán sus correspondientes archivos de objeto compilados (los `.o`).  El objetivo del proceso de compilación es reunir todo el código generado (todos los `.o`) para crear finalmente un archivo ejecutable en la raíz de la jerarquía de directorios del proyecto (que es el producto final del proyecto ficticio `multidoc`).  Para facilitar este proceso, en cada subdirectorio debería crearse un archivo `.a` que contenga la *colección de todos los `.o`* de ese subdirectorio.  Luego, el *directorio padre* de ese subdirectorio podría hacer lo mismo con sus archivos de código, y en su colección incluir a las *colecciones* (los `.a`) de todos sus directorios hijos.

4.  Por ejemplo, suponga que en `multidoc/src/asm/x86` existen dos archivos de código a compilar: `parse.c` y `opcodes.c`.  Suponga además que `opcodes.c` incluye a un archivo de encabezado `opcodes.h`, y que `parse.c` incluye a `parse.h`, y `parse.h` a su vez también incluye a `opcodes.h`.  El directorio `multidoc/src/asm/x86` debería contener un `Makefile` que diga, por ejemplo,

```Makefile
        .PHONY: all
        all: x86.a

        x86.a: parse.o opcodes.o
        	ar rT $@ $?

        parse.o: parse.c parse.h opcodes.h
        opcodes.o: opcodes.c opcodes.h
```

5.  El comando

        ar rT archivo.a elemento1 elemento2 ...

    crea un archivo de colección llamado `archivo.a` que contendrá a `elemento1`, `elemento2`, etc.  Si alguno de esos elementos es a su vez un archivo de colección, se almacenarán sus elementos por separado en vez del archivo completo.  Recuerde que `$@` en una *receta* de un `Makefile` se sustituye por el nombre del archivo que causó que se corriera esa regla, que en este caso, es `asm.a`; además, `$?` se sustituye por aquellas dependencias de la regla que deban actualizarse: por ejemplo, si acabamos de compilar de nuevo a `opcodes.c` y obtuvimos una nueva versión de `opcodes.o`, se actualizará la copia de `opcodes.o` en el archivo de colección `asm.a`, pero si no se actualizó `parse.o`, no se sustituirá su copia en la colección (porque no es necesario hacerlo).

6.  Suponga ahora que se hizo lo mismo en el directorio `multidoc/src/asm/mips`.  En el directorio `multidoc/src/asm` sucede lo mismo, excepto que al archivo de colección habrá que insertarle los elementos de las colecciones de sus subdirectorios:

```Makefile
        .PHONY: all force
        all: asm.a

        asm.a: x86/x86.a mips/mips.a asm.o marmota.o ...
        	ar rT $@ $?

        x86/x86.a: force
        	$(MAKE) -C x86

        mips/mips.a: force
        	$(MAKE) -C mips

        asm.o: asm.c asm.h marmota.h
        marmota.o: marmota.c marmota.h
        ...
```

7.  El nuevo archivo de objeto `x86/x86.a` contendrá todo el código compilado correspondiente a ese directorio y a todos sus subdirectorios (recursivamente).

8.  Note que la regla *phony* “force” se utiliza como prerequisito para los archivos de colección de los subdirectorios; el efecto de esto es que **siempre** se harán las llamadas recursivas a `make` para los subdirectorios.  Esto es necesario porque el `Makefile` de un directorio no tiene información sobre si es necesario actualizar el código compilado en un subdirectorio.  Sin embargo, el `Makefile` que está dentro de ese subdirectorio sí será capaz de determinar esto.  Por lo tanto, la llamada recursiva se hace incondicionalmente, y si había algo que actualizar dentro de un subdirectorio, la invocación recursiva de `make` hará el trabajo y actualizará el archivo de colección de ese subdirectorio.

9.  Finalmente, en el directorio principal del proyecto se compilarán todos los archivos compilables que estén directamente en él, y se enlazarán todos sus archivos de objeto y todos los archivos de colección de los subdirectorios del proyecto.  El resultado de esto debe ser un ejecutable.  En el directorio principal del proyecto podría haber, por ejemplo, este `Makefile`:

```Makefile
        .PHONY: all
        all: multidoc

        multidoc: src/src.a lib/lib.a main.o args.o ...
        	$(CC) $(CPPFLAGS) $(CCFLAGS) -o $@ $^

        src/src.a: force
        	$(MAKE) -C src

        lib/lib.a: force
        	$(MAKE) -C lib

        main.o: main.c main.h args.h
        args.o: args.c args.h
        ...
```

10. Recuerde que `$^` en una receta de una regla se sustituye con los nombres de todos los prerequisitos de la regla separados por espacio.  La receta mostrada en el ejemplo de `Makefile` anterior genera el ejecutable final del proyecto que contiene todo el código compilado de todos los archivos fuente del proyecto.

11. Note que no se incluyó `doc/doc.a` ni `test/test.a` porque presumiblemente esos directorios no se usan en este proyecto para almacenar código fuente.

12. El objetivo de este proyecto es que escriba en C un *build system* que sea capaz de **generar** los `Makefile`s en situaciones como la de este ejemplo, con la restricción de que el procesamiento propio de cada directorio debe hacerse en procesos separados que se comuniquen a través de *pipes* y señales del sistema operativo.

13. Los compiladores de C casi universalmente soportan opciones especiales que hacen que en vez de compilar un archivo de código, generen el código correspondiente a la regla que debería contener un `Makefile` para compilar y mantener actualizado correctamente al archivo de objeto de ese archivo de código.  Por ejemplo, en el caso del archivo de ejemplo `parse.c` anteriormente mencionado, podría ejecutarse en su directorio el comando

        gcc -E -MMD parse.c

    y se generará otro archivo llamado `parse.d` cuyo contenido será

        parse.o: parse.c parse.h opcodes.h

    que es precisamente lo que debe contener el `Makefile` que esté en el directorio de `parse.c`.  Hay varias variaciones de estas opciones; todas comienzan con `-M`, pero algunas generan un archivo con un nombre fijo, otras generan la salida en un archivo cuyo nombre es parte de la opción, otras generan el texto de la regla en la salida estándar del compilador, etc.  El manual de `gcc` describe en detalle todas las opciones que provee para cálculo de dependencias para `Makefile`s.



- - -



# Enunciado

*Esta sección es normativa.*

## Requerimientos

1.  Usted debe desarrollar un programa llamado `rautomake` escrito en el lenguaje de programación C.

2.  `rautomake` será ejecutado dentro de un cierto directorio.  Por ejemplo, suponga que el ejecutable de `rautomake` (que será el resultado de que usted haya compilado su proyecto) reside en

        /home/marmota/USB/CI3825/2012EM/P2/src/rautomake

    y usted ejecuta el comando

        ../src/rautomake

    en un interpretador de línea de comando cuyo directorio actual es

        /home/marmota/USB/CI3825/2012EM/P2/test

    entonces el directorio donde se ejecuta `rautomake` será

        /home/marmota/usb/CI3825/2012EM/P2/test

3.  Es necesario que `rautomake` cree un `Makefile` en un subdirectorio del directorio donde se ejecutó, o en ese mismo directorio, si `rautomake` tiene permiso de lectura, escritura y acceso a ese directorio y a todos sus padres hasta el directorio donde fue ejecutado `rautomake`, inclusive, y en él existen archivos para los que `rautomake` tenga permiso de lectura y cuyos nombres terminen en `.c`, o si es necesario que `rautomake` cree un `Makefile` en algún subdirectorio del directorio en cuestión.

4.  Cada `Makefile` que `rautomake` deba crear deberá contener una regla para cada archivo en el directorio de ese `Makefile` cuyo nombre termine en `.c`.  El contenido de la regla será el mismo que algún compilador de C generaría al pedirle que genere la regla correspondiente a ese archivo para un `Makefile`; debe tener como prerequisitos al menos a todos los archivos incluídos directa o indirectamente por el archivo en cuestión que estén dentro del directorio en el que fue ejecutado `rautomake`, y no debe tener como prerequisito a ningún archivo que no esté incluído ni directa ni indirectamente por el archivo en cuestión.

5.  Cada `Makefile` que `rautomake` deba crear en un directorio distinto a donde fue ejecutado deberá contener las reglas necesarias para crear un archivo de colección con el programa `ar` que contenga la colección de todo el código de objeto generado al compilar cada uno de los archivos de ese directorio cuyo nombre termine en “.c”, junto con todo el código de objeto en archivos de colección de sus subdirectorios en los que `rautomake` deba crear `Makefile`s.

6.  Si `rautomake` debió crear algún `Makefile`, entonces `rautomake` deberá crear un `Makefile` en el directorio donde fue ejecutado.  Ese `Makefile` deberá contener una regla que cree un archivo ejecutable cuyo nombre sea igual al nombre del directorio donde `rautomake` fue ejecutado; ese archivo ejecutable será el resultado de enlazar todos los archivos de objeto correspondientes a archivos en el directorio en cuestión cuyos nombres terminen en “.c”, junto con el contenido de los archivos de colección correspondientes a los subdirectorios del directorio en cuestión en los que `rautomake` debiera generar un `Makefile`.  Los `Makefile`s que `rautomake` debe crear deben permitir que se genere el ejecutable mencionado en este párrafo si se ejecuta la orden `make` en el mismo directorio donde `rautomake` fue llamado, suponiendo que todos los archivos de código compilen y ese ejecutable se pueda generar enlazando los resultados de todos los archivos de objeto resultantes.

7.  `rautomake` será ejecutado únicamente en directorios cuyos nombres estén compuestos únicamente de caracteres alfanuméricos de ASCII, o el caracter “FULL STOP”, también llamado “punto” (“.”); tampoco tendrán caracteres fuera de ese conjunto los nombres de cualquier archivo contenido en los directorios donde `rautomake` sea ejecutado, ni sus subdirectorios, ni los archivos contenidos en sus subdirectorios; tampoco será ninguno de esos nombres exactamente igual a “.c”.  No es necesario que `rautomake` verifique esto.

8.  `rautomake` puede suponer que no existe ningún archivo dentro del directorio donde es ejecutado, ni dentro de ninguno de sus subdirectorios, cuyo nombre termine en “.d”.  Si tales archivos existen, `rautomake` puede eliminarlos o sobreescribirlos para cualquier fin.  `rautomake` no será ejecutado en un directorio donde existan tales archivos y `rautomake` no tenga permiso de lectura y escritura sobre esos archivos.  `rautomake` no será ejecutado en un directorio cuyo nombre termine en “.d”, ni que tenga algún subdirectorio cuyo nombre termine en “.d”.

9.  Un proceso de `rautomake` no puede visitar, abrir ni manipular más de un directorio a lo largo de su ejecución.  Si `rautomake` requiere visitar, abrir o manipular más de un directorio, deberá copiarse a sí mismo y una de sus copias podrá visitar, abrir o manipular un directorio distinto al visitado, abierto o manipulado por el proceso original.

10. La comunicación entre varios procesos resultantes de una ejecución de `rautomake` solo podrá hacerse a través de *pipes* no nominales y/o señales del sistema operativo, a menos que uno de los procesos involucrados sea un compilador de C, en cuyo caso es aceptable la comunicación a través de archivos.

11. A menos que este documento haga una excepción explícita particular, `rautomake` **debe** manejar explícitamente absolutamente todos los efectos y resultados de llamadas al sistema o a bibliotecas que puedan afectar al flujo de ejecución de su implementación de `rautomake`.  En particular, si su implementación de `rautomake` deja de revisar una posible condición de error en el retorno de una llamada al sistema o a alguna biblioteca que utilice, incluyendo la biblioteca estándar de C, se esperará que usted pueda demostrar rigurosamente que el flujo de ejecución de su programa nunca depende de esa condición.

## Entrega

1.  El código principal de `rautomake` debe estar escrito en el lenguaje de programación C en cualquiera de sus versiones, y debe poder compilarse con las herramientas GNU disponibles en la versión estable más reciente al momento de la entrega de Debian GNU/Linux, y, en particular, debe poder compilarse y ejecutarse en las computadoras del LDC.

2.  La entrega consistirá de un archivo en formato `tar.gz` o `tar.bz2` que contenga todo el código que haya desarrollado y sea necesario para compilar su programa.  No debe incluir archivos compilados.  Su proyecto debe poder compilarse extrayendo los archivos del paquete de su entrega, entrando en el directorio generado por la extracción, y ejecutando el comando `make`; es decir que *debe* incluir al menos un `Makefile` para automatizar la compilación de su proyecto.

3.  Aunque en general es preferible que su código sea compatible con los documentos de estandarización más recientes publicados para cada tecnología que utilice, es aceptable que requiera y use extensiones propias de la implementación de las herramientas de la plataforma GNU/Linux tanto del lenguaje de programación C como de la interfaz con el sistema operativo.  Es aceptable que requiera alguna versión de las herramientas del sistema diferente de las que están instaladas y disponibles en la versión estable más reciente al momento de la entrega de Debian GNU/Linux, y en particular de lo que esté instalado y disponible en las computadoras del LDC, pero deberá justificar estos requerimientos adicionales.