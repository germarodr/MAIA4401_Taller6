# Taller 6: APIs

**Autor:** German Rodríguez  
**Curso:** MAIA 4401 — MLOps  
**Fecha:** Agosto 2026

---

## Resumen

En este taller se explora el servicio de predicciones del modelo de *churn* bancario a
través de una API construida con **FastAPI** y ejecutada mediante **Uvicorn**. El flujo
incluye el análisis de la estructura de la API, la configuración de una máquina virtual
**AWS EC2**, la ejecución de pruebas automatizadas con **tox** y la interacción con las
rutas de salud y predicción desde el navegador.

También se actualiza la API para utilizar la versión `0.0.2` del paquete del modelo,
construida en el Taller 5 después de eliminar la característica `Customer_Age`. Finalmente
se comparan las respuestas de la API al omitir una variable eliminada del modelo y otra
variable que todavía es requerida.

Durante la preparación del taller se realizaron o se planificaron las siguientes actividades:

- Exploración de los archivos de configuración, dependencias, esquemas y rutas de la API.
- Lanzamiento y configuración pendiente de una instancia AWS EC2 con Ubuntu 24.04.
- Publicación del proyecto y ejecución de las pruebas dentro de la máquina virtual EC2.
- Prueba de las rutas `/api/v1/health` y `/api/v1/predict` desde la instancia EC2.
- Actualización pendiente del paquete del modelo a `model_abandono-0.0.2-py3-none-any.whl`.
- Prueba pendiente del comportamiento de la API ante variables ausentes en la entrada.

# 1. Exploración de los archivos de la API (Parte 1)

El proyecto `bankchurn-api` está organizado como una aplicación Python independiente.
Su estructura principal es la siguiente:

```
bankchurn-api/
├── app/
│   ├── api.py
│   ├── config.py
│   ├── main.py
│   ├── schemas/
│   │   ├── health.py
│   │   └── predict.py
│   └── tests/
├── model-pkg/
│   └── model_abandono-0.0.1-py3-none-any.whl
├── requirements.txt
├── test_requirements.txt
├── typing_requirements.txt
├── tox.ini
└── Procfile
```

## 1.1 Archivos de dependencias

El archivo `requirements.txt` contiene las dependencias necesarias para ejecutar la API
y cargar el paquete del modelo:

```text
uvicorn>=0.20.0,<0.30.0
fastapi>=0.88.0,<1.0.0
python-multipart>=0.0.5,<0.1.0
typing_extensions>=4.2.0,<5.0.0
loguru>=0.5.3,<1.0.0
```

El archivo `test_requirements.txt` agrega las dependencias necesarias para ejecutar las
pruebas automatizadas. El archivo `typing_requirements.txt` contiene las herramientas de
verificación de tipos y calidad de código. Estas dependencias se instalan de forma
separada para mantener diferenciados los ambientes de ejecución, prueba y revisión.

## 1.2 Ambientes definidos en `tox.ini`

El archivo `tox.ini` define los ambientes `test_app`, `run` y `checks`. Para este taller
son especialmente importantes los dos primeros:

| Ambiente | Dependencias | Comando | Propósito |
|---|---|---|---|
| `test_app` | `-rtest_requirements.txt` | `pytest -vv app/tests/` | Ejecuta las pruebas de la aplicación con salida detallada. |
| `run` | Las mismas dependencias de `test_app` | `python app/main.py` | Inicia el servidor de la API. |
| `checks` | `-rtyping_requirements.txt` | `flake8`, `isort`, `black` y `mypy` | Revisa estilo, formato y tipos. |

El ambiente `run` reutiliza el directorio del ambiente `test_app` mediante la propiedad
`envdir = {toxworkdir}/test_app`. Además, `tox.ini` establece `PYTHONPATH=.` y
`PYTHONHASHSEED=0` para la ejecución de la aplicación y sus pruebas.

## 1.3 Esquemas de la API

### Esquema `Health`

El esquema está definido en `app/schemas/health.py` y representa la respuesta de la ruta
de salud. Contiene tres campos de texto:

| Campo | Tipo | Descripción |
|---|---|---|
| `name` | `str` | Nombre del proyecto o API. |
| `api_version` | `str` | Versión de la aplicación. |
| `model_version` | `str` | Versión del paquete del modelo cargado. |

### Esquemas de predicción

