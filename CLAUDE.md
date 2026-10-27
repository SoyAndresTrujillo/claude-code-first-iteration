# Platziflix - Documentación de Arquitectura y Contexto

> **Documento de Memoria para Claude Code**
> Este archivo contiene toda la información arquitectónica del proyecto Platziflix para servir como contexto en futuras sesiones de desarrollo.

---

## 📋 Tabla de Contenidos

1. [Visión General del Proyecto](#visión-general-del-proyecto)
2. [Estructura del Proyecto](#estructura-del-proyecto)
3. [Stack Tecnológico](#stack-tecnológico)
4. [Arquitectura del Sistema](#arquitectura-del-sistema)
5. [Base de Datos](#base-de-datos)
6. [API REST](#api-rest)
7. [Frontend](#frontend)
8. [Mobile](#mobile)
9. [Patrones y Convenciones](#patrones-y-convenciones)
10. [Comandos Útiles](#comandos-útiles)
11. [Estado Actual del Proyecto](#estado-actual-del-proyecto)
12. [Próximos Pasos](#próximos-pasos)

---

## 🎯 Visión General del Proyecto

**Platziflix** es una plataforma educativa minimalista para cursos en línea que demuestra desarrollo full-stack moderno a través de web y móvil.

**Misión Principal**: Proporcionar una plataforma simple para explorar y tomar cursos con funcionalidad de calificación integrada.

### Componentes Principales

```
┌─────────────────────────────────────┐
│     Clientes (UI Layer)             │
│  • Frontend Web (Next.js)           │
│  • iOS App (Swift/SwiftUI)          │
│  • Android App (Kotlin/Compose)     │
└──────────────┬──────────────────────┘
               │ HTTP/REST
┌──────────────▼──────────────────────┐
│  Backend API (FastAPI)              │
│  • Lógica de negocio                │
│  • Servicio de ratings              │
│  • Gestión de cursos                │
└──────────────┬──────────────────────┘
               │ SQL/TCP
┌──────────────▼──────────────────────┐
│  Base de Datos (PostgreSQL)         │
│  • Cursos y lecciones               │
│  • Profesores                       │
│  • Sistema de ratings               │
└─────────────────────────────────────┘
```

---

## 📁 Estructura del Proyecto

```
claude-code-first-iteration/
├── Backend/                          # FastAPI + PostgreSQL
│   ├── app/
│   │   ├── main.py                   # Punto de entrada FastAPI
│   │   ├── core/                     # Configuración global
│   │   ├── models/                   # Modelos SQLAlchemy (ORM)
│   │   ├── schemas/                  # Schemas Pydantic (validación)
│   │   ├── services/                 # Lógica de negocio
│   │   ├── db/                       # Configuración DB + seed
│   │   ├── alembic/                  # Migraciones
│   │   └── tests/                    # Tests unitarios e integración
│   ├── docker-compose.yml
│   ├── Dockerfile
│   ├── Makefile
│   └── pyproject.toml
│
├── Frontend/                         # Next.js 15 + React 19
│   ├── src/
│   │   ├── app/                      # App Router (páginas)
│   │   │   ├── page.tsx              # Home (lista cursos)
│   │   │   ├── course/[slug]/        # Detalle curso
│   │   │   └── classes/[class_id]/   # Reproductor lección
│   │   ├── components/               # Componentes React
│   │   │   ├── Course/
│   │   │   ├── CourseDetail/
│   │   │   ├── VideoPlayer/
│   │   │   └── StarRating/           # Sistema de estrellas
│   │   ├── services/                 # API clients
│   │   │   └── ratingsApi.ts
│   │   └── types/                    # TypeScript types
│   ├── package.json
│   ├── next.config.ts
│   └── vitest.config.ts
│
├── Mobile/
│   ├── PlatziFlixiOS/                # Swift + SwiftUI
│   │   ├── PlatziFlixiOS/
│   │   │   ├── Data/                 # DTOs, Repositories, Mappers
│   │   │   ├── Domain/               # Models, Protocols
│   │   │   ├── Presentation/         # ViewModels, Views
│   │   │   └── Services/             # NetworkManager
│   │   └── PlatziFlixiOSTests/
│   │
│   └── PlatziFlixAndroid/            # Kotlin + Jetpack Compose
│       ├── app/src/main/java/...
│       │   ├── di/                   # Dependency Injection
│       │   ├── data/                 # DTOs, Repositories, Mappers
│       │   ├── domain/               # Models, Interfaces
│       │   ├── presentation/         # ViewModels, Screens, Components
│       │   └── ui/theme/
│       └── app/build.gradle.kts
│
├── claude.md                         # Este archivo
└── README.md
```

---

## 🛠 Stack Tecnológico

### Backend
- **Framework**: FastAPI 0.104.0+
- **Lenguaje**: Python 3.11+
- **ORM**: SQLAlchemy 2.0+
- **Base de Datos**: PostgreSQL
- **Migraciones**: Alembic 1.13.0+
- **Servidor**: Uvicorn (ASGI)
- **Validación**: Pydantic 2.0+
- **Testing**: pytest, httpx

### Frontend
- **Framework**: Next.js 15.3.3 (App Router)
- **Runtime**: React 19.0.0
- **Lenguaje**: TypeScript
- **Estilos**: SASS/SCSS + CSS Modules
- **Testing**: Vitest + React Testing Library
- **Build**: Turbopack

### iOS
- **Lenguaje**: Swift 5.9+
- **UI**: SwiftUI
- **Networking**: URLSession + Codable
- **Estado**: Combine (@Published, ObservableObject)
- **Arquitectura**: CLEAR (Clean, Layered, Explicit, Adaptive, Responsible)

### Android
- **Lenguaje**: Kotlin 1.9+
- **UI**: Jetpack Compose + Material3
- **Networking**: Retrofit 2.9 + OkHttp 4.12
- **Estado**: StateFlow + Coroutines
- **Arquitectura**: CLEAR + MVI Pattern
- **DI**: Manual (AppModule)

---

## 🏗 Arquitectura del Sistema

### Patrón General: Layered Architecture

Todos los componentes siguen una arquitectura en capas con separación clara de responsabilidades:

#### Backend: Service Layer Pattern

```
┌─────────────────────────────────────┐
│  Routing Layer (main.py)            │  ← Endpoints HTTP
├─────────────────────────────────────┤
│  Validation Layer (Pydantic)        │  ← Schemas de entrada/salida
├─────────────────────────────────────┤
│  Service Layer (services/)          │  ← Lógica de negocio
├─────────────────────────────────────┤
│  ORM Layer (SQLAlchemy)             │  ← Modelos de datos
├─────────────────────────────────────┤
│  Database (PostgreSQL)              │  ← Persistencia
└─────────────────────────────────────┘
```

**Flujo de una petición**:
1. Request → Endpoint en `main.py`
2. Validación → Pydantic Schema
3. Lógica → CourseService
4. Query → SQLAlchemy ORM
5. Database → PostgreSQL
6. Response ← Serialización JSON

#### Mobile: CLEAR Architecture

```
┌─────────────────────────────────────┐
│  Presentation Layer                 │
│  • ViewModels (estado)              │
│  • Views/Screens (UI)               │
├─────────────────────────────────────┤
│  Domain Layer                       │
│  • Models (entidades de negocio)    │
│  • Repository Protocols             │
├─────────────────────────────────────┤
│  Data Layer                         │
│  • DTOs (transferencia de datos)    │
│  • Mappers (DTO → Model)            │
│  • Repository Implementations       │
│  • Network Services                 │
└─────────────────────────────────────┘
```

**Flujo de datos**:
1. View → Emite evento a ViewModel
2. ViewModel → Llama Repository
3. Repository → API HTTP (NetworkService)
4. DTO → Mapper transforma a Model
5. Model → ViewModel actualiza estado
6. Estado → View se re-renderiza

---

## 💾 Base de Datos

### Esquema PostgreSQL

#### Tabla: `teachers`
```sql
CREATE TABLE teachers (
    id INTEGER PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    deleted_at TIMESTAMP NULL
);
CREATE INDEX ix_teachers_email ON teachers(email);
```

#### Tabla: `courses`
```sql
CREATE TABLE courses (
    id INTEGER PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    thumbnail VARCHAR(500) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    deleted_at TIMESTAMP NULL
);
CREATE INDEX ix_courses_slug ON courses(slug);
```

#### Tabla: `lessons` (también llamadas `classes`)
```sql
CREATE TABLE lessons (
    id INTEGER PRIMARY KEY,
    course_id INTEGER NOT NULL REFERENCES courses(id),
    name VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    slug VARCHAR(255) NOT NULL,
    video_url VARCHAR(500) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    deleted_at TIMESTAMP NULL
);
CREATE INDEX ix_lessons_course_id ON lessons(course_id);
```

#### Tabla: `course_teachers` (Many-to-Many)
```sql
CREATE TABLE course_teachers (
    course_id INTEGER REFERENCES courses(id),
    teacher_id INTEGER REFERENCES teachers(id),
    PRIMARY KEY (course_id, teacher_id)
);
```

#### Tabla: `course_ratings` ⭐ NUEVO
```sql
CREATE TABLE course_ratings (
    id INTEGER PRIMARY KEY,
    course_id INTEGER NOT NULL REFERENCES courses(id),
    user_id INTEGER NOT NULL,
    rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    deleted_at TIMESTAMP NULL,
    CONSTRAINT unique_active_rating UNIQUE (course_id, user_id, deleted_at)
);
CREATE INDEX ix_course_ratings_course_id ON course_ratings(course_id);
CREATE INDEX ix_course_ratings_user_id ON course_ratings(user_id);
```

### Relaciones

```
teachers ──┐
           ├── course_teachers ──── courses ──── lessons
teachers ──┘                           │
                                       │
                                  course_ratings
```

### Características de DB

1. **Soft Deletes**: Todos los modelos usan `deleted_at` en lugar de eliminación física
2. **Timestamps**: Automáticos con `created_at` y `updated_at`
3. **Constraints**:
   - Unique en `(course_id, user_id, deleted_at)` permite múltiples ratings soft-deleted
   - Check constraint asegura `rating BETWEEN 1 AND 5`
4. **Índices**: En FKs, slugs, emails para optimización de queries

---

## 🌐 API REST

### Base URL
- **Desarrollo**: `http://localhost:8000`
- **Producción**: TBD

### Endpoints Disponibles

#### Health Check
```http
GET /
GET /health
```

#### Cursos

```http
# Listar todos los cursos con stats de rating
GET /courses
Response: [
  {
    "id": 1,
    "name": "Curso de Python",
    "description": "Aprende Python desde cero",
    "thumbnail": "https://...",
    "slug": "curso-python",
    "average_rating": 4.5,     # Promedio calculado
    "total_ratings": 142        # Total de ratings activos
  }
]

# Obtener detalle de un curso
GET /courses/{slug}
Response: {
  "id": 1,
  "name": "Curso de Python",
  "description": "...",
  "thumbnail": "...",
  "slug": "curso-python",
  "teacher_id": [1, 2],
  "classes": [
    {
      "id": 1,
      "title": "Introducción",
      "description": "...",
      "slug": "introduccion",
      "video": "https://...",
      "duration": 600
    }
  ],
  "average_rating": 4.5,
  "total_ratings": 142,
  "rating_distribution": {
    "1": 5,
    "2": 10,
    "3": 25,
    "4": 50,
    "5": 52
  }
}
```

#### Lecciones/Clases

```http
# Obtener detalle de una lección
GET /classes/{class_id}
Response: {
  "id": 1,
  "title": "Introducción a Python",
  "description": "...",
  "slug": "introduccion-python",
  "video": "https://youtube.com/..."
}
```

#### Ratings ⭐ NUEVO

```http
# Crear o actualizar rating de usuario
POST /courses/{course_id}/ratings
Content-Type: application/json
{
  "user_id": 42,
  "rating": 5
}
Response: 201 Created
{
  "id": 123,
  "course_id": 1,
  "user_id": 42,
  "rating": 5,
  "created_at": "2025-01-17T10:30:00Z",
  "updated_at": "2025-01-17T10:30:00Z"
}

# Obtener estadísticas de ratings
GET /courses/{course_id}/ratings/stats
Response: {
  "average_rating": 4.35,
  "total_ratings": 142,
  "rating_distribution": {
    "1": 5,
    "2": 10,
    "3": 25,
    "4": 50,
    "5": 52
  }
}

# Obtener todos los ratings del curso
GET /courses/{course_id}/ratings
Response: [
  {
    "id": 123,
    "course_id": 1,
    "user_id": 42,
    "rating": 5,
    "created_at": "2025-01-17T10:30:00Z",
    "updated_at": "2025-01-17T10:30:00Z"
  }
]

# Obtener rating de usuario específico
GET /courses/{course_id}/ratings/user/{user_id}
Response: 200 OK (si existe) o 204 No Content (si no existe)

# Actualizar rating existente
PUT /courses/{course_id}/ratings/{user_id}
{
  "rating": 4
}
Response: 200 OK

# Eliminar rating (soft delete)
DELETE /courses/{course_id}/ratings/{user_id}
Response: 204 No Content
```

### Códigos de Estado

- `200 OK` - Operación exitosa
- `201 Created` - Recurso creado
- `204 No Content` - Operación exitosa sin contenido
- `400 Bad Request` - Datos inválidos
- `404 Not Found` - Recurso no encontrado
- `422 Unprocessable Entity` - Error de validación
- `500 Internal Server Error` - Error del servidor

---

## 💻 Frontend

### Tecnología
- Next.js 15 con App Router
- Server Components por defecto
- Client Components marcados con `"use client"`

### Páginas Principales

#### Home (`/`)
```typescript
// src/app/page.tsx
export default async function HomePage() {
  const courses = await fetch('http://localhost:8000/courses')

  return (
    <div className={styles.grid}>
      {courses.map(course => (
        <Course
          key={course.id}
          course={course}
          showRating={true}
        />
      ))}
    </div>
  )
}
```

#### Course Detail (`/course/[slug]`)
```typescript
// src/app/course/[slug]/page.tsx
export default async function CoursePage({ params }) {
  const course = await fetch(`http://localhost:8000/courses/${params.slug}`)

  return (
    <CourseDetail
      course={course}
      showLessons={true}
      showRatingStats={true}
    />
  )
}
```

#### Class Viewer (`/classes/[class_id]`)
```typescript
// src/app/classes/[class_id]/page.tsx
export default async function ClassPage({ params }) {
  const lesson = await fetch(`http://localhost:8000/classes/${params.class_id}`)

  return (
    <VideoPlayer
      videoUrl={lesson.video}
      title={lesson.title}
      description={lesson.description}
    />
  )
}
```

### Componentes Clave

#### StarRating Component
```typescript
// src/components/StarRating/StarRating.tsx
interface StarRatingProps {
  rating: number;           // 0.0 - 5.0
  totalRatings?: number;
  showCount?: boolean;
  size?: 'small' | 'medium' | 'large';
  readonly?: boolean;
  className?: string;
}

export function StarRating({ rating, totalRatings, ...props }: StarRatingProps) {
  // Muestra estrellas con fill parcial para decimales
  // Ejemplo: 4.5 = 4 estrellas llenas + 1 media estrella
}
```

#### Ratings API Service
```typescript
// src/services/ratingsApi.ts
export const ratingsApi = {
  getRatingStats: (courseId: number) => Promise<RatingStats>,
  getCourseRatings: (courseId: number) => Promise<CourseRating[]>,
  getUserRating: (courseId: number, userId: number) => Promise<CourseRating | null>,
  createRating: (courseId: number, data: RatingRequest) => Promise<CourseRating>,
  updateRating: (courseId: number, userId: number, data: RatingRequest) => Promise<CourseRating>,
  deleteRating: (courseId: number, userId: number) => Promise<void>
}
```

### Tipos TypeScript

```typescript
// src/types/index.ts
export interface Course {
  id: number;
  name: string;
  description: string;
  thumbnail: string;
  slug: string;
  average_rating?: number;
  total_ratings?: number;
}

export interface CourseDetail extends Course {
  classes: Class[];
  teacher_id?: number[];
  rating_distribution?: Record<number, number>;
}

export interface Class {
  id: number;
  title: string;
  description: string;
  video: string;
  slug: string;
  duration: number;
}

// src/types/rating.ts
export interface CourseRating {
  id: number;
  course_id: number;
  user_id: number;
  rating: number;
  created_at: string;
  updated_at: string;
}

export interface RatingStats {
  average_rating: number;
  total_ratings: number;
  rating_distribution?: Record<number, number>;
}

export interface RatingRequest {
  user_id: number;
  rating: number; // 1-5
}
```

---

## 📱 Mobile

### iOS (Swift + SwiftUI)

#### Arquitectura CLEAR

**Domain Layer** (`Domain/`):
```swift
// Domain/Models/Course.swift
struct Course: Identifiable, Equatable {
    let id: Int
    let name: String
    let description: String
    let thumbnail: String
    let slug: String
}

// Domain/Repositories/CourseRepositoryProtocol.swift
protocol CourseRepository {
    func getAllCourses() async throws -> [Course]
    func getCourseBySlug(_ slug: String) async throws -> Course
}
```

**Data Layer** (`Data/`):
```swift
// Data/Entities/CourseDTO.swift
struct CourseDTO: Codable {
    let id: Int
    let name: String
    let description: String
    let thumbnail: String
    let slug: String
}

// Data/Mappers/CourseMapper.swift
struct CourseMapper {
    static func toDomain(_ dto: CourseDTO) -> Course {
        Course(
            id: dto.id,
            name: dto.name,
            description: dto.description,
            thumbnail: dto.thumbnail,
            slug: dto.slug
        )
    }
}

// Data/Repositories/RemoteCourseRepository.swift
final class RemoteCourseRepository: CourseRepository {
    private let networkService: NetworkService

    func getAllCourses() async throws -> [Course] {
        let dtos: [CourseDTO] = try await networkService.request(
            CourseAPIEndpoints.getAllCourses
        )
        return dtos.map(CourseMapper.toDomain)
    }
}
```

**Presentation Layer** (`Presentation/`):
```swift
// Presentation/ViewModels/CourseListViewModel.swift
@MainActor
class CourseListViewModel: ObservableObject {
    @Published var courses: [Course] = []
    @Published var isLoading = false
    @Published var errorMessage: String?
    @Published var searchText = ""

    private let repository: CourseRepository

    var filteredCourses: [Course] {
        guard !searchText.isEmpty else { return courses }
        return courses.filter { $0.name.localizedCaseInsensitiveContains(searchText) }
    }

    func loadCourses() async {
        isLoading = true
        defer { isLoading = false }

        do {
            courses = try await repository.getAllCourses()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}

// Presentation/Views/CourseListView.swift
struct CourseListView: View {
    @StateObject private var viewModel = CourseListViewModel(
        repository: AppModule.provideCourseRepository()
    )

    var body: some View {
        NavigationStack {
            List(viewModel.filteredCourses) { course in
                CourseCardView(course: course)
            }
            .searchable(text: $viewModel.searchText)
            .task { await viewModel.loadCourses() }
        }
    }
}
```

### Android (Kotlin + Jetpack Compose)

#### Arquitectura CLEAR + MVI

**Domain Layer** (`domain/`):
```kotlin
// domain/models/Course.kt
data class Course(
    val id: Int,
    val name: String,
    val description: String,
    val thumbnail: String,
    val slug: String
)

// domain/repositories/CourseRepository.kt
interface CourseRepository {
    suspend fun getAllCourses(): Result<List<Course>>
    suspend fun getCourseBySlug(slug: String): Result<Course>
}
```

**Data Layer** (`data/`):
```kotlin
// data/entities/CourseDTO.kt
@Serializable
data class CourseDTO(
    val id: Int,
    val name: String,
    val description: String,
    val thumbnail: String,
    val slug: String
)

// data/mappers/CourseMapper.kt
object CourseMapper {
    fun CourseDTO.toDomain(): Course = Course(
        id = id,
        name = name,
        description = description,
        thumbnail = thumbnail,
        slug = slug
    )
}

// data/repositories/RemoteCourseRepository.kt
class RemoteCourseRepository(
    private val apiService: ApiService
) : CourseRepository {
    override suspend fun getAllCourses(): Result<List<Course>> = runCatching {
        apiService.getAllCourses().map { it.toDomain() }
    }
}

// data/network/ApiService.kt
interface ApiService {
    @GET("courses")
    suspend fun getAllCourses(): List<CourseDTO>

    @GET("courses/{slug}")
    suspend fun getCourseBySlug(@Path("slug") slug: String): CourseDTO
}
```

**Presentation Layer** (`presentation/`):
```kotlin
// presentation/courses/state/CourseListUiState.kt
data class CourseListUiState(
    val isLoading: Boolean = false,
    val isRefreshing: Boolean = false,
    val courses: List<Course> = emptyList(),
    val error: String? = null
)

// presentation/courses/state/CourseListUiEvent.kt
sealed class CourseListUiEvent {
    object LoadCourses : CourseListUiEvent()
    object RefreshCourses : CourseListUiEvent()
    object ClearError : CourseListUiEvent()
}

// presentation/courses/viewmodel/CourseListViewModel.kt
class CourseListViewModel(
    private val repository: CourseRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(CourseListUiState())
    val uiState: StateFlow<CourseListUiState> = _uiState.asStateFlow()

    fun handleEvent(event: CourseListUiEvent) {
        when (event) {
            is CourseListUiEvent.LoadCourses -> loadCourses()
            is CourseListUiEvent.RefreshCourses -> refreshCourses()
            is CourseListUiEvent.ClearError -> clearError()
        }
    }

    private fun loadCourses() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }

            repository.getAllCourses()
                .onSuccess { courses ->
                    _uiState.update { it.copy(
                        courses = courses,
                        isLoading = false,
                        error = null
                    )}
                }
                .onFailure { error ->
                    _uiState.update { it.copy(
                        isLoading = false,
                        error = error.message
                    )}
                }
        }
    }
}

// presentation/courses/screen/CourseListScreen.kt
@Composable
fun CourseListScreen(
    viewModel: CourseListViewModel = viewModel(
        factory = viewModelFactory {
            CourseListViewModel(AppModule.provideCourseRepository())
        }
    )
) {
    val uiState by viewModel.uiState.collectAsState()

    LaunchedEffect(Unit) {
        viewModel.handleEvent(CourseListUiEvent.LoadCourses)
    }

    when {
        uiState.isLoading -> LoadingIndicator()
        uiState.error != null -> ErrorMessage(uiState.error!!)
        else -> CourseList(uiState.courses)
    }
}
```

---

## 🎨 Patrones y Convenciones

### Backend (Python/FastAPI)

#### 1. Service Layer Pattern
```python
# app/services/course_service.py
class CourseService:
    def __init__(self, db: Session):
        self.db = db

    def get_all_courses(self) -> List[Dict]:
        """Lógica de negocio para obtener cursos"""
        courses = self.db.query(Course).filter(
            Course.deleted_at.is_(None)
        ).all()

        # Añadir stats de ratings
        for course in courses:
            stats = self.get_course_rating_stats(course.id)
            course.average_rating = stats["average_rating"]
            course.total_ratings = stats["total_ratings"]

        return courses
```

#### 2. Dependency Injection
```python
# app/main.py
from app.db.base import get_db

@app.get("/courses")
def get_courses(db: Session = Depends(get_db)):
    service = CourseService(db)
    return service.get_all_courses()
```

#### 3. Soft Delete Pattern
```python
# app/models/base.py
class BaseModel:
    deleted_at = Column(DateTime, nullable=True)

    def soft_delete(self):
        self.deleted_at = datetime.utcnow()
```

#### 4. Pydantic Schemas
```python
# app/schemas/rating.py
class RatingRequest(BaseModel):
    user_id: int = Field(..., gt=0)
    rating: int = Field(..., ge=1, le=5)

class RatingResponse(BaseModel):
    id: int
    course_id: int
    user_id: int
    rating: int
    created_at: datetime
    updated_at: datetime

    class Config:
        from_attributes = True
```

### Frontend (TypeScript/React)

#### 1. Server vs Client Components
```typescript
// Server Component (por defecto)
export default async function CoursePage() {
  const data = await fetch(...)
  return <div>{data}</div>
}

// Client Component (interactividad)
'use client'
export function InteractiveRating() {
  const [rating, setRating] = useState(0)
  return <StarRating onClick={setRating} />
}
```

#### 2. CSS Modules
```typescript
// Component.tsx
import styles from './Component.module.scss'

export function Component() {
  return <div className={styles.container}>...</div>
}
```

```scss
// Component.module.scss
.container {
  display: flex;
  gap: 1rem;

  &__item {
    padding: 0.5rem;
  }
}
```

#### 3. Error Handling
```typescript
// services/ratingsApi.ts
async function handleApiResponse<T>(response: Response): Promise<T> {
  if (!response.ok) {
    const error = await response.json().catch(() => ({}))
    throw new Error(error.detail || 'API request failed')
  }
  return response.json()
}
```

### Mobile (Swift/Kotlin)

#### 1. CLEAR Architecture Layers

**Nunca mezcles capas**:
- ❌ View no llama directamente a Repository
- ❌ ViewModel no conoce DTOs
- ✅ View → ViewModel → Repository → NetworkService

#### 2. DTO → Domain Mapping

Siempre usa Mappers:
```swift
// ❌ MAL
let course = Course(dto: courseDTO)

// ✅ BIEN
let course = CourseMapper.toDomain(courseDTO)
```

#### 3. Error Handling

**iOS (Result/async throws)**:
```swift
func getAllCourses() async throws -> [Course] {
    // throws NetworkError
}

// O con Result
func getAllCoursesResult() async -> Result<[Course], Error>
```

**Android (Result<T>)**:
```kotlin
suspend fun getAllCourses(): Result<List<Course>> = runCatching {
    apiService.getAllCourses().map { it.toDomain() }
}
```

---

## ⚡ Comandos Útiles

### Backend

```bash
# Navegar al backend
cd Backend

# Instalar dependencias
pip install -e ".[dev]"

# Ejecutar servidor
python -m uvicorn app.main:app --reload

# Ejecutar tests
pytest

# Crear migración
alembic revision --autogenerate -m "descripción"

# Aplicar migraciones
alembic upgrade head

# Seed database
python -m app.db.seed

# Docker
docker-compose up -d
docker-compose down

# Makefile shortcuts
make install    # Instalar dependencias
make run        # Ejecutar servidor
make test       # Ejecutar tests
make migrate    # Aplicar migraciones
```

### Frontend

```bash
# Navegar al frontend
cd Frontend

# Instalar dependencias
npm install

# Ejecutar desarrollo
npm run dev

# Build producción
npm run build

# Ejecutar tests
npm test

# Ejecutar con Turbopack
npm run dev -- --turbopack

# Linting
npm run lint
```

### iOS

```bash
# Navegar al proyecto iOS
cd Mobile/PlatziFlixiOS

# Abrir en Xcode
open PlatziFlixiOS.xcodeproj

# Desde Xcode:
# - Cmd+R: Run
# - Cmd+U: Run tests
# - Cmd+B: Build
```

### Android

```bash
# Navegar al proyecto Android
cd Mobile/PlatziFlixAndroid

# Build
./gradlew build

# Ejecutar tests
./gradlew test

# Instalar en dispositivo
./gradlew installDebug

# Limpiar build
./gradlew clean

# Desde Android Studio:
# - Run app
# - Run tests
# - Gradle sync
```

---

## 📊 Estado Actual del Proyecto

### Completado ✅

#### Backend
- ✅ Estructura base FastAPI
- ✅ Modelos de DB (Course, Lesson, Teacher)
- ✅ Endpoints CRUD para cursos
- ✅ Sistema de ratings completo
- ✅ Migraciones con Alembic
- ✅ Seeds de datos iniciales
- ✅ Tests unitarios e integración
- ✅ Soft delete pattern
- ✅ Service layer pattern

#### Frontend
- ✅ Configuración Next.js 15
- ✅ Página home con listado de cursos
- ✅ Página detalle de curso
- ✅ Página reproductor de lección
- ✅ Componente StarRating
- ✅ Cliente API para ratings
- ✅ TypeScript tipos completos
- ✅ Tests de componentes

#### Mobile
- ✅ iOS: Arquitectura CLEAR
- ✅ iOS: Listado de cursos
- ✅ iOS: Búsqueda local
- ✅ Android: Arquitectura CLEAR + MVI
- ✅ Android: Listado de cursos
- ✅ Android: Network module con Retrofit

### En Progreso 🚧

#### Backend
- 🚧 Optimización de queries N+1
- 🚧 Documentación OpenAPI completa
- 🚧 Rate limiting

#### Frontend
- 🚧 Integración completa de ratings interactivos
- 🚧 Gestión de estado global
- 🚧 Tests E2E

#### Mobile
- 🚧 iOS: Detalle de curso
- 🚧 iOS: Reproductor de video
- 🚧 Android: Detalle de curso
- 🚧 Android: Reproductor de video

### Pendiente ⏳

#### Sistema
- ⏳ Autenticación de usuarios (JWT)
- ⏳ Sistema de autorización
- ⏳ Modelo de usuario en DB
- ⏳ Paginación de resultados
- ⏳ Búsqueda y filtros (backend)
- ⏳ Sistema de comentarios
- ⏳ Progreso de cursos
- ⏳ Favoritos/Wishlist
- ⏳ Notificaciones
- ⏳ Panel de administración

#### Infraestructura
- ⏳ CI/CD pipelines
- ⏳ Deployment scripts
- ⏳ Monitoring y logging
- ⏳ Backups automáticos
- ⏳ CDN para assets
- ⏳ Environment configs

---

## 🚀 Próximos Pasos

### Prioridad Alta

1. **Sistema de Autenticación**
   - Crear modelo User en backend
   - JWT tokens con FastAPI
   - Endpoints de login/register
   - Protección de endpoints de ratings
   - Frontend: formularios de auth
   - Mobile: gestión de tokens

2. **Completar Mobile Apps**
   - iOS: Pantalla de detalle de curso
   - iOS: Reproductor de video nativo
   - Android: Pantalla de detalle de curso
   - Android: Reproductor de video con ExoPlayer
   - Integrar sistema de ratings en mobile

3. **Ratings Interactivos en Frontend**
   - Componente para enviar ratings
   - Autenticación requerida
   - Feedback visual (loading, success, error)
   - Actualización optimista de UI

### Prioridad Media

4. **Búsqueda y Filtros**
   - Backend: Query params para filtros
   - Frontend: Barra de búsqueda
   - Mobile: Filtros por categoría
   - Indexación para búsqueda rápida

5. **Progreso de Cursos**
   - Tabla `user_course_progress`
   - Track de lecciones completadas
   - Barra de progreso en UI
   - Certificados al completar

6. **Mejoras de UX**
   - Loading skeletons
   - Error boundaries mejorados
   - Toast notifications
   - Optimistic updates
   - Offline support (mobile)

### Prioridad Baja

7. **Features Avanzadas**
   - Sistema de comentarios/reviews
   - Favoritos y wishlists
   - Recomendaciones personalizadas
   - Notificaciones push
   - Gamificación (badges, streaks)

8. **Admin Panel**
   - Dashboard de métricas
   - Gestión de cursos
   - Moderación de ratings
   - Gestión de usuarios

---

## 🔒 Limitaciones Conocidas

### Seguridad
1. **Sin autenticación**: Cualquiera puede crear ratings con cualquier `user_id`
2. **CORS abierto**: Permitir todos los orígenes en desarrollo
3. **Sin rate limiting**: Vulnerable a spam de ratings
4. **Sin validación de ownership**: Cualquiera puede actualizar/eliminar ratings

### Funcionalidad
1. **Videos externos**: Dependencia de YouTube/Vimeo, no hosting propio
2. **Sin búsqueda**: No hay filtros o búsqueda en backend
3. **Sin paginación**: Puede ser lento con muchos cursos
4. **Usuario mock**: Frontend/mobile usan IDs hardcodeados

### Infraestructura
1. **Sin CI/CD**: Deploy manual
2. **Sin monitoring**: No hay logs centralizados
3. **Sin backups**: Riesgo de pérdida de datos
4. **SQLite en dev**: No es PostgreSQL real localmente

### Performance
1. **N+1 queries**: Algunos endpoints pueden optimizarse con eager loading
2. **Sin caching**: Redis/Memcached no implementado
3. **Sin CDN**: Assets servidos directamente
4. **Sin compresión**: Respuestas JSON sin gzip

---

## 📝 Notas de Desarrollo

### Convenciones de Código

#### Backend
- **Imports**: Absolutos desde `app.*`
- **Naming**: snake_case para funciones/variables
- **Type hints**: Obligatorios en funciones públicas
- **Docstrings**: Google style para servicios
- **Max line length**: 100 caracteres

#### Frontend
- **Imports**: Absolutos desde `@/` (alias configurado)
- **Naming**: camelCase para funciones, PascalCase para componentes
- **Types**: Interfaces para props, types para datos
- **File naming**: PascalCase para componentes, camelCase para utils
- **CSS**: BEM-like con CSS Modules

#### Mobile
- **iOS**: Swift naming conventions (camelCase, PascalCase)
- **Android**: Kotlin naming conventions
- **Folder structure**: Seguir CLEAR architecture estrictamente
- **Comments**: Solo para lógica compleja, código debe ser auto-explicativo

### Testing

#### Backend
```bash
# Ejecutar todos los tests
pytest

# Con coverage
pytest --cov=app --cov-report=html

# Solo un archivo
pytest app/tests/test_rating_endpoints.py

# Con verbose
pytest -v
```

#### Frontend
```bash
# Ejecutar tests
npm test

# Watch mode
npm test -- --watch

# Coverage
npm test -- --coverage
```

### Migraciones de DB

```bash
# Crear migración auto
alembic revision --autogenerate -m "add_user_table"

# Migración manual
alembic revision -m "manual_change"

# Aplicar
alembic upgrade head

# Revertir
alembic downgrade -1

# Ver historial
alembic history

# Ver estado actual
alembic current
```

### Git Workflow

```bash
# Feature branch
git checkout -b feature/user-authentication

# Commits semánticos
git commit -m "feat: add user authentication endpoints"
git commit -m "fix: resolve rating constraint error"
git commit -m "docs: update API documentation"

# Tipos de commit:
# feat: nueva funcionalidad
# fix: corrección de bug
# docs: documentación
# style: formato, no afecta código
# refactor: refactorización
# test: añadir tests
# chore: tareas de mantenimiento
```

---

## 🔗 URLs de Referencia

### Documentación Oficial
- **FastAPI**: https://fastapi.tiangolo.com/
- **Next.js**: https://nextjs.org/docs
- **React**: https://react.dev/
- **SwiftUI**: https://developer.apple.com/xcode/swiftui/
- **Jetpack Compose**: https://developer.android.com/jetpack/compose
- **SQLAlchemy**: https://docs.sqlalchemy.org/
- **Pydantic**: https://docs.pydantic.dev/

### Herramientas
- **Alembic**: https://alembic.sqlalchemy.org/
- **Retrofit**: https://square.github.io/retrofit/
- **Vitest**: https://vitest.dev/

### APIs Locales
- **Backend**: http://localhost:8000
- **API Docs**: http://localhost:8000/docs
- **Frontend**: http://localhost:3000

---

## 💡 Tips y Mejores Prácticas

### Backend
1. Siempre usar `Depends(get_db)` para sesiones de DB
2. Validar entrada con Pydantic schemas
3. Usar soft deletes, nunca eliminar datos
4. Separar lógica de negocio en services
5. Escribir tests para cada endpoint

### Frontend
1. Usar Server Components cuando sea posible
2. Client Components solo para interactividad
3. Prefetch data en layouts
4. Error boundaries en cada página
5. Loading states para mejor UX

### Mobile
1. Nunca mezclar capas (View → ViewModel → Repository)
2. Siempre usar Mappers para DTO → Domain
3. Manejar estados de loading/error/success
4. Inyección de dependencias manual pero clara
5. Tests para ViewModels y Repositories

### General
1. Commits semánticos y descriptivos
2. Code review antes de merge
3. Tests antes de deployment
4. Documentar decisiones arquitectónicas
5. Mantener este archivo actualizado

---

## 🆘 Troubleshooting

### Backend no inicia
```bash
# Verificar que PostgreSQL está corriendo
docker-compose ps

# Reconstruir contenedores
docker-compose down -v
docker-compose up --build

# Verificar migraciones
alembic current
alembic upgrade head
```

### Frontend no compila
```bash
# Limpiar cache
rm -rf .next node_modules
npm install
npm run dev
```

### Mobile no conecta a backend
- **iOS Simulator**: Usar `http://localhost:8000`
- **Android Emulator**: Usar `http://10.0.2.2:8000`
- **Dispositivo físico**: Usar IP local `http://192.168.x.x:8000`

### Tests fallan
```bash
# Backend: Verificar DB de test
pytest --setup-show

# Frontend: Limpiar cache
npm test -- --clearCache
```

---

## 📌 Changelog

### 2025-01-17
- ✨ Creado archivo `claude.md` como memoria del proyecto
- ✨ Sistema de ratings completamente implementado
- ✨ Componente StarRating en frontend
- ✨ Arquitectura CLEAR documentada para mobile
- 📝 Documentación completa de API y base de datos

---

**Última actualización**: 2025-01-17
**Versión del documento**: 1.0.0
**Mantenedor**: Claude Code

---

Este documento debe ser actualizado cada vez que se realicen cambios significativos en la arquitectura, nuevas features, o decisiones técnicas importantes.
