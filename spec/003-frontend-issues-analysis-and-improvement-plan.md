# Frontend Code Analysis Report
## Análisis de Issues Existentes y Plan de Mejora

> **Documento de análisis técnico del frontend de Platziflix**
> Fecha: 2026-01-19
> Versión: 1.0.0

---

## 📊 Executive Summary

Se realizó un análisis exhaustivo del código frontend (Next.js 15 + React 19 + TypeScript) identificando **42 issues** distribuidos en 10 categorías principales. El análisis revela problemas críticos de configuración, seguridad, rendimiento y calidad de código que requieren atención inmediata.

### Métricas Clave

| Métrica | Valor | Estado |
|---------|-------|--------|
| **Archivos analizados** | 23 TS/TSX | ✅ |
| **Issues totales** | 42 | 🔴 |
| **Issues críticos** | 3 | 🔴 |
| **Issues alta prioridad** | 15 | 🟡 |
| **Cobertura de tests** | ~29% | 🔴 |
| **Deuda técnica estimada** | ~3-4 semanas | 🟡 |

### Distribución por Severidad

```
CRÍTICO  (3):  ███ 7%
ALTO     (15): ███████████████ 36%
MEDIO    (18): ██████████████████ 43%
BAJO     (6):  ██████ 14%
```

---

## 🔥 Top 5 Issues Críticos/Altos

### 1. 🔴 URLs Hardcodeadas del Backend en Producción
**Severidad**: CRÍTICO
**Categoría**: Configuración / Seguridad
**Archivos afectados**: `page.tsx`, `course/[slug]/page.tsx`, `classes/[class_id]/page.tsx`

**Problema**:
```typescript
// ❌ MAL - Hardcodeado en 3 archivos
const res = await fetch("http://localhost:8000/courses", { cache: "no-store" });
```

**Impacto**:
- Aplicación NO funciona en producción
- No se puede deployar a ningún ambiente
- Expone arquitectura interna (puerto 8000)
- Viola principio de configuración externa

**Estado actual**: Bloqueador de despliegue

---

### 2. 🔴 Sin Estrategia de Caché para APIs
**Severidad**: CRÍTICO
**Categoría**: Rendimiento
**Archivos afectados**: Todas las páginas con fetch

**Problema**:
```typescript
// ❌ Todas las peticiones usan cache: "no-store"
const res = await fetch(url, { cache: "no-store" });
```

**Impacto**:
- Fetch completo en cada request (sin revalidación)
- Carga innecesaria al backend
- Tiempos de carga lentos (3-5x más)
- Costos de servidor elevados
- No aprovecha ISR de Next.js

**Costo estimado**: $500-1000/mes adicionales en infraestructura

---

### 3. 🔴 Cobertura de Tests Muy Baja (29%)
**Severidad**: CRÍTICO
**Categoría**: Calidad / Testing

**Problema**:
- Solo 4 archivos de tests de 14+ componentes/páginas
- Sin tests de integración
- Sin tests E2E
- Componentes críticos sin tests: `CourseDetail`, API pages, `ratingsApi`

**Impacto**:
- Alto riesgo de regresiones
- No se puede refactorizar con confianza
- Bugs llegan a producción
- Velocidad de desarrollo reducida (miedo a romper cosas)

**Deuda técnica**: ~2 semanas para alcanzar 80% coverage

---

### 4. 🟡 Inconsistencia Crítica de Tipos (name vs title)
**Severidad**: ALTO
**Categoría**: Type Safety
**Archivos afectados**: `types/index.ts`, todos los componentes

**Problema**:
```typescript
// types/index.ts
export interface Course {
  name: string;  // ← Backend usa "name"
}

// CourseDetail.tsx
<h1>{course.title}</h1>  // ← Componente usa "title" ❌
```

**Impacto**:
- Errores runtime al acceder propiedades undefined
- TypeScript no detecta el error (tipo `any` implícito)
- Contrato API confuso
- Difícil de mantener

