# Resumen Week 2 — Tu Digital Twin en AWS

## El arco de la semana

Construiste un chatbot que te representa a vos. Arrancó sin memoria (amnésico). Al final de la semana está en producción global en AWS, sin depender de proveedores externos, y con toda la infraestructura automatizada como código.

```
Day 1: chatbot local sin memoria → chatbot local CON memoria (archivos)
Day 2: memoria en archivos → memoria en S3 + Lambda + API Gateway + CloudFront
Day 3: OpenAI (externo) → AWS Bedrock (todo dentro de AWS)
Day 4: clicks manuales en consola → infraestructura como código con Terraform
```

---

## Lo que construiste

Un Digital Twin: un chatbot que actúa como vos cuando alguien visita tu web profesional. Le pasás tu LinkedIn, un resumen tuyo, tu estilo de comunicación — y el modelo responde como si fuera vos.

---

## Conceptos clave con analogías

### El problema de la memoria — la amnesia del modelo

Por defecto, cada llamada a OpenAI es independiente. No hay "estado" entre llamadas. Si no mandás el historial explícitamente, el modelo no sabe que hubo una conversación antes.

**Analogía**: es como hablar por teléfono con alguien que tiene amnesia total. Cada vez que llamas, no te recuerda — ni tu nombre, ni de qué hablaron.

**La solución**: antes de cada llamada a OpenAI, cargás el historial de la conversación y lo incluís en el array de mensajes:

```
Sin memoria:  [system, mensaje_nuevo]
Con memoria:  [system, msg_1, resp_1, msg_2, resp_2, ..., mensaje_nuevo]
```

---

### Memoria basada en archivos — Day 1

Cada conversación se guarda en un archivo JSON en la carpeta `memory/`. El nombre del archivo es el `session_id`.

```
memory/
├── abc123-def456.json   ← conversación de Leo
├── xyz789-ghi012.json   ← conversación de otro usuario
```

**Contenido de un archivo JSON:**
```json
[
  {"role": "user", "content": "Hola, me llamo María"},
  {"role": "assistant", "content": "Hola María, soy el twin de Leo..."},
  {"role": "user", "content": "¿Qué hace Leo?"},
  {"role": "assistant", "content": "Leo es AI Engineer..."}
]
```

**Analogía**: es como un cuaderno donde anotás todo lo que se habló. Antes de cada nueva respuesta, relés el cuaderno entero.

**Problema a escala**: si una conversación tiene 100 mensajes, mandás los 100 en cada llamada. Cada token cuesta dinero y hay un límite máximo. La solución del Day 2: `conversation[-10:]` — solo los últimos 10 mensajes.

---

### Gestión de sesiones — session_id

¿Cómo sabe el backend a cuál de los miles de archivos JSON cargar?

```
Primera vez:
  - Frontend no manda session_id
  - Backend genera uno nuevo: str(uuid.uuid4()) → "a3f2-b1c4-..."
  - Backend devuelve ese ID al frontend
  - Frontend lo guarda en React state

Próximas veces:
  - Frontend manda el mismo session_id en cada request
  - Backend carga el archivo "a3f2-b1c4-....json"
  - Contexto recuperado
```

**Limitación**: el session_id vive en el navegador (React state). Si cerrás la pestaña, se pierde. Para reconocer al mismo usuario entre sesiones distintas necesitarías login + base de datos.

---

### AWS Lambda — el mozo que aparece cuando lo llamás

Lambda es FaaS (Function as a Service). Tu código no corre en un servidor permanente — se ejecuta cuando llega un request y se apaga cuando termina. No pagás por tiempo inactivo.

**Analogía**: es como un mozo que aparece cuando lo llamás, toma el pedido, lo entrega, y desaparece. No está parado todo el día esperando. Solo trabajás cuando hay trabajo.

**Comparación con ECS Fargate (Week 1):**
```
ECS Fargate:  servidor siempre prendido → pagás por hora, incluso sin requests
Lambda:       se enciende con cada request → pagás solo por lo que usás
```

**Cuándo usar Lambda**: tráfico variable (a veces muchos requests, a veces cero). Ideal para APIs.

