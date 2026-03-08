# TaleForge Workspace

Monorepo workspace combining the TaleForge backend and frontend via git submodules.

```
taleforge-workspace/
  backend/    → https://github.com/amitrhodes/taleforge-backend.git
  frontend/   → https://github.com/amitrhodes/taleforge-frontend.git
```

## Submodule Commands

```bash
# Clone workspace with submodules
git clone --recurse-submodules <workspace-url>

# If already cloned without submodules
git submodule update --init --recursive

# Update submodules to latest remote commits
git submodule update --remote --merge

# Commit a submodule pointer update
git add backend frontend && git commit -m "Update submodule refs"
```

## Dev Workflow

Run backend and frontend in separate terminals:

```bash
# Terminal 1 — Backend (Java/Spring Boot, port 8080)
cd backend
mvn spring-boot:run

# Terminal 2 — Frontend (Vite, port 5173)
cd frontend
npm run dev
```

Frontend proxies `/api` → `http://localhost:8080` via Vite config.

## Backend (`backend/`)

**Stack**: Java 21, Spring Boot 3.2.3, PostgreSQL 16, Flyway, JWT (jjwt 0.12.3), Lombok 1.18.38, Maven

```bash
mvn clean package -DskipTests   # build
mvn spring-boot:run             # run
pkill -9 -f "taleforge"         # kill
```

No `mvnw` — use system `mvn` from `/opt/homebrew/bin/mvn`.

### Package Structure

```
src/main/java/com/taleforge/
  controller/   AuthController, AuthorController, BookController,
                SeriesController, SearchController, RequestController
  service/      AuthService, AuthorService, BookService, SeriesService,
                SearchService, JwtService
  repository/   UserRepository, AuthorRepository, BookRepository,
                SeriesRepository, CharacterRepository, LoreItemRepository,
                PageRequestRepository
  model/        User, Author, Book, Series, Character, LoreItem, Genre, PageRequest
  dto/          AuthRequest, AuthResponse, AuthorDto, BookDto, SeriesDto,
                CharacterDto, LoreItemDto, SearchResultDto, PageRequestDto
  config/       SecurityConfig, CorsConfig
  security/     JwtUtil, JwtAuthFilter, UserDetailsServiceImpl
  exception/    GlobalExceptionHandler
src/main/resources/
  application.yml
  db/migration/   V1__init.sql, V2__seed.sql, V3__add_genres.sql, ...
```

### Database

- **PostgreSQL**: DB `taleforge`, user `taleforge_user`, password `Papita67`, port 5432
- **Flyway**: manages all schema changes — never edit `ddl-auto`, it's set to `validate`
- **Key tables**: `users`, `authors`, `series`, `books`, `characters`, `lore_items`, `genres`, `book_genres`, `page_requests`
- **Seed**: test@example.com / password, GRRM / ASOIAF data (author id=1, series id=1, books id=1–7), 69 genres

Flyway repair (after editing an already-applied migration):
```bash
mvn flyway:repair \
  -Dflyway.url=jdbc:postgresql://localhost:5432/taleforge \
  -Dflyway.user=taleforge_user \
  -Dflyway.password=Papita67
```

### API Endpoints (all at `http://localhost:8080`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/auth/register` | public | Register |
| POST | `/api/auth/login` | public | Login → tokens |
| POST | `/api/auth/refresh` | public | Refresh (`X-Refresh-Token` header) |
| GET | `/api/books/{id}` | public | Book details + genres |
| GET | `/api/books/{id}/characters` | public | Characters |
| GET | `/api/books/{id}/lore` | public | Lore/FAQ |
| GET | `/api/authors/{id}` | public | Author details |
| GET | `/api/authors/{id}/books` | public | Books by author |
| GET | `/api/authors/{id}/series` | public | Series by author |
| GET | `/api/series/{id}` | public | Series details |
| GET | `/api/series/{id}/books` | public | Books in series |
| GET | `/api/search?q=&type=` | public | Search (books/authors/series) |
| POST | `/api/requests` | public | Submit content request |

