# Resumen Week 1 — De cero a SaaS en AWS

## El arco de la semana

Empezaste con una función Python de 5 líneas en Vercel. Terminaste con una app médica full-stack containerizada corriendo en AWS.

```
Day 1:   FastAPI básica → Vercel (deploy en 5 minutos)
Day 1.2: Agregar OpenAI → primera respuesta de IA en producción
Day 2:   App full-stack (Next.js + FastAPI) → Vercel con streaming
Day 3:   Auth con Clerk (login, JWT) + suscripciones de pago
Day 4:   Aplicación médica real → todo lo anterior integrado
Day 5:   Docker + ECR + ECS Fargate → producción en AWS
```

---

## Lo que construiste

Un asistente de consultas médicas full-stack con:
- **Frontend**: Next.js (React + TypeScript + Tailwind)
- **Backend**: FastAPI (Python) con streaming de respuestas
- **Auth**: Clerk (login con Google/GitHub, JWT, suscripciones)
- **IA**: OpenAI con streaming en tiempo real
- **Deploy**: ECS Fargate en AWS (con Docker, ECR, IAM)

---

## Day 1 — Tu primera API en producción (Vercel)

### Lo que hiciste

Desplegaste una función FastAPI mínima en Vercel. Tres archivos: `instant.py`, `requirements.txt`, `vercel.json`.

```python
@app.get("/")
def instant():
    return "Live from production!"
```

### Vercel — la lavandería automática

Vercel es como llevar ropa a una lavandería automática. Dejás el código, sale funcionando en producción con HTTPS. No configurás servidores, no gestionás nada.

**Por qué importa**: es el primer salto mental del curso — código local → código accesible desde cualquier parte del mundo.

### Day 1 Parte 2: agregar OpenAI

Mismo deploy, pero ahora el endpoint llama a la API de OpenAI y devuelve una respuesta generada por IA. Primer contacto con:
- `OPENAI_API_KEY` como variable de entorno (nunca en el código)
- `client.chat.completions.create()` — la llamada básica al modelo

```
Browser → Vercel → FastAPI → OpenAI API → respuesta → Vercel → Browser
```

---

## Day 2 — App full-stack con streaming

### Lo que hiciste

Pasaste de una sola función a una app completa:
- Frontend en Next.js (Pages Router) con React + TypeScript + Tailwind
- Backend en FastAPI con un endpoint `/generate` que hace streaming de respuestas
- Todo deployado en Vercel

### Streaming — por qué no esperás al final

Sin streaming: el usuario espera 10 segundos mirando una pantalla vacía hasta que aparece toda la respuesta de una.

Con streaming: las palabras van apareciendo en tiempo real mientras el modelo genera, como si alguien estuviera escribiendo.

**Analogía**: es la diferencia entre esperar que cargue toda una película antes de empezar a verla, o verla en streaming desde el primer segundo.

En código, FastAPI devuelve un `StreamingResponse` que manda los tokens de OpenAI al frontend a medida que llegan.

### Pages Router vs App Router

Day 2 usó **Pages Router** (la forma anterior de Next.js):
```
pages/
  index.tsx    → ruta /
  product.tsx  → ruta /product
```

Week 2 usará **App Router** (la forma moderna):
```
app/
  page.tsx         → ruta /
  about/page.tsx   → ruta /about
```

La lógica de React es la misma. Cambia la estructura de carpetas y algunos detalles de cómo se obtienen datos.

### Diagrama Day 2

```
Browser
  │
  ├── GET / → Vercel → Next.js (HTML + JS)
  │
  └── POST /generate → Vercel → FastAPI
                                    │
                                    └── OpenAI (stream)
                                            │
                                    tokens llegan de a poco
                                            │
                                    Frontend los muestra en tiempo real
```

---

## Day 3 — Autenticación con Clerk

### Lo que hiciste

Agregaste login real a la app. Los usuarios pueden registrarse con Google, GitHub o email. El backend verifica la identidad de cada request con un token JWT.

### Clerk — el portero del edificio

**Analogía**: Clerk es el portero de un edificio corporativo. Antes de entrar, verificás tu identidad. Él te da un carnet (JWT token) que mostrás en cada piso (endpoint) para confirmar que tenés acceso.

Sin Clerk: cualquiera puede llamar a tu API. Con Clerk: solo usuarios autenticados.

### JWT — el carnet de acceso

JWT (JSON Web Token) es un string cifrado que contiene información sobre el usuario. El frontend lo manda en el header de cada request al backend:

```
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
```

El backend verifica con la librería `fastapi_clerk_auth` que el token es válido y no fue alterado.

```
Usuario se loguea → Clerk genera JWT → Frontend lo guarda
Cada request → Frontend manda JWT en header → Backend lo verifica → si ok, responde
```

### Clerk Billing — suscripciones sin código de pagos

