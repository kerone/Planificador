# Planificador de Productividades — Reparto (Columnas Operativas)

Producto HTML autocontenido para planificar la operativa de reparto de las colmenas
(bcn1, vlc1, mad1, alc1, mad3, svq1, pan2, pan3, mad2). Réplica del Excel
**«Desglose Columnas Operativas»** extendida a toda la red y con detalle a nivel día.

**Un solo archivo: [`index.html`](index.html).** Se abre en cualquier navegador, sin
servidor. Los datos se guardan en el navegador (localStorage) y se pueden exportar /
importar como JSON.

## ☁ Nube (opcional): datos en Supabase, HTML en GitHub Pages

La sincronización se configura **en el código** (constante `NUBE_CONFIG` al inicio
del `<script>` de `index.html`): pon la URL del proyecto y la clave `anon` y queda
activa sin ningún botón — guarda en la nube al editar (el ☁ de la cabecera se
ilumina), siembra el documento la primera vez y, al abrir, si la nube va por
delante **se carga sola, sin preguntar** (la nube manda: siempre ves lo de Supabase).
Si cierras la pestaña con cambios aún sin subir, se suben solos al volver a abrir; y si
una subida falla, el ☁ se pone rojo con el motivo y sale un aviso. Así el `index.html`
puede servirse estático desde GitHub Pages y todos los equipos comparten datos.

1. Crea la tabla una vez (SQL Editor de Supabase):

   ```sql
   create table if not exists planificador (
     clave text primary key, data jsonb not null,
     updated_at timestamptz default now()
   );
   alter table planificador enable row level security;
   create policy "acceso anon" on planificador for all using (true) with check (true);
   ```

2. Rellena `NUBE_CONFIG` (url, key) en `index.html` y publícalo.
   ⚠ La política de arriba deja la tabla accesible a cualquiera con la clave anon
   (que además viaja en el HTML): mantén el repositorio/clave dentro del equipo.

Para aplicar un valor a muchos días de golpe (p. ej. productividad 16 del 1 al 15 de
agosto), usa la barra **Editar rango** de Día a día (justo debajo del gráfico):
**arrastra sobre los días** para seleccionar el rango y aplica ahí el campo y el
valor (opcionalmente solo a días con columnas). La tabla diaria sigue siendo
editable celda a celda como siempre.

## El modelo (todo en columnas)

1. **Columnas por día** — de la query de negocio (`daily_business_offer`), editables
   por día.
2. **POM del mes** = máximo de columnas diarias del mes en cada colmena (forzable).
3. **Parque de rutas desde el POM** (fase 1 de la **Carpeta Verde**, todo en rutas
   enteras con techo/ROUNDUP):

   ```
   Rutas TM     = techo(POM × %OfertaTM ÷ prod)   ≤ máx TM
   Rutas largas = techo del resto                  ≤ máximo largas
   Rutas cortas = resto ÷ (prod × 6,5/7,5)         ≤ máximo cortas
   Rutas TP     = resto                            ≤ mín(máx TP, máx TM ÷ 2)
   ```

   `%Oferta TM`: por defecto **manda negocio** — cada día usa su % real de la query
   (mañana ÷ total); la grid por colmena y mes (0,5 general · 0,7 PAN2/PAN3 ·
   0,52 mad3 sep–dic) es el respaldo, o la norma en modo «fijo». Las cortas rinden `prod × 6,5/7,5` (con 15,8 → 13,69).

4. **Operativa del día** (fase 2, con la oferta real; POM editable por día):
   cascada **puente → mañana → largas → cortas**, cada turno con techo y capado a
   su parque de la fase 1:

   ```
   Oferta SM = mañana de la query (editable por día)
   TP día    = mín( techo((Of.SM − TM·prod/2)/prod), %TP·tarde, parque TP )
   Mañana    = techo((Of.SM − TP·prod/2)/prod)                 ≤ parque TM
   Largas    = techo del resto ≤ parque largas · Cortas = resto ≤ parque cortas
   Servido   = Σ rutas × prod (cortas × 6,5/7,5) · Oferta_OK si ≥ oferta
   ```

El motor reproduce la pestaña **Carpeta Verde** del Excel de Productividades en sus
3.209 filas cacheadas (las únicas discrepancias son 14 filas con overrides manuales
del propio Excel).

## Interfaz

Las **colmenas** están en una barra horizontal arriba, cada una con su color
identitario (bcn1 azul, vlc1 verde, mad1 rojo, alc1 amarillo, svq1 morado, mad2
rosa, mad3 naranja, pan2/pan3 pan). El botón **+** al final de la barra crea una
colmena nueva (código + color identitario): nace vacía y en todos los planes —
configura sus máximos por turno en Parámetros y carga sus columnas con el CSV.
Las **secciones** están en el menú vertical
de la izquierda. Los **filtros de mes** (Día a día / Comparativa) van coloreados
por la productividad media del mes: **verde = la más alta del año**, azul = la
más baja (sin rojos: no es una alarma).

## Secciones (por colmena, en el menú lateral)