En `app/schemas/predict.py` se definen dos modelos de Pydantic:

- `MultipleDataInputs`: recibe una lista llamada `inputs`. Cada elemento de la lista
  debe cumplir el esquema `DataInputSchema`, importado desde el paquete del modelo.
- `PredictionResults`: representa la respuesta de la predicción. Incluye `errors`,
  `version` y `predictions`.

El ejemplo incluido en el esquema muestra las variables originales del modelo, entre
ellas `Customer_Age`, `Gender`, `Dependent_count`, `Education_Level`, `Credit_Limit`,
`Total_Trans_Amt` y `Avg_Utilization_Ratio`. Este ejemplo será actualizado al utilizar
la versión `0.0.2`, en la que se eliminó `Customer_Age`.

## 1.4 Argumentos de `uvicorn.run`

En `app/main.py` el servidor se inicia con:

```python
uvicorn.run(app, host="0.0.0.0", port=8001, log_level="debug")
```

Los cuatro argumentos son:

| Argumento | Valor | Función |
|---|---|---|
| `app` | La instancia FastAPI | Indica qué aplicación debe ejecutar Uvicorn. |
| `host` | `0.0.0.0` | Permite aceptar conexiones desde todas las interfaces de red de la máquina virtual. |
| `port` | `8001` | Define el puerto en el que queda disponible el servidor. |
| `log_level` | `debug` | Configura un nivel detallado de mensajes durante la ejecución. |

## 1.5 Configuración de la aplicación

El archivo `app/config.py` define los parámetros globales de la aplicación mediante la
clase `Settings`:

- `API_V1_STR` establece la ruta base `/api/v1`.
- `PROJECT_NAME` define el nombre visible de la API.
- `BACKEND_CORS_ORIGINS` contiene los orígenes permitidos para solicitudes CORS.
- `LoggingSettings` define el nivel de logging, inicialmente `INFO`.
- `InterceptHandler` redirige los mensajes del módulo estándar de logging hacia Loguru.
- `setup_app_logging` configura los loggers de Uvicorn y de la aplicación.

## 1.6 Ejecución de las rutas

### Ruta raíz `/`

La ruta raíz devuelve una respuesta HTML con el mensaje `Welcome to the API` y un enlace
a la documentación interactiva ubicada en `/docs`.

### Ruta `/api/v1/health`

La ruta de salud responde mediante el esquema `Health`. Construye un objeto con el nombre
del proyecto, la versión de la API y la versión del modelo instalado. Su objetivo es
comprobar que el servidor está activo y que el paquete del modelo puede cargarse.

Ejemplo de respuesta esperada:

```json
{
  "name": "Banckchurn API",
  "api_version": "0.1.0",
  "model_version": "0.0.1"
}
```

Los valores exactos de versión deben verificarse en la ejecución realizada.

### Ruta `/api/v1/predict`

La ruta de predicción recibe un objeto con una lista de entradas. FastAPI y Pydantic
validan la estructura recibida; después, el código realiza estas acciones:

1. Convierte las entradas a un `DataFrame` de pandas.
2. Registra la solicitud en los logs.
3. Reemplaza valores `NaN` por `None`.
4. Invoca `make_prediction` del paquete del modelo.
5. Si el modelo devuelve errores de validación, responde con HTTP `400`.
6. Si la validación es correcta, devuelve las predicciones y la versión del modelo.

# 2. Prueba y ejecución de la API (Parte 2)

## 2.1 Lanzamiento de la instancia EC2

Este es el primer paso de ejecución de la Parte 2. Se debe lanzar una instancia AWS EC2
con la configuración recomendada: tipo `t3.small`, sistema operativo Ubuntu 24.04 y
disco de 20 GB.

- **IP pública:** `<COMPLETAR_IP>`
- **Región:** `<COMPLETAR_REGION>`
- **ID de instancia:** `<COMPLETAR_INSTANCE_ID>`

![Consola de AWS EC2 con la instancia en ejecución y el usuario visible](imgs/t6-01-ec2-console.png)

*Figura 2.1 — Consola de AWS EC2 con la máquina virtual en ejecución (numeral 2.1 — 5 pts).* 

## 2.2 Creación y publicación del repositorio

El repositorio público `MAIA4401_Taller6` fue creado en GitHub y contiene los archivos
del taller. La estructura de la raíz es:

