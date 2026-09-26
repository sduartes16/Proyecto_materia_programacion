# Flight Delay Time Statistics Dashboard

Réplica local del laboratorio **4.8 – Flight Delay Time Statistics Dashboard**
(IBM Developer Skills Network · *Data Visualization with Python*, DV0101EN, Módulo 4).

## Requisitos

- Python 3.8 o superior (probado con 3.12.10)
- `pandas`, `dash`, `plotly` (ver `requirements.txt`)
- El archivo `airline_data.csv` en esta misma carpeta (9,4 MB)

## Puesta en marcha

```powershell
# 1. Entorno virtual (ya creado como .venv)
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt

# 2. Ejecutar
.\.venv\Scripts\python.exe flight_delay.py
```

Luego abre <http://127.0.0.1:10000> en el navegador (la app escucha en
`0.0.0.0:10000`). Para detener la aplicación, pulsa `Ctrl+C` en la terminal.

> En el laboratorio original el comando es `python3.8 flight_delay.py` y la
> aplicación se abre con el botón *Launch Application* indicando el puerto.
> Ambas cosas son específicas del Cloud IDE (Theia), no del código.

## Estructura del tablero

| Componente | ID | Dato graficado |
|---|---|---|
| Título | — | `Flight Delay Time Statistics` |
| Entrada de año | `input-year` | Valor por defecto `2010` |
| Gráfico 1 (segmento 1) | `carrier-plot` | `CarrierDelay` por aerolínea y mes |
| Gráfico 2 (segmento 1) | `weather-plot` | `WeatherDelay` |
| Gráfico 3 (segmento 2) | `nas-plot` | `NASDelay` |
| Gráfico 4 (segmento 2) | `security-plot` | `SecurityDelay` |
| Gráfico 5 (segmento 3) | `late-plot` | `LateAircraftDelay` |

El callback `get_graph` recibe el año, delega el cálculo en `compute_info` y
devuelve la lista de las cinco figuras.

## Notas sobre los datos

`airline_data.csv` es una **muestra** del dataset *Airline Reporting Carrier
On-Time Performance*:

- 27.000 filas, 33 aerolíneas, años **1987–2020**
- El enunciado pide el rango **2010–2020**; fuera de ese rango hay años sin
  datos y los gráficos salen vacíos.
- Se lee con `encoding="ISO-8859-1"` y con `dtype=str` en las columnas
  `Div1Airport`, `Div1TailNum`, `Div2Airport`, `Div2TailNum`; conserva esos
  parámetros o pandas emitirá advertencias y tipos inconsistentes.

## Despliegue en una plataforma (Render, Railway, Heroku…)

Los tres archivos que exige el enunciado ya están preparados:

| Archivo | Qué aporta |
|---|---|
| `flight_delay.py` | La app de Dash. Su última línea es `app.run_server(host='0.0.0.0', port=10000)` |
| `requirements.txt` | Dependencias congeladas + `gunicorn` |
| `Procfile` | `web: gunicorn flight_delay:server --bind 0.0.0.0:$PORT` |

**El objeto WSGI.** gunicorn no ejecuta `flight_delay.py` como *script* (por eso
`if __name__ == '__main__'` no se activa en producción); importa el módulo y
busca dentro de él una variable llamada `server`. Por eso el archivo termina con:

```python
server = app.server           # objeto WSGI que consume gunicorn
...
if __name__ == '__main__':
    app.run_server(host='0.0.0.0', port=10000)
```

**Aviso importante sobre `app.run_server`.** Ese método pertenece a Dash 1.x/2.x;
Dash 3+ lo eliminó en favor de `app.run(...)`. Aquí hay instalado **Dash 4.4.1**,
así que la llamada del enunciado fallaría con
`AttributeError`/`ObsoleteAttributeException` si no fuera por el alias de
compatibilidad que se añadió justo antes:

```python
try:
    _ = app.run_server       # Dash 1.x / 2.x
except Exception:            # Dash >= 3 -> ObsoleteAttributeException
    setattr(app, 'run_server', app.run)
```

Detalle fino: **no sirve `hasattr(app, 'run_server')`** para decidir esto. Dash 3+
intercepta el atributo y lanza `ObsoleteAttributeException` (ver
`dash/_obsolete.py`) en vez de `AttributeError`, y `hasattr()` solo absorbe
`AttributeError`, así que la excepción se propaga y tumba el arranque. De ahí el
`try/except`.

**Verificado** en este equipo: `python flight_delay.py` imprime
`Dash is running on http://0.0.0.0:10000/` y responde `HTTP 200`; sirviendo
`flight_delay.server` con un servidor WSGI real, `GET /_dash-layout` devuelve los
cinco gráficos, el campo de año y el título, y `/_dash-dependencies` registra el
callback.

**Ojo con `pip freeze` en Windows.** En PowerShell 5.1 `pip freeze > requirements.txt`
escribe el archivo en **UTF-16LE** (con BOM), y en el servidor Linux
`pip install -r requirements.txt` falla al leerlo. Usa una de estas dos formas:

```powershell
pip freeze | Set-Content -Encoding utf8 requirements.txt
# o, si ya lo generaste mal, conviértelo desde Python:
python -c "s=open('requirements.txt',encoding='utf-16').read();open('requirements.txt','w',encoding='utf-8',newline='\n').write(s)"
```

