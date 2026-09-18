# Handoff del kernel Exynos 7885 - estado real al 2026-09-18

## Objetivo

Construir un kernel Linux 4.4 para Samsung Galaxy A7 2018 (`a7y18lte`, Exynos 7885), sin overclock, conservando el hardware stock y portando mejoras compatibles del fork `sacksso/android_kernel_samsung_a7y18lte_testing`.

El objetivo inmediato es que GitHub Actions genere `Image`, DTB y ZIP AnyKernel3. La validacion en el telefono viene despues.

## Error mas reciente del CI

Archivo de referencia: `contexto AI/36_Compile kernel.txt`.

El workflow del 2026-09-18 fallo aqui:

```text
DTC     arch/arm64/boot/dts/exynos/exynos7885-a7y18lte_eur_open_00.dtb
Error: ../arch/arm64/boot/dts/exynos/display-lcd_a7y18_common.dtsi:12.1-2 syntax error
FATAL ERROR: Unable to parse input tree
make[2]: *** [scripts/Makefile.lib:319: arch/arm64/boot/dts/exynos/exynos7885-a7y18lte_eur_open_00.dtb] Error 1
make[1]: *** [arch/arm64/Makefile:162: dtbs] Error 2
make: *** [Makefile:152: sub-make] Error 2
```

No fue un fallo de GCC, CPUFreq, los nuevos gobernadores, GPU, WireGuard ni touch. El log demuestra que esos subsistemas llegaron a compilar. El primer error fatal fue exclusivamente el parser DTC.

## Causa raiz del fallo DTS

El workflow activo clona:

```yaml
repository: sacksso/sack_kernel_samsung_exynos7885_testing
ref: lineage-18.1
path: kernel
```

El archivo que esta en el checkout remoto contiene un root node adicional:

```dts
/ {
        decon_board_s6e3fa7: decon_board_s6e3fa7 {
```

Ese archivo se incluye desde `exynos7885-a7y18lte_common.dtsi`, que ya esta dentro del root node principal. Por eso DTC falla en la linea 12.

La copia local correcta ya esta preparada en:

```text
arch/arm64/boot/dts/exynos/display-lcd_a7y18_common.dtsi
```

Su comienzo correcto es:

```dts
decon_board_s6e3fa7: decon_board_s6e3fa7 {
```

y no debe comenzar con `/ {`.

### Importante: el arreglo esta solo local

La rama local `lineage-18.1` esta en commit `0260617a0fb9`, pero el arreglo DTS y muchas mejoras estan como cambios sin commit. GitHub Actions no ve cambios locales: solo ve lo que se publique en `origin/lineage-18.1`.

Antes de lanzar otra build hay que publicar un commit limpio que incluya, como minimo, el DTS corregido y el workflow que usa el fork.

No incluir en ese commit:

- `out/`
- `contexto AI/`
- `contexto_actual_kernel_oc.txt`
- archivos generados temporales de WireGuard como `.wireguard-fetch-lock`, `.check`, `compat/dstmetadata`, `compat/skb_array` o `compat/version`

## Base y remotos actuales

Workspace:

```text
/workspaces/sack_kernel_samsung_exynos7885_testing
```

Rama:

```text
lineage-18.1
```

Remoto principal:

```text
origin https://github.com/sacksso/sack_kernel_samsung_exynos7885_testing
```

Upstream del kernel:

```text
upstream https://github.com/exynos7885-dev/kernel_samsung_exynos7885.git
```

Version del kernel:

```text
Linux 4.4.302
```

Toolchain del workflow:

```text
LineageOS GCC 6.4.1 AArch64
CROSS_COMPILE=$GITHUB_WORKSPACE/toolchain/aarch64/bin/aarch64-linux-gnu-
CROSS_COMPILE_ARM32=arm-linux-gnueabi-
```

## Overclock: estado final

El plan cambio: se abandono el overclock.

Se eliminaron los forzados de CPU OC de `drivers/cpufreq/exynos-acme.c`:

- `policy->max = 2288000`
- `policy->cpuinfo.max_freq = 2288000`
- `target_freq = 2288000`
- forzados de `domain->max_freq`
- `boost_max_freqs[0] = 2288000`
- notifiers `late_initcall` para reescribir limites
- bypass de `cpufreq_frequency_table_verify()`

El ACME debe usar los limites y tablas normales de CAL/DT. Las apariciones de `2288000` o `1690000` que siguen en tablas DTS historicas no fueron introducidas por la logica activa de OC y no deben reinterpretarse como una solicitud para forzar esas frecuencias.

Los workflows activos no deben contener:

```text
cpu_max_c
2288000
2184000
CPU OC
OC source
overclock
```

## Mejoras portadas o preparadas

### CPUFreq

Se portaron del fork fuente tres gobernadores opcionales:

```text
drivers/cpufreq/cpufreq_bioshock.c
drivers/cpufreq/cpufreq_blu_active.c
drivers/cpufreq/cpufreq_darkness.c
```

Se registraron en:

```text
drivers/cpufreq/Kconfig
drivers/cpufreq/Makefile
```

El `defconfig` activa los tres como built-in:

```text
CONFIG_CPU_FREQ_GOV_BLU_ACTIVE=y
CONFIG_CPU_FREQ_GOV_BIOSHOCK=y
CONFIG_CPU_FREQ_GOV_DARKNESS=y
```

Pero el default es unicamente schedutil:

```text
CONFIG_CPU_FREQ_DEFAULT_GOV_SCHEDUTIL=y
```