**Cuándo NO usar Lambda**: procesos que necesitan correr constantemente (como N8N, que necesita un servidor siempre vivo escuchando webhooks).

---

### Mangum — el traductor entre Lambda y FastAPI

Lambda espera una función Python con esta firma:
```python
def handler(event, context):
    ...
```

FastAPI es un servidor web completo con rutas, middlewares, etc. No tiene esa firma. Mangum traduce.

```python
# Sin Mangum: FastAPI no puede correr en Lambda
# Con Mangum: una línea y listo
from mangum import Mangum
from server import app

handler = Mangum(app)  # Mangum envuelve FastAPI y expone el handler que Lambda espera
```

**Analogía**: Mangum es como un adaptador de enchufe. FastAPI es un enchufe europeo. Lambda es un tomacorriente americano. Mangum hace que encajen.

---

### API Gateway — la recepción del edificio

API Gateway es el punto de entrada público de tu API. Recibe los requests HTTP de internet y los redirige a Lambda.

**Analogía**: es la recepcionista de un edificio corporativo. Vos llegás desde la calle, ella te manda al piso y oficina correctos. También maneja quién puede entrar (CORS, autenticación, rate limiting).

**Flujo de un request:**
```
Browser → URL de API Gateway → API Gateway → Lambda → FastAPI → OpenAI
                                                              → respuesta vuelve por el mismo camino
```

**CORS en dos lugares**: configurás CORS tanto en API Gateway como en FastAPI con `CORSMiddleware`. Si falta en cualquiera de los dos, el navegador bloquea los requests. Dos capas, las dos necesarias.

---

### S3 — el almacenamiento con dos roles distintos

En Week 2, S3 se usa para dos cosas completamente distintas:

```
twin-memory-xxx/          ← bucket de memoria
  abc123.json             ← historial de conversación de una sesión
  xyz789.json             ← historial de otra sesión

twin-frontend-xxx/        ← bucket del frontend
  index.html
  _next/static/...        ← archivos JS/CSS compilados de Next.js
```

**Analogía**: el primer bucket es un archivo físico donde guardás documentos (conversaciones). El segundo es una estantería pública donde exponés folletos (la web).

**El switch `USE_S3`**: una variable de entorno que permite usar el mismo código en desarrollo (archivos locales) y producción (S3):
```python
USE_S3 = os.getenv("USE_S3", "false").lower() == "true"
# false → guarda en carpeta memory/ local (igual que Day 1)
# true  → guarda en el bucket S3 (producción en AWS)
```

---

### CloudFront — la red de distribución global

CloudFront es una CDN (Content Delivery Network). Tiene servidores en todo el mundo. Cuando alguien en Japón abre tu web, CloudFront sirve el contenido desde el nodo más cercano a Japón en lugar de viajar hasta tu bucket S3 en us-east-1.

**Analogía**: es como tener sucursales en todo el mundo. En vez de que todos los clientes viajes a la casa central (S3), hay una sucursal cerca de cada uno.

**Beneficios concretos**:
- HTTPS automático (S3 solo da HTTP por defecto)
- Carga más rápida (contenido cacheado cerca del usuario)
- Menos carga en S3

---

### boto3 — el control remoto de AWS desde Python

boto3 es el SDK oficial de AWS para Python. Con boto3 podés hacer desde código todo lo que hacés en la consola de AWS.

```python
import boto3

s3_client = boto3.client("s3")

# Leer un archivo de S3
response = s3_client.get_object(Bucket="mi-bucket", Key="conversacion.json")
data = json.loads(response["Body"].read())

# Escribir un archivo en S3
s3_client.put_object(Bucket="mi-bucket", Key="conversacion.json", Body=json.dumps(data))
```

**Analogía**: boto3 es como el mando a distancia del ecosistema de AWS. En lugar de hacer click en la consola, ejecutás comandos en Python.

---

### Por qué Docker para empaquetar Lambda

Lambda corre en Linux/AMD64. Si instalás dependencias Python en Windows o Mac, algunos paquetes tienen binarios compilados para tu sistema operativo — que no funcionan en Linux.