```
repo/
├── app/
├── model-pkg/
├── requirements.txt
├── test_requirements.txt
├── typing_requirements.txt
├── tox.ini
└── Procfile
```

El repositorio se actualizó mediante:

```bash
git add .
git commit -m "API inicial de predicción bankchurn"
git push
```

- **URL del repositorio:** `<COMPLETAR_URL_REPOSITORIO>`

## 2.3 Conexión a la máquina virtual

Se debe establecer la conexión SSH con la llave privada entregada por AWS:

```bash
ssh -i /ruta/a/llave.pem ubuntu@<IP_PUBLICA>
```

![Terminal con la conexión SSH exitosa a la máquina virtual](imgs/t6-02-ssh.png)

*Figura 2.3 — Conexión SSH exitosa a la VM (numeral 2.3 — 5 pts).*

## 2.4 – 2.7 Instalación de herramientas

Dentro de la máquina virtual EC2 se actualizarán los paquetes y se instalarán pip, unzip
y la herramienta para crear ambientes virtuales:

```bash
sudo apt update
sudo apt install python3-pip
sudo apt install zip unzip
sudo apt install python3.12-venv
```

## 2.8 – 2.9 Creación y activación del ambiente virtual

```bash
python3 -m venv /home/ubuntu/env-api
source /home/ubuntu/env-api/bin/activate
```

La activación se verificó observando el prefijo `(env-api)` en la terminal.

## 2.10 Clonación del repositorio

Dentro de la VM EC2 se clonará el repositorio publicado en GitHub:

```bash
git clone <URL_DEL_REPOSITORIO>
cd <NOMBRE_DEL_REPOSITORIO>
ls
```

La estructura se comprobó verificando que las carpetas `app` y `model-pkg`, junto con
los archivos de configuración, estuvieran en la raíz del repositorio.

## 2.11 Instalación de tox

Dentro de la VM EC2 se instalará Tox y se configurará el `PATH`:

```bash
pip install tox
sudo apt-get install tox
export PATH=$PATH:/home/ubuntu/.local/bin
tox --version
```

La versión instalada de tox fue `<COMPLETAR_VERSION_TOX>`.

## 2.12 Ejecución de las pruebas de la API

```bash
tox run -e test_app
```

Este comando se ejecutará dentro de la VM EC2. Tox creará o reutilizará el ambiente
aislado, instalará las dependencias de prueba y ejecutará `pytest -vv app/tests/`.
La captura y el resultado que se incluirán aquí deben corresponder a esa ejecución.

![Salida de tox run -e test_app con las pruebas superadas](imgs/t6-03-tox-test-app.png)

*Figura 2.12 — Ejecución del ambiente de pruebas de la API (numeral 2.12 — 10 pts).*

Resultado en EC2: `<COMPLETAR_SALIDA_Y_NUMERO_DE_PRUEBAS>`.

## 2.13 Ejecución de la API

```bash
tox run -e run
```

Este comando se ejecutará dentro de la VM EC2. El ambiente `run` ejecutará
`python app/main.py` y dejará el servidor Uvicorn escuchando en el puerto `8001`.

![Salida de tox run -e run y servidor Uvicorn en ejecución](imgs/t6-04-tox-run.png)

*Figura 2.13 — Ejecución del ambiente de ejecución de la API (numeral 2.13 — 10 pts).*

## 2.14 Apertura del puerto 8001

En el grupo de seguridad asociado a la instancia EC2 se agregará una regla de entrada con
las siguientes características:

| Campo | Valor |
|---|---|
| Tipo | TCP personalizado |
| Puerto | `8001` |
| Origen | Anywhere IPv4 (`0.0.0.0/0`) |

Esta regla permitirá acceder al servidor desde un navegador externo.

## 2.15 – 2.16 Acceso y prueba desde el navegador

Una vez iniciado el servidor en la VM y abierto el puerto, la aplicación se abrirá en:

```text
http://<IP_PUBLICA>:8001
```

La documentación interactiva se consultó en:

```text
http://<IP_PUBLICA>:8001/docs
```

Desde Swagger UI se ejecutará primero `/api/v1/health` y posteriormente
`/api/v1/predict`, utilizando `Try it out` y `Execute`.

![Ejecución exitosa de una predicción desde la documentación de FastAPI](imgs/t6-05-prediction-success.png)

