# kernel_workflows (fork de @TheSillyOk)

Fork de [TheSillyOk/kernel_workflows](https://github.com/TheSillyOk/kernel_workflows) adaptado para compilar
**únicamente el kernel SM6125 (LineageOS 23.2 / Android 16) del Xiaomi Mi A3 (laurel_sprout), con soporte Droidspaces.**

- Workflow: [.github/workflows/build_kernels.yml](.github/workflows/build_kernels.yml)
- Lanzamiento: pestaña **Actions -> Kernel Build -> Run workflow** (`workflow_dispatch`)
- Ejemplo de run exitoso: https://github.com/criollojoel10/kernel_workflows/actions/runs/32607208965

## Qué compila este fork

| Variante | Kernel | Fuente | Android |
|---|---|---|---|
| Normal | SM6125 `lineage-23.2` | [LineageOS/android_kernel_xiaomi_sm6125](https://github.com/LineageOS/android_kernel_xiaomi_sm6125/tree/lineage-23.2) | 16 (LOS 23.2) |
| SukiSU (ReSukiSU, manual hooks) | idem | idem | 16 |
| xxKSU (backslashxx/KernelSU) | idem | idem | 16 |

Defconfig: `vendor/trinket-perf_defconfig` + fragments `vendor/xiaomi-trinket.config vendor/laurel_sprout.config`.
AnyKernel3: rama `sm6125` de TheSillyOk/anykernel. Artefactos con nombre `YYYY.MM.DD-MiA3[-SukiSU|-xxKSU]-SM6125-23.2-<run>`.

Se eliminó del matrix todo lo demás respecto al upstream: **Ginkgo (Redmi Note 8)** y los kernels **NoName 18.1 / VIC**.

## Soporte Droidspaces (non-GKI / legacy 4.14)

Añadido según la guía oficial:
https://github.com/ravindu644/Droidspaces-OSS/blob/main/Documentation/Kernel-Configuration.md#configuring-non-gki-kernels-legacy-kernels

Paso nuevo en el workflow: **"Add Droidspaces Support (non-GKI)"**, que por cada build:

1. Inyecta las configs obligatorias en el defconfig (comentando la definición previa si existía):

   - IPC: `SYSCTL`, `SYSVIPC`, `POSIX_MQUEUE`
   - Namespaces: `NAMESPACES`, `PID_NS`, `UTS_NS`, `IPC_NS`, `NET_NS`, `USER_NS`
   - Seccomp: `SECCOMP`, `SECCOMP_FILTER`
   - Cgroups: `CGROUPS`, `CGROUP_DEVICE`, `CGROUP_PIDS`, `MEMCG`, `CGROUP_SCHED`, `FAIR_GROUP_SCHED`, `CGROUP_FREEZER`, `CGROUP_NET_PRIO`
   - FS: `DEVTMPFS`, `OVERLAY_FS`, `TMPFS_POSIX_ACL`, `TMPFS_XATTR`
   - Firmware: `FW_LOADER`, `FW_LOADER_USER_HELPER`, `FW_LOADER_COMPRESS`
   - Red/NAT: `VETH`, `BRIDGE`, `NETFILTER*`, `NF_CONNTRACK*`, `IP_NF_*`, `NFTABLES`, `MASQUERADE`, etc.
   - Compatibilidad: `ANDROID_PARANOID_NETWORK=n`

2. Aplica los parches non-GKI de `Droidspaces-OSS/Documentation/resources/kernel-patches/non-GKI/`
   con verificación previa (`patch --dry-run`): si un parche no aplica a esta fuente se omite con un warning.
   - `01.fix_kernel_panic_in_xt_qtaguid.patch` → **se omite** porque LineageOS eliminó `xt_qtaguid` de lineage-23.2.
   - `02.fix_restore cgroup file prefix handling.patch` → aplica OK.

Verificación tras flashear: app Droidspaces → Settings → Requirements → Check Requirements (o `su -c droidspaces check`).

## Diferencias técnicas vs upstream

1. **Matrix reducido a MiA3**: fuera Ginkgo, NoName-18.1 y NoName-VIC.
2. **`base_kernel_patches`**: nueva clave opcional por entrada del matrix que sobreescribe los parches base
   `fix_lto.patch ptrace_fix.patch kpatch_fix.patch`.
   - Motivo: LineageOS integró upstream el fix de LTO symversions (`scripts/Makefile.build`) que trae `fix_lto.patch`,
     por lo que ya no aplica y rompía el build ("Hunk #1 FAILED at 503"). Para SM6125 se usan solo
     `ptrace_fix.patch kpatch_fix.patch` (verificados contra lineage-23.2).
3. **Cron diario desactivado** (`schedule` comentado): los builds son manuales.
4. **Notificación Telegram** (`continue-on-error: true`): este fork no tiene el secret `TELEGRAM_BOT_TOKEN`.

## Parches aplicados a SM6125 (orden)

1. Base: `ptrace_fix.patch`, `kpatch_fix.patch` ([TheSillyOk/kernel_ls_patches](https://github.com/TheSillyOk/kernel_ls_patches))
2. Droidspaces non-GKI (condicional, ver arriba)
3. Según variante KSU:
   - SukiSU: setup.sh de ReSukiSU + `scope-min-hooks-v1.9.patch` + `selinux_stuff.patch`
   - xxKSU: setup.sh de backslashxx/KernelSU

Todos verificados con dry-run contra `lineage-23.2` (2026-08).

## Diagnóstico del bootloop (2026-08-23) — CAUSA RAÍZ

Síntoma: el release `2026/08/23-r1` (run 32607208965) instalaba pero bootloopeaba,
mientras que `2026.07.31-MiA3-SukiSU-SM6125-23.2-681` (run upstream #681) funciona.

Investigación (todo vía API de GitHub):

1. **La rama `lineage-23.2` estuvo congelada del 25-may al 14-ago** en
   `44758a7220f29c0b73009a8a45b0d86e335970d3`. Los runs upstream #681–#695
   (31-jul → 14-ago, todos exitosos) compilaron TODOS ese mismo SHA.
2. El **14-ago 17:35 UTC hubo un único push que reescribió la historia completa**
   de la rama (nuevo tip `153f55afe4...`). Las fechas de commit internas son
   engañosas: p.ej. el fix LTO (`ad3d4bba`, fechado 03-jul) llegó realmente en ese
   push de agosto.
3. **Los runs upstream #696–#703 (15–22 ago) fallan TODOS** al compilar:
   `fix_lto.patch` ya no aplica sobre la historia nueva. Es decir, *nadie*
   había compilado jamás el árbol post-push; nuestro release booteloop fue
   literalmente el primer build exitoso de ese código sin probar.
4. Además, el bloque Droidspaces original forzaba configs destructivas
   (`USER_NS=y`, `FW_LOADER_USER_HELPER=y`, `BRIDGE_NETFILTER=y`,
   `ANDROID_PARANOID_NETWORK=n`, `PID_NS=y`...) sobre decisiones explícitas del
   vendor, y ReSukiSU se tomaba de HEAD móvil (agosto) en vez del commit de
   julio que usa el build bueno.

**Causa más probable**: source post-force-push nunca validado + flips agresivos
de config. No es atribuible a un único cambio sin pruebas en dispositivo
(last_kmsg/pstore lo diría), pero la corrección elimina ambas variables.

## Solución aplicada

1. **Kernel pineado a `44758a72`** (`kernel_commit` en el matrix): el SHA exacto
   que produjo tu zip bueno y el último release estable upstream
   (`2026/08/14-test`). Sigue alcanzable desde el repo oficial de LineageOS,
   aunque ya no sea el tip de ninguna rama.
2. **Parches base completos restaurados** (`fix_lto` + `ptrace_fix` +
   `kpatch_fix`): sobre `44758a72` aplican limpio, idéntico al run #681.
3. **ReSukiSU pineado a `88dbc78`** (el HEAD de main del 30-jul, el mismo que
   usó el run #681).
4. **Paso Droidspaces no-destructivo** ("bootloop-safe"):
   - Requisitos fatales según la doc (namespaces, seccomp, cgroups, devtmpfs...)
     se fuerzan a `=y` solo si el defconfig los tiene apagados o ausentes.
     En este kernel el único flip real es `PID_NS=y` (requisito fatal de
     Droidspaces; venía desactivado por el vendor).
   - Opcionales (OVERLAY_FS, NF_TABLES, NAT extras, USER_NS, FW_LOADER_USER_HELPER...)
     solo se añaden si el símbolo no existe en el defconfig; las decisiones del
     vendor (p.ej. `BRIDGE_NETFILTER` desactivado) **se respetan**.
   - `CONFIG_ANDROID_PARANOID_NETWORK=n`: requisito documentado para que los
     contenedores tengan red.
   - Se retiró `CONFIG_CGROUP_NET_PRIO`: no compila en este árbol
     (`netprio_cgroup.h: no member named 'id'`) y no es requisito.
5. Corregido también el upload de assets: con un solo build variant,
   `download-artifact@v8` extrae sin carpeta contenedora y `upload-assets.sh`
   publicaba `META-INF.zip`/`tools.zip`; ahora se fuerza `pattern: "*"` +
   `merge-multiple: false`.

Run corregido: https://github.com/criollojoel10/kernel_workflows/actions/runs/32615867985
Release: https://github.com/criollojoel10/kernel_workflows/releases/tag/2026/08/23-r1
Asset: `2026.08.23-MiA3-SukiSU-SM6125-23.2-6.zip`

Si aún hubiera bootloop tras esto, siguiente paso de bisección en dispositivo:
flashear primero sin el paso Droidspaces (referencia pura) y luego con él;
y revisar `PID_NS=y` como único flip restante.

# Créditos

Workflow original, kernels y parches: [@TheSillyOk](https://github.com/TheSillyOk) y colaboradores (ver README upstream).
Droidspaces configs/parches: [@ravindu644](https://github.com/ravindu644/Droidspaces-OSS).