La solución: usar Docker con la imagen oficial de Lambda para instalar las dependencias:

```bash
docker run --platform linux/amd64 public.ecr.aws/lambda/python:3.13 \
  pip install -r requirements.txt --target /var/task/lambda-package
```

**Analogía**: es como empaquetar comida para un viaje usando las mismas condiciones del destino. No cocinás la comida en tu cocina y esperás que llegue bien — usás las condiciones del lugar donde se va a consumir.

---

## App Router vs Pages Router (Next.js)

Week 1 usó Pages Router. Week 2 usó App Router. La diferencia práctica:

```
Pages Router (Week 1):        App Router (Week 2):
pages/                        app/
  index.tsx → /                 page.tsx → /
  about.tsx → /about            about/
                                  page.tsx → /about
```

App Router es más moderno. Los componentes son Server Components por defecto (renderizan en el servidor). Para componentes que usan estado o eventos del navegador, se necesita `'use client'` al principio del archivo.

---

## Diagrama de arquitectura completo

### Day 1 — Local

```
Browser
  │
  ├── GET localhost:3000 → Next.js (frontend)
  │                         └── componente twin.tsx
  │
  └── POST localhost:8000/chat → FastAPI (backend)
                                   │
                                   ├── load_conversation()  ←→  memory/abc123.json
                                   ├── llamada a OpenAI
                                   └── save_conversation()  →   memory/abc123.json
```

### Day 2 — Producción en AWS

```
Developer
  │
  ├── uv run deploy.py  →  lambda-deployment.zip
  │                              │
  │                              └── sube a Lambda
  │
  └── npm run build → out/ → aws s3 sync → S3 (frontend)


Usuario
  │  HTTPS
  ▼
CloudFront (CDN global)
  │
  ├── /  →  S3 (frontend estático: HTML, CSS, JS de Next.js)
  │
  └── /chat  →  API Gateway
                     │
                     ▼
                  Lambda (FastAPI + Mangum)
                     │
                     ├── OpenAI API  (genera la respuesta)
                     └── S3 memory  (guarda/carga historial JSON)
```

---

## El contexto enriquecido del Twin

En Day 2, el sistema prompt que se le manda a OpenAI ya no es solo `me.txt`. Es un contexto construido desde cuatro fuentes:

```
facts.json     → datos estructurados (nombre, rol, skills, educación)
summary.txt    → párrafo de presentación personal
style.txt      → cómo habla, qué tono usa
linkedin.pdf   → experiencia laboral completa extraída del PDF

Todo junto → context.py → prompt() → sistema prompt del modelo
```

---

## Comparación Day 1 vs Day 2

| | Day 1 (local) | Day 2 (AWS) |
|---|---|---|
| Dónde corre el backend | tu máquina | Lambda |
| Dónde se guarda la memoria | archivos locales | S3 |
| Dónde está el frontend | localhost:3000 | S3 + CloudFront |
| HTTPS | No | Sí (CloudFront) |
| URL pública | No | Sí |
| Costo | $0 | ~$0-5/mes |
| Historia mandada a OpenAI | Completa | Solo últimos 10 mensajes |

---

## Gotchas que te van a salvar tiempo

1. **CORS en dos lugares** — en API Gateway Y en FastAPI. Falta uno → el navegador bloquea
2. **`conversation[-10:]`** — no mandás todo el historial, solo los últimos 10 mensajes. El historial completo igual se guarda en S3
3. **Lambda timeout default = 3 segundos** — OpenAI puede tardar más. Ajustarlo a 30 segundos
4. **Handler correcto**: `lambda_handler.handler`, no `server.app` ni `handler.handler`
5. **USE_S3 en mayúsculas**: la variable de entorno se lee con `.lower() == "true"`. Poner `true` minúscula o `True` — ambos funcionan con ese código
6. **CloudFront cachea el frontend** — si subís cambios a S3 y no ves nada nuevo, crear una invalidación `/*` en CloudFront
7. **El session_id se pierde al cerrar la pestaña** — esto es por diseño en esta implementación. Sin login no hay persistencia entre sesiones
8. **Política de origen en CloudFront: Solo HTTP** — el bucket S3 web estático corre en HTTP, no HTTPS. CloudFront recibe HTTPS del usuario y hace HTTP al origen S3. Si ponés HTTPS-only hacia el origen → error