Los tres gobernadores fueron compilados correctamente en una prueba local del subarbol `drivers/cpufreq` con el toolchain disponible. No cambian las tablas de frecuencia ni hacen overclock.

### BFQ

Se portaron los archivos nativos:

```text
block/bfq-cgroup.c
block/bfq-ioc.c
block/bfq-iosched.c
block/bfq-sched.c
block/bfq.h
```

Y la integracion en:

```text
block/Kconfig.iosched
block/Makefile
```

El defconfig contiene:

```text
CONFIG_IOSCHED_BFQ=y
CONFIG_DEFAULT_BFQ=y
CONFIG_DEFAULT_IOSCHED="bfq"
```

### WireGuard y NCM

Se portaron:

```text
net/wireguard/
net/ncm/
```

Con referencias en:

```text
net/Kconfig
net/Makefile
```

El defconfig contiene:

```text
CONFIG_WIREGUARD=y
# CONFIG_WIREGUARD_DEBUG is not set
```

### ZRAM y red

Se activo:

```text
CONFIG_ZRAM_LZ4_COMPRESS=y
CONFIG_NET_SCH_FQ=y
CONFIG_TCP_CONG_CUBIC=y
```

BBR continua opcional y el workflow permite que este ausente.

### Touch

Se portaron cambios del fork fuente en:

```text
drivers/input/input.c
drivers/input/touchscreen/sec_ts/sec_ts.c
drivers/input/touchscreen/imagis_40xx/ist40xx.c
```

Incluyen la API exportada de `input_enable_device()`/`input_disable_device()`, gestos `KEY_BLACK_UI_GESTURE` y cambios de sec_ts/Imagis. El watchdog adicional de Imagis se aplica desde el workflow antes de compilar.

### GPU

No se porto un driver GPU de otro dispositivo.

El fork fuente usa el mismo stack Mali de la familia actual, no agrega un gobernador GPU nuevo. Se ajusto el limite seguro de DTS a:

```dts
gpu_max_clock = <1100000>;
gpu_max_clock_limit = <1100000>;
```

No usar blobs GPU de otro Exynos sin verificar correspondencia de GPU, driver kernel, HAL userspace, firmware y Android.

## Workflows

El workflow recomendado es:

```text
.github/workflows/16_Compile_kernel_testing.yml
```

`16_Compile_kernel_testing_final.yml` esta sincronizado con el anterior y contiene el mismo contenido. Ambos pasan:

```text
YAML_OK run_blocks=29
bash -n para cada bloque run
```

El archivo `16_Compile_kernel_testing.yml` en la raiz es una copia de referencia y puede estar atrasada respecto al workflow activo de `.github/workflows/`.

El workflow activo debe usar el fork propio y GCC 6.4.1. BFQ y WireGuard ya estan en el arbol, por lo que no deben copiarse otra vez desde `donor` durante CI.

## Cambios de compatibilidad GCC moderno

Estos cambios locales se hicieron para poder validar partes del arbol con GCC moderno, pero el CI usa GCC 6.4.1:

```text
kernel/cpu.c
kernel/extable.c
mm/page_alloc.c
```

Corrigen warnings modernos como `-Werror=array-compare`, `-Werror=address` y `-Werror=dangling-pointer`. No confundirlos con el error actual de DTC.

## Validaciones ya realizadas

- `defconfig` genera correctamente en un directorio O separado.
- Los tres gobernadores CPU compilan en el subarbol `drivers/cpufreq`.
- BFQ, WireGuard y NCM aparecen integrados.
- Ambos workflows pasan YAML y `bash -n`.
- `git diff --check` pasa despues de limpiar whitespace de BFQ.
- El CI alcanzo la compilacion de CPUFreq, GPU, WireGuard, touch y muchos drivers.
- El CI fallo unicamente al compilar `exynos7885-a7y18lte_eur_open_00.dtb` por el DTS antiguo del checkout remoto.

## Siguiente accion recomendada

1. Revisar `git diff` y separar archivos fuente validos de artefactos generados.
2. Crear un commit limpio con el DTS corregido y las mejoras fuente/workflow.
3. Publicar en `origin/lineage-18.1`.
4. Lanzar `.github/workflows/16_Compile_kernel_testing.yml` manualmente.
5. Si el siguiente CI falla, usar el primer `error:` real del log, no el ultimo `make Error 2`.
6. Solo despues de obtener `Image` y DTB, crear/descargar el ZIP AnyKernel3.

## Prompt breve para otra IA

Estoy trabajando en un kernel Linux 4.4.302 para Samsung Galaxy A7 2018 (`a7y18lte`, Exynos 7885). El workflow usa GCC 6.4.1 de Lineage y clona `sacksso/sack_kernel_samsung_exynos7885_testing` en la rama `lineage-18.1`. Se abandono el overclock: ACME debe permanecer stock y usar CAL/DT normalmente. Se portaron BFQ, WireGuard, NCM, tres gobernadores CPU opcionales (`bioshock`, `blu_active`, `darkness`) con `schedutil` como default, cambios de touch, zram-LZ4 y un limite GPU seguro de 1100 MHz. La ultima compilacion de CI compilo CPUFreq/GPU/WireGuard/touch y fallo exclusivamente en DTC: `display-lcd_a7y18_common.dtsi:12.1-2 syntax error`, porque el checkout remoto aun tenia `/ {` envolviendo un include DTS. La copia local correcta elimina ese root wrapper. El arreglo local debe publicarse en `origin/lineage-18.1` antes de relanzar CI. No usar blobs GPU de otro dispositivo ni reintroducir OC. Separar artefactos generados de los cambios fuente antes de hacer commit.