| Sección | Función |
|---|---|
| **Día a día** | La tabla del Excel por día, con filtros por mes y gráficos. **Negocio**, productividad, **POM** y Oferta SM editables por día. Muestra **Negocio** (lo que pide negocio) y **Servido** (lo que la flota reparte de verdad = Σ rutas × productividad) por separado: casi nunca coinciden por el redondeo del nº de rutas (100 col ÷ 16,9 = 5,9 → 6 rutas → 6 × 16,9 ≈ 101 servidas), y la columna **Δ neg** muestra el ±. Las columnas por turno (Col TM/TT/TP) son rutas × prod y suman al Servido. Muestra **Servido**, **Δ neg** y **% Cob**; TT-L / TT-C por separado; y **Limita** = el turno que agota su máximo (*faltan largas* / *cortas* / *puente* / *vehículos tarde* / *tope de vehículos*). Un anillo de cobertura y KPIs resumen el periodo filtrado. |
| **Mensual** | POM automático (máx. del mes) con override, gráfico POM vs capacidad y **columnas sin servir por mes**, y el desglose del POM por turno. |
| **Parámetros** | **Máximo de vehículos** por turno y mes, cada uno parametrizable: TM, **TT largas**, **TT cortas** y TP (12 meses) — son las «calles» de la Carpeta Verde. Productividad por mes de las rutas de 8 h (media de los días; editarla fija el mes) — las cortas rinden siempre prod × 6,5/7,5 —, criterio TP y % TP, criterio Oferta SM y la parrilla de carga. |
| **Oleadas** | Gantt de la jornada (06:00–00:00) con las oleadas de carga por turno según la parrilla. |
| **Comparativa** | Día a día plan vs PMR, **cada lado con sus vehículos**: el plan con los que calcula el motor, PMR con los reales del CSV (`Workers`). Carga el export de turnos con «Importar CSV → Cargar en Comparativa» (se guarda con el plan, no lo modifica ni recalcula) y muestra KPIs, gráfico de diferencia conmutable (**vehículos o productividad**, PMR − plan) y tabla diaria con prod plan/PMR, vehículos por turno M·P·T de cada lado, **Δ vehículos por turno** (+ rojo = PMR usa más / − verde = usa menos) y **columnas Negocio / plan / PMR**, con filtro por mes. Las barras de todos los gráficos son clicables: llevan al día o al mes de la tabla. Los gráficos se dibujan al ancho real de la pantalla y se redimensionan solos. El botón «Vaciar» borra los datos PMR cargados (el plan no se toca). |
| **Guía** | Resumen del modelo y de qué se edita. |

## Datos iniciales y año nuevo

- Productividades por día 2026: del Excel «Productividades Reparto 2026».
- Columnas por día 2026: foto de la query de negocio del 08/07/2026.
- Máximos de vehículos por turno y mes: derivados del mismo Excel.
- **+ Año** crea el 2027 copiando productividades y calles; las columnas se
  dejan vacías (se cargan de la query), se copian o se copian escaladas +X %.
  La copia de valores diarios (productividades y columnas) se alinea **por día de
  la semana**: cada día del año nuevo toma el día equivalente del base (lunes ↔
  lunes, ±3 días; en los bordes del año salta una semana), así el patrón semanal
  no se desplaza. Al importar después el CSV de negocio, los días presentes en el
  CSV machacan los valores copiados.

## Importar CSV (auto-detecta el formato)

El botón «Importar CSV» acepta dos exports y detecta solo cuál le pasas:

1. **Columnas de negocio** — la query `daily_business_offer` (abajo).
2. **Productividades por turno** — el export de PMR
   (`Origin, Shift, Shift Type, Service Date, Planned Columns, Workers, …`).
   Dos acciones: **Cargar en Comparativa** guarda los datos con el plan para
   verlos día a día en la pestaña Comparativa, sin modificar nada; **Aplicar**
   sincroniza la productividad diaria de las 8 h (la **configurada** del CSV,
   campo `Productivity`, ponderada por trabajadores de mañana + puente + tarde
   larga — así coincide con la query, p.ej. 16,75, y no con el 16,76 que sale de
   dividir las columnas planificadas redondeadas) y los
   **máximos de vehículos por turno/mes** (se amplían a los vehículos que PMR usó
   en el mes cuando superan los del plan; nunca se reducen solos), con alcance a
   elegir: solo desde hoy (resto de año) o todo el rango del CSV. Si el CSV es de
   otro año que el plan activo, «Cargar en Comparativa» ofrece cambiar de plan.

Para las columnas, exporta la consulta de siempre a CSV:

```sql
SELECT * FROM `prod-mercadona.venta_mart.daily_business_offer`
WHERE center IN ('vlc1','bcn1','mad1','alc1','svq1','mad2','pan2','pan3','mad3')
  AND service_date >= '2027-01-01'
ORDER BY center, service_date
```

Usa `total_columns` (o `adjusted_total_columns` con la casilla). Opcionalmente puede
fijar la Oferta SM de cada día con `morning_columns` de la query.
