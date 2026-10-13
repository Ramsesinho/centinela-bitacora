# Proyecto Centinela · Fase 1: arquitectura y bitácora de ingeniería

**Autor:** Ramses Adames · **Fecha:** septiembre de 2026 · **Estado:** Fase 1 cerrada (detección sin IA)

Centinela es un **laboratorio personal** de ingeniería de detección: un pipeline determinista que recibe
telemetría de ataques reales simulados, la procesa en contenedores aislados, la evalúa con reglas Sigma y
mide con honestidad qué detecta y qué no. Todo con **datos públicos** (OTRF Security-Datasets, licencia
MIT). No es un SOC en producción.

La Fase 1 cubre la detección **sin IA**. La Fase 2 añadirá un analista de IA para el triaje, diseñado
para no dejarse engañar por los registros que lee.

---

## Capa 0 · La máquina de trabajo y su red

**Objetivo:** que la máquina virtual de trabajo (Ubuntu 24.04, disco cifrado) y todo lo que corre dentro
salgan a internet, pero no puedan llegar a la red de casa ni a la PC anfitriona.

- Cortafuegos `nftables` dentro de la VM (tabla `inet vm_guard`, cadenas de salida y de reenvío, así que
  cubre también a los contenedores): descarta los rangos privados y locales (`10.0.0.0/8`,
  `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `127.0.0.0/8`) y todo IPv6; solo permite el DNS y
  el DHCP de VirtualBox.
- El archivo se instaló como una línea en base64 y se comprobó con su huella SHA-256 antes de activarlo;
  sintaxis validada con `nft -c` y servicio persistente.
- **Por qué hizo falta:** una regla del Firewall de Windows bloqueaba el router, pero la prueba demostró que
  **no protegía a la propia PC** (desde la VM, el puerto 445 del anfitrión respondía `succeeded`). El
  cortafuegos dentro de la VM cerró ese hueco.

| Prueba desde la VM | Resultado |
|---|---|
| Internet | `INTERNET OK` |
| PC anfitriona, puerto 445 | `timed out` (antes del arreglo: `succeeded`) |
| Router, puerto 80 | `timed out` |
| Un contenedor hacia la PC anfitriona | `BLOQUEADO` |

## Capa 1 · Salida controlada para los agentes de IA

**Objetivo:** que el agente de IA del equipo solo pueda hablar con la API de su proveedor, y nada más.

- El agente corre en una red de Docker **sin salida propia**; su única salida es un proxy Squid con **lista
  permitida** (solo `api.deepseek.com`, solo HTTPS).
- **Lo que no está en la lista se niega antes de consultar el DNS.** Así un nombre inventado para sacar
  datos (por ejemplo `datos.dominio-atacante.com`) no genera ninguna consulta DNS. Hallazgo de la revisión
  adversarial, corregido y probado.

| Prueba desde el contenedor del agente | Resultado |
|---|---|
| Destino permitido | Llega (la API responde `401` porque la prueba no manda clave) |
| Dominio no permitido | `CONNECT tunnel failed, response 403` |
| IP directa | `403` |
| Sin usar el proxy | `Could not resolve host` |
| PC anfitriona | `403` |
| Fuga por DNS (`tcpdump` en la interfaz de la VM) | Solo se ve la consulta de control; **ninguna** del dominio negado |

## Capa 2 · Aislamiento de los contenedores que procesan la telemetría

**Objetivo:** los registros de un ataque son material hostil. Quien los procesa no debe poder dañar la
máquina ni agotar sus recursos.

| Recurso | Límite | Qué pasa al intentar romperlo |
|---|---|---|
| Red | `--network none` | Sin interfaces salvo loopback: `ENETUNREACH` |
| DNS | Sin servidores | Falla la resolución (`gaierror`) |
| Usuario | `1000:1000`, nunca root | Sin privilegios |
| Capacidades de Linux | `--cap-drop ALL` | Capacidades efectivas en `0x0` |
| Sistema de archivos | Raíz de solo lectura | `EROFS` al escribir; solo `/tmp` y la carpeta de salida |
| Procesos | `pids.max = 256` | Limita la creación de procesos (bombas de procesos) |
| Memoria | `memory.max = 2 GiB` | Tope de 2 147 483 648 bytes |
| CPU | 2 núcleos | Cuota `200000 100000` |
| Tiempo | Tope por paso | Un paso de 3 s que dormía 60 s se cortó con código 137 a los 3–4 s |

**Cómo se prueba que la prueba funciona (control negativo):** la misma batería se ejecuta **sin** las
protecciones, y cada comprobación tiene que detectar que la suya falta. Una prueba que no puede fallar no
prueba nada.

- Comprobaciones OK con las protecciones: **59**
- Detecciones del control negativo sin ellas: **18** (9 por imagen)
- Fallos: **0**

## Capa 3 · Una ejecución incompleta es un error, nunca «sin amenazas»

**Objetivo:** que un fallo del sistema nunca se confunda con «no hubo ataques».

- **Incidente:** en la primera detección, el paso de Zircolite no encontró su base de datos
  (`sqlite3.OperationalError: unable to open database file`): Zircolite 3.8.1 la guarda con otro nombre y
  su filtro de eventos seguía encendido.
- **Respuesta del sistema:** `EJECUCION INCOMPLETA (código 1)`, y nada de esa carpeta cuenta como
  resultado.
- **Corrección:** localizar la base con la regla de nombres de Zircolite y exigir que haya exactamente una,
  apagar el filtro de eventos y comprobar que el recuento de eventos cargados coincide. Una prueba reproduce
  el error original.

## Capa 4 · Ingesta, detección y el caso Mimikatz

Cada archivo de datos se descarga por commit fijado y se verifica con su huella SHA-256; la normalización
guarda la huella de los eventos y la del original.

| Escenario (OTRF) | Eventos Sysmon | Alertas SigmaHQ | Alertas regla propia |
|---|---|---|---|
| `empire_launcher_vbs` (ejecución) | 1190 de 2067 líneas | 4 | 0 |
| `empire_mimikatz_logonpasswords` (robo de credenciales) | 679 de 6026 líneas | **0** | **1** |
| `covenant_psremoting_command` (movimiento lateral) | 3351 de 4284 líneas | 99 | 0 |

Rechazados: 0 en los tres.

### Por qué SigmaHQ no vio a Mimikatz

El ataque está en los datos: `powershell.exe` abre `lsass.exe` con permiso de lectura de memoria
(`GrantedAccess 0x1010`), y en su pila de llamadas aparece código **sin archivo en disco** (`UNKNOWN`),
la huella de Mimikatz inyectado en memoria. **Ninguna de las 15 reglas Sysmon de acceso a LSASS de
SigmaHQ cubre ese patrón:** una no incluye `0x1010` y además excluye todo lo que está en `System32`; otra
exige otros permisos.

Aparte se encontró otro punto ciego: si una regla usa un campo que el evento no trae dentro de un `NOT`,
la evaluación en SQLite da NULL y descarta la fila. No fue la causa de este caso, pero en la validación le
costó a SigmaHQ dos detecciones reales.

### La regla propia

Detecta la **lectura de la memoria de LSASS cuando quien llama directamente a la API de Windows es código
sin módulo en disco**. Se afinó en dos pasos:

- **Versión 1:** 9 falsas alarmas en el escenario de movimiento lateral. Eran el `Get-Process` del atacante
  ejecutado por PowerShell remoto, que llega a LSASS a través de .NET sin robar credenciales.
- **Versión 2:** exige que el llamador directo sea código sin módulo. Resultado 0, 1 y 0 en los tres
  escenarios.
- **Versión 2.1:** solo cuenta como API de Windows un `ntdll.dll` que esté en `System32` o `SysWOW64` (un
  `ntdll.dll` falso en otra carpeta ya no engaña a la regla). Congelada con su huella antes de la validación.

### Validación externa (10 conjuntos de OTRF que no se usaron para escribir la regla)

Las etiquetas de lo que es ataque se subieron a git **antes** de mirar las alertas.

| | Solo SigmaHQ | Solo la regla propia | Las dos juntas |
|---|---|---|---|
| Escenarios con alguna alerta sobre el atacante (de 10) | 7 | 2 | **9** |
| Volcados en memoria de LSASS detectados (de 2) | 0 | 2 | 2 |
| Falsas alarmas | 0 | 0 | 0 |

- Las **27 alertas** caen sobre eventos del atacante.
- La condición de «lectura de memoria» evita **1478 falsas alarmas** en `empire_mimikatz_sam_access`, donde
  Mimikatz abre LSASS muchas veces sin leer su memoria (duplica el token de SYSTEM).
- La regla propia y SigmaHQ se complementan: los volcados por otras vías (Dumpert, `comsvcs`, el
  Administrador de tareas) los cubre SigmaHQ.

## Capa 5 · Revisión humana, manifiestos y evaluación final

**Objetivo:** que la IA no se evalúe a sí misma.

- Cada escenario tiene un **manifiesto**: qué eventos son el ataque, en qué ventana de tiempo y qué reglas
  se espera que salten. Lo preparó el agente constructor, lo **verificó Codex** de forma independiente y lo
  **revisé y firmé yo** tras comprobar cada evento. La evaluación **se niega a calcular** si un manifiesto no
  está firmado o si su huella no coincide con los eventos.
- Las 104 alertas se agrupan en **4 incidentes** (1 de ejecución, 1 de Mimikatz y 2 de movimiento lateral).

| Escenario | Solo SigmaHQ | SigmaHQ + regla propia |
|---|---|---|
| `empire_launcher_vbs` | Detectado | Detectado |
| `empire_mimikatz_logonpasswords` | **No detectado** | Detectado |
| `covenant_psremoting_command` | Detectado | Detectado |
| **Cobertura de escenarios** | **2 de 3** | **3 de 3** |

Toda la Fase 1 se reproduce desde cero con un solo comando (`./centinela.sh fase1`).

**Ejecuciones oficiales (13 de octubre de 2026),** con el código ya revisado (commit `7f9856a`):
`20261013T165150Z-07yxYu` (Fase 1) y `validacion-20261013T165557Z-9TTeSD` (validación externa). Antes de
ejecutarlas, Codex hizo una revisión final que pidió endurecer cómo se leen y verifican los datos: no seguir
enlaces, comprobar las huellas y fijar la plataforma de las imágenes. Con esos cambios, **todas las cifras
de esta bitácora se repiten exactamente**, y las alertas y los incidentes son idénticos byte a byte a los de
septiembre.

## Límites (lo que este laboratorio no demuestra)

- **Muestra pequeña:** 3 escenarios de desarrollo y 10 de validación. La única prueba de la regla propia
  totalmente independiente es un escenario (`empire_over_pth_patch_lsass`, 2 de 2 alertas).
- **La regla se formuló después de estudiar el caso** que la motivó; por eso su acierto en ese escenario no
  cuenta como prueba independiente.
- **Evasión conocida:** si la llamada pasa por un módulo con archivo en disco, la regla no salta.
- **Solo Sysmon** y reglas de nivel `medium` o superior: un escenario sin alerta no demuestra que ninguna
  regla lo detecte. El mismo acceso de Mimikatz aparece en el canal Security, que queda para una fase
  posterior.
- `empire_mimikatz_sam_access` no lo detecta nadie, ni SigmaHQ ni la regla propia: queda anotado, sin
  ajustar nada para taparlo.
- Los «manifiestos firmados» son revisión humana registrada y huellas SHA-256 de los eventos; **no son
  firmas criptográficas**.

## Conclusiones

1. Las reglas públicas son un buen punto de partida, pero **tienen huecos medibles**; aquí uno de ellos era
   justo un robo de credenciales en memoria.
2. Una regla propia basada en la **técnica** (quién llama realmente a la API) cerró ese hueco sin falsas
   alarmas en la validación, y complementa a SigmaHQ en vez de sustituirla.
3. **Probar las pruebas** (control negativo) y **fallar cerrado** importan tanto como detectar.
4. La revisión humana y una segunda IA revisora evitan que un agente se evalúe a sí mismo.

**Siguiente:** Fase 2, un analista de IA para el triaje, con una batería de registros envenenados que mide
si el informe final se deja engañar.

---

*Laboratorio personal con datos públicos de [OTRF Security-Datasets](https://github.com/OTRF/Security-Datasets)
(MIT). Detección con [Zircolite](https://github.com/wagga40/Zircolite) y reglas de
[SigmaHQ](https://github.com/SigmaHQ/sigma) (Detection Rule License 1.1).*
