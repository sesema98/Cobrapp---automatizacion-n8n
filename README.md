# CobrApp - Automatizacion de pagos con n8n + Python

Proyecto practico para registrar automaticamente pagos enviados como capturas de Yape o Plin desde un grupo de Telegram.

## Arquitectura

```text
Usuario envia captura
        |
        v
Grupo de Telegram
        |
        v
n8n recibe el mensaje y descarga la imagen
        |
        v
FastAPI procesa OCR con Python
        |
        v
Excel guarda el pago en pagos/pagos.xlsx
        |
        v
n8n confirma el registro en Telegram
```

## Tecnologias

- n8n como orquestador del flujo.
- Telegram Bot API para recibir capturas y enviar respuestas.
- FastAPI para exponer la API OCR.
- OpenCV + pytesseract para procesar imagenes.
- openpyxl para registrar pagos en Excel.
- Docker Compose para ejecutar todo localmente.

## Estructura

```text
.
├── docker-compose.yml
├── README.md
├── .env.example
├── capturas/
├── n8n/
│   └── flujo-cobrapp-telegram.json
├── pagos/
│   └── pagos.xlsx
└── python-api/
    ├── Dockerfile
    ├── main.py
    └── requirements.txt
```

## Ejecutar con Docker

Desde la carpeta del proyecto:

```bash
docker compose up --build
```

Servicios:

- API Python: <http://localhost:8000>
- Documentacion Swagger: <http://localhost:8000/docs>
- n8n: <http://localhost:5678>

Credenciales n8n configuradas en `docker-compose.yml`:

```text
usuario: admin
clave: admin123
```

## Endpoints de la API

### POST /procesar-imagen

Recibe una imagen como `multipart/form-data`, la procesa con OCR y registra el pago si es valido.

Ejemplo con curl:

```bash
curl -X POST http://localhost:8000/procesar-imagen \
  -F "file=@capturas/pago-yape.png"
```

Respuesta esperada:

```json
{
  "valido": true,
  "fecha": "01/05/2026",
  "hora": "18:35",
  "nombre": "Juan Perez",
  "monto": "50.00",
  "tipo": "Yape",
  "operacion": "123456789",
  "estado": "Registrado"
}
```

### POST /procesar-imagen-base64

Recibe una imagen en base64.

```json
{
  "imagen_base64": "data:image/png;base64,..."
}
```

### GET /pagos

Lista todos los pagos registrados en el Excel.

```bash
curl http://localhost:8000/pagos
```

### GET /reporte

Devuelve el total recaudado del dia.

```bash
curl http://localhost:8000/reporte
```

## Configurar Telegram

1. Abrir Telegram y buscar `@BotFather`.
2. Ejecutar `/newbot`.
3. Copiar el token generado.
4. Agregar el bot al grupo del negocio.
5. En n8n, crear una credencial de Telegram usando ese token.
6. Importar el archivo `n8n/flujo-cobrapp-telegram.json`.
7. Reemplazar las credenciales `REEMPLAZAR` por la credencial real de Telegram.
8. Para el reporte diario, reemplazar `REEMPLAZAR_CHAT_ID` por el ID del grupo.

## Flujo n8n

El archivo exportado esta en:

```text
n8n/flujo-cobrapp-telegram.json
```

El flujo contiene:

- `Telegram Trigger`: recibe mensajes del grupo.
- `IF - Tiene imagen`: valida si el mensaje trae foto.
- `Telegram - Descargar imagen`: descarga la captura.
- `HTTP - Procesar OCR`: envia la imagen a FastAPI.
- `IF - Pago valido`: valida la respuesta.
- `Telegram - Confirmar pago`: confirma el registro.
- `Schedule - Reporte diario`: ejecuta el reporte a las 11:59 PM.
- `Telegram - Enviar reporte`: envia el total recaudado al grupo.

## Excel generado

La API crea automaticamente:

```text
pagos/pagos.xlsx
```

Columnas:

```text
Fecha | Hora | Nombre | Monto | Tipo | N Operacion | Estado | Registrado por | Texto OCR
```

El campo `Estado` puede ser:

- `Registrado`: pago nuevo.
- `Duplicado`: ya existia el mismo numero de operacion.
- `No registrado`: la imagen no tenia datos suficientes.

## Criterios de validacion del pago

La API considera valido un pago cuando encuentra:

- Tipo: Yape o Plin.
- Monto.
- Numero de operacion.

Si falta alguno, responde `valido: false` y n8n envia un aviso al grupo.

## Pruebas recomendadas

1. Levantar los contenedores con `docker compose up --build`.
2. Abrir <http://localhost:8000/docs>.
3. Probar `POST /procesar-imagen` con una captura real.
4. Verificar que se cree `pagos/pagos.xlsx`.
5. Importar el flujo en n8n.
6. Enviar una captura al grupo de Telegram.
7. Confirmar que el bot responda y que el Excel tenga la nueva fila.
8. Probar `GET /reporte`.

## Notas

- Para una demo academica, Telegram es mas simple que WhatsApp.
- Si el OCR no detecta bien el texto, usar capturas nitidas y sin recortes.
- Si se ejecuta sin Docker, se debe instalar Tesseract OCR manualmente en el sistema.
