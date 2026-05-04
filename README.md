# CobrApp - Automatizacion de pagos con n8n + Python

CobrApp es un sistema de automatizacion para registrar pagos enviados como capturas de Yape o Plin desde un grupo de Telegram.

El flujo recibe una imagen, la procesa con OCR usando Python, extrae los datos principales del comprobante, registra el pago en un archivo Excel y responde automaticamente al grupo de Telegram.

## Arquitectura

```text
Usuario envia captura Yape/Plin
        |
        v
Grupo de Telegram
        |
        v
n8n recibe el mensaje
        |
        v
n8n descarga la imagen desde Telegram
        |
        v
FastAPI procesa la imagen con OCR
        |
        v
Python extrae monto, nombre, fecha, tipo y operacion
        |
        v
Excel registra el pago en pagos/pagos.xlsx
        |
        v
n8n confirma el registro en Telegram
```

## Tecnologias

- **Telegram Bot API**: recibe capturas y envia respuestas.
- **n8n**: orquesta el flujo de automatizacion.
- **FastAPI**: API REST para procesar imagenes.
- **pytesseract**: OCR para extraer texto.
- **OpenCV + Pillow**: preprocesamiento de imagenes.
- **openpyxl**: escritura y lectura del archivo Excel.
- **Docker Compose**: ejecucion local de n8n y la API.
- **Cloudflare Tunnel**: URL HTTPS temporal para que Telegram pueda enviar webhooks a n8n local.

## Estructura del proyecto

```text
.
├── .env.example
├── .gitignore
├── README.md
├── diagrama-flujo-cobrapp.html
├── docker-compose.yml
├── capturas/
│   └── .gitkeep
├── n8n/
│   └── flujo-cobrapp-telegram.json
├── pagos/
│   └── .gitkeep
└── python-api/
    ├── Dockerfile
    ├── main.py
    └── requirements.txt
```

## Requisitos para replicar

Para ejecutar el proyecto en otra computadora se necesita:

- Docker Desktop instalado y abierto.
- Git instalado.
- Cuenta de Telegram.
- Bot creado con `@BotFather`.
- Acceso a internet.

No es necesario instalar Python localmente, porque la API corre dentro de Docker.

## Clonar el proyecto

```bash
git clone https://github.com/sesema98/Cobrapp---automatizacion-n8n.git
cd Cobrapp---automatizacion-n8n
```

## Configurar variables de entorno

Copiar el archivo de ejemplo:

```bash
cp .env.example .env
```

En Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

El archivo `.env.example` contiene:

```text
TELEGRAM_BOT_TOKEN=coloca_aqui_el_token_de_botfather
N8N_USER=admin
N8N_PASSWORD=admin123
PYTHON_API_URL=http://python-api:8000
```

El token de Telegram se configura directamente como credencial dentro de n8n. El `.env` queda como referencia para documentar credenciales y URLs.

## Levantar n8n y la API

Desde la carpeta del proyecto:

```bash
docker compose up -d --build
```

Verificar contenedores:

```bash
docker ps --filter "name=cobrapp"
```

Servicios locales:

```text
API FastAPI: http://localhost:8000
Swagger API: http://localhost:8000/docs
n8n local:   http://localhost:5678
```

Credenciales locales de n8n:

```text
usuario: admin
clave: admin123
```

## Crear el bot de Telegram

1. Abrir Telegram.
2. Buscar `@BotFather`.
3. Ejecutar:

```text
/newbot
```

4. Asignar nombre y usuario al bot.
5. Copiar el token generado.
6. Crear un grupo de Telegram.
7. Agregar el bot al grupo.

Para que el bot pueda leer mensajes del grupo:

1. Abrir `@BotFather`.
2. Ejecutar:

```text
/setprivacy
```

3. Seleccionar el bot.
4. Elegir `Disable`.
5. Sacar y volver a agregar el bot al grupo si no escucha mensajes.
6. Opcionalmente, hacerlo administrador del grupo.

## Crear URL HTTPS para Telegram

Telegram exige webhooks con HTTPS. Como n8n corre localmente, se usa Cloudflare Tunnel.

Ejecutar:

```bash
docker run -d --name cobrapp-cloudflared \
  --network trabajo1-automatizacion-serva_default \
  cloudflare/cloudflared:latest tunnel \
  --no-autoupdate \
  --url http://cobrapp-n8n:5678
```

Ver la URL generada:

```bash
docker logs cobrapp-cloudflared --tail 80
```

Se vera una URL parecida a:

```text
https://ejemplo-temporal.trycloudflare.com
```

Configurar esa URL en `docker-compose.yml`:

```yaml
WEBHOOK_URL=https://ejemplo-temporal.trycloudflare.com/
N8N_EDITOR_BASE_URL=https://ejemplo-temporal.trycloudflare.com/
```

Reiniciar n8n:

```bash
docker compose restart n8n
```

Luego abrir n8n usando la URL HTTPS generada.

> Nota: la URL de Cloudflare Tunnel es temporal. Si se reinicia el tunel, puede cambiar y se debe actualizar nuevamente.

## Configurar credencial de Telegram en n8n

1. Entrar a n8n.
2. Ir a **Credentials**.
3. Crear una credencial **Telegram API**.
4. Pegar el token del bot.
5. Guardar.

## Importar el flujo n8n

En n8n:

1. Ir a **Workflows**.
2. Importar el archivo:

