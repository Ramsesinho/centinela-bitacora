# Centinela · bitácora de ingeniería

Bitácora pública de **Centinela**, un laboratorio personal de ingeniería de detección: telemetría de
ataques reales simulados, procesada en contenedores aislados, evaluada con reglas Sigma y medida con
honestidad, incluido lo que no detecta.

- **[Fase 1: arquitectura y bitácora de ingeniería](BITACORA-FASE-1.md)** · detección sin IA, una regla
  propia que detecta Mimikatz en memoria donde las reglas públicas no lo ven, y validación con 10 conjuntos
  de datos no vistos.
- **[Fase 2: un analista de IA que lee registros escritos por el atacante](BITACORA-FASE-2.md)** · los mismos
  42 ataques contra un montaje ingenuo y contra Centinela: el ingenuo obedeció órdenes escondidas en los
  registros en 8 salidas y Centinela en ninguna, sin que los hechos del informe cambiaran nunca; y por qué la
  medición automática y la revisión humana no bastaron para separarlos.
- **[Fase 3: informes que el atacante no puede reescribir](BITACORA-FASE-3.md)** · informes técnico y ejecutivo en
  HTML sin JavaScript, donde el texto del atacante y el de la IA no pueden ejecutar nada ni hacerse pasar por el
  sistema, y donde nada de la IA se publica sin revisión humana. Ejemplos de la ejecución oficial en
  [`informes-fase-3/`](informes-fase-3/).

Solo datos públicos ([OTRF Security-Datasets](https://github.com/OTRF/Security-Datasets), MIT). El código
del laboratorio se publicará cuando cierre la fase y pase su revisión.

**Autor:** Ramses Adames, estudiante de ciberseguridad (Panamá).