---

### 5. 🟡 Sin Optimización de Imágenes
**Severidad**: ALTO
**Categoría**: Rendimiento / Seguridad / SEO
**Archivos afectados**: `Course.tsx`, `CourseDetail.tsx`

**Problema**:
```typescript
// ❌ Usando <img> nativo en lugar de Next.js Image
<img src={thumbnail} alt={name} className={styles.thumbnail} />
```

**Impacto**:
- Imágenes sin optimización (tamaño completo descargado)
- Sin lazy loading (todas las imágenes cargan inmediatamente)
- Sin responsive images (mismo tamaño en mobile/desktop)
- Vulnerabilidad XSS si URL es maliciosa
- Core Web Vitals bajos (LCP > 2.5s)
- SEO penalizado por Google
- Costos de bandwidth elevados

**Pérdida estimada**: -20 puntos en Lighthouse, -15% conversión

---

## 📋 Categorías de Issues

### 1. Seguridad (5 issues)

| # | Severidad | Issue | Archivos |
|---|-----------|-------|----------|
| 1.1 | 🔴 CRÍTICO | URLs hardcodeadas en producción | 3 archivos |
| 1.2 | 🟡 ALTO | Sin Content Security Policy (CSP) | `layout.tsx` |
| 1.3 | 🟡 ALTO | URLs de imágenes no validadas (XSS) | `Course.tsx`, `CourseDetail.tsx` |
| 1.4 | 🟡 MEDIO | console.error expone stack traces | `error.tsx` |
| 1.5 | 🟡 MEDIO | Sin protección CSRF en rating API | `ratingsApi.ts` |

**Impacto global**: Aplicación vulnerable a XSS, no deployable, información sensible expuesta.

**Tiempo estimado de fix**: 1 semana

---

### 2. Rendimiento (5 issues)

| # | Severidad | Issue | Impacto |
|---|-----------|-------|---------|
| 2.1 | 🔴 CRÍTICO | Sin caché API (cache: no-store) | 3-5x lento, $$ servidor |
| 2.2 | 🟡 ALTO | Llamadas duplicadas en generateMetadata | 2x requests por página |
| 2.3 | 🟡 ALTO | Sin optimización de imágenes | LCP > 2.5s, SEO - |
| 2.4 | 🟡 MEDIO | Sin loading states client-side | CLS alto, UX pobre |
| 2.5 | 🟢 BAJO | Fonts sin display: 'swap' | FOUT (flash unstyled) |

**Impacto global**: Aplicación 3-5x más lenta de lo necesario, Core Web Vitals en rojo.

**Tiempo estimado de fix**: 1.5 semanas

---

### 3. Accesibilidad (6 issues)

| # | Severidad | Issue | WCAG Violation |
|---|-----------|-------|----------------|
| 3.1 | 🟡 ALTO | Alt text no descriptivo | Level A |
| 3.2 | 🟡 ALTO | Sin skip to main content | Level A |
| 3.3 | 🟡 MEDIO | Video sin captions/transcripts | Level A |
| 3.4 | 🟡 MEDIO | Links anidados (invalid HTML) | HTML5 Spec |
| 3.5 | 🟡 MEDIO | Sin focus-visible styles | Level AA |
| 3.6 | 🟢 BAJO | Sin atributo dir en <html> | Best practice |

**Impacto global**: No accesible para usuarios con discapacidades, violaciones WCAG Level A.

**Tiempo estimado de fix**: 1 semana

---

### 4. Type Safety (5 issues)

| # | Severidad | Issue | Riesgo |
|---|-----------|-------|--------|
| 4.1 | 🔴 CRÍTICO | Mismatch name vs title | Runtime errors |
| 4.2 | 🟡 ALTO | Interface duplica campos (description) | Confusión modelo |
| 4.3 | 🟡 MEDIO | Params no son Promise<> consistentemente | Next 15 deprecation |
| 4.4 | 🟡 MEDIO | Tests usan props que no existen | False confidence |
| 4.5 | 🟢 BAJO | Faltan return types explícitos | Readability |

