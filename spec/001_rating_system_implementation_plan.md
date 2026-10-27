# Análisis Técnico: Sistema de Ratings Interactivo (Opción A - MVP Sin Auth)

**Documento generado por**: Agent Architect
**Fecha**: 2026-01-18
**Versión**: 1.0.0
**Estado**: Listo para implementación

---

## Problema

Implementar funcionalidad de rating interactiva en el frontend de Platziflix sin sistema de autenticación, permitiendo a los usuarios calificar cursos y visualizar distribución de ratings, usando localStorage como workaround temporal para identificación de usuarios.

---

## Impacto Arquitectural

### Backend
- ✅ **Sin cambios** - Todos los endpoints necesarios ya existen y funcionan
- ✅ API acepta `user_id` en requests (validación básica ya implementada)

### Frontend
**Nuevos componentes** (a crear):
- `RatingInput` - Input interactivo de estrellas
- `RatingDistribution` - Histograma de distribución
- `UserIdManager` - Utility hook para gestión de localStorage

**Componentes a modificar**:
- `CourseDetail` - Integrar nuevos componentes
- `StarRating` - Potencialmente extender para modo interactivo

**Nuevos servicios/utils**:
- `userIdService.ts` - Gestión de user_id en localStorage
- `ratingHelpers.ts` - Funciones auxiliares para cálculos

### Base de datos
- ✅ **Sin cambios** - Esquema ya soporta todos los casos de uso

---

## Propuesta de Solución

### Arquitectura de Componentes

```
CourseDetail Page
│
├─ StarRating (readonly)           # Ya existe - muestra promedio
│
├─ RatingInput (interactive) ──────┤ # NUEVO - permite calificar
│  └─ userIdService                │
│                                   │
└─ RatingDistribution ──────────────┤ # NUEVO - muestra histograma
   └─ RatingBar (sub-component)    │
```

### Flujo de Datos

```
1. User clicks star → RatingInput
2. RatingInput lee user_id de localStorage
3. Si no existe user_id → genera uno y guarda
4. POST/PUT a /courses/{id}/ratings con {user_id, rating}
5. Backend retorna rating actualizado
6. Frontend actualiza UI optimísticamente
7. Recarga stats desde /courses/{id}/ratings/stats
8. RatingDistribution se actualiza automáticamente
```

---

## Plan de Implementación

### Fase 1: Infraestructura (2-3 horas)

#### Task 1.1: UserID Service (45 min)
**Archivo**: `Frontend/src/services/userIdService.ts`

```typescript
// Frontend/src/services/userIdService.ts

const USER_ID_KEY = 'platziflix_user_id';

export const userIdService = {
  /**
   * Obtiene el user_id del localStorage o genera uno nuevo
   */
  getUserId(): number {
    const stored = localStorage.getItem(USER_ID_KEY);

    if (stored) {
      const userId = parseInt(stored, 10);
      if (!isNaN(userId) && userId > 0) {
        return userId;
      }
    }

    // Generar ID aleatorio entre 1000 y 999999
    const newUserId = Math.floor(Math.random() * 999000) + 1000;
    this.setUserId(newUserId);
    return newUserId;
  },

  /**
   * Guarda un user_id en localStorage
   */
  setUserId(userId: number): void {
    localStorage.setItem(USER_ID_KEY, userId.toString());
  },

  /**
   * Limpia el user_id (útil para testing/reset)
   */
  clearUserId(): void {
    localStorage.removeItem(USER_ID_KEY);
  },

  /**
   * Verifica si existe un user_id
   */
  hasUserId(): boolean {
    return localStorage.getItem(USER_ID_KEY) !== null;
  }
};
```

**Tests**: `Frontend/src/services/userIdService.test.ts`

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { userIdService } from './userIdService';

describe('userIdService', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('should generate a new user ID if none exists', () => {
    const userId = userIdService.getUserId();
    expect(userId).toBeGreaterThanOrEqual(1000);
    expect(userId).toBeLessThanOrEqual(999999);
  });

  it('should return the same user ID on subsequent calls', () => {
    const userId1 = userIdService.getUserId();
    const userId2 = userIdService.getUserId();
    expect(userId1).toBe(userId2);
  });

  it('should allow setting a custom user ID', () => {
    userIdService.setUserId(12345);
    expect(userIdService.getUserId()).toBe(12345);
  });

  it('should clear user ID', () => {
    userIdService.getUserId();
    userIdService.clearUserId();
    expect(userIdService.hasUserId()).toBe(false);
  });
});
```

#### Task 1.2: Rating Helpers (30 min)
**Archivo**: `Frontend/src/utils/ratingHelpers.ts`

```typescript
// Frontend/src/utils/ratingHelpers.ts

