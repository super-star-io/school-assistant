# school-assistant
Kiosko educativo digital para bibliotecas escolares

# ANÁLISIS TÉCNICO DETALLADO DEL PROYECTO: ASISTENTE EDUCATIVO CON IA

---

## 1. ARQUITECTURA DEL SISTEMA

### Arquitectura de Tres Capas

El proyecto seguirá una **arquitectura de tres capas clásica** optimizada para desarrollo individual y operación con costos mínimos. 

#### **Capa de Presentación (Frontend)**
La capa de presentación estará construida con **React 18**, aprovechando hooks modernos y context API para gestión de estado. Esta decisión se toma por tu experiencia previa con el framework y porque permite desarrollo rápido de interfaces interactivas. La aplicación será una **SPA (Single Page Application)** que se comunicará con el backend mediante API REST.

#### **Capa de Aplicación (Backend)**
La capa de aplicación consistirá en un servidor **Node.js con Express.js**. Esta combinación es ideal porque:
- Permite reutilizar conocimiento de JavaScript en ambos lados
- Facilita el manejo de operaciones asíncronas (crítico para las llamadas a APIs de IA)
- Tiene un ecosistema maduro con millones de paquetes disponibles

El servidor implementará:
- Autenticación JWT
- Middleware para rate limiting
- Sistema de logging
- Lógica de negocio para gestionar prompts y conversaciones

#### **Capa de Datos (Database)**
La capa de datos utilizará **PostgreSQL** como base de datos principal. Aunque SQLite podría parecer suficiente para un MVP, PostgreSQL ofrece ventajas importantes:
- ✅ Mejor manejo de concurrencia cuando múltiples estudiantes usen el sistema simultáneamente
- ✅ Capacidades analíticas superiores que necesitaremos para generar insights para los maestros
- ✅ Facilidad para migrar a producción (Supabase ofrece PostgreSQL gestionado en su tier gratuito)

---

## 2. INTEGRACIÓN DE INTELIGENCIA ARTIFICIAL

### Selección de Proveedor de IA

El componente de IA será el **corazón del sistema**. Tenemos tres opciones principales:

| Opción | Ventajas | Consideraciones |
|--------|----------|-----------------|
| **Claude API** (Anthropic) | Excelente en tareas educativas, $5 créditos iniciales (~1000 consultas), API simple | Recomendado para MVP |
| **GPT-4** (OpenAI) | Muy potente, ampliamente probado | Costos ligeramente más altos |
| **Llama 2** (Local) | Completamente gratuito, privacidad total | Requiere setup técnico adicional |

**Recomendación:** Comenzar con **Claude API** por su capacidad pedagógica superior y créditos iniciales.

### Arquitectura de Prompts

La arquitectura de prompts será **modular y parametrizable**. Crearemos un sistema de plantillas donde cada prompt tendrá variables dinámicas:

```javascript
// Ejemplo de estructura de prompt
const basePrompt = `
Eres un tutor de ${subject} para estudiantes de ${grade} grado en México.
Tu objetivo es ayudar al estudiante a aprender, NO darle la respuesta directamente.
Usa el método socrático: haz preguntas que lo guíen a descubrir la solución.
Adapta tu lenguaje para que sea comprensible para un niño de ${age} años.
`;
```

**Variables dinámicas incluyen:**
- 📚 Grado escolar del estudiante
- 🎯 Materia actual
- 💬 Contexto de la conversación previa
- 🎮 Modo de interacción seleccionado

### Manejo del Contexto Conversacional

Implementaremos una **ventana de contexto deslizante** que incluya los últimos 5-7 intercambios, pero comprimiendo mensajes antiguos para no exceder límites de tokens. 

**Metadatos almacenados por conversación:**
- 🔖 Tema específico
- 🔢 Número de intentos antes de comprender
- ⭐ Nivel de satisfacción al final

---

## 3. DISEÑO DE BASE DE DATOS

### Esquema Optimizado para Operación y Análisis

El esquema de base de datos estará optimizado tanto para operación en tiempo real como para análisis posterior.