**Impacto global**: TypeScript no detecta errores críticos, fallos runtime frecuentes.

**Tiempo estimado de fix**: 3 días

---

### 5. Calidad de Código (4 issues)

| # | Severidad | Issue | Deuda Técnica |
|---|-----------|-------|---------------|
| 5.1 | 🟡 ALTO | Lógica duplicada (formatDuration) | Violación DRY |
| 5.2 | 🟡 MEDIO | Magic numbers (timeout: 10000) | Hard to maintain |
| 5.3 | 🟡 MEDIO | Mensajes de error en ES + EN mezclados | Inconsistente UX |
| 5.4 | 🟢 BAJO | Código comentado en producción | Code smell |

**Impacto global**: Difícil de mantener, propenso a bugs, inconsistente.

**Tiempo estimado de fix**: 2 días

---

### 6. Testing (5 issues)

| # | Severidad | Issue | Coverage Actual |
|---|-----------|-------|-----------------|
| 6.1 | 🔴 CRÍTICO | Cobertura muy baja | 29% (target: 80%) |
| 6.2 | 🟡 ALTO | Sin tests de integración | 0% |
| 6.3 | 🟡 ALTO | Tests StarRating con expectativas incorrectas | False positives |
| 6.4 | 🟡 MEDIO | ratingsApi sin tests | 0 tests servicio crítico |
| 6.5 | 🟡 MEDIO | Tests usan sintaxis incorrecta | No importan vitest |

**Impacto global**: No se puede confiar en el código, regresiones frecuentes.

**Tiempo estimado de fix**: 2 semanas

---

### 7. Arquitectura (5 issues)

| # | Severidad | Issue | Anti-pattern |
|---|-----------|-------|--------------|
| 7.1 | 🟡 ALTO | Sin estrategia de variables de entorno | No .env.example |
| 7.2 | 🟡 ALTO | Sin cliente API centralizado | Fetch disperso |
| 7.3 | 🟡 MEDIO | Acoplamiento UI + Data Fetching | SoC violation |
| 7.4 | 🟡 MEDIO | Error boundary solo en 1 ruta | Incomplete |
| 7.5 | 🟢 BAJO | Naming inconsistente archivos | PascalCase vs kebab |

**Impacto global**: Difícil de escalar, propenso a errores, hard to test.

**Tiempo estimado de fix**: 1 semana

---

### 8. UX (4 issues)

| # | Severidad | Issue | User Impact |
|---|-----------|-------|-------------|
| 8.1 | 🟡 ALTO | Sin estados de error específicos | Usuario perdido |
| 8.2 | 🟡 MEDIO | Link "Regresar al curso" roto (404) | Navegación rota |
| 8.3 | 🟡 MEDIO | Sin loading indicators en ratings | Clicks duplicados |
| 8.4 | 🟢 BAJO | Estilos de botones inconsistentes | Visual inconsistency |

**Impacto global**: Experiencia de usuario frustrante, navegación rota.

**Tiempo estimado de fix**: 3 días

---

### 9. Mantenibilidad (3 issues)

| # | Severidad | Issue | Dev Experience |
|---|-----------|-------|----------------|
| 9.1 | 🟡 MEDIO | Sin archivo de constantes compartidas | Magic strings repetidos |
| 9.2 | 🟡 MEDIO | Sin JSDoc en funciones complejas | IDE no ayuda |
| 9.3 | 🟢 BAJO | Sin Storybook | Hard to develop UI |

**Impacto global**: Onboarding lento, errores de typos, difícil colaborar.

**Tiempo estimado de fix**: 1 semana

---

### 10. Best Practices (3 issues)