export interface RatingDistributionData {
  rating: number;
  count: number;
  percentage: number;
}

/**
 * Transforma rating_distribution del backend a formato para gráfica
 */
export function transformDistribution(
  distribution: Record<number, number> | undefined,
  totalRatings: number
): RatingDistributionData[] {
  if (!distribution || totalRatings === 0) {
    return [1, 2, 3, 4, 5].map(rating => ({
      rating,
      count: 0,
      percentage: 0
    }));
  }

  return [5, 4, 3, 2, 1].map(rating => {
    const count = distribution[rating] || 0;
    const percentage = totalRatings > 0
      ? Math.round((count / totalRatings) * 100)
      : 0;

    return { rating, count, percentage };
  });
}

/**
 * Formatea el número de ratings para display
 */
export function formatRatingCount(count: number): string {
  if (count === 0) return 'Sin calificaciones';
  if (count === 1) return '1 calificación';
  if (count < 1000) return `${count} calificaciones`;

  const k = Math.floor(count / 1000);
  const remainder = count % 1000;

  if (remainder === 0) return `${k}k calificaciones`;
  return `${k}.${Math.floor(remainder / 100)}k calificaciones`;
}

/**
 * Calcula el promedio de ratings (útil para optimistic updates)
 */
export function calculateAverage(
  distribution: Record<number, number>
): number {
  const entries = Object.entries(distribution);
  if (entries.length === 0) return 0;

  const totalSum = entries.reduce((sum, [rating, count]) => {
    return sum + (parseInt(rating) * count);
  }, 0);

  const totalCount = entries.reduce((sum, [, count]) => sum + count, 0);

  return totalCount > 0 ? totalSum / totalCount : 0;
}
```

**Tests**: `Frontend/src/utils/ratingHelpers.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import {
  transformDistribution,
  formatRatingCount,
  calculateAverage
} from './ratingHelpers';

describe('ratingHelpers', () => {
  describe('transformDistribution', () => {
    it('should transform backend distribution to chart data', () => {
      const distribution = { 1: 5, 2: 10, 3: 25, 4: 50, 5: 52 };
      const result = transformDistribution(distribution, 142);

      expect(result).toHaveLength(5);
      expect(result[0]).toEqual({ rating: 5, count: 52, percentage: 37 });
      expect(result[4]).toEqual({ rating: 1, count: 5, percentage: 4 });
    });

    it('should handle empty distribution', () => {
      const result = transformDistribution(undefined, 0);
      expect(result).toHaveLength(5);
      expect(result.every(d => d.count === 0)).toBe(true);
    });
  });

  describe('formatRatingCount', () => {
    it('should format rating counts correctly', () => {
      expect(formatRatingCount(0)).toBe('Sin calificaciones');
      expect(formatRatingCount(1)).toBe('1 calificación');
      expect(formatRatingCount(42)).toBe('42 calificaciones');
      expect(formatRatingCount(1000)).toBe('1k calificaciones');
      expect(formatRatingCount(1500)).toBe('1.5k calificaciones');
    });
  });

  describe('calculateAverage', () => {
    it('should calculate correct average', () => {
      const distribution = { 1: 1, 2: 1, 3: 1, 4: 1, 5: 1 };
      expect(calculateAverage(distribution)).toBe(3);
    });

    it('should handle weighted distribution', () => {
      const distribution = { 5: 10 }; // 10 ratings of 5 stars
      expect(calculateAverage(distribution)).toBe(5);
    });
  });
});
```

#### Task 1.3: Custom Hooks (45 min)
**Archivo**: `Frontend/src/hooks/useUserRating.ts`

```typescript
// Frontend/src/hooks/useUserRating.ts
'use client'

import { useState, useEffect } from 'react';
import { ratingsApi } from '@/services/ratingsApi';
import { userIdService } from '@/services/userIdService';
import type { CourseRating } from '@/types/rating';