---

## Day 3 — De OpenAI a AWS Bedrock

### ¿Qué es AWS Bedrock?

Es el servicio de IA de AWS. En lugar de llamar a la API de OpenAI (servidor externo), llamás a modelos de IA que corren dentro de tu propia infraestructura de AWS.

**Analogía**: antes pedías comida a un delivery externo (OpenAI). Ahora tenés cocina propia dentro del edificio (Bedrock). La comida puede ser similar, pero todo queda adentro — más rápido, más seguro, una sola factura.

**Ventajas concretas**:
- Menor latencia (el request no sale de AWS)
- Seguridad: los datos no salen de tu cuenta
- Facturación unificada (todo en una cuenta de AWS)
- Integración nativa con IAM, VPC, CloudWatch

---

### Los modelos Nova — la familia de AWS

AWS tiene su propia familia de modelos llamada **Nova**:

```
Nova Micro  → el más rápido y barato. Bueno para Q&A simple
Nova Lite   → equilibrado, recomendado para producción (el que usás por defecto)
Nova Pro    → el más capaz, para razonamiento complejo. Más caro
```

El modelo se elige con una variable de entorno `BEDROCK_MODEL_ID`. Cambiar de modelo = cambiar una variable, sin tocar código.

---

### El cambio en el código — de OpenAI a Bedrock

Day 2 usaba el cliente de OpenAI:
```python
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(model="gpt-4o-mini", messages=messages)
text = response.choices[0].message.content
```

Day 3 usa boto3 para conectarse a Bedrock:
```python
bedrock_client = boto3.client("bedrock-runtime", region_name="us-east-1")
response = bedrock_client.converse(modelId=BEDROCK_MODEL_ID, messages=messages)
text = response["output"]["message"]["content"][0]["text"]
```

**Diferencia importante en el formato de mensajes**: OpenAI acepta strings simples en `content`. Bedrock usa una lista de objetos:

```python
# OpenAI
{"role": "user", "content": "Hola"}

# Bedrock
{"role": "user", "content": [{"text": "Hola"}]}
```

**El system prompt en Bedrock**: Bedrock no tiene un campo "system" directo en la API `converse`. La convención es pasarlo como primer mensaje de usuario con el prefijo `"System: ..."`:

```python
messages.append({
    "role": "user",
    "content": [{"text": f"System: {prompt()}"}]
})
```

---

### Permisos necesarios

Dos lugares donde hay que agregar permisos para Bedrock:

```
1. Grupo IAM TwinAccess (para el usuario aiengineer)
   → AmazonBedrockFullAccess

2. Rol de ejecución de Lambda (para que la función pueda llamar a Bedrock)
   → AmazonBedrockFullAccess
```

Sin el permiso en el rol de Lambda → `AccessDeniedException` cuando la función intenta llamar a Bedrock.

---

### Regiones y perfiles de inferencia — el gotcha más feo de Day 3

Los modelos Bedrock no están disponibles en todas las regiones. Además, existe el concepto de **inference profiles** (perfiles de inferencia): cuando usás el prefijo `us.` o `eu.` en el ID del modelo, AWS distribuye el request entre varias regiones para mejor disponibilidad.

```
Sin perfil:   amazon.nova-lite-v1:0       → solo la región donde está tu Lambda
Con perfil:   us.amazon.nova-lite-v1:0    → distribuye entre us-east-1, us-east-2, us-west-2
```

**El problema**: si usás el perfil `us.`, necesitás tener acceso concedido al modelo en **todas** las regiones del perfil (us-east-1, us-east-2, us-west-2). Si falta una → error.

**Regla práctica**: si obtenés un error de Bedrock, probá añadir el prefijo `us.` o `eu.` al ID del modelo y asegurate de haber solicitado acceso en las regiones correspondientes.

---

### CloudWatch — ver qué está pasando en producción

