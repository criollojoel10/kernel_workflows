# kernel_workflows (fork de @TheSillyOk)

Fork de [TheSillyOk/kernel_workflows](https://github.com/TheSillyOk/kernel_workflows) adaptado para compilar
**únicamente el kernel SM6125 (LineageOS 23.2 / Android 16) del Xiaomi Mi A3 (laurel_sprout),
pineado al último source validado en dispositivo, con soporte [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS).**

- Workflow: [.github/workflows/build_kernels.yml](.github/workflows/build_kernels.yml)
- Lanzamiento manual: pestaña **Actions -> Kernel Build -> Run workflow** (`workflow_dispatch`)
- Release actual: [`2026/08/23-r1`](https://github.com/criollojoel10/kernel_workflows/releases/tag/2026/08/23-r1)
  -> `2026.08.23-MiA3-SukiSU-SM6125-23.2-6.zip`
  ([run 32615867985](https://github.com/criollojoel10/kernel_workflows/actions/runs/32615867985))

## Qué compila este fork

| Variante | Kernel source | KSU | Android |
|---|---|---|---|
| SukiSU (ReSukiSU, manual hooks) | LineageOS `lineage-23.2` @ `44758a7220f29c0b73009a8a45b0d86e335970d3` | ReSukiSU @ `88dbc78` (main, 30-jul-2026) | 16 (LOS 23.2) |

- Defconfig: `vendor/trinket-perf_defconfig` (+ fragments `vendor/xiaomi-trinket.config vendor/laurel_sprout.config`)
- AnyKernel3: rama `sm6125` de TheSillyOk/anykernel
- Artefacto/release: `YYYY.MM.DD-MiA3-SukiSU-SM6125-23.2-<run>.zip`
- Instalación probada por el propietario del fork vía `adb sideload`

Fuera del matrix respecto al upstream: **Ginkgo (Redmi Note 8)**, kernels **NoName 18.1/VIC**,
y las variantes **Normal** y **xxKSU** (comentadas en el YAML; solo se valida en dispositivo la variante SukiSU).

## Por qué el kernel está pineado a `44758a72`

La rama oficial `lineage-23.2` estuvo **congelada del 25-may al 14-ago-2026** en ese SHA.
Todos los builds upstream exitosos (#681–#695) y el release estable `2026/08/14-test` salieron de él.

El **14-ago-2026 17:35 UTC**, un único push **reescritó la historia completa** de la rama
(nuevo tip `153f55afe4...`). Consecuencias verificadas:

- Los runs upstream #696–#703 (15–22 ago) **fallan todos al compilar** (`fix_lto.patch` ya no aplica).
- Nadie había compilado con éxito el árbol post-push hasta nuestro primer intento,
  que resultó en un kernel que **bootea en loop** en dispositivo real.
- Las fechas internas de los commits engañan: p.ej. el fix LTO (`ad3d4bba`, fechado 03-jul)
  llegó realmente en ese push de agosto.

Por eso el workflow clona `lineage-23.2` y luego hace checkout forzado a
`44758a72` (`kernel_commit` en el matrix). El SHA sigue siendo alcanzable desde el repo oficial
de LineageOS aunque ya no sea tip de ninguna rama. Para actualizar en el futuro:
validar primero en dispositivo un build del nuevo tip y solo entonces cambiar `kernel_commit`.

## Soporte Droidspaces (non-GKI / legacy 4.14)

Según la guía oficial:
https://github.com/ravindu644/Droidspaces-OSS/blob/main/Documentation/Kernel-Configuration.md#configuring-non-gki-kernels-legacy-kernels

Paso del workflow: **"Add Droidspaces Support (non-GKI)"**, con política **no-destructiva**
(para no alterar el comportamiento de arranque del kernel validado):

1. **Requisitos fatales** de la doc se fuerzan a `=y` únicamente si el defconfig los tiene
   apagados o ausentes: namespaces (`PID_NS`, `UTS_NS`, `IPC_NS`, `NET_NS`...), seccomp,
   cgroups (`CGROUP_DEVICE`, `CGROUP_PIDS`...), `DEVTMPFS`, IPC (`SYSVIPC`, `POSIX_MQUEUE`)...
   - En este kernel el único flip real es **`PID_NS=y`** (requisito fatal de Droidspaces,
     venía explícitamente desactivado por el vendor). Todo lo demás ya estaba activo o es no-op.
2. **Opcionales** (OVERLAY_FS, NF_TABLES, extras NAT, USER_NS, FW_LOADER_USER_HELPER...)
   solo se añaden si el símbolo **no existe** en el defconfig; las decisiones del vendor se respetan.
3. **`CONFIG_ANDROID_PARANOID_NETWORK=n`** siempre (requisito documentado para red en contenedores).
4. Se retiró `CONFIG_CGROUP_NET_PRIO`: **no compila** en este árbol
   (`netprio_cgroup.h:35: no member named 'id' in 'struct cgroup'`) y no es requisito fatal.
5. Parches non-GKI de Droidspaces-OSS con verificación previa (`patch --dry-run`);
   lo que no aplique a esta fuente se omite con warning:
   - `01.fix_kernel_panic_in_xt_qtaguid.patch` → omitido (`xt_qtaguid` no existe en este árbol)
   - `02.fix_restore cgroup file prefix handling.patch` → aplica OK

Verificación tras flashear: app Droidspaces → Settings → Requirements → Check Requirements
(o `su -c droidspaces check`). Nota: sin NAT completo, usar modo de red host/"none"
si algún controlador opcional quedó fuera.

## Parches aplicados a SM6125 (orden)

1. Base: `fix_lto.patch`, `ptrace_fix.patch`, `kpatch_fix.patch`
   ([TheSillyOk/kernel_ls_patches](https://github.com/TheSillyOk/kernel_ls_patches)) —
   verificados con dry-run contra `44758a72`; aplican igual que en el run #681.
2. Droidspaces non-GKI (condicional, ver arriba)
3. ReSukiSU pineado: setup.sh @ `88dbc78` + `scope-min-hooks-v1.9.patch` + `selinux_stuff.patch`

## Diferencias técnicas vs upstream

1. Matrix reducido: solo MiA3 SM6125-23.2 × variante SukiSU (Normal/xxKSU comentadas en el YAML).
2. **`kernel_commit`**: pin del kernel a `44758a72` tras el force-push del 14-ago (ver arriba).
3. **`ksu_commit`**: pin de ReSukiSU a `88dbc78` (los HEAD móviles de los repos KSU son otra
   variable de riesgo; el build bueno usaba ese commit exacto).
4. Paso Droidspaces no-destructivo (ver arriba) + retirado `CGROUP_NET_PRIO` (no compila).
5. `Download Artifacts` con `pattern: "*"` + `merge-multiple: false`: con un solo artefacto,
   `download-artifact@v8` extrae sin carpeta contenedora y `upload-assets.sh` de upstream
   publicaba `META-INF.zip`/`tools.zip` en vez del zip de AnyKernel3.
6. Cron diario desactivado (`schedule` comentado): builds manuales.
7. Telegram `continue-on-error: true` (este fork no tiene `TELEGRAM_BOT_TOKEN`).

## Historial de incidentes

- **Run [#32597340269](https://github.com/criollojoel10/kernel_workflows/actions/runs/32597340269)** (falla de CI):
  `fix_lto.patch` obsoleto sobre el nuevo history post-force-push. Resuelto primero saltando el parche;
  hoy es irrelevante porque el pin vuelve al estado donde el parche aplica.
- **Release `2026/08/23-r1` v1** (bootloop): construido desde el tip reescrito nunca probado +
  configs Droidspaces agresivas + ReSukiSU móvil. Release borrado y reconstruido con el fix completo.
- **Run [#32609173650](https://github.com/criollojoel10/kernel_workflows/actions/runs/32609173650)** (falla de CI):
  pin intermedio a `dbc6d0da` (estado demasiado antiguo dentro de la historia nueva: sin targets
  `vendor/*..config` en kbuild). Sustituido por el pin correcto a `44758a72`.
- **Run [#32615674277](https://github.com/criollojoel10/kernel_workflows/actions/runs/32615674277)** (falla de CI):
  `CGROUP_NET_PRIO=y` no compila en este árbol. Retirado del set.
- **Assets rotos en release**: `META-INF.zip`/`tools.zip` por comportamiento de
  `download-artifact@v8` con artefacto único. Corregido en el workflow y reparado el release
  subiendo `2026.08.23-MiA3-SukiSU-SM6125-23.2-6.zip` manualmente.

## Si volviera a bootloopear (plan de bisección)

1. Flashear referencia pura sin el paso Droidspaces (mismo pin): si arranca, el problema está
   en las configs/parches de Droidspaces.
2. Sospechosos restantes por orden: `PID_NS=y` (único flip fatal), parche cgroup ds02, extras NAT.
3. Sacar `last_kmsg`/pstore tras el bootloop para confirmar el subsistema.

# Créditos

- Workflow original, kernels y parches: [@TheSillyOk](https://github.com/TheSillyOk) y colaboradores (README upstream)
- Droidspaces configs/parches: [@ravindu644](https://github.com/ravindu644/Droidspaces-OSS)