export function useUserRating(courseId: number) {
  const [userRating, setUserRating] = useState<number | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const userId = userIdService.getUserId();

  useEffect(() => {
    loadUserRating();
  }, [courseId, userId]);

  async function loadUserRating() {
    try {
      setIsLoading(true);
      setError(null);

      const rating = await ratingsApi.getUserRating(courseId, userId);
      setUserRating(rating?.rating ?? null);
    } catch (err) {
      console.error('Error loading user rating:', err);
      setError(err instanceof Error ? err.message : 'Error desconocido');
      setUserRating(null);
    } finally {
      setIsLoading(false);
    }
  }

  async function submitRating(rating: number): Promise<boolean> {
    try {
      setError(null);

      if (userRating === null) {
        // Create new rating
        await ratingsApi.createRating(courseId, { user_id: userId, rating });
      } else {
        // Update existing rating
        await ratingsApi.updateRating(courseId, userId, { user_id: userId, rating });
      }

      setUserRating(rating);
      return true;
    } catch (err) {
      console.error('Error submitting rating:', err);
      setError(err instanceof Error ? err.message : 'Error al guardar');
      return false;
    }
  }

  async function deleteRating(): Promise<boolean> {
    try {
      setError(null);
      await ratingsApi.deleteRating(courseId, userId);
      setUserRating(null);
      return true;
    } catch (err) {
      console.error('Error deleting rating:', err);
      setError(err instanceof Error ? err.message : 'Error al eliminar');
      return false;
    }
  }

  return {
    userRating,
    isLoading,
    error,
    submitRating,
    deleteRating,
    refresh: loadUserRating
  };
}
```

---

### Fase 2: Componentes Visuales (3-4 horas)

#### Task 2.1: RatingInput Component (1.5 horas)
**Archivo**: `Frontend/src/components/RatingInput/RatingInput.tsx`

```typescript
// Frontend/src/components/RatingInput/RatingInput.tsx
'use client'

import { useState } from 'react';
import styles from './RatingInput.module.scss';

export interface RatingInputProps {
  currentRating?: number | null;
  onRate: (rating: number) => Promise<boolean>;
  onDelete?: () => Promise<boolean>;
  disabled?: boolean;
  size?: 'small' | 'medium' | 'large';
  showLabel?: boolean;
  className?: string;
}

export function RatingInput({
  currentRating = null,
  onRate,
  onDelete,
  disabled = false,
  size = 'medium',
  showLabel = true,
  className = ''
}: RatingInputProps) {
  const [hoveredRating, setHoveredRating] = useState<number | null>(null);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const displayRating = hoveredRating ?? currentRating ?? 0;

  const handleClick = async (rating: number) => {
    if (disabled || isSubmitting) return;

    setIsSubmitting(true);

    // Si el usuario clickea la misma estrella, eliminar rating
    if (rating === currentRating && onDelete) {
      await onDelete();
    } else {
      await onRate(rating);
    }

    setIsSubmitting(false);
  };

  const labels = {
    0: 'Sin calificar',
    1: 'Muy malo',
    2: 'Malo',
    3: 'Regular',
    4: 'Bueno',
    5: 'Excelente'
  };

  return (
    <div className={`${styles.ratingInput} ${styles[size]} ${className}`}>
      <div className={styles.stars}>
        {[1, 2, 3, 4, 5].map((star) => (
          <button
            key={star}
            type="button"
            className={`${styles.star} ${star <= displayRating ? styles.filled : ''}`}
            onClick={() => handleClick(star)}
            onMouseEnter={() => !disabled && setHoveredRating(star)}
            onMouseLeave={() => setHoveredRating(null)}
            disabled={disabled || isSubmitting}
            aria-label={`Calificar con ${star} estrella${star > 1 ? 's' : ''}`}
          >
            <svg viewBox="0 0 24 24" fill="currentColor">
              <path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z" />
            </svg>
          </button>
        ))}
      </div>

      {showLabel && (
        <span className={styles.label}>
          {isSubmitting ? 'Guardando...' : labels[displayRating]}
        </span>
      )}

      {currentRating && !isSubmitting && (
        <button
          type="button"
          className={styles.clearButton}
          onClick={() => onDelete?.()}
          disabled={disabled}
          aria-label="Eliminar calificación"
        >
          Eliminar
        </button>
      )}
    </div>
  );
}
```

**Estilos**: `Frontend/src/components/RatingInput/RatingInput.module.scss`

```scss
// Frontend/src/components/RatingInput/RatingInput.module.scss

