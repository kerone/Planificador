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
agosto), usa la barra **Editar rango** de la vista Plan (justo debajo del gráfico):
**arrastra sobre los días** para seleccionar el rango y aplica ahí el campo y el
valor (opcionalmente solo a días con columnas). La tabla diaria sigue siendo
editable celda a celda como siempre.

## El modelo (todo en columnas)

1. **Columnas por día** — de la query de negocio (`daily_business_offer`), editables
   por día.
2. **POM del mes** = máximo de columnas diarias del mes en cada colmena (forzable).
3. **Cortes y parque desde el POM** (pestaña **Organizador Operativo**, todo en
   rutas enteras con techo/ROUNDUP). Entradas por mes: máx TM (H), máx TT total
   I = largas + cortas, máx TP (J, se usa tal cual) y máx cortas (K):

   ```
   Corte N   = mín(cap. mañana, cap. tarde) × 2      (reparto simétrico 50/50)
   Corte O   = capacidad TM+TT      ·  P = capacidad máxima (con TP)
   Rutas TM  = techo(50% POM ÷ prod) hasta N; entre N y O, techo de lo que la
               tarde completa no cubre; por encima de O, todas las calles TM
   Largas    = techo del resto ≤ máx largas
   Cortas    = resto ÷ (prod × 6,5/7,5) ≤ máx cortas · TP = resto ≤ máx TP
   Techo SM  = techo₁₀(TM·prod + TP·prod/2)     (X, redondeado a decenas)
   Veh. POM  = máx(TM+TP, largas+cortas+TP)     (Z)
   ```

4. **Operativa del día** (con la oferta real; POM editable por día): cascada
   **puente → mañana → largas → cortas**, cada turno capado a su parque:

   ```
   Oferta SM = mín(mañana de la query, techo SM X)   (editable por día)
   TP día    = mín( techo((Of.SM − TM·prod/2)/prod), %TP·tarde, parque TP )
   Mañana    = techo((Of.SM − TP·prod/2)/prod)                 ≤ parque TM
   Largas    = techo del resto ≤ parque largas · Cortas = resto ≤ parque cortas
   Servido   = Σ rutas × prod (cortas × 6,5/7,5) · Oferta_OK si ≥ oferta
   ```

El motor reproduce la pestaña **Organizador Operativo** del Excel en sus 3.285
filas: coincidencia exacta en las 2.644 filas sanas de todas las columnas (las
641 restantes las gobierna una variante antigua del propio Excel con un signo
cambiado — R > máximo y largas negativas — que aquí se implementa corregida,
como en las filas nuevas de la hoja).

## Interfaz

Las **colmenas** están en una barra horizontal arriba, cada una con su color
identitario (bcn1 azul, vlc1 verde, mad1 rojo, alc1 amarillo, svq1 morado, mad2
rosa, mad3 naranja, pan2/pan3 pan). El botón **+** al final de la barra crea una
colmena nueva (código + color identitario): nace vacía y en todos los planes —
configura sus máximos por turno en Parámetros y carga sus columnas con el CSV.
El menú lateral se lee como el flujo de trabajo — **Plan · Parámetros ·
Oleadas · Plan vs PMR** — y la cabecera agrupa lo demás: **Importar CSV** (la
operación frecuente), **Copia ▾** (descargar/restaurar el JSON de respaldo),
— el año del plan se elige o se crea en el desplegable del chip **AÑO** —,
**?** (la Guía, que se abre encima de donde estés) y el ☁ de la nube. Los
**filtros de mes** (Plan / Plan vs PMR) van coloreados por la productividad
media del mes: **verde = la más alta del año**, azul = la más baja (sin rojos:
no es una alarma).

## Secciones (por colmena, en el menú lateral)