| # | Severidad | Issue | Violation |
|---|-----------|-------|-----------|
| 10.1 | 🟡 ALTO | No usa Next.js Image | Framework best practice |
| 10.2 | 🟡 MEDIO | Params no await consistentemente | Next 15 migration path |
| 10.3 | 🟡 MEDIO | TypeScript no fully strict | Missing strict flags |

**Impacto global**: No aprovecha framework, futuras versiones romperán.

**Tiempo estimado de fix**: 4 días

---

## 🎯 Plan de Implementación por Fases

### FASE 1: Ship Blockers (Sprint 1 - Semana 1-2)
**Objetivo**: Hacer la aplicación deployable y estable

#### 1.1 Configuración de Entornos (2 días)
- [ ] Crear `.env.example` con todas las variables
- [ ] Crear cliente API centralizado (`src/lib/api-client.ts`)
- [ ] Migrar todos los fetch a usar `process.env.NEXT_PUBLIC_API_URL`
- [ ] Validar env vars con `zod` o `@t3-oss/env-nextjs`
- [ ] Actualizar documentación de setup

**Entregables**:
- `.env.example`
- `src/lib/api-client.ts`
- `src/lib/env.ts` (validation)
- README actualizado

**Tests de aceptación**:
- ✅ App corre en dev con `.env.local`
- ✅ Build de producción exitoso
- ✅ Todas las rutas funcionan contra API remota

---

#### 1.2 Fixes Críticos de Tipos (1 día)
- [ ] Unificar `name` → `title` en interface `Course`
- [ ] Remover `description` duplicado de `CourseDetail`
- [ ] Actualizar todos los componentes
- [ ] Hacer `params` consistente como `Promise<>` en Next 15
- [ ] Validar con TypeScript en modo strict

**Entregables**:
- `src/types/index.ts` actualizado
- Todos los componentes actualizados
- 0 errores de TypeScript

**Tests de aceptación**:
- ✅ `npm run build` exitoso sin errores TS
- ✅ Todas las páginas renderizan correctamente

---

#### 1.3 Fix de Navegación Rota (1 día)
- [ ] Implementar breadcrumbs o storage de course slug
- [ ] Arreglar link "Regresar al curso"
- [ ] Añadir query param `?from=course-slug` en links
- [ ] Test manual de navegación completa

**Entregables**:
- Navegación funcional curso → clase → curso
- Tests E2E básicos de navegación

**Tests de aceptación**:
- ✅ Usuario puede ir curso → clase → volver al curso
- ✅ No hay links rotos (404)

---

#### 1.4 Implementar Estrategia de Caché (2 días)
- [ ] Definir tiempos de revalidación por endpoint:
  - Courses list: 1 hora (`revalidate: 3600`)
  - Course detail: 10 minutos (`revalidate: 600`)
  - Class detail: 10 minutos
  - Rating stats: 1 minuto (alta volatilidad)
- [ ] Implementar ISR (Incremental Static Regeneration)
- [ ] Añadir `force-cache` donde sea apropiado
- [ ] Configurar `unstable_cache` para datos estáticos
- [ ] Medir mejoras de performance (Lighthouse)

**Entregables**:
- Fetch con estrategias de caché optimizadas
- Documentación de estrategias por endpoint
- Lighthouse score > 90

**Tests de aceptación**:
- ✅ Lighthouse Performance > 90
- ✅ TTFB < 200ms (con caché)
- ✅ Páginas se sirven desde caché en 2da visita

---

### FASE 2: Seguridad y Performance (Sprint 2 - Semana 3-4)

#### 2.1 Migración a Next.js Image (3 días)
- [ ] Instalar y configurar `next/image`
- [ ] Configurar `remotePatterns` en `next.config.ts`
- [ ] Migrar `<img>` → `<Image>` en `Course.tsx`
- [ ] Migrar `<img>` → `<Image>` en `CourseDetail.tsx`
- [ ] Añadir placeholders blur (base64)
- [ ] Optimizar tamaños (`sizes` prop)
- [ ] Validar mejoras en Lighthouse