El `requirements.txt` actual está en UTF-8 sin BOM y `pip install --dry-run -r
requirements.txt` lo resuelve sin errores. `gunicorn` va **sin pin** (la última
versión en PyPI es 26.2.0 y pide Python ≥ 3.10): dentro de Windows se instala
pero no se puede ejecutar (depende del módulo `fcntl`), y un pin inventado
rompería el build del servidor.

**El CSV también viaja al servidor.** `flight_delay.py` lee `airline_data.csv`
con `Path(__file__).with_name(...)`, así que el repositorio debe incluirlo
(9,4 MB, dentro de los límites de GitHub y de la mayoría de plataformas). El
`.gitignore` excluye `.venv/`, que no debe subirse.

**gunicorn no se puede probar en Windows.** Para una prueba local parecida a
producción usa `waitress` (`waitress-serve --port=10000 flight_delay:server`) o
simplemente `python flight_delay.py`.

**Fija la versión de Python del servidor.** El *freeze* dejó `numpy==2.5.3`, que
exige Python ≥ 3.12 y solo publica wheels para cp312–cp315 (con 3.10/3.11 el
build intentaría compilar desde fuente y fallaría). Ya está resuelto con el
archivo `.python-version` en la raíz del repo, que contiene `3.12` (Render
también acepta la variable `PYTHON_VERSION`, pero ahí el valor debe ser
completo, p. ej. `3.12.10`).

## Despliegue en Render, paso a paso

### 1. Repositorio en GitHub (ya publicado)

El proyecto vive en <https://github.com/ccristbo/dashboard>, rama `main`, con el
remoto ya configurado. Para futuros cambios basta:

```powershell
git add -A
git commit -m "descripcion del cambio"
git push
```

Nota histórica: ese repositorio ya tenía un *commit* inicial con un `README.md`
de dos líneas creado por GitHub. Los dos historiales no compartían ancestro, así
que se integraron con `git merge origin/main --allow-unrelated-histories` y el
conflicto de `README.md` se resolvió a favor de esta versión. No se reescribió
nada: el `Initial commit` sigue en el historial.

### 2. Crear el servicio en Render

**Opción A (recomendada, automática).** Ya existe `render.yaml`, así que:
*Render Dashboard → New → Blueprint → seleccionar el repositorio → Apply*.
Render crea el servicio con todo configurado.

**Opción B (manual).** *New → Web Service → conectar GitHub → elegir el repo* y
rellenar:

| Campo | Valor |
|---|---|
| Language / Runtime | Python |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `gunicorn flight_delay:server --bind 0.0.0.0:$PORT` |
| Instance Type | **Free** |
| Health Check Path | `/` (opcional) |

### 3. Detalles de Render que hay que tener presentes

- **Render no lee el `Procfile`** (solo Heroku lo hace). En Render el comando de
  arranque sale del campo *Start Command* o del `startCommand` de `render.yaml`.
  El `Procfile` se deja en el repo porque es inocuo y sirve para otras
  plataformas.
- **Puerto.** Render inyecta la variable `PORT` (su valor por defecto es
  justamente **10000**, igual que el del laboratorio) y exige que el servidor
  escuche en `0.0.0.0`. Por eso el comando de producción usa
  `--bind 0.0.0.0:$PORT`; la línea
  `app.run_server(host='0.0.0.0', port=10000)` solo se ejecuta en local
  (`python flight_delay.py`), porque gunicorn **importa** el módulo y nunca
  entra en el bloque `if __name__ == '__main__'`.
- **El plan por defecto de un servicio nuevo es `0.5c-512mb` (de pago).** En el
  flujo manual hay que elegir *Free* a mano; en `render.yaml` ya está declarado
  `plan: free`.
- **Plan Free = se duerme.** Tras un rato sin visitas la instancia se suspende y
  el siguiente acceso tarda varias decenas de segundos en responder (arranque en
  frío, que además vuelve a leer el CSV de 9,4 MB).
- **Sistema de archivos efímero**, pero aquí no importa: los datos van dentro del
  repositorio y la app no escribe nada.
- **Si el deploy falla**, revisa los *Logs*: los fallos típicos son que el
  intérprete elegido no tenga wheels (por eso el `.python-version`) o que el
  comando de arranque no apunte a `flight_delay:server`.

### 4. Comprobar que quedó bien

Abre la URL `https://<nombre-del-servicio>.onrender.com` y verifica que salen los
cinco gráficos al escribir `2010` en el campo del año. En local, la prueba
equivalente sin gunicorn es (requiere instalar `waitress`, que no está en
`requirements.txt` porque no se usa en producción):

```powershell
.\.venv\Scripts\python.exe -m pip install waitress
.\.venv\Scripts\python.exe -m waitress --port=10000 flight_delay:server
```

## Diferencias frente al laboratorio original

1. Los datos se leen de `airline_data.csv` local en vez de la URL de S3
   (funciona sin internet y arranca al instante).
2. No se instala `httpx==0.20`: ese pin resolvía el proxy del Cloud IDE y en
   local es innecesario.
3. Se añadió una guarda en `get_graph` para que vaciar el campo de año no rompa
   el callback con `ValueError`.
4. Se añadió el objeto WSGI `server = app.server` y un alias de compatibilidad
   para `app.run_server`, requisitos de la versión de Dash instalada (4.4.1) y
   del despliegue con gunicorn. Ver la sección *Despliegue*.

## Ejercicios del laboratorio (pendientes, si quieres practicar)

1. Cambiar el título a `"Flight Details Statistics Dashboard"` con `font-size` 35.
2. Renombrar el archivo a `flight_details.py` y relanzar.
3. Detener la app con `Ctrl+C`.