| Sección | Función |
|---|---|
| **Plan** | Dos niveles con los chips de mes como zoom. Con **AÑO**: el nivel mensual — KPIs, **POM por mes** editable (máx. del mes, con override), gráfico POM vs capacidad, **columnas sin servir por mes**, desglose del POM por turno y la tabla anual completa. Con un **mes**: la tabla del Excel por día, con gráficos. **Negocio**, productividad, **POM** y Oferta SM editables por día. Muestra **Negocio** (lo que pide negocio) y **Servido** (lo que la flota reparte de verdad = Σ rutas × productividad) por separado: casi nunca coinciden por el redondeo del nº de rutas (100 col ÷ 16,9 = 5,9 → 6 rutas → 6 × 16,9 ≈ 101 servidas), y la columna **Dif** muestra el ± (en rojo solo si sale de la tolerancia). Las columnas por turno (Col TM/TT/TP) son rutas × prod y suman al Servido. Muestra **Servido**, **Dif** y **% Cob**; TT-L / TT-C por separado; y **Limita** = el motivo del KO (*faltan largas* / *cortas* / *puente* / *vehículos tarde* / *tope de vehículos* / *exceso*). Un día es **KO** cuando lo servido se desvía del negocio más de la **tolerancia ±%** (parámetro por colmena, 2 % por defecto), por defecto o por exceso. Un anillo de cobertura y KPIs resumen el periodo filtrado. En ambos niveles, la barra **Editar rango** aplica un valor a un rango de días arrastrando, y la gráfica de **Productividad** (línea con valores) permite comparar **años** (elige planes con los chips) o ver **todas las colmenas** juntas. Comparando dos años, la propia gráfica anota en cada mes el **Δ de vehículos que exige el POM** (+N verde = el año nuevo necesita más flota · −N rojo = menos): si baja la productividad, ves al momento cuántos vehículos más te pide el POM. |
| **Parámetros** | Una tarjeta con los **máximos de vehículos** por turno y mes (TM, **TT largas**, **TT cortas** y TP — los H·I·J·K del Organizador; el TP se usa tal cual) y la **productividad por mes** de las rutas de 8 h (las cortas rinden siempre prod × 6,5/7,5). La **tolerancia KO** (± % de desviación sobre negocio que se admite antes de marcar un día en rojo, 2 % por defecto) es editable. El único criterio restante (el **% máx del turno puente**) va plegado en **«Criterios avanzados»** — el %TM y el criterio de Oferta SM ya no existen: los formula el Organizador (50/50 con cortes, y Of.SM = mín(query, techo X)). |
| **Oleadas** | Gantt de la jornada (06:00–00:00) con las oleadas de carga por turno. La **parrilla de carga y las duraciones por tipo de ruta** se configuran aquí mismo, plegadas bajo el gantt: editas y ves el efecto encima. |
| **Plan vs PMR** | Día a día plan vs PMR, **cada lado con sus vehículos**: el plan con los que calcula el motor, PMR con los reales del CSV (`Workers`). Carga el export de turnos con «Importar CSV → Cargar en Plan vs PMR» (se guarda con el plan, no lo modifica ni recalcula) y muestra KPIs, gráfico de diferencia conmutable (**vehículos o productividad**, PMR − plan) y tabla diaria con prod plan/PMR, vehículos por turno M·P·T de cada lado, **Δ vehículos por turno** (+ rojo = PMR usa más / − verde = usa menos) y **columnas Negocio / plan / PMR**, con filtro por mes. Las barras de todos los gráficos son clicables: llevan al día o al mes de la tabla. Los gráficos se dibujan al ancho real de la pantalla y se redimensionan solos. El botón «Vaciar» borra los datos PMR cargados (el plan no se toca). |

La **Guía** (resumen del modelo y de qué se edita) vive en el botón **?** de la
cabecera y se abre como ventana sobre la vista actual.

## Datos iniciales y año nuevo

- Productividades por día 2026: del Excel «Productividades Reparto 2026».
- Columnas por día 2026: foto de la query de negocio del 08/07/2026.
- Máximos de vehículos por turno y mes: derivados del mismo Excel.
- **AÑO ▾ → + Nuevo año…** (en la barra de meses) crea el 2027 copiando
  productividades y máximos; las columnas se
 dejan vacías (se cargan de la query), se copian o se copian escaladas +X %.
  El año activo también se cambia desde ese desplegable.
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
   Dos acciones: **Cargar en Plan vs PMR** guarda los datos con el plan para
   verlos día a día en esa pestaña, sin modificar nada; **Aplicar**
   sincroniza la productividad diaria de las 8 h (la **configurada** del CSV,
   campo `Productivity`, ponderada por trabajadores de mañana + puente + tarde
   larga — así coincide con la query, p.ej. 16,75, y no con el 16,76 que sale de
   dividir las columnas planificadas redondeadas) y los
   **máximos de vehículos por turno/mes** (se amplían a los vehículos que PMR usó
   en el mes cuando superan los del plan; nunca se reducen solos), con alcance a
   elegir: solo desde hoy (resto de año) o todo el rango del CSV. Si el CSV es de
   otro año que el plan activo, «Cargar en Plan vs PMR» ofrece cambiar de plan.

Para las columnas, exporta la consulta de siempre a CSV:

```sql
SELECT * FROM `prod-mercadona.venta_mart.daily_business_offer`
WHERE center IN ('vlc1','bcn1','mad1','alc1','svq1','mad2','pan2','pan3','mad3')
  AND service_date >= '2027-01-01'
ORDER BY center, service_date
```

Usa `total_columns` (o `adjusted_total_columns` con la casilla). Opcionalmente puede
fijar la Oferta SM de cada día con `morning_columns` de la query.