*Figura 2.16 — Respuesta satisfactoria de la ruta de predicción (numeral 2.16 — 10 pts).*

La respuesta exitosa que se documentará será:

```json
{
  "errors": null,
  "version": "<COMPLETAR_VERSION_MODELO>",
  "predictions": [<COMPLETAR_PREDICCION>]
}
```

Al finalizar la prueba se detendrá el servidor con `Ctrl+C`.

# 3. Modificación de la API para el modelo 0.0.2 (Parte 3)

En el Taller 5 se modificó el modelo para eliminar la característica `Customer_Age` y se
generó el paquete que se incorporará a la API:

```text
model_abandono-0.0.2-py3-none-any.whl
```

## 3.1 Incorporación del nuevo paquete

Se copiará el archivo `.whl` a `model-pkg/` y se retirará o reemplazará la versión
anterior. La carpeta deberá quedar así:

```text
model-pkg/
└── model_abandono-0.0.2-py3-none-any.whl
```

## 3.2 Actualización de `requirements.txt`

Se actualizará la dependencia local para que la API utilice la versión `0.0.2` del
modelo. La línea final deberá quedar:

```text
./model-pkg/model_abandono-0.0.2-py3-none-any.whl
```

La actualización se publicará en GitHub:

```bash
git add .
git commit -m "Actualizar API al modelo 0.0.2"
git push
```

## 3.3 Actualización en la máquina virtual

Dentro de la VM EC2 se actualizará el repositorio:

```bash
git pull
```

Se verificará que el repositorio contenga el nuevo archivo `.whl` y que
`requirements.txt` referencie la versión `0.0.2`.

## 3.4 Repetición de las pruebas con tox

Esta ejecución se realizará dentro de la VM EC2:

```bash
tox run -e test_app
```

![Pruebas de la API después de actualizar el modelo a 0.0.2](imgs/t6-06-tox-test-v020.png)

*Figura 3.4 — Pruebas posteriores a la actualización del paquete (numeral 3.6 — evidencia complementaria).*

Resultado en EC2: `<COMPLETAR_SALIDA_Y_NUMERO_DE_PRUEBAS>`.

## 3.5 Repetición de la ejecución y prueba en el navegador

```bash
tox run -e run
```

Se volverá a acceder a la documentación mediante:

```text
http://<IP_PUBLICA>:8001/docs
```

La ruta `/api/v1/health` permitirá verificar que la versión del modelo reportada sea
`0.0.2`.

## 3.6 Predicción omitiendo una variable eliminada

Se eliminará `Customer_Age` del objeto de entrada, porque esta característica ya no forma
parte del modelo `0.0.2`.

![Predicción sin la variable Customer_Age](imgs/t6-07-prediction-without-removed-feature.png)

*Figura 3.6 — Resultado al eliminar una variable que ya no utiliza el modelo (numeral 3.8 — 10 pts).*

Resultado observado en EC2: `<COMPLETAR_RESULTADO>`.

## 3.7 Predicción omitiendo una variable todavía requerida

Se eliminará la variable `<COMPLETAR_VARIABLE_NO_ELIMINADA>` del objeto de entrada. Esta
variable sí continúa siendo necesaria para el modelo `0.0.2`.

![Predicción sin una variable todavía requerida](imgs/t6-08-prediction-without-required-feature.png)

*Figura 3.7 — Resultado al eliminar una variable que aún requiere el modelo (numeral 3.9 — 10 pts).*

Resultado observado en EC2: `<COMPLETAR_RESULTADO>`.

# 4. Conclusiones

El taller permitió comprobar el ciclo básico de exposición de un modelo de machine
learning mediante una API. La organización en esquemas Pydantic separa la validación de
entradas y salidas, mientras que FastAPI publica automáticamente una interfaz de
interacción en `/docs`.

La configuración de tox permitió reproducir tanto las pruebas como la ejecución del
servidor en ambientes controlados. El despliegue en EC2 agregó los pasos necesarios para
instalar dependencias, configurar un ambiente virtual, publicar el puerto `8001` y
acceder a la API desde fuera de la máquina.

Finalmente, la actualización del paquete a `0.0.2` permitirá observar la relación entre
el contrato de entrada de la API y las características esperadas por el modelo. Se
comparará el resultado de omitir una variable eliminada del paquete con el resultado de
omitir una variable que todavía es requerida.
