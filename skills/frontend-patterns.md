---
name: frontend-patterns
description: Frontend development patterns for Inertia.js with React, TypeScript, state management, performance optimization, and UI best practices in Laravel applications.
---

# Frontend Development Patterns

Modern frontend patterns for Laravel with Inertia.js and React.

## Inertia.js Data Flow

```
Laravel Controller
    | (Inertia::render with props)
React Page Component
    | (props drilling or context)
Child Components
```

## Component Patterns

### Composition Over Inheritance

```typescript
// Good: Component composition
interface CardProps {
  children: React.ReactNode
  variant?: 'default' | 'outlined'
}

export function Card({ children, variant = 'default' }: CardProps) {
  return <div className={`card card-${variant}`}>{children}</div>
}

export function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>
}

export function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>
}

// Usage
<Card>
  <CardHeader>Title</CardHeader>
  <CardBody>Content</CardBody>
</Card>
```

### Inertia Page Component

```typescript
import { Head } from '@inertiajs/react'

interface Market {
  id: number
  name: string
  description: string
  status: 'active' | 'resolved' | 'closed'
}

interface Props {
  markets: Market[]
  filters: {
    status: string | null
    search: string | null
  }
}

export default function MarketsIndex({ markets, filters }: Props) {
  return (
    <>
      <Head title="Markets" />

      <div className="container mx-auto py-8">
        <h1 className="text-2xl font-bold mb-6">Markets</h1>

        <div className="grid gap-4">
          {markets.map((market) => (
            <MarketCard key={market.id} market={market} />
          ))}
        </div>
      </div>
    </>
  )
}
```

### Compound Components

```typescript
interface TabsContextValue {
  activeTab: string
  setActiveTab: (tab: string) => void
}

const TabsContext = createContext<TabsContextValue | undefined>(undefined)

export function Tabs({ children, defaultTab }: {
  children: React.ReactNode
  defaultTab: string
}) {
  const [activeTab, setActiveTab] = useState(defaultTab)

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      {children}
    </TabsContext.Provider>
  )
}

export function TabList({ children }: { children: React.ReactNode }) {
  return <div className="tab-list">{children}</div>
}

export function Tab({ id, children }: { id: string, children: React.ReactNode }) {
  const context = useContext(TabsContext)
  if (!context) throw new Error('Tab must be used within Tabs')

  return (
    <button
      className={context.activeTab === id ? 'active' : ''}
      onClick={() => context.setActiveTab(id)}
    >
      {children}
    </button>
  )
}

// Usage
<Tabs defaultTab="overview">
  <TabList>
    <Tab id="overview">Overview</Tab>
    <Tab id="details">Details</Tab>
  </TabList>
</Tabs>
```

## Inertia.js Patterns

### Form Handling with useForm

```typescript
import { useForm } from '@inertiajs/react'

interface FormData {
  name: string
  description: string
  end_date: string
  category_ids: number[]
}

export function CreateMarketForm() {
  const { data, setData, post, processing, errors, reset } = useForm<FormData>({
    name: '',
    description: '',
    end_date: '',
    category_ids: [],
  })

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    post(route('markets.store'), {
      onSuccess: () => reset(),
    })
  }

  return (
    <form onSubmit={handleSubmit}>
      <div className="mb-4">
        <label htmlFor="name">Market Name</label>
        <input
          id="name"
          type="text"
          value={data.name}
          onChange={(e) => setData('name', e.target.value)}
          className={errors.name ? 'border-red-500' : ''}
        />
        {errors.name && (
          <p className="text-red-500 text-sm mt-1">{errors.name}</p>
        )}
      </div>

      <div className="mb-4">
        <label htmlFor="description">Description</label>
        <textarea
          id="description"
          value={data.description}
          onChange={(e) => setData('description', e.target.value)}
          className={errors.description ? 'border-red-500' : ''}
        />
        {errors.description && (
          <p className="text-red-500 text-sm mt-1">{errors.description}</p>
        )}
      </div>

      <button type="submit" disabled={processing}>
        {processing ? 'Creating...' : 'Create Market'}
      </button>
    </form>
  )
}
```