.ratingInput {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  align-items: flex-start;

  .stars {
    display: flex;
    gap: 0.25rem;
  }

  .star {
    background: none;
    border: none;
    cursor: pointer;
    padding: 0;
    transition: all 0.2s ease;
    color: #d1d5db; // gray-300

    &:hover:not(:disabled) {
      transform: scale(1.1);
      color: #fbbf24; // yellow-400
    }

    &.filled {
      color: #fbbf24; // yellow-400
    }

    &:disabled {
      cursor: not-allowed;
      opacity: 0.5;
    }

    svg {
      display: block;
      width: 100%;
      height: 100%;
    }
  }

  .label {
    font-size: 0.875rem;
    color: #6b7280; // gray-500
    font-weight: 500;
  }

  .clearButton {
    font-size: 0.75rem;
    color: #ef4444; // red-500
    background: none;
    border: none;
    cursor: pointer;
    padding: 0.25rem 0;
    text-decoration: underline;

    &:hover:not(:disabled) {
      color: #dc2626; // red-600
    }

    &:disabled {
      cursor: not-allowed;
      opacity: 0.5;
    }
  }

  // Size variants
  &.small .star svg {
    width: 1.25rem;
    height: 1.25rem;
  }

  &.medium .star svg {
    width: 2rem;
    height: 2rem;
  }

  &.large .star svg {
    width: 2.5rem;
    height: 2.5rem;
  }
}
```

**Tests**: `Frontend/src/components/RatingInput/RatingInput.test.tsx`

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen, fireEvent } from '@testing-library/react';
import { RatingInput } from './RatingInput';

describe('RatingInput', () => {
  it('should render 5 stars', () => {
    const onRate = vi.fn();
    render(<RatingInput onRate={onRate} />);

    const stars = screen.getAllByRole('button', { name: /Calificar con/i });
    expect(stars).toHaveLength(5);
  });

  it('should call onRate when star is clicked', async () => {
    const onRate = vi.fn().mockResolvedValue(true);
    render(<RatingInput onRate={onRate} />);

    const fourthStar = screen.getByLabelText('Calificar con 4 estrellas');
    fireEvent.click(fourthStar);

    expect(onRate).toHaveBeenCalledWith(4);
  });

  it('should show current rating as filled', () => {
    const onRate = vi.fn();
    const { container } = render(
      <RatingInput currentRating={3} onRate={onRate} />
    );

    const filledStars = container.querySelectorAll('.filled');
    expect(filledStars).toHaveLength(3);
  });

  it('should show delete button when rated', () => {
    const onRate = vi.fn();
    const onDelete = vi.fn();

    render(
      <RatingInput
        currentRating={5}
        onRate={onRate}
        onDelete={onDelete}
      />
    );

    const deleteButton = screen.getByLabelText('Eliminar calificación');
    expect(deleteButton).toBeInTheDocument();
  });

  it('should be disabled when disabled prop is true', () => {
    const onRate = vi.fn();
    render(<RatingInput onRate={onRate} disabled />);

    const stars = screen.getAllByRole('button', { name: /Calificar con/i });
    stars.forEach(star => {
      expect(star).toBeDisabled();
    });
  });
});
```

#### Task 2.2: RatingDistribution Component (1.5 horas)
**Archivo**: `Frontend/src/components/RatingDistribution/RatingDistribution.tsx`

```typescript
// Frontend/src/components/RatingDistribution/RatingDistribution.tsx
'use client'

import { transformDistribution, formatRatingCount } from '@/utils/ratingHelpers';
import type { RatingStats } from '@/types/rating';
import styles from './RatingDistribution.module.scss';

export interface RatingDistributionProps {
  stats: RatingStats;
  className?: string;
  showTitle?: boolean;
}

export function RatingDistribution({
  stats,
  className = '',
  showTitle = true
}: RatingDistributionProps) {
  const distribution = transformDistribution(
    stats.rating_distribution,
    stats.total_ratings
  );

  return (
    <div className={`${styles.distribution} ${className}`}>
      {showTitle && (
        <h3 className={styles.title}>Distribución de calificaciones</h3>
      )}

      <div className={styles.bars}>
        {distribution.map(({ rating, count, percentage }) => (
          <div key={rating} className={styles.barRow}>
            <span className={styles.ratingLabel}>{rating} ★</span>

            <div className={styles.barContainer}>
              <div
                className={styles.barFill}
                style={{ width: `${percentage}%` }}
                aria-label={`${percentage}% de calificaciones de ${rating} estrellas`}
              />
            </div>

            <span className={styles.count}>
              {count > 0 ? count.toLocaleString() : '0'}
            </span>
          </div>
        ))}
      </div>

      <p className={styles.total}>
        {formatRatingCount(stats.total_ratings)}
      </p>
    </div>
  );
}
```

