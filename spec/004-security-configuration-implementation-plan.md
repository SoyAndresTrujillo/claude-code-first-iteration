# Plan de Implementación: Seguridad y Configuración Frontend
## Detalle Técnico de Implementación

> **Documento de plan específico para issues de seguridad y configuración**
> Fecha: 2026-01-19
> Versión: 1.0.0
> Basado en: spec/003-frontend-issues-analysis-and-improvement-plan.md

---

## 📋 Índice

1. [Resumen Ejecutivo](#resumen-ejecutivo)
2. [FASE 1: Configuración Base](#fase-1-configuración-base)
3. [FASE 2: Seguridad de Red](#fase-2-seguridad-de-red)
4. [FASE 3: Seguridad de Contenido](#fase-3-seguridad-de-contenido)
5. [FASE 4: Logging Seguro](#fase-4-logging-seguro)
6. [FASE 5: Protección CSRF](#fase-5-protección-csrf)
7. [Orden de Implementación](#orden-de-implementación)
8. [Testing y Validación](#testing-y-validación)
9. [Checklist Final](#checklist-final)

---

## 🎯 Resumen Ejecutivo

### Alcance
Este plan cubre la implementación de **7 issues críticos** de seguridad y configuración:

1. ✅ Variables de entorno y configuración externa
2. ✅ Cliente API centralizado
3. ✅ Content Security Policy (CSP)
4. ✅ Validación de URLs de imágenes
5. ✅ Logging seguro (sin stack traces)
6. ✅ Protección CSRF
7. ✅ Estrategia de deployment seguro

### Tiempo Estimado
- **Mínimo**: 12 horas
- **Máximo**: 18 horas
- **Promedio**: 14-16 horas (2 días de trabajo)

### Dependencias
- Backend debe implementar Origin validation (Issue 1.5)
- Deployment platform con soporte de env vars

---

## 🔧 FASE 1: Configuración Base

### Issue 1.1: Variables de Entorno

**Severidad**: CRÍTICO
**Tiempo estimado**: 2-3 horas
**Dependencias**: Ninguna (PRIMERO)
**Riesgo**: Bajo

#### Contexto Técnico

**Problema actual**:
```typescript
// ❌ Hardcodeado en 3 archivos
const res = await fetch("http://localhost:8000/courses");
```

**Por qué es un riesgo**:
- No funciona en producción (localhost:8000 no existe)
- Imposible deployar a staging/production
- Viola principio de configuración externa (12 factor app)

**Impacto en producción**:
- 100% de probabilidad de fallo
- Aplicación completamente inutilizable
- BLOQUEADOR DE DEPLOYMENT

#### Plan de Implementación

**Paso 1: Crear archivo `.env.example`**

```bash
cd Frontend
touch .env.example
```

Contenido de `.env.example`:
```bash
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:8000

# Environment
NODE_ENV=development

# Optional: Analytics, Monitoring (futuro)
# NEXT_PUBLIC_SENTRY_DSN=
# NEXT_PUBLIC_GA_ID=
```

**Validación**:
- [ ] Archivo creado en raíz de Frontend
- [ ] Comentarios claros para cada variable
- [ ] Committed to git

---

**Paso 2: Crear archivo de validación de entorno**

```bash
mkdir -p src/lib/env
touch src/lib/env/config.ts
```

Contenido de `src/lib/env/config.ts`:
```typescript
/**
 * Environment Configuration with Runtime Validation
 *
 * Uses zod for type-safe environment variable validation.
 * Fails fast if required variables are missing.
 */

import { z } from 'zod';

/**
 * Schema for environment variables
 */
const envSchema = z.object({
  // API Configuration
  NEXT_PUBLIC_API_URL: z.string().url({
    message: 'NEXT_PUBLIC_API_URL must be a valid URL',
  }),

  // Node environment
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
});

/**
 * Parse and validate environment variables
 */
function validateEnv() {
  const parsed = envSchema.safeParse({
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
    NODE_ENV: process.env.NODE_ENV,
  });

  if (!parsed.success) {
    console.error('❌ Invalid environment variables:');
    console.error(parsed.error.flatten().fieldErrors);
    throw new Error('Invalid environment variables');
  }

  return parsed.data;
}

/**
 * Validated environment configuration
 *
 * @example
 * import { env } from '@/lib/env/config';
 * fetch(env.NEXT_PUBLIC_API_URL + '/courses');
 */
export const env = validateEnv();

/**
 * Type-safe environment variables
 */
export type Env = z.infer<typeof envSchema>;
```

**Comandos**:
```bash
npm install zod
```

**Validación**:
- [ ] zod instalado
- [ ] Schema definido correctamente
- [ ] Validación ejecuta al importar
- [ ] TypeScript infiere tipos correctamente

---

**Paso 3: Migrar fetch calls a usar env**

**Archivo 1**: `src/app/page.tsx`

Cambiar:
```typescript
// ❌ ANTES
const res = await fetch("http://localhost:8000/courses", {
  cache: "no-store",
});
```

Por:
```typescript
// ✅ DESPUÉS
import { env } from '@/lib/env/config';

const res = await fetch(`${env.NEXT_PUBLIC_API_URL}/courses`, {
  cache: "no-store",
});
```

**Archivo 2**: `src/app/course/[slug]/page.tsx`

Cambiar:
```typescript
// ❌ ANTES
const res = await fetch(`http://localhost:8000/courses/${slug}`, {
  cache: "no-store",
});
```

Por:
```typescript
// ✅ DESPUÉS
import { env } from '@/lib/env/config';

const res = await fetch(`${env.NEXT_PUBLIC_API_URL}/courses/${slug}`, {
  cache: "no-store",
});
```

**Archivo 3**: `src/app/classes/[class_id]/page.tsx`

Similar cambio.

**Archivo 4**: `src/services/ratingsApi.ts`

Cambiar:
```typescript
// ❌ ANTES
const BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000';
```

Por:
```typescript
// ✅ DESPUÉS
import { env } from '@/lib/env/config';

const BASE_URL = env.NEXT_PUBLIC_API_URL;
```

**Validación**:
- [ ] 0 URLs hardcodeadas (grep "localhost:8000")
- [ ] Todos los imports agregados
- [ ] TypeScript compila sin errores

---

**Paso 4: Crear archivos de entorno locales**

```bash
cd Frontend
touch .env.local
```

Contenido de `.env.local`:
```bash
NEXT_PUBLIC_API_URL=http://localhost:8000
NODE_ENV=development
```

```bash
# NO commitear .env.local
echo ".env.local" >> .gitignore
```

**Validación**:
- [ ] `.env.local` creado
- [ ] Agregado a `.gitignore`
- [ ] Variables coinciden con `.env.example`

---

**Paso 5: Documentar en README**

Actualizar `Frontend/README.md`:

```markdown
## Setup

### 1. Install dependencies

\`\`\`bash
npm install
\`\`\`

### 2. Configure environment variables

\`\`\`bash
cp .env.example .env.local
\`\`\`

Edit `.env.local` and set:

- `NEXT_PUBLIC_API_URL`: Backend API URL (default: http://localhost:8000)

### 3. Run development server

\`\`\`bash
npm run dev
\`\`\`

## Environment Variables

See `.env.example` for all available variables.

**Required**:
- `NEXT_PUBLIC_API_URL`: Backend API base URL

**Optional**:
- `NODE_ENV`: Environment (development/production/test)

⚠️ **Never commit `.env.local`** - it contains local configuration.
```

---

#### Criterios de Aceptación

- [ ] `.env.example` creado y documentado
- [ ] Validación de env con zod implementada
- [ ] 0 URLs hardcodeadas en código
- [ ] `.env.local` en `.gitignore`
- [ ] README actualizado con instrucciones
- [ ] `npm run dev` funciona
- [ ] `npm run build` funciona
- [ ] TypeScript sin errores

#### Testing y Validación

**Test 1: Validación funciona**
```bash
# Borrar NEXT_PUBLIC_API_URL
rm .env.local
npm run dev

# Debe fallar con mensaje claro:
# ❌ Invalid environment variables:
# NEXT_PUBLIC_API_URL: Required
```

**Test 2: URL inválida detectada**
```bash
# En .env.local
NEXT_PUBLIC_API_URL=not-a-url

npm run dev
# Debe fallar: "must be a valid URL"
```

**Test 3: App funciona con env correcto**
```bash
# Restaurar .env.local
NEXT_PUBLIC_API_URL=http://localhost:8000

npm run dev
# Debe funcionar normalmente
```

**Test 4: No hay hardcoded URLs**
```bash
grep -r "localhost:8000" src/
# Debe retornar 0 resultados (excepto comentarios)
```

---

### Issue 1.6: Cliente API Centralizado

**Severidad**: ALTO
**Tiempo estimado**: 3-4 horas
**Dependencias**: Issue 1.1 (Variables de Entorno)
**Riesgo**: Medio

#### Contexto Técnico

**Problema actual**:
- Fetch disperso en múltiples archivos
- Lógica duplicada de error handling
- Difícil agregar headers globales (auth, CSRF)
- No hay retry logic centralizada

**Impacto**:
- Difícil de mantener
- Inconsistente manejo de errores
- No preparado para autenticación

#### Plan de Implementación

**Paso 1: Crear estructura de API client**

```bash
mkdir -p src/lib/api
touch src/lib/api/client.ts
touch src/lib/api/types.ts
touch src/lib/api/errors.ts
```

**Paso 2: Implementar API client base**

Ver agente anterior para código completo (client.ts, types.ts, errors.ts).

**Paso 3: Migrar ratingsApi a usar nuevo client**

```bash
# Refactorizar src/services/ratingsApi.ts
```

**Paso 4: Crear wrappers para fetch en páginas**

```bash
touch src/lib/api/courses.ts
touch src/lib/api/classes.ts
```

**Paso 5: Tests del API client**

```bash
touch src/lib/api/__tests__/client.test.ts
```

#### Criterios de Aceptación

- [ ] API client centralizado creado
- [ ] Manejo de errores unificado
- [ ] Timeout configurado
- [ ] Headers centralizados
- [ ] ratingsApi migrado
- [ ] Tests > 80% coverage
- [ ] Documentación JSDoc completa

---

## 🔒 FASE 2: Seguridad de Red

### Issue 1.2: Content Security Policy (CSP)

**Severidad**: ALTO
**Tiempo estimado**: 2-3 horas
**Dependencias**: Issue 1.1
**Riesgo**: Alto (puede romper funcionalidad)

#### Contexto Técnico

**Problema**: Sin CSP, la aplicación es vulnerable a XSS attacks.

**Impacto**: Código malicioso puede ejecutarse en el contexto del usuario.

#### Plan de Implementación

**Paso 1: Definir política CSP**

Crear `src/lib/security/csp.ts`:

```typescript
/**
 * Content Security Policy Configuration
 */

export function getCSP() {
  const csp = [
    "default-src 'self'",
    "script-src 'self' 'unsafe-eval' 'unsafe-inline'", // Next.js requires unsafe-inline for dev
    "style-src 'self' 'unsafe-inline'",
    "img-src 'self' data: https:",
    "font-src 'self' data:",
    "connect-src 'self' http://localhost:8000",
    "media-src 'self' https://www.youtube.com https://player.vimeo.com",
    "frame-src 'self' https://www.youtube.com https://player.vimeo.com",
  ];

  return csp.join('; ');
}
```

**Paso 2: Configurar en next.config.ts**

```typescript
const nextConfig = {
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: getCSP(),
          },
        ],
      },
    ];
  },
};
```

#### Criterios de Aceptación

- [ ] CSP configurado
- [ ] Videos de YouTube funcionan
- [ ] Imágenes externas permitidas
- [ ] No hay errores CSP en consola
- [ ] Report-only testeado primero

---

## 📊 Orden de Implementación Recomendado

```
┌─────────────────────────────────────────────┐
│ SEMANA 1                                    │
├─────────────────────────────────────────────┤
│ Día 1-2: FASE 1                             │
│   ✓ 1.1 Variables de Entorno (3h)          │
│   ✓ 1.6 Cliente API (4h)                   │
├─────────────────────────────────────────────┤
│ Día 3: FASE 2                               │
│   ✓ 1.2 CSP (3h)                           │
│   ✓ 1.3 Validación URLs (3h)              │
├─────────────────────────────────────────────┤
│ Día 4: FASE 3-4                             │
│   ✓ 1.4 Logging Seguro (2h)               │
│   ✓ 1.5 CSRF Prep (3h)                    │
├─────────────────────────────────────────────┤
│ Día 5: Testing & Docs                       │
│   ✓ Tests de seguridad                     │
│   ✓ Documentación                          │
│   ✓ Review final                           │
└─────────────────────────────────────────────┘

Total: 16-18 horas (~1 semana)
```

---

## ✅ Checklist Final

### Antes de considerar completo:

#### Configuración
- [ ] `.env.example` creado y documentado
- [ ] Validación con zod implementada
- [ ] 0 URLs hardcodeadas (`grep -r "localhost"`)
- [ ] README con instrucciones de setup

#### API Client
- [ ] Cliente centralizado creado
- [ ] Error handling unificado
- [ ] Todos los fetch migrados
- [ ] Tests > 80% coverage

#### Seguridad
- [ ] CSP configurado
- [ ] URLs de imágenes validadas
- [ ] Logger reemplaza console.*
- [ ] Origin header incluido

#### Testing
- [ ] Unit tests pasan
- [ ] Integration tests creados
- [ ] Security scan sin issues críticos
- [ ] Lighthouse > 90

#### Documentación
- [ ] README actualizado
- [ ] SECURITY.md creado
- [ ] JSDoc en funciones públicas
- [ ] Cambios de backend documentados

#### Deployment
- [ ] Build de producción exitoso
- [ ] Env vars configuradas en platform
- [ ] Security headers verificados
- [ ] Smoke tests en staging

---

## 📚 Referencias

- [OWASP CSP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html)
- [OWASP CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [Next.js Security Headers](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy)
- [12 Factor App Config](https://12factor.net/config)

---

**Última actualización**: 2026-01-19
**Versión**: 1.0.0
**Autor**: Claude Code - Frontend Agent
**Revisor**: [Pendiente]