# school-assistant
Kiosko educativo digital para bibliotecas escolares

# ANÁLISIS TÉCNICO DETALLADO DEL PROYECTO: ASISTENTE EDUCATIVO CON IA

---

# 📘 Análisis Técnico Detallado del Proyecto

## Asistente Educativo con Inteligencia Artificial

---

## 🧱 1. Arquitectura del Sistema

El proyecto seguirá una **arquitectura de tres capas**, optimizada para desarrollo individual, escalabilidad progresiva y **costos mínimos de operación**.

### 🎨 Capa de Presentación (Frontend)

* Construida con **React 18**
* Uso de **Hooks modernos** y **Context API** para gestión de estado
* Aplicación tipo **SPA (Single Page Application)**
* Comunicación con backend mediante **API REST**

**Justificación:**

* Aprovecha tu experiencia previa
* Permite desarrollo rápido de interfaces interactivas
* Ecosistema estable y ampliamente adoptado

---

### ⚙️ Capa de Aplicación (Backend)

* Servidor **Node.js + Express.js**

**Responsabilidades principales:**

* Autenticación con **JWT**
* Middleware de **rate limiting**
* Sistema de **logging**
* Lógica de negocio para prompts y conversaciones

**Ventajas:**

* JavaScript en frontend y backend
* Manejo eficiente de operaciones asíncronas (clave para IA)
* Ecosistema maduro con miles de librerías

---

### 🗄️ Capa de Datos

* Base de datos **PostgreSQL**

**¿Por qué PostgreSQL y no SQLite?**

* Mejor manejo de concurrencia
* Capacidades analíticas avanzadas
* Escalabilidad sencilla hacia producción
* Compatible con **Supabase (tier gratuito)**

---

## 🤖 2. Integración de Inteligencia Artificial

La IA es el **corazón del sistema**.

### 🔌 Opciones evaluadas

* Claude API (Anthropic) ✅ **Recomendada para el MVP**
* GPT-4 (OpenAI)
* Llama 2 (local)

**Razones para elegir Claude:**

* Excelente desempeño en contextos educativos
* $5 USD en créditos iniciales (~1000 consultas)
* API clara y bien documentada

---

### 🧠 Arquitectura de Prompts

* Sistema **modular y parametrizable**
* Variables dinámicas:

  * Grado escolar
  * Materia
  * Contexto previo
  * Modo educativo

**Rol base del asistente:**

> *“Eres un tutor que enseña a pensar, no que da respuestas directas.”*

**Modos especializados:**

* Explicación conceptual
* Generación de ejercicios
* Ayuda con tareas
* Repaso para examen

---

### 🗂️ Manejo de Contexto Conversacional

* Ventana deslizante de **5–7 mensajes recientes**
* Compresión de mensajes antiguos para ahorrar tokens
* Metadatos por conversación:

  * Tema
  * Número de intentos
  * Nivel de satisfacción

---

## 🧩 3. Diseño de Base de Datos

Diseñado para **uso en tiempo real** y **análisis educativo posterior**.

### 👤 Tabla `users`

* id (UUID)
* email
* nombre
* rol (estudiante / maestro / admin)
* grado escolar
* escuela

---

### 💬 Tabla `conversations`

* Usuario
* Materia y tema
* Fecha inicio / fin
* Calificación de satisfacción

📊 Permite detectar:

* Temas más difíciles
* Tiempo promedio por consulta
* Horarios de mayor uso

---

### 📝 Tabla `messages`

* Autor y timestamp
* Contenido
* Tokens consumidos
* Tiempo de respuesta de la API
* Flags de utilidad / regeneración

---

### 📈 Tabla `usage_metrics`

* Estadísticas diarias pre-calculadas
* Consultas totales
* Tokens usados
* Promedio de satisfacción

⏰ Actualizada mediante **job nocturno**

---

### ♻️ Tabla `common_questions`

* Caché inteligente de preguntas frecuentes
* Reduce costos y mejora tiempos de respuesta

Ejemplo:

> "¿Cómo se calcula el área de un triángulo?"

---

## 💸 4. Estrategia de Control de Costos

Se implementan **5 niveles de protección**:

### 1️⃣ Rate Limiting

* Máx. **10 consultas diarias por estudiante**
* Reset diario automático

### 2️⃣ Límite de Tokens

* Máx. **500 tokens por respuesta**

### 3️⃣ Caché Multicapa

* Memoria (sesión)
* Base de datos (`common_questions`)
* Caché de prompts del proveedor IA

### 4️⃣ Monitoreo y Alertas

* Dashboard con gasto en tiempo real
* Alerta al llegar al **80% del presupuesto**

### 5️⃣ Fallback Local

* Cambio automático a **Llama 2 local**
* Costo cero en emergencia

---

## 🎨 5. Interfaz y Experiencia de Usuario (UX)

Diseñada para **niños de 9 a 12 años**.

### 🎒 Interfaz para Estudiantes

* Chat ocupa el **80% del viewport**
* Input grande y visible
* Elementos superiores simples:

  * Selector de materia
  * Nueva consulta
  * Consultas restantes

### 🧑‍🏫 Panel para Maestros

* Dashboard con:

  * Uso semanal 📈
  * Temas más consultados 📊
  * Estudiantes activos 👥
  * Satisfacción promedio ⭐

---

### 📱 Responsive Design

* Optimizado para móviles
* Botones grandes
* Input no cubierto por teclado

---

## 🧠 6. Sistema de Prompts Educativos

Define la **calidad pedagógica** del asistente.

### 📜 Principios Base

* Método socrático
* No dar respuestas directas
* Adaptar lenguaje al grado
* Ayuda progresiva tras varios intentos

### 🔄 Variables Dinámicas

* Grado, edad, materia, tema
* Número de intentos
* Nivel de frustración

---

### 🎯 Modos Educativos

* **Explicación:** analogías cotidianas
* **Práctica:** ejercicios guiados
* **Tarea:** preguntas orientadoras
* **Repaso:** diagnóstico y refuerzo

---

## 🔐 7. Seguridad y Privacidad

* Autenticación JWT (24h)
* Contraseñas con bcrypt (salt 12)
* HTTPS obligatorio
* Variables sensibles en `.env`

### 🧒 Protección de Menores

* Cuentas creadas por maestros
* Sin email personal del niño
* Control estricto de acceso a conversaciones

---

## 🚀 8. Deployment y DevOps

### 🧑‍💻 Control de Versiones

* GitHub
* Ramas: `main`, `develop`, `feature/*`

### 🌐 Hosting

* Frontend: **Vercel**
* Backend: **Render / Railway**
* DB: **Render o Supabase**

### 📊 Monitoreo

* Logs con Winston o Pino
* Health check `/health`

---

## 🧪 9. Estrategia de Testing

### Backend

* Unit tests con **Jest**
* Integration tests con **Nock**

### Frontend

* Testing manual inicial
* Tests críticos con React Testing Library

### 👀 Testing Real

* Pruebas tempranas con estudiantes reales

---

## ⚡ 10. Optimización de Performance

* Lazy loading de rutas
* Streaming de respuestas IA (SSE)
* Índices en DB
* Paginación en listas largas
* CDN para assets estáticos

---

📌 **Conclusión:**
Esta arquitectura prioriza **claridad pedagógica, control de costos, seguridad y escalabilidad**, permitiendo validar rápidamente el MVP sin comprometer una futura evolución del sistema.