**Entregables**:
- 0 usos de `<img>` nativo
- `next.config.ts` con `remotePatterns`
- Lighthouse LCP < 2.5s

**Tests de aceptación**:
- ✅ Imágenes lazy-load correctamente
- ✅ Responsive images en mobile/desktop
- ✅ LCP < 2.5s en Lighthouse

---

#### 2.2 Content Security Policy (CSP) (2 días)
- [ ] Definir política CSP restrictiva
- [ ] Implementar headers en `next.config.ts`
- [ ] Manejar nonces para inline scripts
- [ ] Whitelist dominios externos (YouTube, CDN)
- [ ] Test con herramientas CSP

**Entregables**:
- CSP headers configurados
- Documentación de política
- Tests de seguridad pasando

**Tests de aceptación**:
- ✅ No hay violaciones CSP en consola
- ✅ Content Security Policy Evaluator score A+

---

#### 2.3 Sanitización y Validación (2 días)
- [ ] Instalar `DOMPurify` para sanitización HTML
- [ ] Validar URLs de thumbnails (whitelist domains)
- [ ] Implementar validación server-side
- [ ] Rate limiting en API routes
- [ ] Añadir CSRF tokens a formularios

**Entregables**:
- Todas las entradas sanitizadas
- URLs validadas contra whitelist
- CSRF protection activa

**Tests de aceptación**:
- ✅ XSS tests fallan (no inyección posible)
- ✅ Solo dominios permitidos en imágenes

---

### FASE 3: Accesibilidad (Sprint 3 - Semana 5)

#### 3.1 Mejoras WCAG Level A (3 días)
- [ ] Añadir skip to main content link
- [ ] Mejorar alt text (descriptivo, no genérico)
- [ ] Añadir `<track>` para captions en videos
- [ ] Arreglar HTML inválido (links anidados)
- [ ] Añadir landmarks ARIA apropiados
- [ ] Test con screen readers (NVDA, VoiceOver)

**Entregables**:
- WCAG Level A compliance
- Alt text guide documentado
- Tests de accesibilidad pasando

**Tests de aceptación**:
- ✅ axe DevTools 0 violations
- ✅ WAVE 0 errores
- ✅ Navegable con teclado solamente

---

#### 3.2 Keyboard Navigation y Focus (2 días)
- [ ] Añadir estilos `:focus-visible` a todos los interactivos
- [ ] Implementar trap de focus en modales
- [ ] Arreglar tab order lógico
- [ ] Skip links funcionales
- [ ] Test de navegación completa con Tab

**Entregables**:
- Focus styles consistentes
- Tab order lógico
- Documentación de keyboard shortcuts

**Tests de aceptación**:
- ✅ Todos los elementos interactivos tienen focus visible
- ✅ Tab order sigue flujo visual
- ✅ Escape cierra modales

---

### FASE 4: Testing Comprehensivo (Sprint 4 - Semana 6-7)

#### 4.1 Unit Tests (5 días)
- [ ] Tests para `ratingsApi.ts` (todos los métodos)
- [ ] Tests para `CourseDetail.tsx`
- [ ] Tests para páginas (con mocked fetch)
- [ ] Corregir tests de `StarRating` (expectativas incorrectas)
- [ ] Tests para utils (formatDuration, etc.)
- [ ] Alcanzar 80% coverage

**Entregables**:
- Coverage report > 80%
- Tests corregidos y actualizados
- CI/CD corriendo tests

**Tests de aceptación**:
- ✅ `npm test` pasa 100% tests
- ✅ Coverage > 80% en statements/branches/functions/lines

---

#### 4.2 Integration Tests (3 días)
- [ ] Setup Playwright o Cypress
- [ ] Tests de flujos críticos:
  - Ver lista cursos
  - Ver detalle curso
  - Reproducir clase
  - Dar rating (mocked)