Day 3 Parte 2 agregó suscripciones de pago. Clerk se integra con Stripe internamente. Vos solo:
1. Definís un plan en el dashboard de Clerk (nombre, precio, key)
2. Usás el componente `<Protect>` en el frontend para envolver el contenido premium
3. Clerk maneja el checkout, el estado de suscripción y la renovación

**Analogía**: es como tener una caja registradora ya instalada — solo definís los precios, no construís el sistema de pagos.

### Variables de entorno en Vercel

Para que Clerk funcione en producción necesitás las keys en Vercel:
```
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY  → va al frontend (puede ser pública)
CLERK_SECRET_KEY                   → solo en el backend (nunca exponer)
CLERK_JWKS_URL                     → para verificar tokens JWT en el backend
```

La convención `NEXT_PUBLIC_` en Next.js significa que esa variable será visible en el código del browser. Sin ese prefijo, solo el servidor puede leerla.

### Diagrama Day 3

```
Usuario
  │
  └── Intenta acceder → ¿está logueado? (Clerk verifica)
        │
        ├── No → pantalla de login (Google / GitHub / Email)
        │              │
        │         Login exitoso → Clerk genera JWT
        │
        └── Sí → ¿tiene suscripción activa? (Clerk Billing verifica)
                      │
                      ├── No → muestra tabla de precios (PricingTable)
                      │
                      └── Sí → accede al generador de ideas
                                      │
                                      └── POST /generate con JWT en header
                                                  │
                                            Backend verifica JWT → OpenAI → stream
```

---

## Day 4 — Aplicación médica real

### Lo que hiciste

Tomaste toda la infraestructura de los días anteriores y la aplicaste a un caso de uso concreto: un asistente para médicos. El doctor ingresa notas de consulta, la app genera resumen, próximos pasos y un email para el paciente.

### Por qué importa este día

Day 4 no introduce tecnología nueva — consolida todo lo anterior en un producto real. Es el ejercicio de integración.

**Lo que reusa**:
- Streaming de OpenAI (Day 2)
- Autenticación JWT con Clerk (Day 3)
- FastAPI + Next.js + Vercel (Days 1-3)

**Lo nuevo**:
- `Pydantic BaseModel` con campos reales (`patient_name`, `date_of_visit`, `notes`)
- System prompt específico del dominio — el modelo recibe instrucciones para devolver exactamente tres secciones
- Formulario estructurado con selector de fecha en el frontend

### System prompt — las instrucciones del modelo

El system prompt es lo que le decís al modelo quién es y cómo debe responder. En el asistente médico:

```
"You are given notes from a doctor. Reply with exactly three sections:
### Summary of visit for the doctor's records
### Next steps for the doctor
### Draft of email to patient in patient-friendly language"
```

Si el system prompt es bueno, el output del modelo es predecible y estructurado. Si es vago, el modelo improvisa.

### Pydantic — validación de datos automática

`Pydantic BaseModel` define la estructura exacta de los datos que puede recibir un endpoint. Si el frontend manda datos con formato incorrecto, FastAPI devuelve un error 422 automáticamente — sin que vos escribas ninguna validación.

```python
class Visit(BaseModel):
    patient_name: str      # obligatorio, debe ser string
    date_of_visit: str     # obligatorio
    notes: str             # obligatorio
```

**Analogía**: es como un formulario con campos obligatorios — si falta uno o tiene el tipo incorrecto, no se puede enviar.

### Diagrama Day 4

```
Doctor llena formulario
  │  (paciente, fecha, notas de consulta)
  ▼
Frontend valida campos → POST /generate con JWT
  │
  ▼
Backend (FastAPI)
  │  verifica JWT (Clerk)
  │  construye prompt con los datos del formulario
  ▼
OpenAI (streaming)
  │
  ▼
Tres secciones aparecen en tiempo real:
  1. Resumen del médico
  2. Próximos pasos
  3. Email para el paciente
```

---

## Day 5 — Docker + ECR + ECS Fargate

### Lo que hiciste

Dejaste Vercel y deployaste la app en AWS usando contenedores. Esta es la arquitectura que usan empresas reales para apps en producción.

### Por qué salir de Vercel

Vercel es perfecto para arrancar. Pero tiene limitaciones: no soporta cualquier tipo de contenedor, no da control sobre la infraestructura, y no escala bien para apps muy custom. ECS Fargate da control total: elegís la imagen, el entorno, los recursos, los permisos.

### Docker — el container de barco

Docker empaqueta todo lo que la app necesita (código + dependencias + configuración) en una imagen que corre igual en cualquier máquina.

**Multi-stage build** (lo que usaste): el Dockerfile tiene dos etapas:
1. Primera etapa: construye el frontend Next.js como archivos estáticos
2. Segunda etapa: corre FastAPI y sirve tanto la API como esos archivos

Un solo contenedor para todo.