#### **Tabla: `users`**
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    role ENUM('student', 'teacher', 'admin') NOT NULL,
    grade_level INTEGER, -- Para estudiantes
    school_id UUID REFERENCES schools(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**Justificación de UUID:** Facilita eventualmente la fusión de datos de múltiples instancias si escalamos a varias escuelas.

#### **Tabla: `conversations`**
```sql
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) NOT NULL,
    subject VARCHAR(100) NOT NULL, -- matematicas, ciencias, etc.
    topic VARCHAR(255), -- Tema específico
    started_at TIMESTAMP DEFAULT NOW(),
    ended_at TIMESTAMP,
    satisfaction_rating INTEGER CHECK (satisfaction_rating BETWEEN 1 AND 5),
    INDEX idx_user_id (user_id),
    INDEX idx_started_at (started_at)
);
```

**Esta tabla permite identificar patrones:**
- ❓ Qué temas generan más dudas
- ⏱️ Cuánto tiempo toma resolver cada tipo de consulta
- 🕐 En qué horarios los estudiantes más estudian

#### **Tabla: `messages`**
```sql
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID REFERENCES conversations(id) NOT NULL,
    role ENUM('user', 'assistant') NOT NULL,
    content TEXT NOT NULL,
    tokens_used INTEGER, -- Para control de costos
    response_time_ms INTEGER, -- Tiempo de respuesta de la API
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_conversation_id (conversation_id),
    INDEX idx_created_at (created_at)
);
```

#### **Tabla: `usage_metrics`** (Pre-calculada)
```sql
CREATE TABLE usage_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) NOT NULL,
    date DATE NOT NULL,
    queries_count INTEGER DEFAULT 0,
    total_tokens INTEGER DEFAULT 0,
    avg_satisfaction DECIMAL(3,2),
    topics_consulted JSONB, -- Array de temas
    UNIQUE(user_id, date),
    INDEX idx_date (date)
);
```

**⚡ Ventaja:** No hacemos queries pesadas en tiempo real cuando un maestro abre su dashboard. Un job nocturno actualiza estos números.

#### **Tabla: `common_questions`** (Caché Inteligente)
```sql
CREATE TABLE common_questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_hash VARCHAR(64) UNIQUE NOT NULL, -- Hash MD5 de la pregunta
    question_text TEXT NOT NULL,
    best_answer TEXT NOT NULL,
    subject VARCHAR(100),
    grade_level INTEGER,
    times_served INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**💡 Beneficio:** Cuando múltiples estudiantes hacen esencialmente la misma pregunta, servimos una respuesta pre-generada. Esto reduce costos de API y mejora tiempos de respuesta.

---

## 4. ESTRATEGIA DE CONTROL DE COSTOS

### Sistema de Cinco Niveles de Protección

Este es quizás el aspecto más crítico para la viabilidad del proyecto. Las APIs de IA cobran por token, y sin controles adecuados, podrías agotar tu presupuesto en días.

#### **Nivel 1: Rate Limiting Estricto**
```javascript
const DAILY_QUERY_LIMIT = 10; // Por estudiante

app.use('/api/chat', rateLimitMiddleware({
  windowMs: 24 * 60 * 60 * 1000, // 24 horas
  max: DAILY_QUERY_LIMIT,
  message: 'Has alcanzado tu límite de consultas por hoy. ¡Vuelve mañana! 😊'
}));
```

**📊 Justificación:** 10 consultas diarias es suficiente para uso genuino educativo pero previene abuso.

#### **Nivel 2: Límite de Tokens por Respuesta**
```javascript
const AI_CONFIG = {
  model: 'claude-sonnet-4',
  max_tokens: 500, // Límite por respuesta
  temperature: 0.7
};
```

**🎯 Ventaja doble:** Ahorra presupuesto Y mejora pedagogía (explicaciones concisas son mejores).

#### **Nivel 3: Sistema de Caché Multicapa**

```
┌─────────────────────────────────────┐
│  Caché L1: En Memoria (Redis/Map)   │ ← Preguntas idénticas en sesión
├─────────────────────────────────────┤
│  Caché L2: Base de Datos            │ ← common_questions (preguntas frecuentes)
├─────────────────────────────────────┤
│  Caché L3: Prompt Cache de Claude   │ ← Contexto repetido
└─────────────────────────────────────┘
```

#### **Nivel 4: Monitoreo y Alertas**

Dashboard en tiempo real mostrando:
- 💵 Dólares gastados hasta ahora
- 📈 Tokens consumidos por día
- 📊 Proyección de gasto mensual
- 🚨 Alerta al 80% del presupuesto

#### **Nivel 5: Fallback a Modelo Local**

```javascript
async function getAIResponse(prompt) {
  try {
    if (budgetExceeded()) {
      return await getLlamaLocalResponse(prompt); // Plan B: Gratis
    }
    return await getClaudeResponse(prompt); // Plan A: Mejor calidad
  } catch (error) {
    logger.error('AI API failed', error);
    return await getLlamaLocalResponse(prompt); // Fallback automático
  }
}
```

**🛡️ Garantía:** El sistema NUNCA dejará de funcionar por falta de presupuesto.

---

## 5. DISEÑO DE INTERFAZ Y EXPERIENCIA DE USUARIO

### Principios de Diseño para Niños de 9-12 años

#### **Interfaz para Estudiantes: Minimalismo Radical**

```
┌─────────────────────────────────────────────────┐
│  [📚 Matemáticas ▼]  [🔄 Nueva]  [⚡ 7/10 hoy]  │
├─────────────────────────────────────────────────┤
│                                                  │
│  🤖 ¡Hola! ¿En qué puedo ayudarte?             │
│                                                  │
│                                                  │
│  👤 ¿Cómo saco el área de un círculo?          │
│                                                  │
│                                                  │
│  🤖 ¡Buena pregunta! Primero, ¿sabes qué es    │
│     el radio de un círculo?                     │
│                                                  │
├─────────────────────────────────────────────────┤
│  [Escribe tu pregunta aquí...        ] [Enviar]│
└─────────────────────────────────────────────────┘
```

**Elementos clave:**
- 🎨 Paleta amigable: azules claros, verde para asistente, blanco para estudiante
- 📱 80% del viewport para chat
- 🎯 Solo 3 elementos arriba: materia, nueva consulta, límite diario
- ⚡ CERO complejidad que pueda confundir

#### **Panel para Maestros: Rico en Información**

```
┌──────────────────────────────────────────────────────────┐
│  Dashboard - Matemáticas 5to A                          │
├──────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │ 📈 Uso 7d   │  │ 🔥 Temas    │  │ 👥 Top Activos │  │
│  │   [gráfica] │  │ [barras]    │  │  1. Ana (45)   │  │
│  │             │  │             │  │  2. Luis (38)  │  │
│  └─────────────┘  └─────────────┘  └────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │ 📋 Conversaciones Recientes                        │ │
│  │ [Filtros: Estudiante ▼] [Tema ▼] [Fecha ▼]       │ │
│  ├────────────────────────────────────────────────────┤ │
│  │ Ana - Fracciones - Hace 2 horas - ⭐⭐⭐⭐⭐      │ │
│  │ Luis - Geometría - Hace 3 horas - ⭐⭐⭐⭐        │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Widgets principales:**
1. 📊 Uso en últimos 7 días (gráfica de línea)
2. 📚 Temas más consultados (gráfica de barras)
3. 👥 Estudiantes más activos (lista top 5)
4. ⭐ Calificación promedio de satisfacción

### Diseño Responsive para Móviles

**Consideraciones críticas:**
- 📱 Chat adaptado a pantallas pequeñas
- ⌨️ Teclado virtual no cubre el input
- 👆 Botones suficientemente grandes (min 44x44px)
- 🎯 React facilita esto con conditional rendering

```javascript
// Ejemplo de diseño responsive
const isMobile = window.innerWidth < 768;

return (
  <div className={isMobile ? "chat-mobile" : "chat-desktop"}>
    {/* Contenido adaptativo */}
  </div>
);
```

---

## 6. SISTEMA DE PROMPTS EDUCATIVOS

### Codificando Buenas Prácticas Pedagógicas

El prompt engineering será donde se define realmente la **calidad pedagógica** del asistente.

#### **System Prompt Base**

```javascript
const SYSTEM_PROMPT = `
Eres un tutor educativo especializado en educación primaria en México.

PRINCIPIOS PEDAGÓGICOS:
• Usa el método socrático - no des respuestas directas
• Haz preguntas que guíen al estudiante a descubrir la solución
• Si después de 3 intentos no avanza, ofrece más ayuda directa
• SIEMPRE explica el razonamiento detrás de cada paso

ADAPTACIÓN AL NIVEL:
• 4to grado (9-10 años): Evita términos técnicos, usa ejemplos concretos
• 5to grado (10-11 años): Introduce conceptos formales gradualmente
• 6to grado (11-12 años): Más formalismo pero siempre verificando comprensión

CONTEXTO CULTURAL:
• Usa ejemplos del contexto mexicano cuando sea posible
• Monedas: pesos mexicanos, no dólares
• Medidas: sistema métrico
• Situaciones: tacos, fútbol, celebraciones mexicanas

DETECCIÓN DE FRUSTRACIÓN:
• Si el estudiante dice "no entiendo", "es muy difícil", "estoy confundido"
  → Cambia a modo más paciente y explicativo
• Si lleva >20 minutos sin progreso
  → Sugiere descanso o consultar al maestro humano
`;
```

#### **Prompts Especializados por Modo**

##### **Modo 1: Explicación Conceptual**
```javascript
const EXPLANATION_MODE = `
${SYSTEM_PROMPT}

TAREA ACTUAL: Explicar el concepto "${concept}"

ESTRATEGIA:
1. Desglosa el concepto en 3-4 partes simples
2. Usa una analogía con algo de la vida diaria del estudiante
3. Da un ejemplo numérico concreto
4. Verifica comprensión con una pregunta diferente

EJEMPLO DE INTERACCIÓN:
Estudiante: "¿Qué es una fracción?"
Tú: "¡Buena pregunta! Imagina que tienes una pizza completa. 
Si la cortas en 4 partes iguales y tomas 1 pedazo, 
tienes 1/4 (un cuarto) de la pizza.
¿Qué crees que significa 2/4?"
`;
```

##### **Modo 2: Generación de Ejercicios**
```javascript
const PRACTICE_MODE = `
${SYSTEM_PROMPT}

TAREA ACTUAL: Generar ejercicio de práctica sobre "${topic}"

REGLAS:
• Genera UN ejercicio similar al que el estudiante está trabajando
• Cambia los números pero mantén la estructura
• NO resuelvas el ejercicio, solo preséntalo
• Espera la respuesta del estudiante
• Si es correcta: felicita y explica por qué está bien
• Si es incorrecta: señala qué parte está correcta y qué necesita revisar

FORMATO:
"Ahora intenta tú este ejercicio similar:
[Ejercicio aquí]

Tómate tu tiempo y dime tu respuesta cuando estés listo 😊"
`;
```

##### **Modo 3: Ayuda con Tarea**
```javascript
const HOMEWORK_MODE = `
${SYSTEM_PROMPT}

TAREA ACTUAL: Ayudar con tarea sin dar respuesta directa

REGLA DE ORO: NUNCA des la respuesta completa

ESTRATEGIA SOCRÁTICA:
1. "¿Qué necesitas encontrar en este problema?"
2. "¿Qué datos te dan?"
3. "¿Qué fórmula o proceso conoces que podría ayudarte?"
4. "¿Cuál crees que sería el primer paso?"
5. Guía paso a paso, pero que el estudiante haga el trabajo

SI EL ESTUDIANTE INSISTE EN LA RESPUESTA:
"Entiendo que quieres terminar rápido, pero mi trabajo es ayudarte 
a APRENDER, no solo darte la respuesta. Si te la doy, no aprenderás 
y en el examen no estaré allí para ayudarte. ¿Vamos paso a paso juntos?"
`;
```

##### **Modo 4: Repaso para Examen**
```javascript
const REVIEW_MODE = `
${SYSTEM_PROMPT}

TAREA ACTUAL: Repaso para examen de "${subject}"

ESTRATEGIA DIAGNÓSTICA:
1. Hacer 3-5 preguntas breves sobre diferentes aspectos del tema
2. Basado en respuestas, identificar qué domina y qué necesita repasar
3. Enfocar el repaso en las áreas débiles
4. Terminar con las áreas fuertes para dar confianza

ESTRUCTURA:
"¡Vamos a prepararnos para tu examen! Primero quiero ver qué tanto dominas.
Te haré unas preguntas rápidas y basándome en tus respuestas, 
nos enfocaremos en lo que más necesites repasar. ¿Listo?"

[Después del diagnóstico]
"Veo que dominas bien [X], ¡excelente! 
Practiquemos más [Y] que necesita un poco más de repaso."
`;
```

### Variables Dinámicas en Prompts

```javascript
function buildContextualPrompt(conversation) {
  const vars = {
    GRADO: conversation.user.gradeLevel,
    EDAD: calculateAge(conversation.user.gradeLevel),
    MATERIA: conversation.subject,
    TEMA: conversation.topic,
    NUM_INTENTOS: conversation.messages.length / 2,
    FRUSTRATION_LEVEL: detectFrustration(conversation.messages),
    ULTIMA_PREGUNTA: conversation.messages[conversation.messages.length - 1].content
  };
  
  return interpolatePrompt(BASE_PROMPT, vars);
}
```

---

## 7. SEGURIDAD Y PRIVACIDAD

### Protección de Datos de Menores

Aunque son niños y el proyecto es educativo, debemos ser **extremadamente cuidadosos** con datos personales.

#### **Autenticación Robusta**

```javascript
// Configuración JWT
const JWT_CONFIG = {
  expiresIn: '24h',
  algorithm: 'HS256'
};

// Hashing de contraseñas
const BCRYPT_SALT_ROUNDS = 12;

// Para menores: cuentas creadas por maestro
// NO requieren email personal - usan identificadores escolares
```

#### **Comunicación Segura**

- 🔒 **HTTPS exclusivamente** (Render provee certificados gratuitos)
- 🔐 **Variables sensibles en .env** (NUNCA en código)
- 📝 **.gitignore configurado** para secretos

```bash
# .env (ejemplo)
DATABASE_URL=postgresql://user:pass@host:5432/db
JWT_SECRET=super_secret_key_change_in_production
CLAUDE_API_KEY=sk-ant-api03-...
NODE_ENV=production
```

#### **Control de Acceso a Conversaciones**

```javascript
// Middleware de autorización
async function canAccessConversation(userId, conversationId) {
  const conversation = await db.conversations.findById(conversationId);
  
  // Solo el estudiante o su maestro asignado pueden ver
  if (conversation.userId === userId) return true;
  
  const user = await db.users.findById(userId);
  if (user.role === 'teacher' && user.school_id === conversation.user.school_id) {
    return true;
  }
  
  return false; // Ni siquiera admin sin autorización expresa
}
```

#### **Soft Delete con Ventana de Recuperación**

```sql
ALTER TABLE conversations ADD COLUMN deleted_at TIMESTAMP;
ALTER TABLE conversations ADD COLUMN deleted_by UUID;

-- Borrado lógico
UPDATE conversations 
SET deleted_at = NOW(), deleted_by = current_user_id
WHERE id = conversation_id;

-- Job que corre diariamente para borrado permanente
DELETE FROM conversations 
WHERE deleted_at < NOW() - INTERVAL '30 days';
```

#### **Cumplimiento Legal**

**Documentos necesarios:**
1. 📄 **Términos de Uso** (adaptado al contexto educativo mexicano)
2. 🔒 **Aviso de Privacidad** simple y claro
3. ✍️ **Consentimiento institucional** firmado por director
4. 📧 **Carta a padres** explicando:
   - Qué datos se recopilan (nombre, grado, preguntas/respuestas)
   - Para qué se usan (mejorar aprendizaje, reportes anónimos)
   - Cómo se protegen (no se comparten, se eliminan bajo solicitud)

---

## 8. PLAN DE DEPLOYMENT Y DEVOPS

### Infraestructura sin Complejidad Innecesaria

Para el MVP no necesitamos Kubernetes ni microservicios, pero sí **procesos claros**.

#### **Estrategia de Branching Git**

```
main (producción)
  ├── develop (staging)
  │    ├── feature/chat-interface
  │    ├── feature/teacher-dashboard
  │    └── feature/ai-integration
  └── hotfix/urgent-bug-fix
```

**Workflow:**
1. Desarrollo en `feature/*` branches
2. Merge a `develop` → Deploy automático a staging
3. Testing en staging
4. Merge a `main` → Deploy automático a producción

#### **Deployment del Frontend**

**Plataforma:** Vercel (gratuito ilimitado)

```yaml
# vercel.json
{
  "buildCommand": "npm run build",
  "outputDirectory": "build",
  "framework": "react",
  "env": {
    "REACT_APP_API_URL": "https://api-educativa.onrender.com"
  }
}
```

**Dominio resultante:** `asistente-educativo.vercel.app`

#### **Deployment del Backend**

**Plataforma:** Render o Railway (750 hrs/mes gratis)

```yaml
# render.yaml
services:
  - type: web
    name: api-educativa
    env: node
    buildCommand: npm install
    startCommand: npm start
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        sync: false # Variable secreta
      - key: CLAUDE_API_KEY
        sync: false
```

#### **Base de Datos**

**Opciones:**
1. **Render PostgreSQL** (gratis hasta 1GB)
2. **Supabase** (gratis hasta 500MB + features adicionales)

**Recomendación:** Supabase por las herramientas adicionales (dashboard, logs, backups automáticos)

#### **Logging Estructurado**

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// Uso
logger.info('API call', { endpoint: '/chat', userId, tokensUsed: 245 });
logger.error('AI API failed', { error: err.message, stack: err.stack });
```

**Qué logear:**
- ✅ Cada request (timestamp, endpoint, userId)
- ✅ Errores con stack traces completos
- ✅ Métricas de tokens consumidos
- ✅ Eventos importantes (registros, conversaciones completadas)
- ❌ NO logear contraseñas, API keys, contenido sensible

#### **Health Checks**

```javascript
// GET /health
app.get('/health', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    aiAPI: await checkAIAPI(),
    diskSpace: await checkDiskSpace()
  };
  
  const allHealthy = Object.values(checks).every(c => c.status === 'ok');
  
  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'healthy' : 'unhealthy',
    checks,
    timestamp: new Date().toISOString()
  });
});
```

---

## 9. ESTRATEGIA DE TESTING

### Testing Pragmático para Timeline Ajustado

No podemos escribir tests exhaustivos, pero sí **cubriremos lo crítico**.

#### **Backend: Unit Tests con Jest**

```javascript
// tests/prompts.test.js
describe('Prompt Builder', () => {
  test('should interpolate variables correctly', () => {
    const prompt = buildContextualPrompt({
      user: { gradeLevel: 5 },
      subject: 'matematicas',
      topic: 'fracciones'
    });
    
    expect(prompt).toContain('5to grado');
    expect(prompt).toContain('matematicas');
  });
  
  test('should detect frustration from messages', () => {
    const messages = [
      { role: 'user', content: 'no entiendo nada' },
      { role: 'user', content: 'esto es muy difícil' }
    ];
    
    expect(detectFrustration(messages)).toBe('high');
  });
});
```

#### **Integration Tests con Mocks**

```javascript
// tests/chat.integration.test.js
const nock = require('nock');

