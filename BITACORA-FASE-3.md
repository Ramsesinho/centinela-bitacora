# Proyecto Centinela · Fase 3: informes que el atacante no puede reescribir

**Autor:** Ramses Adames · **Fecha:** noviembre de 2026 · **Estado:** Fase 3 cerrada

En la Fase 2, el analista de IA aprendió a no obedecer al atacante. Ahora toca lo que lee una persona: el **informe**.
Un informe de seguridad es un sitio peligroso, porque copia textos que escribió el atacante (líneas de comandos, nombres
de archivo, usuarios) y textos que escribió una IA. La pregunta de la fase:

> ¿Se puede entregar un informe en HTML, para un analista y para gerencia, en el que ni el atacante ni la IA puedan
> ejecutar nada, hacerse pasar por el sistema ni publicar algo que una persona no aprobó?

Todo con **datos públicos** (OTRF Security-Datasets, MIT) sobre los mismos 4 incidentes de las fases anteriores. Es un
laboratorio, no un SOC en producción.

**Los informes de la ejecución oficial, tal cual salieron:**
- [informe técnico](https://ramsesinho.github.io/centinela-bitacora/informes-fase-3/tecnico.html), para el analista;
- [informe ejecutivo](https://ramsesinho.github.io/centinela-bitacora/informes-fase-3/ejecutivo.html), una hoja para
  gerencia.

---

## Tres informes, tres lectores

- **Técnico:** cada incidente con sus alertas, la cronología, el árbol de procesos y las hipótesis de la IA que aprobó
  el revisor.
- **Ejecutivo:** una hoja A4 en lenguaje sencillo. **No lleva ni una palabra escrita por la IA**: las frases salen de
  una lista fija, escrita y aprobada a mano, una por tipo de alerta; si una regla no tiene frase, se muestra su título.
- **Comparación con y sin IA:** los mismos hechos con y sin las hipótesis, para ver qué aportó el modelo.

## Capa 1 · El texto del atacante es dato, nunca código ni rótulo

- **Sin JavaScript y sin nada externo:** cada página declara `Content-Security-Policy: default-src 'none'` y solo carga
  su hoja de estilos, fijada por huella. El programa se niega a generar el informe si esa hoja cambia un solo byte.
- **Todo se escapa.** Los caracteres invisibles o que invierten el texto (por ejemplo U+202E, que hace que `gnp.exe` se
  lea al revés) se muestran como su código (una barra invertida, «u» y «202e»), nunca se aplican.
- **El texto del atacante va siempre en su propio bloque**, con un borde distinto y la etiqueta «texto de los
  registros», y se muestra dentro de ese bloque. No puede imitar un rótulo del informe.
- **Se probó con un corpus hostil:** las trampas de la batería de la Fase 2, etiquetas HTML, comentarios, enlaces,
  `<svg onload>`, una cadena de 1 MB y pilas de acentos combinados. La estructura del HTML no cambia con nada de eso: si
  cambiara, las pruebas fallan. Aparte, y a mano, se midió en el navegador un valor con 2000 caracteres U+202E seguidos:
  cabe en la pantalla de un móvil y al imprimir se reparte entre hojas sin perder texto. Ese caso no está en las pruebas
  automáticas.

## Capa 2 · Nada de la IA se publica sin una persona

- La IA (DeepSeek) analiza cada incidente y **propone hipótesis** que tienen que citar alertas reales. Igual que en la
  Fase 2, no puede tocar los hechos, la gravedad ni el estado.
- Cada hipótesis recibe un identificador ligado a su texto exacto. **Una persona decide** en un archivo de revisión
  `aceptar` o `rechazar`, con motivo obligatorio (en esta ejecución, adoptando la propuesta de un asistente; ver «Lo que
  no salió bien»). El informe publicable **solo muestra las aceptadas**, cuenta las
  rechazadas y se niega a generarse si queda una sin decidir, si falta un motivo o si el archivo trae trucos (claves
  repetidas, alias de YAML…).
- La cabecera de cada informe lleva la huella del archivo de revisión que lo autorizó.

## Capa 3 · Una ejecución oficial que se puede comprobar

- Sale de una **copia limpia** del código en un commit exacto, con los datos de la Fase 1 copiados, un manifiesto de
  huellas y el permiso de escritura quitado (eso no la hace inmutable frente a su dueño; ver «Límites»).
- Un comprobador final revisa que nada cambió: el manifiesto, que sigue sin permiso de escritura, el árbol de git
  limpio, que antes de la llamada a la IA hubo una prueba de red oficial con el mismo código y las mismas imágenes, el
  coste y que cada página publicada es la que se generó.
- **Tope de gasto en el código: 0,05 dólares.** Si la API falla, el incidente queda «triaje no disponible» y la
  ejecución lo dice. No se inventa nada.

---

## Resultados de la ejecución oficial

| Incidente | Alertas | Hipótesis propuestas | Aceptadas | Rechazadas |
|---|---:|---:|---:|---:|
| Covenant, puesto 5 (PowerShell remoto) | 7 | 5 | 5 | 0 |
| Covenant, puesto 6 (llega la sesión remota) | 92 | 5 | 4 | 1 |
| Mimikatz (robo de contraseñas en memoria) | 1 | 3 | 2 | 1 |
| Lanzador de Empire (VBS) | 4 | 5 | 2 | 3 |
| **Total** | **104** | **18** | **13** | **5** |

- **Coste:** 4 llamadas, **0,0011 dólares** (0,0011274720 exactos). Repetir el triaje sin red, con las respuestas
  grabadas, da exactamente los mismos bytes.
- **Ninguna hipótesis citó una alerta inventada:** el validador no descartó ninguna (0 de 18), igual que en la Fase 2.
- **Las 5 rechazadas afirmaban más de lo que muestran las alertas:** un robo de credenciales donde solo hay una
  petición de acceso (en la Fase 1 se vio que era un `Get-Process`), una lectura de memoria que el registro no muestra, un
  registro desactivado, una descarga que la regla no exige y una detección atribuida a la regla equivocada.
- **Lo que aportó la IA fue unir señales, no detectar nada nuevo.** Cada regla ve una cosa suelta; el modelo señaló que
  cuatro de ellas las hizo el mismo programa (`GruntHTTP.exe`) y las ordenó en una historia: un programa que aloja
  PowerShell y abre una sesión remota. En el incidente de Mimikatz, en cambio, apenas añadió nada a lo que ya dice la
  regla.

## Lo que no salió bien, publicado igual

- **La revisión humana fue la aprobación de una propuesta.** Las 18 decisiones las propuso un asistente (Claude), que
  leyó cada hipótesis contra las alertas que cita y contra lo que comprueba cada regla. Yo las leí y las adopté sin
  cambios. Es mi decisión, pero no un juicio desde cero.
- **El informe de comparación oficial se corta al imprimir.** Lo encontró la prueba de usuario (Antigravity): una
  huella de 64 caracteres sin partir ensanchaba las tablas y en A4 se perdía la columna «Con IA». Está arreglado en el
  código, pero rehacerlo obligaba a repetir toda la ejecución oficial con otra llamada a la IA y otra revisión. Por eso
  esa página no se publica aquí.
- **El procedimiento tenía un paso en mal orden.** La prueba de red iba antes de un paso que reconstruye una imagen de
  Docker. El control funcionó: la llamada a la IA se negó a salir **antes de usar la clave**, sin gasto. El orden ya está
  corregido.
- **La medición del arquitecto no vio el corte de la comparación.** Medía si la página se ensanchaba, no cada tabla
  contra su caja. Ahora se mide así.

## Límites

- Datos de laboratorio de 2020 y 4 incidentes: el informe enseña la forma, no la escala.
- Un HTML seguro no protege de un PDF o de un correo que alguien arme después copiando el texto.
- Un registro puede usar letras de otro alfabeto que se parecen a las nuestras (homoglifos): se muestran tal cual.
- El nombre del modelo es el que devuelve la API; no prueba que los pesos sean los mismos que en la Fase 2.
- Quitar el permiso de escritura no impide que el dueño de los archivos cambie algo y lo deshaga durante la ejecución:
  el comprobador final ve los cambios que persisten, no ese.

## Cómo se revisó

Cada paso lo revisó otra IA (Codex) buscando fallos, en la máquina virtual y desde Windows. La revisión final de toda la
fase no dejó bloqueos: sus dos hallazgos se comprobaron y resultaron no ser fallos. Los informes los recorrió además
otra IA como probadora (Antigravity), en móvil, escritorio, modo oscuro e impresión, midiendo el contraste de cada
texto.

**Siguiente:** un portal para gerencia con inicio de sesión y roles, que mostrará estos mismos informes.