CloudWatch es el sistema de observabilidad de AWS. Con Bedrock integrado, podés monitorear:

```
Lambda:
  - Invocaciones: cuántas veces se llamó la función
  - Duración: cuánto tardó cada ejecución
  - Errores: cuántas fallaron

Bedrock:
  - InvocationLatency: cuánto tarda el modelo en responder
  - InputTokenCount / OutputTokenCount: cuántos tokens se consumieron (= costo)
```

**Para ver logs**: CloudWatch → Log groups → `/aws/lambda/twin-api` → el stream más reciente.

**Alerta de presupuesto**: se puede configurar en AWS Billing → Budgets para recibir un email cuando el gasto supera cierto límite. Recomendado para no tener sorpresas.

---

### Diagrama Day 3

```
Usuario
  │  HTTPS
  ▼
CloudFront (CDN)
  │
  ├── S3 (frontend estático)
  │
  └── API Gateway
           │
           ▼
        Lambda (FastAPI + Mangum)
           │
           ├── AWS Bedrock (Nova Lite/Micro/Pro)   ← reemplaza OpenAI
           └── S3 memory (historial de sesión)
```

Todo queda dentro de AWS. El único servicio externo que desaparece es OpenAI.

---

### Comparación Day 2 vs Day 3

| | Day 2 | Day 3 |
|---|---|---|
| Proveedor de IA | OpenAI (externo) | AWS Bedrock (interno) |
| Cliente Python | `openai` library | `boto3` |
| Modelo | gpt-4o-mini | amazon.nova-lite-v1:0 |
| Facturación | cuenta OpenAI separada | todo en AWS |
| Latencia | llamada a servidor externo | dentro de AWS |
| Control de costos | Cost Explorer de OpenAI | CloudWatch + AWS Cost Explorer |

---

### Gotchas Day 3

1. **Perfiles de inferencia** (`us.` / `eu.` prefijo) — si los usás, necesitás acceso en todas las regiones del perfil
2. **Formato de mensajes distinto a OpenAI** — `content` es `[{"text": "..."}]`, no un string
3. **System prompt como user message** — Bedrock `converse` no tiene campo "system" directo
4. **Dos lugares para dar permisos** — grupo IAM del usuario Y rol de ejecución de Lambda
5. **`bedrock-runtime`** — el nombre del servicio boto3 es `bedrock-runtime`, no `bedrock`
6. **Región del cliente Bedrock** — hay que especificar `region_name` explícitamente; toma el valor de `DEFAULT_AWS_REGION`

---

## Day 4 — Terraform: infraestructura como código

### Lo que hiciste

Borraste todo lo que habías creado manualmente en la consola de AWS (Lambda, API Gateway, S3, CloudFront) y lo recreaste con Terraform. Ahora toda la infraestructura vive en archivos de texto versionables, y deployar un entorno nuevo es un solo comando.

---

### El problema de hacer todo a mano

En Day 2 y Day 3 pasaste horas clickeando en la consola de AWS — crear buckets, configurar Lambda, definir rutas en API Gateway, esperar que CloudFront se propagara. Si necesitás hacer lo mismo para un entorno de test, o si un compañero quiere replicar tu setup, tenés que hacerlo todo de nuevo.

**Analogía**: hacer click en la consola de AWS es como construir una casa ladrillo a ladrillo sin planos. Terraform es tener los planos — cualquiera puede construir la misma casa exacta, en cualquier momento, en cualquier lugar.

---

### Terraform — qué es y cómo funciona

Terraform es una herramienta que toma archivos de configuración `.tf` y crea (o destruye) infraestructura real en AWS.

```
Vos escribís:     resource "aws_s3_bucket" "memory" { bucket = "twin-dev-memory-..." }
Terraform hace:   crea ese bucket en AWS por vos
```

El ciclo básico:
```
terraform init    → descarga los plugins (provider de AWS)
terraform plan    → te muestra qué va a crear/modificar/destruir (sin hacerlo)
terraform apply   → ejecuta los cambios en AWS
terraform destroy → elimina todo lo que creó
```