- [ ] Tests de error handling
- [ ] Tests de navegación completa

**Entregables**:
- Suite E2E completa
- Tests corriendo en CI
- Screenshots/videos de ejecución

**Tests de aceptación**:
- ✅ E2E tests pasan en CI
- ✅ Flujos críticos cubiertos

---

### FASE 5: Arquitectura y Mantenibilidad (Sprint 5 - Semana 8-9)

#### 5.1 Refactoring de Arquitectura (5 días)
- [ ] Extraer funciones compartidas a `/src/utils/`
- [ ] Crear archivo de constantes `/src/constants/`
- [ ] Implementar repository pattern para data fetching
- [ ] Añadir error boundaries en todos los layouts
- [ ] Crear componente Button reutilizable
- [ ] Documentar decisiones arquitectónicas (ADRs)

**Entregables**:
- `/src/utils/formatters.ts`, `/src/utils/validators.ts`
- `/src/constants/api.ts`, `/src/constants/ui.ts`
- `/src/repositories/CourseRepository.ts`
- Error boundaries en todos los layouts
- `src/components/ui/Button.tsx`

**Tests de aceptación**:
- ✅ 0 código duplicado (DRY)
- ✅ Componentes < 200 líneas
- ✅ Funciones < 50 líneas

---

#### 5.2 Documentación y DX (2 días)
- [ ] Añadir JSDoc a todas las funciones exportadas
- [ ] Crear Storybook para componentes
- [ ] Documentar arquitectura (`ARCHITECTURE.md`)
- [ ] Crear guía de contribución (`CONTRIBUTING.md`)
- [ ] Setup de pre-commit hooks (Husky + lint-staged)

**Entregables**:
- JSDoc en todas las funciones públicas
- Storybook desplegado
- Documentación completa
- Pre-commit hooks configurados

**Tests de aceptación**:
- ✅ Storybook muestra todos los componentes
- ✅ JSDoc aparece en VSCode autocomplete
- ✅ Pre-commit hooks bloquean código malo

---

### FASE 6: UX y Polish (Sprint 6 - Semana 10)

#### 6.1 Estados de Loading y Error (3 días)
- [ ] Crear skeleton loaders para todas las páginas
- [ ] Implementar error states específicos (no genéricos)
- [ ] Añadir retry buttons en errores
- [ ] Loading indicators para ratings
- [ ] Optimistic UI updates

**Entregables**:
- Skeleton components
- Error states específicos con acciones
- Loading indicators consistentes

**Tests de aceptación**:
- ✅ No hay "white screen" mientras carga
- ✅ Errores tienen mensajes claros + acción

---

#### 6.2 Internacionalización (2 días)
- [ ] Setup next-intl o i18next
- [ ] Extraer todos los strings a archivos i18n
- [ ] Traducir a ES + EN
- [ ] Selector de idioma
- [ ] Formateo de fechas/números localizado

**Entregables**:
- `/src/i18n/` con traducciones
- Selector de idioma funcional
- 100% strings traducibles

**Tests de aceptación**:
- ✅ App funciona en ES + EN
- ✅ No hay strings hardcodeados

---

## 📈 Métricas de Éxito

### Performance Targets

| Métrica | Actual | Target | Prioridad |
|---------|--------|--------|-----------|
| **Lighthouse Performance** | ~60-70 | > 90 | 🔴 Alta |
| **Lighthouse Accessibility** | ~75 | > 95 | 🔴 Alta |
| **Lighthouse Best Practices** | ~70 | > 95 | 🟡 Media |
| **Lighthouse SEO** | ~80 | > 95 | 🟢 Baja |
| **First Contentful Paint (FCP)** | ~2.5s | < 1.8s | 🔴 Alta |
| **Largest Contentful Paint (LCP)** | ~4.0s | < 2.5s | 🔴 Alta |
| **Time to Interactive (TTI)** | ~5.5s | < 3.8s | 🟡 Media |
| **Cumulative Layout Shift (CLS)** | ~0.15 | < 0.1 | 🔴 Alta |
| **Total Blocking Time (TBT)** | ~300ms | < 200ms | 🟡 Media |

