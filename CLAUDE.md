# CLAUDE.md — Pipeline de sociedades del Diario Oficial de Chile

## Contexto

Proyecto personal (no comercial, sin relación con el empleador) para construir un
dataset estructurado y consultable de constituciones, modificaciones y disoluciones
de sociedades en Chile, a partir de las publicaciones del Diario Oficial.

El objetivo es el desafío técnico: extracción de información desde prosa notarial,
resolución de entidades y análisis de la demografía empresarial chilena. No es
un buscador de sociedades más.

**Estado actual: exploración de la fuente. No hay código escrito todavía.**

## Fuente y marco legal

Los extractos de constitución, modificación y disolución de sociedades se publican
obligatoriamente en el Diario Oficial (art. 4° Ley N° 20.494), de lunes a sábado.
Edición electrónica desde el 17 de agosto de 2016; antes existe edición impresa
digitalizada (desde 1877) y la sección Empresas y Cooperativas electrónica desde
el 29 de marzo de 2011.

### Condiciones de uso (verificadas)

https://www.diariooficial.interior.gob.cl/condiciones/

La reutilización está **expresamente permitida**, incluso comercial, y el Diario
Oficial declara que no otorga derechos exclusivos a nadie. Las cuatro condiciones
obligatorias son:

1. No alterar el contenido de la publicación.
2. No desnaturalizar el sentido de la información.
3. Informar de forma clara y visible que la información fue obtenida del Diario Oficial.
4. Mencionar la fecha del Diario Oficial en que consta la información.

**Implicancia para el diseño:** guardar siempre `fecha_publicacion` y `cve` por
registro, y mostrar atribución visible en cualquier salida pública. Nunca editar
el texto del extracto en la base; las correcciones van en campos derivados aparte.

### Restricción de datos personales

Los extractos contienen nombres y RUT de socios personas naturales. Aunque la
fuente sea pública, la Ley 21.719 aplica al tratamiento.

**Regla de diseño adoptada:** indexar y exponer por empresa. NO construir páginas
públicas buscables por nombre de persona natural. El grafo de socios se mantiene
como análisis interno o se publica solo agregado.

Falta revisar: https://www.diariooficial.interior.gob.cl/politica/

## Estructura de la fuente (verificada)

### Índice diario

```
https://www.diariooficial.interior.gob.cl/sociedades-web/indice-de-sociedades/{TAXID}{AAAAMMDD}/
```

HTML plano, sin sesión, sin JavaScript, sin token. Devuelve una lista de
`<a>` con razón social como texto y URL del PDF como href.

`{TAXID}` es el `term_taxonomy_id` de WordPress y **codifica el tipo societario,
no es constante en el tiempo**. Valores confirmados:

| TAXID | Sección                              | Visto en |
|-------|--------------------------------------|----------|
| `10`  | Sociedades de Responsabilidad Limitada | 2016 |
| `05`  | Sociedades de Responsabilidad Limitada | 2023 |
| `17`  | Sociedades por Acciones y Otras        | 2020 |
| `40`  | Sociedades / Cooperativas              | 2026 |

**Riesgo conocido:** hubo renumeración de taxonomías entre 2016 y 2023. Un crawler
con prefijo fijo pierde períodos completos en silencio. Antes de crawlear en serio,
mapear qué TAXID existen por año barriendo un rango de IDs contra fechas conocidas
y registrando el título que devuelve la página (`Indice de Sociedades / <nombre>`).

### El backend es WordPress

La página filtra la consulta SQL en el HTML (bug de ellos; no cambia lo que se
puede hacer, todo el contenido es público y reutilizable):

```sql
select wpp.id, wpp.post_excerpt as glosa, wpp.guid as url
from wp_posts wpp, wp_term_relationships wtr
where wpp.post_mime_type = 'application/pdf'
  and DATE_FORMAT(wpp.post_date,'%Y%m%d') = 'AAAAMMDD'
  and wpp.id = wtr.object_id
  and wtr.term_taxonomy_id = TAXID
order by wpp.post_excerpt ASC
```

Cada extracto es un PDF adjunto individual (`post_mime_type = 'application/pdf'`),
la razón social vive en `post_excerpt` y la URL en `guid`.