describe('Chat Flow', () => {
  beforeEach(() => {
    // Mock de Claude API
    nock('https://api.anthropic.com')
      .post('/v1/messages')
      .reply(200, {
        content: [{ type: 'text', text: 'Respuesta de prueba' }]
      });
  });
  
  test('complete chat flow', async () => {
    const user = await createTestUser();
    const token = await loginUser(user);
    
    const response = await request(app)
      .post('/api/chat')
      .set('Authorization', `Bearer ${token}`)
      .send({ message: '¿Qué es una fracción?' });
    
    expect(response.status).toBe(200);
    expect(response.body.message).toBeDefined();
    expect(response.body.tokensUsed).toBeGreaterThan(0);
  });
});
```

#### **Frontend: Testing con React Testing Library**

```javascript
// tests/ChatComponent.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import ChatComponent from '../components/ChatComponent';

test('send button is disabled when input is empty', () => {
  render(<ChatComponent />);
  const sendButton = screen.getByRole('button', { name: /enviar/i });
  
  expect(sendButton).toBeDisabled();
});

test('displays query limit correctly', () => {
  render(<ChatComponent queriesLeft={7} queryLimit={10} />);
  
  expect(screen.getByText(/7\/10/i)).toBeInTheDocument();
});
```

#### **Testing con Usuarios Reales (Más Importante)**

```
PROTOCOLO DE TESTING TEMPRANO:
┌────────────────────────────────────────────┐
│ Semana 3: Prototipo funcional             │
│ ↓                                          │
│ Seleccionar 2-3 estudiantes voluntarios   │
│ ↓                                          │
│ Sesión supervisada de 30 minutos          │
│ ↓                                          │
│ Observar:                                  │
│  • ¿Dónde se confunden?                   │
│  • ¿Qué funciona bien?                    │
│  • ¿Qué preguntan que no anticipamos?    │
│ ↓                                          │
│ Iterar basado en observaciones            │
└────────────────────────────────────────────┘
```

**💡 Insight clave:** Observar a un niño real usando tu app por 30 minutos da más insights que 1000 unit tests.

---

## 10. OPTIMIZACIONES DE PERFORMANCE

### Experiencia Rápida y Fluida

La aplicación debe sentirse **instantánea y responsiva**.

#### **Frontend: Lazy Loading y Code Splitting**

```javascript
// Carga perezosa de rutas menos usadas
import { lazy, Suspense } from 'react';

