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

## Nota sobre las fuentes

El repo oficial [LineageOS/android_kernel_xiaomi_sm6125](https://github.com/LineageOS/android_kernel_xiaomi_sm6125)
**no está archivado**; rama `lineage-23.2` activa (último push 2026-08). El fallo original del run
[#32597340269](https://github.com/criollojoel10/kernel_workflows/actions/runs/32597340269) no fue el repo,
sino el parche obsoleto `fix_lto.patch` (ver arriba). Si LineageOS cambia más archivos, revisar si los parches
de `kernel_ls_patches` siguen aplicando y actualizar `base_kernel_patches` en consecuencia.

# Créditos

Workflow original, kernels y parches: [@TheSillyOk](https://github.com/TheSillyOk) y colaboradores (ver README upstream).
Droidspaces configs/parches: [@ravindu644](https://github.com/ravindu644/Droidspaces-OSS).