**Estilos**: `Frontend/src/components/RatingDistribution/RatingDistribution.module.scss`

```scss
// Frontend/src/components/RatingDistribution/RatingDistribution.module.scss

.distribution {
  display: flex;
  flex-direction: column;
  gap: 1rem;

  .title {
    font-size: 1.125rem;
    font-weight: 600;
    color: #111827; // gray-900
    margin: 0;
  }

  .bars {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .barRow {
    display: grid;
    grid-template-columns: 3rem 1fr 3rem;
    align-items: center;
    gap: 0.75rem;
  }

  .ratingLabel {
    font-size: 0.875rem;
    font-weight: 500;
    color: #6b7280; // gray-500
    text-align: right;
  }

  .barContainer {
    height: 1.25rem;
    background: #f3f4f6; // gray-100
    border-radius: 0.375rem;
    overflow: hidden;
    position: relative;
  }

  .barFill {
    height: 100%;
    background: linear-gradient(90deg, #fbbf24 0%, #f59e0b 100%);
    transition: width 0.3s ease;
    border-radius: 0.375rem;
    min-width: 2px; // Mostrar algo incluso con 0%
  }

  .count {
    font-size: 0.875rem;
    font-weight: 500;
    color: #374151; // gray-700
    text-align: left;
  }

  .total {
    font-size: 0.875rem;
    color: #6b7280; // gray-500
    text-align: center;
    margin: 0.5rem 0 0 0;
  }
}

// Dark mode support
@media (prefers-color-scheme: dark) {
  .distribution {
    .title {
      color: #f9fafb; // gray-50
    }

    .ratingLabel,
    .total {
      color: #9ca3af; // gray-400
    }

    .count {
      color: #d1d5db; // gray-300
    }

    .barContainer {
      background: #374151; // gray-700
    }
  }
}
```

**Tests**: `Frontend/src/components/RatingDistribution/RatingDistribution.test.tsx`

```typescript
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/react';
import { RatingDistribution } from './RatingDistribution';
import type { RatingStats } from '@/types/rating';

describe('RatingDistribution', () => {
  const mockStats: RatingStats = {
    average_rating: 4.35,
    total_ratings: 142,
    rating_distribution: {
      1: 5,
      2: 10,
      3: 25,
      4: 50,
      5: 52
    }
  };

  it('should render all rating bars', () => {
    const { container } = render(<RatingDistribution stats={mockStats} />);

    const bars = container.querySelectorAll('.barRow');
    expect(bars).toHaveLength(5);
  });

  it('should display correct counts', () => {
    render(<RatingDistribution stats={mockStats} />);

    expect(screen.getByText('52')).toBeInTheDocument(); // 5 stars
    expect(screen.getByText('5')).toBeInTheDocument();  // 1 star
  });

  it('should display total ratings formatted', () => {
    render(<RatingDistribution stats={mockStats} />);

    expect(screen.getByText('142 calificaciones')).toBeInTheDocument();
  });

  it('should handle zero ratings', () => {
    const emptyStats: RatingStats = {
      average_rating: 0,
      total_ratings: 0
    };

    render(<RatingDistribution stats={emptyStats} />);

    expect(screen.getByText('Sin calificaciones')).toBeInTheDocument();
  });

  it('should show title when showTitle is true', () => {
    render(<RatingDistribution stats={mockStats} showTitle />);

    expect(screen.getByText('Distribución de calificaciones')).toBeInTheDocument();
  });

  it('should hide title when showTitle is false', () => {
    render(<RatingDistribution stats={mockStats} showTitle={false} />);

    expect(screen.queryByText('Distribución de calificaciones')).not.toBeInTheDocument();
  });
});
```

---

### Fase 3: Integración (2 horas)

#### Task 3.1: Actualizar CourseDetail Page (1.5 horas)
**Archivo**: `Frontend/src/app/course/[slug]/page.tsx`

```typescript
// Frontend/src/app/course/[slug]/page.tsx
import { CourseDetail } from '@/components/CourseDetail/CourseDetail';
import { RatingSection } from '@/components/RatingSection/RatingSection'; // NUEVO
import type { CourseDetail as CourseDetailType } from '@/types';

async function getCourseBySlug(slug: string): Promise<CourseDetailType> {
  const res = await fetch(`http://localhost:8000/courses/${slug}`, {
    cache: 'no-store'
  });

  if (!res.ok) {
    throw new Error('Failed to fetch course');
  }

  return res.json();
}

