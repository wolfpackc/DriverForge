# DriverForge

Laboratorio para escribir drivers WDM de software para Windows en C, ver qué ocurre dentro del kernel cuando un programa los usa, y compilarlos y firmarlos de verdad.

El mismo código C tiene dos destinos:

1. Simulación. Un intérprete de un subconjunto realista de C ejecuta tu driver dentro de un kernel de Windows 11 x64 simulado, con su administrador de E/S, el Object Manager, el Security Reference Monitor, el pool, los IRQL y SEH. Los procesos de usuario son scripts que se ejecutan sentencia a sentencia.
2. Compilación real. El mismo `.c` se compila con WDK y MSVC (o con MinGW-w64), se enlaza como driver nativo, se firma con un certificado de pruebas y se generan `install.cmd` y `uninstall.cmd`. DriverForge nunca carga drivers: eso lo haces tú, en una máquina de pruebas.

## Inicio rápido

El repositorio no incluye el ejecutable. Compílalo siguiendo el apartado [Compilar DriverForge](#compilar-driverforge) y sigue desde ahí.

1. Ejecuta `DriverForge.exe`. No necesita administrador. Se abre la plantilla "Hola, driver".
2. Pulsa "Cargar Hello (sc start, simulado)", o directamente el botón de ejecutar: si no hay ningún driver cargado, lo carga antes de la primera sentencia. Se crea el `DRIVER_OBJECT`, se ejecuta `DriverEntry` y en el lienzo aparecen el `DEVICE_OBJECT`, su descriptor de seguridad y el enlace simbólico (pestaña Object Manager, como en WinObj).
3. Pulsa Ejecutar (F10) para lanzar la primera sentencia de `cliente.exe`. En el panel "Flujo de la petición" verás el camino completo: `CreateFileW`, ntdll (`\\.\Hello` pasa a `\??\Hello`), el Object Manager (DosDevices de la sesión, `\GLOBAL??\Hello`, el enlace, `\Device\Hello`), el SRM (el token frente al descriptor de seguridad), el `FILE_OBJECT`, `IRP_MJ_CREATE`, tu rutina `HelloCreateClose`, `IoCompleteRequest` y el handle.
4. Sigue sentencia a sentencia o con "Todo" (F5). Los errores salen en rojo con su causa y su solución, y algunos traen un arreglo rápido que repite la sentencia sin empezar de cero.
5. Compilar .sys real (Ctrl+B) compila y firma, pero no carga nada. Deja el `.sys` en `Documentos\DriverForge\<escenario>\<driver>\`.

## Cambios en la 1.0.2

- Copiar, cortar, pegar, deshacer y seleccionar todo funcionan con teclado, con los botones de la cabecera del editor y con el menú contextual. Ctrl+Mayús+F reformatea la sangría.
- Los fragmentos y las plantillas vuelven a responder al arrastrar y al doble clic. Antes una zona de desplazamiento se tragaba los clics.
- Ejecutar carga automáticamente los drivers que no están cargados antes de la primera sentencia.
- El recorrido completo se dibuja en el lienzo con saltos numerados: proceso, enlace simbólico, `DEVICE_OBJECT`, SRM, `FILE_OBJECT`, IRP, `DRIVER_OBJECT.MajorFunction[]` y tu rutina. En `DeviceIoControl` pasa por el handle, el `HANDLE_TABLE_ENTRY`, el `FILE_OBJECT`, el `DEVICE_OBJECT` y el `DRIVER_OBJECT`.
- El estado de cada `FILE_OBJECT` y de cada IRP se ve con colores: amarillo si está en curso, verde si se ha aceptado o completado y rojo si se ha rechazado o cerrado.

## Qué incluye

- Editor con resaltado, diagnósticos al escribir, autocompletado (Ctrl+Espacio), ayuda de parámetros con valores sugeridos en los que se puede hacer clic (por ejemplo el acceso de `CreateFileW` o el método de `CTL_CODE`), deshacer y rehacer, comentar (Ctrl+/) y duplicar línea (Ctrl+D).
- Fragmentos sueltos y agrupados (plantillas completas) para el driver y para el proceso. Se arrastran al editor o se insertan con doble clic.
- Lienzo de objetos con cajas arrastrables que muestran todos los campos (los secundarios, plegados): punteros como flechas, `MajorFunction[]` completo, `DeviceExtension` tipada con tu `struct`, bloques de pool, `SystemBuffer`, MDL, `EPROCESS`, `TOKEN`, `ETHREAD`, la tabla de handles con sus entradas y, si quieres, las cabeceras `OBJECT_HEADER`. Un pulso animado recorre el flujo de cada petición.
- Edición de campos con doble clic en un campo o en el SDDL de un descriptor. La pestaña de código de cada caja enseña las líneas que crearon o tocaron el objeto y los cambios manuales: en rojo si corrigen el error de la última operación y en azul si no.
- Varios drivers y varios procesos por escenario. Cada proceso tiene su token: usuario estándar, administrador filtrado por UAC, administrador elevado, integridad baja o SYSTEM.
- Errores fieles a Windows: ruta incorrecta, enlace olvidado, `DO_DEVICE_INITIALIZING`, dispositivo exclusivo, acceso denegado por DACL o por integridad (NO_WRITE_UP), un IOCTL que exige `FILE_WRITE_ACCESS`, `MajorFunction` sin asignar, IRP sin completar (proceso zombi), `STATUS_PENDING` sin `IoMarkIrpPending`, fuga de información por un `Information` mayor que el búfer o por bytes de pool sin inicializar, descarga pendiente con handles abiertos, `DriverUnload` ausente (error 1052) y fugas de dispositivos, enlaces y pool al descargar. También reproduce pantallazos azules con su código real: 0x3B, 0x7E, 0x50, 0x19, 0x44, 0xC2, 0xD5, 0xBE, 0xCE, 0x7F, 0x4A, 0x133, 0x1E y otros.
- Pantallazo azul al estilo de Windows 11, con explicación, parámetros y un botón que lleva al código.
- Nueve plantillas de escenario integradas y plantillas propias, escenarios `.dfs` que se guardan y se abren, y un PDF explicativo con el código, los flujos, los objetos, el espacio de nombres, la salida y el pantallazo azul.
- Tema claro y oscuro, ajustes y paleta de comandos (Ctrl+K).

## Compilación y firma reales

| Paso | WDK + MSVC | MinGW-w64 |
|---|---|---|
| Detección | `vswhere` y `KitsRoot10` (registro) | `C:\msys64`, WinLibs, PATH o descarga |
| Compilar | `cl /kernel /GS- ...` | `gcc ... -Wl,--subsystem,native -Wl,--entry,DriverEntry` y `df_compat.h` |
| Enlazar | `link /DRIVER /SUBSYSTEM:NATIVE` con `ntoskrnl.lib hal.lib wdmsec.lib` | `-lntoskrnl -lhal` |
| Firma | Certificado autofirmado de firma de código (RSA 2048, SHA-256) en `CurrentUser\My` y `SignerSignEx2`, o el `signtool` del WDK | Igual |

Si no encuentra compilador, DriverForge ofrece descargar MinGW-w64 (WinLibs) en tu perfil, sin administrador.

MinGW tiene dos limitaciones que el registro avisa. GCC no implementa `__try/__except`: compila, pero no captura excepciones. Y `IoCreateDeviceSecure` usa una implementación propia con un subconjunto de SDDL. Si tu driver usa SEH, compílalo con WDK y MSVC.

Para cargar el `.sys` en una máquina de pruebas: desactiva Secure Boot, ejecuta `activar_testsigning.cmd`, reinicia y ejecuta `install.cmd`.

## Fidelidad de la simulación

Los datos (direcciones, PID, LUID, SID de usuario) son inventados pero coherentes: el pool está en `0xFFFFC305...`, las imágenes en `0xFFFFF806...` y las pilas de usuario en `0x000000A5...`. Las estructuras siguen, de forma ilustrativa, el diseño de Windows 11 23H2 x64. Las reglas sí son las de Windows: análisis de rutas, reparse de enlaces, `FILE_DEVICE_SECURE_OPEN`, mapeo genérico, MIC, los métodos `METHOD_*`, la copia de `Information` bytes, CLEANUP y CLOSE, las descargas diferidas y demás.

Quedan fuera de alcance, para una versión 2, las pilas de drivers, la comunicación entre drivers (`IoCallDriver` a otro driver), PnP y KMDF. Esas llamadas compilan de verdad y en la simulación se marcan como "no simulado".

## Autodiagnóstico

```
DriverForge.exe --selftest C:\ruta\salida [--build]
```

Ejecuta todas las plantillas en el simulador, genera un PDF por plantilla, detecta el compilador y, con `--build`, compila y firma `Hello.sys` y `cliente.exe`. Deja `selftest.json` y `selftest.log` en la carpeta de salida.

## Compilar DriverForge

Desde Linux, con mingw-w64 13 o posterior:

```
./build.sh
```

Genera `build/DriverForge.exe`, estático y sin dependencias.

El núcleo (analizador, intérprete y simulador) es C++20 portable. `make -C tests` compila y ejecuta las pruebas nativas en Linux.

## Estructura

- `src/core`: el simulador (analizador de C, intérprete, kernel simulado).
- `src/ui`: la interfaz, con Direct2D.
- `src/build`: la compilación y la firma reales.
- `tests`: las pruebas del núcleo.
- `res`: icono, manifiesto y recursos.