**El archivo de estado (`terraform.tfstate`)**: Terraform guarda en este archivo el "mapa" de qué recursos creó y cuáles son sus IDs reales en AWS. Es crítico — si lo perdés, Terraform ya no sabe qué maneja. Por eso **nunca se sube a git**.

---

### Variables y tfvars — parametrizar la infraestructura

En lugar de hardcodear valores, Terraform usa variables:

```hcl
variable "environment" {
  type = string   # puede ser "dev", "test" o "prod"
}

# El recurso usa la variable
resource "aws_lambda_function" "api" {
  function_name = "twin-${var.environment}-api"
}
```

Los valores concretos van en archivos `.tfvars`:
```hcl
# terraform.tfvars (para dev)
environment      = "dev"
bedrock_model_id = "amazon.nova-micro-v1:0"

# prod.tfvars (para producción)
environment      = "prod"
bedrock_model_id = "amazon.nova-lite-v1:0"
```

**Analogía**: las variables son como los parámetros de una función — la lógica es la misma, solo cambian los valores de entrada.

---

### Workspaces — múltiples entornos con el mismo código

Un workspace es un estado de Terraform aislado. Podés tener `dev`, `test` y `prod` corriendo al mismo tiempo, cada uno con sus propios recursos en AWS, sin que se pisen.

```
terraform workspace new dev   → crea workspace dev
terraform workspace new test  → crea workspace test

# Cada workspace tiene su propio tfstate:
terraform.tfstate.d/
  ├── dev/terraform.tfstate
  ├── test/terraform.tfstate
  └── prod/terraform.tfstate
```

Los recursos se nombran automáticamente con el prefijo del entorno:
```
dev:  twin-dev-api,  twin-dev-memory,  twin-dev-frontend
test: twin-test-api, twin-test-memory, twin-test-frontend
prod: twin-prod-api, twin-prod-memory, twin-prod-frontend
```

---

### El script de deploy — un solo comando

Day 4 creó scripts (`deploy.sh` para Mac/Linux, `deploy.ps1` para Windows) que automatizan todo el proceso:

```
./scripts/deploy.sh dev   (o .\scripts\deploy.ps1 -Environment dev)

Lo que hace en orden:
  1. uv run deploy.py         → construye el zip de Lambda con Docker
  2. terraform init           → descarga plugins
  3. terraform workspace      → selecciona o crea el workspace del entorno
  4. terraform apply          → crea/actualiza toda la infraestructura en AWS
  5. npm run build            → compila el frontend Next.js
  6. aws s3 sync              → sube el frontend al bucket S3
  7. imprime las URLs         → CloudFront URL, API Gateway URL
```

Lo que antes tomaba una hora de clicks, ahora tarda lo que tarda el apply.

---

### Lo que Terraform gestiona ahora

```
terraform/
  versions.tf      → versión de Terraform y del provider AWS
  variables.tf     → declaración de todas las variables
  main.tf          → los recursos reales (S3, Lambda, API Gateway, CloudFront, IAM)
  outputs.tf       → los valores que imprime al terminar (URLs, nombres de buckets)
  terraform.tfvars → valores por defecto (dev)
  prod.tfvars      → valores para producción (opcional)
```

Todo lo que antes configurabas a mano en la consola — permisos IAM del rol Lambda, CORS en API Gateway, política del bucket S3, certificado SSL — ahora está en `main.tf`.

---

### Diagrama Day 4

```
Antes (Days 2-3):                     Ahora (Day 4):
  AWS Console                           deploy.sh / deploy.ps1
    → crear S3 (×2)                       → uv run deploy.py
    → crear Lambda                        → terraform apply
    → crear API Gateway                       → S3 (×2)
    → crear CloudFront                        → Lambda + IAM role
    → configurar CORS                         → API Gateway + rutas
    → permisos IAM                            → CloudFront
    → ...                                 → npm build + s3 sync
  ~1 hora de clicks                     ~5 minutos, repetible infinitas veces
```

---

### Dominio personalizado (opcional)