export default async function CoursePage({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const course = await getCourseBySlug(slug);

  return (
    <div>
      <CourseDetail course={course} />

      {/* NUEVO: Rating Section */}
      <RatingSection
        courseId={course.id}
        initialStats={{
          average_rating: course.average_rating || 0,
          total_ratings: course.total_ratings || 0,
          rating_distribution: course.rating_distribution
        }}
      />
    </div>
  );
}
```

#### Task 3.2: Crear RatingSection Component (30 min)
**Archivo**: `Frontend/src/components/RatingSection/RatingSection.tsx`

```typescript
// Frontend/src/components/RatingSection/RatingSection.tsx
'use client'

import { useState, useEffect } from 'react';
import { StarRating } from '@/components/StarRating/StarRating';
import { RatingInput } from '@/components/RatingInput/RatingInput';
import { RatingDistribution } from '@/components/RatingDistribution/RatingDistribution';
import { useUserRating } from '@/hooks/useUserRating';
import { ratingsApi } from '@/services/ratingsApi';
import type { RatingStats } from '@/types/rating';
import styles from './RatingSection.module.scss';

export interface RatingSectionProps {
  courseId: number;
  initialStats: RatingStats;
}

export function RatingSection({ courseId, initialStats }: RatingSectionProps) {
  const [stats, setStats] = useState<RatingStats>(initialStats);
  const { userRating, isLoading, error, submitRating, deleteRating } = useUserRating(courseId);

  // Refrescar stats después de que el usuario califique
  const handleRatingSubmit = async (rating: number) => {
    const success = await submitRating(rating);

    if (success) {
      // Refrescar stats desde el servidor
      await refreshStats();
    }

    return success;
  };

  const handleRatingDelete = async () => {
    const success = await deleteRating();

    if (success) {
      await refreshStats();
    }

    return success;
  };

  const refreshStats = async () => {
    try {
      const newStats = await ratingsApi.getRatingStats(courseId);
      setStats(newStats);
    } catch (err) {
      console.error('Error refreshing stats:', err);
    }
  };

  return (
    <section className={styles.ratingSection}>
      <div className={styles.container}>
        <div className={styles.header}>
          <h2 className={styles.title}>Calificaciones del curso</h2>

          <div className={styles.averageDisplay}>
            <span className={styles.averageNumber}>
              {stats.average_rating.toFixed(1)}
            </span>
            <StarRating
              rating={stats.average_rating}
              totalRatings={stats.total_ratings}
              showCount
              size="medium"
            />
          </div>
        </div>

        <div className={styles.content}>
          <div className={styles.leftColumn}>
            <RatingDistribution stats={stats} showTitle={false} />
          </div>

          <div className={styles.rightColumn}>
            <div className={styles.userRatingCard}>
              <h3 className={styles.cardTitle}>Tu calificación</h3>

              {isLoading ? (
                <p className={styles.loading}>Cargando...</p>
              ) : (
                <RatingInput
                  currentRating={userRating}
                  onRate={handleRatingSubmit}
                  onDelete={handleRatingDelete}
                  size="large"
                  showLabel
                />
              )}

              {error && (
                <p className={styles.error}>{error}</p>
              )}
            </div>
          </div>
        </div>
      </div>
    </section>
  );
}
```

**Estilos**: `Frontend/src/components/RatingSection/RatingSection.module.scss`

```scss
// Frontend/src/components/RatingSection/RatingSection.module.scss

.ratingSection {
  padding: 3rem 0;
  background: #f9fafb; // gray-50

  .container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 1.5rem;
  }

  .header {
    margin-bottom: 2rem;

    .title {
      font-size: 1.875rem;
      font-weight: 700;
      color: #111827; // gray-900
      margin: 0 0 1rem 0;
    }

    .averageDisplay {
      display: flex;
      align-items: center;
      gap: 1rem;

      .averageNumber {
        font-size: 3rem;
        font-weight: 700;
        color: #111827; // gray-900
      }
    }
  }

  .content {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 3rem;

    @media (max-width: 768px) {
      grid-template-columns: 1fr;
      gap: 2rem;
    }
  }

  .leftColumn {
    // Distribution takes full width
  }

  .rightColumn {
    // User rating input
  }

  .userRatingCard {
    background: white;
    border-radius: 0.75rem;
    padding: 2rem;
    box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1);

    .cardTitle {
      font-size: 1.25rem;
      font-weight: 600;
      color: #111827; // gray-900
      margin: 0 0 1.5rem 0;
    }

    .loading {
      color: #6b7280; // gray-500
      font-style: italic;
    }

    .error {
      color: #ef4444; // red-500
      font-size: 0.875rem;
      margin-top: 0.5rem;
    }
  }
}
```

---

### Fase 4: Testing y Polish (1.5 horas)

#### Task 4.1: Tests de Integración (45 min)
**Archivo**: `Frontend/src/components/RatingSection/RatingSection.test.tsx`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { RatingSection } from './RatingSection';
import { ratingsApi } from '@/services/ratingsApi';
import { userIdService } from '@/services/userIdService';

vi.mock('@/services/ratingsApi');
vi.mock('@/services/userIdService');

describe('RatingSection Integration', () => {
  const mockStats = {
    average_rating: 4.5,
    total_ratings: 100,
    rating_distribution: { 1: 5, 2: 10, 3: 15, 4: 30, 5: 40 }
  };

  beforeEach(() => {
    vi.clearAllMocks();
    vi.mocked(userIdService.getUserId).mockReturnValue(12345);
  });

  it('should render rating section with stats', () => {
    render(<RatingSection courseId={1} initialStats={mockStats} />);

    expect(screen.getByText('Calificaciones del curso')).toBeInTheDocument();
    expect(screen.getByText('4.5')).toBeInTheDocument();
  });

  it('should allow user to submit a rating', async () => {
    vi.mocked(ratingsApi.getUserRating).mockResolvedValue(null);
    vi.mocked(ratingsApi.createRating).mockResolvedValue({
      id: 1,
      course_id: 1,
      user_id: 12345,
      rating: 5,
      created_at: new Date().toISOString(),
      updated_at: new Date().toISOString()
    });

    render(<RatingSection courseId={1} initialStats={mockStats} />);

    await waitFor(() => {
      expect(screen.getByLabelText('Calificar con 5 estrellas')).toBeInTheDocument();
    });

    const fiveStar = screen.getByLabelText('Calificar con 5 estrellas');
    fireEvent.click(fiveStar);

    await waitFor(() => {
      expect(ratingsApi.createRating).toHaveBeenCalledWith(1, {
        user_id: 12345,
        rating: 5
      });
    });
  });
});
```

