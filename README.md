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