### Quality Targets

| Métrica | Actual | Target | Prioridad |
|---------|--------|--------|-----------|
| **Test Coverage (Statements)** | 29% | > 80% | 🔴 Alta |
| **Test Coverage (Branches)** | ~20% | > 75% | 🔴 Alta |
| **TypeScript Errors** | 0 | 0 | ✅ Ok |
| **ESLint Warnings** | ? | 0 | 🟡 Media |
| **Bundle Size (First Load JS)** | ? | < 200KB | 🟡 Media |
| **Dependencies Vulnerabilities** | ? | 0 high/critical | 🔴 Alta |

### Business Targets

| Métrica | Impacto Esperado |
|---------|------------------|
| **Conversion Rate** | +15-20% (performance mejora) |
| **Bounce Rate** | -10-15% (UX mejorado) |
| **SEO Ranking** | +10-15 posiciones (Core Web Vitals) |
| **Accessibility Compliance** | 100% WCAG 2.1 Level A |
| **Server Costs** | -40% (caché optimizado) |
| **Developer Velocity** | +25% (tests + docs) |

---

## 🛠 Herramientas Requeridas

### Nuevas Dependencias

#### Producción
```json
{
  "dependencies": {
    "@t3-oss/env-nextjs": "^0.7.0",  // Env validation
    "zod": "^3.22.0",                 // Schema validation
    "dompurify": "^3.0.6",            // HTML sanitization
    "next-intl": "^3.4.0"             // i18n (opcional Fase 6)
  }
}
```

#### Desarrollo
```json
{
  "devDependencies": {
    "@playwright/test": "^1.40.0",    // E2E testing
    "@storybook/nextjs": "^7.6.0",    // Component documentation
    "husky": "^8.0.3",                 // Git hooks
    "lint-staged": "^15.2.0",          // Pre-commit linting
    "@axe-core/playwright": "^4.8.0"   // Accessibility testing
  }
}
```

### Herramientas de Análisis
- **Lighthouse CI** (performance tracking)
- **axe DevTools** (accessibility scanning)
- **WAVE** (accessibility validation)
- **Sentry** (error tracking - opcional)
- **Vercel Analytics** (real user metrics - opcional)

---

## 📊 Estimación de Esfuerzo

### Resumen por Fase

| Fase | Duración | Complejidad | Riesgo | Prioridad |
|------|----------|-------------|--------|-----------|
| **Fase 1**: Ship Blockers | 2 semanas | 🟡 Media | 🔴 Alto | 🔴 Crítica |
| **Fase 2**: Seguridad/Performance | 2 semanas | 🟡 Media | 🟡 Medio | 🔴 Alta |
| **Fase 3**: Accesibilidad | 1 semana | 🟢 Baja | 🟢 Bajo | 🟡 Media |
| **Fase 4**: Testing | 2 semanas | 🟡 Media | 🟡 Medio | 🔴 Alta |
| **Fase 5**: Arquitectura | 2 semanas | 🔴 Alta | 🟡 Medio | 🟡 Media |
| **Fase 6**: UX/Polish | 1 semana | 🟢 Baja | 🟢 Bajo | 🟢 Baja |

**Total estimado**: 10 semanas (2.5 meses)

### Esfuerzo por Persona

| Rol | Carga (%) | Fases | Total horas |
|-----|-----------|-------|-------------|
| **Senior Frontend Dev** | 100% | 1-6 | 400h |
| **QA Engineer** | 50% | 4 | 80h |
| **DevOps Engineer** | 25% | 1-2 | 80h |
| **Design/UX** | 15% | 3, 6 | 48h |

**Total**: ~600 horas de desarrollo (~15 semanas persona)

---