```text
n8n/flujo-cobrapp-telegram.json
```

3. En cada nodo de Telegram, seleccionar la credencial creada:

```text
Telegram Trigger
Telegram - Descargar imagen
Telegram - Confirmar pago
Telegram - Imagen invalida
Telegram - Enviar reporte
```

4. En el nodo `Telegram - Enviar reporte`, configurar el `Chat ID` del grupo.

El `Chat ID` se puede obtener ejecutando el trigger y revisando:

```text
message.chat.id
```

## Flujo n8n

El flujo contiene:

- `Telegram Trigger`: recibe mensajes del grupo.
- `IF - Tiene imagen`: valida si el mensaje contiene foto o imagen como documento.
- `Telegram - Descargar imagen`: descarga la imagen desde Telegram.
- `HTTP - Procesar OCR`: envia la imagen a FastAPI.
- `IF - Pago valido`: valida la respuesta de la API.
- `Telegram - Confirmar pago`: confirma el pago al grupo.
- `Telegram - Imagen invalida`: avisa si no se pudo registrar.
- `Schedule - Reporte diario`: ejecuta el reporte diario.
- `HTTP - Obtener reporte`: consulta el total recaudado.
- `Telegram - Enviar reporte`: envia el resumen al grupo.

## Endpoints de la API

### GET /

Verifica que la API este activa.

```bash
curl http://localhost:8000/
```

### POST /procesar-imagen

Recibe una imagen como `multipart/form-data`, aplica OCR y registra el pago si es valido.

```bash
curl -X POST http://localhost:8000/procesar-imagen \
  -F "file=@capturas/pago-yape.png"
```

Respuesta esperada:

```json
{
  "valido": true,
  "fecha": "03/05/2026",
  "hora": "20:46",
  "nombre": "Gabriela Soto",
  "monto": "10",
  "tipo": "Yape",
  "operacion": "24814930",
  "estado": "Registrado"
}
```

### POST /procesar-imagen-base64

Recibe una imagen codificada en base64.

```json
{
  "imagen_base64": "data:image/png;base64,..."
}
```

### GET /pagos

Lista los pagos guardados en Excel.

```bash
curl http://localhost:8000/pagos
```

### GET /reporte

Devuelve cantidad de pagos y total recaudado del dia.

```bash
curl http://localhost:8000/reporte
```

## Excel generado

La API crea automaticamente:

```text
pagos/pagos.xlsx
```

Columnas:

```text
Fecha | Hora | Nombre | Monto | Tipo | N Operacion | Estado | Registrado por | Texto OCR
```

Estados posibles:

- `Registrado`: pago nuevo.
- `Duplicado`: el numero de operacion ya existe.
- `No registrado`: faltan datos o la imagen no corresponde a un pago.
- `Excel bloqueado`: el archivo esta abierto y Windows no permite escribir.

> Importante: cerrar `pagos.xlsx` antes de probar. Si esta abierto, la API no puede guardar cambios.

## Criterios de validacion del pago

La API considera valido un pago cuando detecta:

- Tipo de pago: Yape o Plin.
- Monto.
- Numero de operacion.

Adicionalmente intenta extraer:

- Nombre del pagador.
- Fecha.
- Hora.

## Probar el sistema completo

1. Levantar Docker:

```bash
docker compose up -d --build
```

2. Crear el tunel HTTPS con Cloudflare.
3. Actualizar `WEBHOOK_URL` y reiniciar n8n.
4. Abrir n8n con la URL HTTPS.
5. Importar el flujo.
6. Configurar la credencial de Telegram.
7. Ejecutar el workflow o publicarlo.
8. Enviar una captura Yape/Plin al grupo.
9. Verificar que el bot responda.
10. Revisar `pagos/pagos.xlsx`.
11. Probar el reporte diario.

## Probar el reporte diario manualmente

En n8n, ejecutar en orden:

```text
Schedule - Reporte diario
HTTP - Obtener reporte
Telegram - Enviar reporte
```

Tambien puede probarse desde navegador:

```text
http://localhost:8000/reporte
```

## Diagrama del flujo

Abrir:

```text
diagrama-flujo-cobrapp.html
```

Este archivo muestra graficamente:

- Telegram.
- n8n.
- FastAPI.
- OCR.
- Excel.
- Confirmacion.
- Reporte diario.

## Comandos utiles

Ver contenedores:

```bash
docker ps --filter "name=cobrapp"
```

Ver logs de la API:

```bash
docker logs cobrapp-python-api --tail 100
```

Ver logs de n8n:

```bash
docker logs cobrapp-n8n --tail 100
```

Ver logs del tunel:

```bash
docker logs cobrapp-cloudflared --tail 80
```

Reiniciar n8n:

```bash
docker compose restart n8n
```

Detener el proyecto:

```bash
docker compose down
docker rm -f cobrapp-cloudflared
```

## Notas tecnicas

- Telegram no acepta webhooks `http://localhost`, por eso se usa Cloudflare Tunnel.
- La URL `trycloudflare.com` es temporal.
- n8n no permite ejecutar al mismo tiempo un `Telegram Trigger` en modo prueba y publicado.
- Si el bot no escucha mensajes del grupo, revisar `/setprivacy` en BotFather.
- Si el OCR falla, usar capturas nitidas, sin recortes extremos y con monto/operacion visibles.
- Si Excel esta abierto, cerrar el archivo antes de registrar pagos.