Day 4 también incluye configuración para conectar tu propio dominio (`tudominio.com`) al twin. Requiere Route 53 para DNS, ACM para el certificado SSL y CloudFront configurado con el dominio. Solo se activa si ponés `use_custom_domain = true` en `prod.tfvars`. Para aprender, no hace falta.

---

### Gotchas Day 4

1. **`terraform.tfstate` nunca a git** — contiene IDs internos de AWS y puede tener info sensible
2. **`*.tfvars` tampoco a git** — pueden tener valores secretos. El `.gitignore` del día los excluye
3. **Destruir antes de recrear** — si borrás un recurso del `.tf` y hacés `apply`, Terraform lo destruye en AWS. Hay que tener cuidado con `terraform destroy`
4. **Vaciar S3 antes de destruir** — AWS no deja borrar un bucket con objetos adentro. Los scripts `destroy.sh/ps1` lo hacen automáticamente
5. **`terraform plan` antes de `apply`** — siempre conviene ver qué va a cambiar antes de ejecutarlo, sobre todo en prod
6. **CloudFront tarda** — crear o destruir una distribución CloudFront puede tardar 5-15 minutos

---

## Day 5 — CI/CD con GitHub Actions

### Lo que hiciste

Conectaste el repositorio de GitHub con AWS para que cada `git push` a `main` despliegue automáticamente el twin en el entorno `dev`. Crear o destruir entornos se hace desde la interfaz web de GitHub, sin tocar la terminal.

---

### CI/CD — qué significa y por qué importa

**CI** (Continuous Integration): cada vez que subís código, se ejecutan pasos automáticos (build, tests).  
**CD** (Continuous Deployment): si todo pasa, el código se despliega automáticamente a producción.

**Analogía**: es como tener un asistente que cada vez que terminás de escribir un documento, lo revisa, lo imprime y lo manda al cliente automáticamente. Vos solo escribís — el resto pasa solo.

Sin CI/CD: `escribir código → empaquetar → subir a Lambda → sincronizar S3 → invalidar CloudFront` — todo a mano cada vez.  
Con CI/CD: `git push` → todo lo anterior pasa automático en minutos.

---

### GitHub Actions — el motor del CI/CD

GitHub Actions es el sistema de automatización de GitHub. Cuando ocurre un evento (push, click en botón), ejecuta un **workflow** — una secuencia de pasos definida en un archivo `.yml`.

```
.github/workflows/
  deploy.yml   → se ejecuta en cada push a main (o manualmente)
  destroy.yml  → solo manual, con confirmación
```

Cada workflow corre en una máquina virtual de GitHub (Ubuntu), instala las herramientas necesarias (Python, Node, Terraform, uv) y ejecuta el mismo script que antes corrías en tu máquina.

---

### OIDC — autenticación sin guardar claves

El problema: GitHub Actions necesita credenciales de AWS para crear recursos. La solución naive sería guardar `AWS_ACCESS_KEY_ID` y `AWS_SECRET_ACCESS_KEY` como secrets. El problema: son claves de larga duración — si se filtran, alguien tiene acceso permanente a tu cuenta.

**OIDC** (OpenID Connect) es la solución moderna. En lugar de claves permanentes:

```
GitHub Actions arranca → le pide un token temporal a GitHub
→ ese token se presenta a AWS
→ AWS verifica que viene del repo correcto
→ AWS entrega credenciales temporales válidas solo por esa sesión
→ el job hace su trabajo y las credenciales expiran solas
```

**Analogía**: en lugar de darle una llave permanente a un repartidor, le das un código QR de un solo uso válido por 30 minutos. Si alguien roba el código QR después, ya no sirve.

---

### Estado remoto de Terraform en S3

En Day 4 el `terraform.tfstate` vivía en tu máquina. Problema: si GitHub Actions corre Terraform en una máquina virtual efímera, el estado se pierde al terminar el job.

La solución: guardar el estado en S3 con bloqueo en DynamoDB.

```
S3 bucket (twin-terraform-state-[account-id])
  └── dev/terraform.tfstate
  └── test/terraform.tfstate
  └── prod/terraform.tfstate

DynamoDB tabla (twin-terraform-locks)
  └── evita que dos deploys corran en paralelo y corrompan el estado
```