### Navigation with Inertia Link

```typescript
import { Link, router } from '@inertiajs/react'

// Declarative navigation
<Link href={route('markets.show', market.id)} className="text-blue-600">
  {market.name}
</Link>

// Preserve scroll position
<Link href={route('markets.index')} preserveScroll>
  Back to Markets
</Link>

// Preserve state
<Link href={route('markets.index', { page: 2 })} preserveState>
  Page 2
</Link>

// Programmatic navigation
function handleClick() {
  router.visit(route('markets.show', market.id))
}

// With method
function handleDelete() {
  router.delete(route('markets.destroy', market.id), {
    onBefore: () => confirm('Are you sure?'),
    onSuccess: () => toast.success('Deleted!'),
  })
}
```

### Shared Data Access

```typescript
import { usePage } from '@inertiajs/react'

interface SharedData {
  auth: {
    user: {
      id: number
      name: string
      email: string
    } | null
  }
  flash: {
    success?: string
    error?: string
  }
}

export function useAuth() {
  const { auth } = usePage<SharedData>().props
  return auth
}

export function useFlash() {
  const { flash } = usePage<SharedData>().props
  return flash
}

// Usage in component
function Header() {
  const { user } = useAuth()

  return (
    <header>
      {user ? (
        <span>Welcome, {user.name}</span>
      ) : (
        <Link href={route('login')}>Login</Link>
      )}
    </header>
  )
}
```

### Real-time Validation

```typescript
import { useForm } from '@inertiajs/react'
import { useCallback } from 'react'
import debounce from 'lodash/debounce'

export function CreateMarketForm() {
  const { data, setData, post, processing, errors, clearErrors } = useForm({
    name: '',
    description: '',
  })

  // Debounced server-side validation
  const validateField = useCallback(
    debounce((field: string, value: string) => {
      router.post(route('markets.validate'), {
        [field]: value,
      }, {
        preserveState: true,
        preserveScroll: true,
        only: ['errors'],
      })
    }, 500),
    []
  )

  const handleChange = (field: keyof typeof data, value: string) => {
    setData(field, value)
    clearErrors(field)
    validateField(field, value)
  }

  return (
    <form onSubmit={(e) => { e.preventDefault(); post(route('markets.store')) }}>
      <input
        value={data.name}
        onChange={(e) => handleChange('name', e.target.value)}
      />
      {errors.name && <span className="error">{errors.name}</span>}
    </form>
  )
}
```

## Custom Hooks Patterns

### State Management Hook

```typescript
export function useToggle(initialValue = false): [boolean, () => void] {
  const [value, setValue] = useState(initialValue)

  const toggle = useCallback(() => {
    setValue((v) => !v)
  }, [])

  return [value, toggle]
}

// Usage
const [isOpen, toggleOpen] = useToggle()
```

### Debounce Hook

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// Usage with Inertia
function MarketSearch() {
  const [search, setSearch] = useState('')
  const debouncedSearch = useDebounce(search, 500)

  useEffect(() => {
    if (debouncedSearch) {
      router.get(route('markets.index'), {
        search: debouncedSearch,
      }, {
        preserveState: true,
        preserveScroll: true,
      })
    }
  }, [debouncedSearch])

  return (
    <input
      type="text"
      value={search}
      onChange={(e) => setSearch(e.target.value)}
      placeholder="Search markets..."
    />
  )
}
```

### Local Storage Hook

```typescript
export function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === 'undefined') {
      return initialValue
    }
    try {
      const item = window.localStorage.getItem(key)
      return item ? JSON.parse(item) : initialValue
    } catch {
      return initialValue
    }
  })

  const setValue = useCallback((value: T | ((val: T) => T)) => {
    setStoredValue((prev) => {
      const valueToStore = value instanceof Function ? value(prev) : value
      window.localStorage.setItem(key, JSON.stringify(valueToStore))
      return valueToStore
    })
  }, [key])

  return [storedValue, setValue] as const
}