const Chat = lazy(() => import('./pages/Chat'));
const TeacherDashboard = lazy(() => import('./pages/TeacherDashboard'));
const AdminPanel = lazy(() => import('./pages/AdminPanel'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Chat />} /> {/* Carga inmediata */}
        <Route path="/teacher" element={<TeacherDashboard />} />
        <Route path="/admin" element={<AdminPanel />} />
      </Routes>
    </Suspense>
  );
}
```

#### **Backend: Streaming de Respuestas de IA**

```javascript
// Server-Sent Events para respuestas en tiempo real
app.post('/api/chat/stream', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  
  const stream = await anthropic.messages.stream({
    model: 'claude-sonnet-4',
    messages: req.body.messages,
    max_tokens: 500
  });
  
  for await (const chunk of stream) {
    if (chunk.type === 'content_block_delta') {
      // Enviar cada token al frontend en tiempo real
      res.write(`data: ${JSON.stringify({ token: chunk.delta.text })}\n\n`);
    }
  }
  
  res.write('data: [DONE]\n\n');
  res.end();
});
```

**🎯 Resultado:** El estudiante ve que el asistente está "escribiendo" en tiempo real, como ChatGPT, en vez de esperar 3-5 segundos viendo un spinner.

#### **Base de Datos: Índices Estratégicos**

```sql
-- Índices críticos para queries frecuentes
CREATE INDEX idx_users_id ON users(id);
CREATE INDEX idx_conversations_user_id ON conversations(user_id);
CREATE INDEX idx_conversations_started_at ON conversations(started_at);
CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_usage_metrics_date ON usage_metrics(date);
CREATE INDEX idx_usage_metrics_user_date ON usage_metrics(user_id, date);