#### Task 4.2: Accessibility Audit (30 min)
- Verificar contraste de colores (WCAG AA)
- Añadir `aria-labels` faltantes
- Testar navegación por teclado
- Validar con screen readers

**Checklist**:
```typescript
// Accesibilidad checklist
const a11yChecklist = {
  keyboard: {
    // ✓ Tab navigation funciona
    // ✓ Enter/Space activan botones
    // ✓ Escape cierra modales (si existen)
  },
  screenReader: {
    // ✓ Todos los botones tienen aria-label
    // ✓ Estados de loading anunciados
    // ✓ Errores anunciados con role="alert"
  },
  visual: {
    // ✓ Contraste > 4.5:1 para texto normal
    // ✓ Contraste > 3:1 para texto grande
    // ✓ Focus visible en todos los interactivos
  }
};
```

#### Task 4.3: Performance Optimization (15 min)
- Lazy load RatingSection si está below the fold
- Memoizar cálculos de distribución
- Optimistic updates para mejor UX

```typescript
// Optimistic update example
async function handleRatingSubmit(rating: number) {
  // Update UI immediately
  setStats(prev => {
    const newDistribution = { ...prev.rating_distribution };
    newDistribution[rating] = (newDistribution[rating] || 0) + 1;

    return {
      ...prev,
      total_ratings: prev.total_ratings + 1,
      rating_distribution: newDistribution,
      average_rating: calculateAverage(newDistribution)
    };
  });

  // Then sync with server
  const success = await submitRating(rating);

  if (!success) {
    // Revert on failure
    await refreshStats();
  }

  return success;
}
```

---

## Estructura de Archivos Final

```
Frontend/src/
├── app/
│   └── course/[slug]/
│       └── page.tsx                    # ✏️ MODIFICADO
│
├── components/
│   ├── RatingInput/                    # ✨ NUEVO
│   │   ├── RatingInput.tsx
│   │   ├── RatingInput.module.scss
│   │   └── RatingInput.test.tsx
│   │
│   ├── RatingDistribution/             # ✨ NUEVO
│   │   ├── RatingDistribution.tsx
│   │   ├── RatingDistribution.module.scss
│   │   └── RatingDistribution.test.tsx
│   │
│   └── RatingSection/                  # ✨ NUEVO
│       ├── RatingSection.tsx
│       ├── RatingSection.module.scss
│       └── RatingSection.test.tsx
│
├── hooks/
│   └── useUserRating.ts                # ✨ NUEVO
│
├── services/
│   ├── ratingsApi.ts                   # ✅ YA EXISTE
│   └── userIdService.ts                # ✨ NUEVO
│
└── utils/
    ├── ratingHelpers.ts                # ✨ NUEVO
    └── ratingHelpers.test.ts           # ✨ NUEVO
```