**Analogía**: el tfstate es como el plano actualizado del edificio. En Day 4 lo guardabas en tu escritorio. En Day 5 lo subís a una carpeta compartida en la nube para que cualquier máquina (incluyendo GitHub Actions) pueda leerlo y modificarlo.

---

### Los secrets de GitHub

Las credenciales que GitHub Actions necesita se guardan como secrets en el repositorio (Settings → Secrets → Actions). Nunca aparecen en los logs.

```
AWS_ROLE_ARN          → el ARN del rol IAM que GitHub puede asumir vía OIDC
DEFAULT_AWS_REGION    → us-east-1
AWS_ACCOUNT_ID        → el ID de 12 dígitos de tu cuenta AWS
```

---

### El flujo completo de un deploy automático

```
1. git push origin main
        │
        ▼
2. GitHub detecta el push → activa deploy.yml
        │
        ▼
3. GitHub Actions arranca una VM Ubuntu
        │
        ├── instala Python, Node, uv, Terraform
        ├── obtiene credenciales temporales de AWS vía OIDC
        ├── corre scripts/deploy.sh dev
        │       ├── uv run deploy.py  → lambda-deployment.zip
        │       ├── terraform apply   → crea/actualiza infra en AWS
        │       ├── npm run build     → compila Next.js
        │       └── aws s3 sync       → sube frontend a S3
        └── imprime URLs en el resumen del workflow
```

El deploy manual (test, prod) funciona igual pero se dispara desde el botón "Run workflow" en la pestaña Actions de GitHub, eligiendo el entorno.

---

### Diagrama Day 5 — arquitectura final

```
Developer
  │
  └── git push → GitHub
                    │
                    ├── deploy.yml → GitHub Actions VM
                    │                    │
                    │              OIDC token → AWS
                    │                    │
                    │              scripts/deploy.sh dev
                    │                    │
                    │         ┌──────────┴──────────┐
                    │         │    Terraform apply   │
                    │         │    S3 state bucket   │
                    │         └──────────┬──────────┘
                    │                   │
                    │         Infraestructura AWS
                    │           ├── dev
                    │           ├── test
                    │           └── prod
                    │
                    └── destroy.yml → destruye entornos desde GitHub UI
                                      (requiere escribir nombre del entorno para confirmar)

Cada entorno:
  CloudFront → S3 (frontend)
            → API Gateway → Lambda → Bedrock (Nova Micro)
                                   → S3 (memoria)
```

---

### Comparación Day 4 vs Day 5

| | Day 4 | Day 5 |
|---|---|---|
| Quién corre el deploy | vos, desde tu terminal | GitHub Actions, automático |
| Estado de Terraform | archivo local | S3 + DynamoDB |
| Credenciales AWS | configuradas en tu máquina | OIDC temporal, sin claves guardadas |
| Deploy en dev | manual (`.\scripts\deploy.ps1`) | automático al hacer push |
| Deploy en test/prod | manual | manual desde GitHub UI |
| Destruir entornos | scripts locales | desde GitHub UI con confirmación |

---

### Gotchas Day 5

1. **OIDC puede ya existir** — si ya existe el proveedor OIDC en tu cuenta AWS, hay que importarlo con `terraform import` antes de crear el rol, si no Terraform falla
2. **`terraform_wrapper: false`** — en el workflow yml, esta opción es necesaria para que los outputs de Terraform sean strings puros y no se rompan los scripts
3. **El nombre del repo en OIDC debe ser exacto** — formato `usuario/repositorio`, sin URL, sin `.git`
4. **DynamoDB para bloqueo** — si dos workflows corren en paralelo y ambos intentan modificar el estado, DynamoDB evita que se corrompan; si un job muere a mitad, puede quedar un lock colgado — se desbloquea con `terraform force-unlock LOCK_ID`
5. **`backend-setup.tf` y `github-oidc.tf` son temporales** — se crean, se aplican, y se borran. No deben quedar en el repo final
6. **El bucket de estado no se destruye con los entornos** — es global, persiste entre deploys. Costo: ~$0.02/mes