-- Índice compuesto para dashboard de maestro
CREATE INDEX idx_conversations_school_date 
ON conversations(school_id, started_at DESC);
```

#### **Paginación Inteligente**

```javascript
// Historial de conversaciones: 20 a la vez
app.get('/api/conversations', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = 20;
  const offset = (page - 1) * limit;
  
  const conversations = await db.conversations.findMany({
    where: { userId: req.user.id },
    orderBy: { startedAt: 'desc' },
    take: limit,
    skip: offset
  });
  
  const total = await db.conversations.count({
    where: { userId: req.user.id }
  });
  
  res.json({
    conversations,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  });
});
```

#### **CDN para Assets Estáticos**

```javascript
// Vercel incluye CDN automático, pero para imágenes:
// Usar Cloudinary (tier gratuito)

const cloudinary = require('cloudinary').v2;

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});

// Optimización automática de imágenes
const avatarUrl = cloudinary.url('avatar.jpg', {
  width: 100,
  height: 100,
  crop: 'fill',
  quality: 'auto',
  format: 'auto' // WebP en navegadores compatibles
});
```

---

## RESUMEN: STACK TÉCNICO COMPLETO

```
┌─────────────────────────────────────────────────────┐
│                     FRONTEND                        │
│  React 18 + Tailwind CSS → Vercel (GRATIS)        │
└──────────────────┬──────────────────────────────────┘
                   │ HTTPS / REST API
┌──────────────────▼──────────────────────────────────┐
│                     BACKEND                         │
│  Node.js + Express → Render/Railway (GRATIS)       │
│  • JWT Auth                                         │
│  • Rate Limiting                                    │
│  • Winston Logging                                  │
│  • Prompt Engine                                    │
└──────────────────┬──────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
┌───────▼─────────┐   ┌──────▼──────────┐
│   PostgreSQL    │   │   Claude API    │
│   (Supabase)    │   │   (Anthropic)   │
│     GRATIS      │   │  $5 créditos    │
└─────────────────┘   └─────────────────┘
```

**💰 Costo total operativo:** $0 - $30 USD por 3 meses  
**⚡ Tiempo de desarrollo:** 12 semanas (15-20 hrs/semana)  
**📊 Capacidad:** 50-100 estudiantes simultáneos  
**🎯 Escalabilidad:** Preparado para 1000+ estudiantes con ajustes mínimos

---

¿Te parece completo este análisis técnico? ¿Quieres que profundice en alguna sección específica o que agregue algún aspecto que haya faltado?