```
Sin Docker:  "instalá Python 3.11, estas libs, estas vars de entorno..."
Con Docker:  una imagen → corre igual en tu PC, en AWS, en cualquier lado
```

### ECR — la bodega privada de imágenes

ECR (Elastic Container Registry) es donde guardás tus imágenes Docker dentro de tu cuenta de AWS. Como Docker Hub pero privado.

```
tu máquina → docker build → imagen local
imagen local → docker push → ECR
ECR → ECS descarga la imagen cuando levanta una task
```

### La jerarquía de ECS

```
Cluster         = la ciudad        (agrupa todo)
Service         = el gerente       (mantiene N tasks corriendo, reinicia si fallan)
Task Definition = el blueprint     (imagen, CPU, memoria, variables de entorno)
Task            = el container     (la instancia real corriendo ahora mismo)
```

**Fargate** significa que no gestionás servidores — AWS corre el container en su infraestructura. Solo definís cuánta CPU y memoria necesitás.

### IAM — el sistema de llaves

AWS no da permisos automáticamente. El rol `ecsTaskExecutionRole` necesita dos llaves explícitas:

```
ecsTaskExecutionRole
  ├── AmazonEC2ContainerRegistryReadOnly  → puede bajar la imagen de ECR
  └── CloudWatchLogsFullAccess            → puede escribir logs

Sin estas dos → AccessDeniedException → task no levanta
```

**Analogía**: es como darle a un empleado acceso al edificio, pero sin la llave al archivo ni al cuaderno de registros.

### Diagrama Day 5

```
Developer
  │
  ├── docker build --platform linux/amd64 → imagen local
  └── docker push → ECR (almacén de imágenes en AWS)

                   AWS
┌──────────────────────────────────────────────────────┐
│  Cluster                                             │
│    └── Service (mantiene 1 task corriendo)           │
│          └── Task Definition (blueprint)             │
│                └── Task ← imagen de ECR              │
│                      │                               │
└──────────────────────────────────────────────────────┘
                        │
                 Public IP:8000
                        │
                 Security Group (firewall: puerto 8000)
                        │
                     Browser
```

### Flujo de deploy y actualización

**Primera vez:**
```
build → push ECR → crear cluster → task definition → security group → service
```

**Actualizar la app:**
```
build nuevo → push ECR (mismo tag :latest) → Service: "Force new deployment"
```

---

## Comparación de la semana completa

| | Day 1-2 | Day 3 | Day 4 | Day 5 |
|---|---|---|---|---|
| Deploy | Vercel | Vercel | Vercel | ECS Fargate |
| Auth | No | Clerk + JWT | Clerk + JWT | Clerk + JWT |
| IA | OpenAI | OpenAI streaming | OpenAI streaming | OpenAI streaming |
| Contenedor | No | No | No | Docker |
| Control infra | Bajo | Bajo | Bajo | Alto |
| Costo | Gratis | Gratis | Gratis | ~$9/mes |

---

## Comparación Vercel vs ECS Fargate

| | Vercel | ECS Fargate |
|---|---|---|
| Tipo | PaaS | CaaS |
| Setup | 5 minutos | ~1 hora |
| HTTPS | Automático | No (necesita ALB) |
| URL fija | Sí | No (IP cambia) |
| Costo | Gratis (hobby) | ~$9/mes |
| Control | Bajo | Alto |
| Uso en industria | Startups, proyectos | Producción real |

**Regla práctica**: Vercel para validar y arrancar rápido. ECS Fargate cuando necesitás containers, más control, o el equipo ya usa AWS.

---

## Gotchas que te van a salvar tiempo

1. **`NEXT_PUBLIC_` prefix** — sin ese prefijo la variable de entorno no llega al browser en Next.js
2. **`--platform linux/amd64`** — obligatorio al hacer `docker build` en Windows/Mac para que la imagen funcione en AWS (Linux x86)
3. **"Solo Fargate"** — en la consola AWS dice "Solo Fargate", no "AWS Fargate"
4. **Permisos IAM no son automáticos** — `ecsTaskExecutionRole` necesita las dos políticas agregadas a mano
5. **Public IP: ENABLED** — si no lo activás al crear el Service, el container no tiene IP pública
6. **Load balancing: None** — para aprender no hace falta ALB; si lo dejás activo cobra más
7. **Circuit breaker** — "Deployment circuit breaker was triggered" es síntoma, no causa. El error real está en los logs de la task con estado "Stopped"
8. **JWT timeout de 60 segundos** — si OpenAI tarda más de 60s, Clerk invalida el token. Fix disponible en `community_contributions/jwt_token_60s_fix.md`

---

## Cómo pausar ECS para no gastar

ECS no tiene "pause" nativo. La solución: `Desired tasks = 0` en el Service. No se cobra cuando hay 0 tasks corriendo. Para volver a levantar: cambiar a `1`.