// Usage
const [theme, setTheme] = useLocalStorage('theme', 'light')
```

## State Management Patterns

### Context + Reducer Pattern

```typescript
interface State {
  markets: Market[]
  selectedMarket: Market | null
  loading: boolean
}

type Action =
  | { type: 'SET_MARKETS'; payload: Market[] }
  | { type: 'SELECT_MARKET'; payload: Market }
  | { type: 'SET_LOADING'; payload: boolean }

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'SET_MARKETS':
      return { ...state, markets: action.payload }
    case 'SELECT_MARKET':
      return { ...state, selectedMarket: action.payload }
    case 'SET_LOADING':
      return { ...state, loading: action.payload }
    default:
      return state
  }
}

const MarketContext = createContext<{
  state: State
  dispatch: Dispatch<Action>
} | undefined>(undefined)

export function MarketProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, {
    markets: [],
    selectedMarket: null,
    loading: false,
  })

  return (
    <MarketContext.Provider value={{ state, dispatch }}>
      {children}
    </MarketContext.Provider>
  )
}

export function useMarkets() {
  const context = useContext(MarketContext)
  if (!context) throw new Error('useMarkets must be used within MarketProvider')
  return context
}
```

## Performance Optimization

### Memoization

```typescript
// useMemo for expensive computations
const sortedMarkets = useMemo(() => {
  return markets.sort((a, b) => b.volume - a.volume)
}, [markets])

// useCallback for functions passed to children
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])

// React.memo for pure components
export const MarketCard = React.memo<MarketCardProps>(({ market }) => {
  return (
    <div className="market-card">
      <h3>{market.name}</h3>
      <p>{market.description}</p>
    </div>
  )
})
```

### Code Splitting & Lazy Loading

```typescript
import { lazy, Suspense } from 'react'

// Lazy load heavy components
const HeavyChart = lazy(() => import('./HeavyChart'))
const ThreeJsBackground = lazy(() => import('./ThreeJsBackground'))

export function Dashboard() {
  return (
    <div>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart data={data} />
      </Suspense>

      <Suspense fallback={null}>
        <ThreeJsBackground />
      </Suspense>
    </div>
  )
}
```

### Virtualization for Long Lists

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

export function VirtualMarketList({ markets }: { markets: Market[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: markets.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100,
    overscan: 5,
  })

  return (
    <div ref={parentRef} className="h-[600px] overflow-auto">
      <div
        style={{
          height: `${virtualizer.getTotalSize()}px`,
          position: 'relative',
        }}
      >
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <MarketCard market={markets[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

## Form Handling Patterns

### Controlled Form with Inertia

```typescript
import { useForm } from '@inertiajs/react'

interface FormData {
  name: string
  description: string
  end_date: string
}

export function CreateMarketForm() {
  const { data, setData, post, processing, errors } = useForm<FormData>({
    name: '',
    description: '',
    end_date: '',
  })

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    post(route('markets.store'))
  }

  return (
    <form onSubmit={handleSubmit}>
      <div className="mb-4">
        <label htmlFor="name" className="block text-sm font-medium">
          Market Name
        </label>
        <input
          id="name"
          type="text"
          value={data.name}
          onChange={(e) => setData('name', e.target.value)}
          className={`mt-1 block w-full rounded-md ${
            errors.name ? 'border-red-500' : 'border-gray-300'
          }`}
        />
        {errors.name && (
          <p className="mt-1 text-sm text-red-500">{errors.name}</p>
        )}
      </div>

      <div className="mb-4">
        <label htmlFor="description" className="block text-sm font-medium">
          Description
        </label>
        <textarea
          id="description"
          value={data.description}
          onChange={(e) => setData('description', e.target.value)}
          rows={4}
          className={`mt-1 block w-full rounded-md ${
            errors.description ? 'border-red-500' : 'border-gray-300'
          }`}
        />
        {errors.description && (
          <p className="mt-1 text-sm text-red-500">{errors.description}</p>
        )}
      </div>

      <button
        type="submit"
        disabled={processing}
        className="px-4 py-2 bg-blue-600 text-white rounded-md disabled:opacity-50"
      >
        {processing ? 'Creating...' : 'Create Market'}
      </button>
    </form>
  )
}
```

## Error Boundary Pattern

```typescript
interface ErrorBoundaryState {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = {
    hasError: false,
    error: null,
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Error boundary caught:', error, errorInfo)
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback p-8 text-center">
          <h2 className="text-xl font-bold text-red-600">Something went wrong</h2>
          <p className="mt-2 text-gray-600">{this.state.error?.message}</p>
          <button
            onClick={() => this.setState({ hasError: false })}
            className="mt-4 px-4 py-2 bg-blue-600 text-white rounded"
          >
            Try again
          </button>
        </div>
      )
    }

    return this.props.children
  }
}