---

## Resumen de Tareas

### Desglose de Tiempo

| Fase | Tarea | Tiempo Estimado | Dependencias |
|------|-------|----------------|--------------|
| **Fase 1** | UserID Service | 45 min | - |
| | Rating Helpers | 30 min | - |
| | useUserRating Hook | 45 min | UserID Service, ratingsApi |
| **Fase 2** | RatingInput Component | 1.5 hrs | useUserRating |
| | RatingDistribution | 1.5 hrs | Rating Helpers |
| **Fase 3** | RatingSection | 30 min | RatingInput, RatingDistribution |
| | Update CourseDetail | 1 hr | RatingSection |
| **Fase 4** | Integration Tests | 45 min | Todos los componentes |
| | Accessibility | 30 min | - |
| | Performance | 15 min | - |
| **TOTAL** | | **~8 horas** | |

### Orden de Implementación

```
1. userIdService.ts         (no deps)
2. ratingHelpers.ts         (no deps)
3. useUserRating.ts         (deps: userIdService, ratingsApi)
4. RatingInput              (deps: useUserRating)
5. RatingDistribution       (deps: ratingHelpers)
6. RatingSection            (deps: RatingInput, RatingDistribution)
7. Update CourseDetail      (deps: RatingSection)
8. Tests                    (deps: all)
```

---

## Estrategia de Testing

### Unit Tests (Vitest)
- ✅ `userIdService` - localStorage operations
- ✅ `ratingHelpers` - transformations and calculations
- ✅ `RatingInput` - user interactions
- ✅ `RatingDistribution` - visual rendering
- ✅ `useUserRating` - hook logic

### Integration Tests
- ✅ `RatingSection` - full user flow
- ✅ API calls mocked con `vi.mock`
- ✅ Optimistic updates behavior

### E2E Tests (Opcional - Playwright)
```typescript
test('User can rate a course', async ({ page }) => {
  await page.goto('/course/curso-python');

  await page.click('[aria-label="Calificar con 5 estrellas"]');

  await expect(page.locator('.averageNumber')).toContainText('5.0');
});
```

---

## Consideraciones Finales

### Limitaciones de la Solución MVP
1. **Sin autenticación real**: Usar localStorage es temporal
2. **Múltiples ratings**: Un usuario puede calificar múltiples veces limpiando localStorage
3. **Sin validación de identidad**: Cualquiera puede usar cualquier user_id
4. **Sin persistencia cross-device**: Rating se pierde en otro dispositivo

### Mejoras Futuras (Post-MVP)
1. Implementar autenticación JWT
2. Asociar ratings a usuarios autenticados
3. Historial de cambios de rating
4. Moderación de ratings
5. Análisis de sentimiento en reviews textuales

### Notas de Seguridad
- No confiar en `user_id` del frontend
- Backend debe validar permisos en fase de auth
- Rate limiting necesario en producción
- CORS debe configurarse correctamente

---

## Criterios de Aceptación

### Funcionalidad
- [ ] Usuario puede ver su rating previo al cargar la página
- [ ] Usuario puede calificar de 1 a 5 estrellas
- [ ] Usuario puede cambiar su rating existente
- [ ] Usuario puede eliminar su rating
- [ ] Estadísticas se actualizan después de calificar
- [ ] Distribución muestra porcentajes correctos
- [ ] Promedio se calcula correctamente

### UX
- [ ] Feedback visual inmediato (hover, click)
- [ ] Loading states durante requests
- [ ] Error handling con mensajes claros
- [ ] Responsive en mobile y desktop
- [ ] Accesible con teclado
- [ ] Compatible con screen readers

### Performance
- [ ] No hay flickering en carga inicial
- [ ] Optimistic updates se sienten instantáneos
- [ ] Sin re-renders innecesarios
- [ ] Lazy loading de componentes pesados

### Testing
- [ ] >80% code coverage
- [ ] Todos los tests pasan
- [ ] No warnings en consola
- [ ] Build de producción exitoso

---

**Fin del documento**