No list endpoints (`GET /api/books`, etc.) — by design.

JWT: access token 15 min (in-memory on client), refresh token 7 days (localStorage).

### Known Gotchas

- **Lombok on Java 25**: requires `lombok.version=1.18.38` + explicit `annotationProcessorPaths` in `maven-compiler-plugin`
- **Flyway**: do NOT add `flyway-database-postgresql` dep — pg support is built into `flyway-core`
- **DB permissions**: `taleforge_user` needs `GRANT USAGE, CREATE ON SCHEMA public`
- **Auth errors**: `BadCredentialsException` → HTTP 401 via `GlobalExceptionHandler`

---

## Frontend (`frontend/`)

**Stack**: React 18, TypeScript, Vite 5, React Router v6, TanStack Query v5, Axios, Tailwind CSS v3, Lucide React

```bash
npm run dev     # Vite dev server at http://localhost:5173
npm run build   # tsc + vite build
npm run lint    # ESLint
```

### Structure

```
src/
  App.tsx                  # QueryClientProvider + BrowserRouter + Routes
  main.tsx
  index.css                # Tailwind base imports
  styles/tokens.css        # CSS custom properties (mirrors Tailwind tokens)
  types/index.ts           # Book, Author, Series, Character, LoreItem, SearchResult, AuthUser

  pages/
    HomePage.tsx           # Search + filter + grid/list results
    BookPage.tsx
    AuthorPage.tsx
    SeriesPage.tsx

  components/
    layout/
      PageLayout.tsx       # Sticky header + max-width wrapper + BackToTop + ToastContainer
      StickyHeader.tsx
    ui/
      Button.tsx
      Card.tsx             # CardBook, CardAuthor, CardSeries
      Chip.tsx             # variants: default, ongoing, planned
      SearchBar.tsx        # Controlled input with 300ms debounce
      Toast.tsx            # showToast({ type, message }) + ToastContainer
      Accordion.tsx
      BackToTop.tsx

  hooks/
    useAuth.ts             # login / logout, in-memory user state
    useSearch.ts           # URL-param-driven search
    useAccordion.ts
    useBackToTop.ts

  services/
    api.ts                 # Axios (baseURL: /api) + JWT Bearer interceptor + 401 refresh logic
    auth.ts                # login(), register(), logout()
    books.ts, authors.ts, series.ts, search.ts
```

### Routes

| Path | Page |
|------|------|
| `/` | HomePage |
| `/books/:id` | BookPage |
| `/authors/:id` | AuthorPage |
| `/series/:id` | SeriesPage |

### Auth Flow

- Access token: in-memory module variable in `api.ts`; set via `setAccessToken(token)`
- Refresh token: `localStorage` key `refreshToken`
- 401 interceptor: auto-refresh via `POST /api/auth/refresh` with `X-Refresh-Token`, retries original request; skips `/auth/*` routes

### Design Tokens (Tailwind)

| Token | Value |
|-------|-------|
| `bg-bg` | `#0B0F13` |
| `bg-surface-card` | `#12171D` |
| `bg-surface-subtle` | `#0E1318` |
| `text-accent` | `#9BD3F7` |
| `text-text-primary` | `#F2F5F7` |
| `text-text-secondary` | `#C7D0D8` |
| `border-border` | `#26303A` |
| `rounded-card` | `16px` |
| `rounded-input` | `12px` |
| `rounded-chip` | `8px` |
| `max-w-content` | `1100px` |

### Key Conventions

- Pages use `PageLayout` wrapper
- Search state lives in URL params (`?q=...&type=...&view=grid|list`)
- `SearchBar` debounces `onChange` by 300ms
- Toast via `showToast({ type: 'success'|'error'|'info', message })` from `components/ui/Toast`
- Cards link via React Router `<Link>`
- Use `type="text"` not `type="search"` (avoids double clear buttons)
- TanStack Query config: `staleTime: 5min, retry: 1`