// Usage
<ErrorBoundary>
  <App />
</ErrorBoundary>
```

## Accessibility Patterns

### Keyboard Navigation

```typescript
export function Dropdown({ options, onSelect }: DropdownProps) {
  const [isOpen, setIsOpen] = useState(false)
  const [activeIndex, setActiveIndex] = useState(0)

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault()
        setActiveIndex((i) => Math.min(i + 1, options.length - 1))
        break
      case 'ArrowUp':
        e.preventDefault()
        setActiveIndex((i) => Math.max(i - 1, 0))
        break
      case 'Enter':
        e.preventDefault()
        onSelect(options[activeIndex])
        setIsOpen(false)
        break
      case 'Escape':
        setIsOpen(false)
        break
    }
  }

  return (
    <div
      role="combobox"
      aria-expanded={isOpen}
      aria-haspopup="listbox"
      onKeyDown={handleKeyDown}
    >
      {/* Dropdown implementation */}
    </div>
  )
}
```

### Focus Management

```typescript
export function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)
  const previousFocusRef = useRef<HTMLElement | null>(null)

  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement
      modalRef.current?.focus()
    } else {
      previousFocusRef.current?.focus()
    }
  }, [isOpen])

  if (!isOpen) return null

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      tabIndex={-1}
      onKeyDown={(e) => e.key === 'Escape' && onClose()}
      className="fixed inset-0 z-50 flex items-center justify-center"
    >
      <div className="absolute inset-0 bg-black/50" onClick={onClose} />
      <div className="relative bg-white rounded-lg p-6 max-w-md w-full">
        {children}
      </div>
    </div>
  )
}
```

## File Organization

### Component Structure

```
resources/js/
├── Components/        # Reusable React components
│   ├── UI/           # Base UI components (Button, Input, etc.)
│   ├── Forms/        # Form components
│   └── Layouts/      # Layout components
├── Hooks/            # Custom React hooks
├── Layouts/          # Page layouts
├── Pages/            # Inertia pages (match routes)
│   ├── Markets/
│   │   ├── Index.tsx
│   │   ├── Show.tsx
│   │   └── Create.tsx
│   └── Auth/
│       ├── Login.tsx
│       └── Register.tsx
├── Types/            # TypeScript types
│   ├── index.d.ts
│   └── models.ts
└── Utils/            # Utility functions
```

### Type Definitions

```typescript
// resources/js/Types/models.ts
export interface User {
  id: number
  name: string
  email: string
  created_at: string
}

export interface Market {
  id: number
  name: string
  description: string
  status: 'active' | 'resolved' | 'closed'
  end_date: string
  user?: User
  orders_count?: number
}

// resources/js/Types/index.d.ts
import { PageProps as InertiaPageProps } from '@inertiajs/core'

export interface PageProps extends InertiaPageProps {
  auth: {
    user: User | null
  }
  flash: {
    success?: string
    error?: string
  }
}

declare module '@inertiajs/react' {
  export function usePage<T extends PageProps>(): { props: T }
}
```

**Remember**: Modern frontend patterns enable maintainable, performant user interfaces. Use Inertia.js conventions for seamless Laravel integration.