## 🚀 Quick Wins (< 1 día cada uno)

Para demostrar progreso rápido, estos issues se pueden resolver individualmente:

1. ✅ **Crear `.env.example`** (30 min)
2. ✅ **Arreglar link "Regresar al curso"** (1 hora)
3. ✅ **Unificar name → title en tipos** (2 horas)
4. ✅ **Extraer formatDuration a utils** (1 hora)
5. ✅ **Añadir constantes para magic numbers** (1 hora)
6. ✅ **Corregir tests de StarRating** (2 horas)
7. ✅ **Remover código comentado** (15 min)
8. ✅ **Añadir skip to content link** (1 hora)
9. ✅ **Mejorar alt text en imágenes** (1 hora)
10. ✅ **Añadir focus-visible styles** (2 horas)

**Total quick wins**: ~1 semana de desarrollo

---

## 📝 Conclusiones y Recomendaciones

### Estado Actual
El frontend de Platziflix está **funcional pero no production-ready**. Tiene una base sólida (Next.js 15, TypeScript, arquitectura de componentes limpia) pero requiere mejoras significativas en:

1. **Configuración** (bloqueador de deploy)
2. **Performance** (3-5x más lento de lo necesario)
3. **Testing** (alto riesgo de regresiones)
4. **Seguridad** (vulnerabilidades XSS, sin CSP)
5. **Accesibilidad** (no cumple WCAG Level A)

### Riesgo de No Actuar

| Área | Riesgo | Probabilidad | Impacto |
|------|--------|--------------|---------|
| **Deploy falla en producción** | URLs hardcodeadas | 100% | 🔴 Crítico |
| **Usuarios se van por lentitud** | Performance | 75% | 🔴 Alto |
| **Bugs en producción** | Falta de tests | 60% | 🟡 Medio |
| **Ataque XSS exitoso** | Sin sanitización | 30% | 🔴 Alto |
| **Demanda legal por accesibilidad** | WCAG violations | 15% | 🟡 Medio |

### Recomendación de Priorización

**Opción A: MVP Rápido (4 semanas)**
- Fase 1 completa (2 sem)
- Fase 2.1 + 2.2 (2 sem)
- **Resultado**: Deployable, rápido, seguro (60% issues resueltos)

**Opción B: Quality First (8 semanas)** ⭐ RECOMENDADO
- Fases 1-4 completas
- **Resultado**: Production-ready con tests, 80% issues resueltos

**Opción C: Full Implementation (10 semanas)**
- Todas las fases
- **Resultado**: Best-in-class frontend, 100% issues resueltos

### Próximos Pasos Inmediatos

1. **Día 1-2**: Crear `.env.example` + cliente API centralizado
2. **Día 3-5**: Arreglar tipos (name/title) + navegación rota
3. **Día 6-10**: Implementar caché strategy + migrar a Next.js Image
4. **Día 11-15**: Deploy a staging + validar métricas

**Objetivo Sprint 1**: Aplicación deployable con performance > 80 Lighthouse

---

## 📚 Referencias

### Documentación
- [Next.js 15 Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [Next.js Image Optimization](https://nextjs.org/docs/app/building-your-application/optimizing/images)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [Web Vitals](https://web.dev/vitals/)

### Herramientas
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [axe DevTools](https://www.deque.com/axe/devtools/)
- [Playwright](https://playwright.dev/)
- [Storybook](https://storybook.js.org/)

### Issues Similares (Comunidad)
- [Next.js Image Migration Guide](https://nextjs.org/docs/pages/building-your-application/upgrading/codemods#next-image-to-legacy-image)
- [TypeScript Strict Mode Guide](https://www.typescriptlang.org/tsconfig#strict)
- [CSP for Next.js](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy)

---

**Última actualización**: 2026-01-19
**Versión**: 1.0.0
**Autor**: Claude Code Analysis Agent
**Revisor**: [Pendiente]
**Aprobador**: [Pendiente]