**Pendiente de probar:** si `/wp-json/wp/v2/media?after=...&before=...` está
habilitado. Si responde, se reemplaza el crawl HTML por descarga JSON paginada
y filtrable por fecha, que es mucho mejor punto de partida.

### Nombre del PDF (metadatos gratis)

```
https://www.diariooficial.interior.gob.cl/media/{AAAA}/{MM}/{DD}/1019761_M_LTDA_20160428.pdf
                                                              │       │  │     │
                                                              │       │  │     └─ fecha
                                                              │       │  └─────── forma societaria
                                                              │       └────────── tipo de acto
                                                              └────────────────── id interno
```

Confirmado: `M` = modificación, `LTDA` = responsabilidad limitada. Presumible pero
**no verificado**: `C` = constitución, `D` = disolución, más códigos `SPA`, `SA`,
`EIRL`, `COOP`. Verificar antes de asumir.

Observación del 28/04/2016 (TAXID 10): los 93 registros eran todos `_M_`. Las
constituciones deben estar bajo otra taxonomía ese mismo día. Confirmar cómo se
separan actos y formas societarias entre secciones.

## Lo que falta averiguar (en orden)

1. **¿Los PDFs traen texto seleccionable o son imagen?** Define todo el resto del
   diseño (parsing directo vs OCR previo). Es lo primero que hay que probar.
2. ¿Está abierta la REST API de WordPress?
3. Mapa completo de TAXID por año y de códigos de acto/forma en el nombre de archivo.
4. Qué tan uniforme es la redacción del extracto entre notarías.
5. Si el Registro de Empresas y Sociedades (RES, "empresa en un día", Ministerio de
   Economía) expone lo mismo ya estructurado. Muchas sociedades nuevas se crean por
   esa vía y podrían no requerir parsing. **El corpus completo requiere ambas fuentes.**

## Esquema objetivo (borrador)

Campos desde el índice, sin abrir el PDF:
`razon_social`, `fecha_publicacion`, `url_pdf`, `id_interno`, `tipo_acto`,
`forma_societaria`, `taxid_seccion`

Campos que solo están dentro del PDF:
`cve`, `capital`, `moneda_capital`, `socios[]` (nombre, RUT, participación),
`objeto_social`, `notaria`, `fecha_escritura`, `domicilio`, `duracion`,
`administracion`

Derivados:
`empresa_id` (para encadenar constitución → modificaciones → disolución),
`rubro_inferido`, `capital_normalizado_uf`

## Decisiones tomadas

- Crawl respetuoso: delay entre requests, caché local de HTML y PDFs crudos, nunca
  re-descargar lo ya bajado. El histórico son ~3.100 días hábiles desde 2016 y no
  hay ninguna prisa.
- Guardar el PDF crudo siempre. El parsing se rehace; la descarga no debería.
- Separar capas: descarga → texto crudo → extracción estructurada → entidades
  resueltas. Cada capa es idempotente y reprocesable sin volver a la fuente.
- Regex para campos duros (fechas, montos, RUT) y modelo para lo semántico (objeto
  social, socios y participaciones). No forzar regex donde la prosa es libre.
- Validar volúmenes contra el ISOC (Índice de Sociedades Constituidas) que el propio
  Diario Oficial publica en su informe IPALE, que combina Diario Oficial + RES. Sirve
  como cifra oficial de control para detectar si el crawler está perdiendo registros.

## Contexto de quien trabaja en esto

Perfil de data/analytics con experiencia en GCP (BigQuery, Cloud Functions,
Dataform), modelamiento estadístico y record linkage probabilístico con Splink.
La resolución de entidades sobre nombres y RUT es terreno conocido; el parsing de
prosa notarial no lo es. No hace falta explicar conceptos básicos de datos ni de
Python.

## Cosas que NO se hacen en este proyecto

- No se scrapea la Consulta Unificada de Causas del Poder Judicial (condiciones de
  uso prohíben la reproducción con fines comerciales o contra derechos de terceros).
  Fuente distinta, proyecto distinto.
- No se publica una vista buscable por persona natural.
- No se altera el texto de los extractos.
