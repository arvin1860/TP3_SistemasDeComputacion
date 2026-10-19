# TP3_SistemasDeComputacion


## Introducción teórica

**UEFI** (Unified Extensible Firmware Interface) es la especificación que reemplaza al BIOS tradicional como interfaz entre el firmware y el sistema operativo. A diferencia del BIOS, que trabajaba en modo real de 16 bits, con 1 MB de memoria direccionable y acceso directo a puertos de hardware, UEFI ofrece un entorno pre-OS moderno y modular, compatible con procesadores de 32 y 64 bits, con su propio gestor de memoria, consola y servicios.

**PI (Platform Initialization)** es la arquitectura interna del firmware. Define las fases por las que pasa la plataforma desde el reset hasta cargar el sistema operativo: **SEC** (contexto mínimo y raíz de confianza), **PEI** (inicialización de memoria y chipset), **DXE** (carga de drivers y servicios), **BDS** (selección del dispositivo de arranque) y **RT** (runtime, tras `ExitBootServices()`).

UEFI abstrae el hardware mediante **Protocolos**, bloques de funciones identificados por un GUID y agrupados en **Handles**. El código accede a ellos a través de la **System Table**, que expone los *Boot Services* (solo durante el arranque) y los *Runtime Services* (también disponibles con el SO en ejecución). Las decisiones de arranque se guardan en variables **NVRAM** (`Boot####`, `BootOrder`).

Todo el código ejecutable (drivers, aplicaciones, bootloaders) usa el formato **PE/COFF**, con información de reubicación para cargarse en cualquier dirección. **Secure Boot** verifica las firmas de esos binarios antes de ejecutarlos y establece una cadena de confianza.

Desde la perspectiva de seguridad, el firmware corre con el máximo privilegio y antes que cualquier defensa del SO, por lo que es un objetivo valioso para bootkits y malware persistente.

En este trabajo se explora el entorno UEFI desde la Shell (TP1), se desarrolla y analiza una aplicación nativa PE/COFF (TP2) y se ejecuta en hardware físico sorteando Secure Boot (TP3).

## Descripción de los trabajos prácticos

### TP1: Exploración del entorno UEFI y la Shell

**Objetivo:** ver cómo UEFI abstrae el hardware y gestiona la configuración antes de que cargue el sistema operativo.

Se arrancó **QEMU con OVMF** (firmware UEFI de código abierto) y se exploró la **UEFI Shell**:

- **`map` y `dh -b`:** listan los *handles* y *protocolos*. En lugar de puertos fijos, el hardware se accede mediante interfaces identificadas por GUID.
- **`dmpstore` y `set -v`:** muestran las variables **NVRAM** y las variables de entorno de la Shell. Se creó la variable `TestSeguridad` y se observaron `Boot####` y `BootOrder`, que usa el Boot Manager (fase BDS) para decidir qué arrancar.
- **`memmap -b`, `pci -b`, `drivers -b`:** permiten hacer *footprinting* de la plataforma: mapa de memoria (con las regiones `RuntimeServicesCode`, que persisten tras `ExitBootServices()`), dispositivos PCI del chipset emulado y drivers cargados.

### TP2: Desarrollo, compilación y análisis de seguridad

**Objetivo:** crear una aplicación nativa UEFI en C, entender el formato PE/COFF y analizarla con un descompilador.

1. **Desarrollo:** `aplicacion.c` usa `gnu-efi`. Imprime texto con `SystemTable->ConOut->OutputString` (no con `printf`, porque no hay libc ni sistema operativo) y contiene un byte `0xCC` (opcode de `INT3`) declarado `volatile`, que luego se compara.
2. **Compilación en tres pasos:** `gcc` genera el objeto, `ld` genera un ELF compartido (`.so`) y `objcopy --target=efi-app-x86_64` lo reempaqueta como **PE/COFF** (`.efi`). Con `file`, `objdump -x` y `readelf -h` se verificó el formato (Subsystem = EFI application, `ImageBase` 0, tabla de relocaciones).
3. **Ejecución en QEMU:** la app se corrió desde la Shell (`FS0:\> aplicacion.efi`) y mostró los dos mensajes esperados.
4. **Análisis en Ghidra:** se importó el `.efi` y se localizó `efi_main`. En el listado aparecen `MOVZX EAX, byte ptr [RBP + local_9]` y `CMP AL, 0xCC`. El decompilador solo mostró las llamadas `efi_call1/efi_call2` y no el `if`, porque supone que el valor es constante y no respeta `volatile`. Además, `0xCC` (204 sin signo) se interpreta como `-52` cuando se trata como `char` con signo.

### TP3: Ejecución en hardware físico (bare metal)

**Objetivo:** llevar el binario compilado a una computadora real (Lenovo T450), sorteando las restricciones de Secure Boot.

1. **Pendrive:** se formateó en FAT32 con la estructura `EFI/BOOT/BOOTX64.EFI` (la UEFI Shell de TianoCore como cargador por defecto) y se copió `aplicacion.efi` en la raíz.
2. **Configuración del firmware:** en el setup (F1) se deshabilitó **Secure Boot**, porque ni la Shell ni la aplicación están firmadas por una entidad de confianza (Microsoft/Lenovo), y se seleccionó **UEFI Only**.
3. **Ejecución:** con F12 se arrancó desde el USB, se entró al filesystem del pendrive y se ejecutó la aplicación, que imprimió sus mensajes directamente en pantalla, fuera de cualquier sistema operativo.


Se lanza QEMU con `-s -S` (servidor GDB en el puerto 1234, CPU pausada), se conecta GDB con `target remote localhost:1234`, se obtiene la dirección base de la imagen cargada y se cargan los símbolos con `add-symbol-file`. El objetivo es frenar la ejecución en `CMP AL, 0xCC` y mostrar el registro `AL = 0xCC`.
