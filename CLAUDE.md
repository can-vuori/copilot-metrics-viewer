# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GitHub Copilot Metrics Viewer is a Nuxt 3 application that displays GitHub Copilot usage analytics for organizations and enterprises. Built with Vue 3, TypeScript, Vuetify, and Chart.js, it visualizes data from the GitHub Copilot Metrics API.

## Development Commands

### Initial Setup
```bash
npm install  # Takes ~3 minutes, includes postinstall script (nuxt prepare)
```

### Development
```bash
npm run dev           # Start development server at http://localhost:3000
npm run build         # Production build (~30 seconds)
npm run preview       # Preview production build
```

### Testing
```bash
npm test              # Run unit tests with Vitest (83 tests)
npm run test:e2e      # Run E2E tests with Playwright
npm run typecheck     # Type checking (currently has 18 known errors)
```

### Code Quality
```bash
npm run lint          # Run ESLint (currently has 43 known errors)
npm run lint:fix      # Auto-fix linting issues
```

### Docker
```bash
docker build -t copilot-metrics-viewer .
docker run -p 8080:80 --env-file ./.env copilot-metrics-viewer
```

## Architecture

### Application Structure

**Frontend (app/)**
- **pages/**: Single page application (index.vue)
- **components/**: Vue components for metrics visualization
  - `MetricsViewer.vue` - Main metrics dashboard with acceptance rates, suggestions, and active users
  - `SeatsAnalysisViewer.vue` - Seat allocation and usage analysis
  - `CopilotChatViewer.vue` - Chat metrics visualization
  - `AgentModeViewer.vue` - GitHub.com integration and model analytics
  - `DateRangeSelector.vue` - Date filtering interface
  - `MainComponent.vue` - Primary layout wrapper
- **model/**: TypeScript data models and converters
  - `Copilot_Metrics.ts` - Core metrics type definitions
  - `MetricsToUsageConverter.ts` - Converts between API formats (old metrics → new usage)
  - `Options.ts` - Query parameter handling and scope management
  - `Seat.ts`, `Breakdown.ts` - Supporting types
- **utils/**: Frontend utilities (csvExport.ts, dateUtils.ts)

**Backend (server/)**
- **api/**: API endpoints
  - `metrics.ts` - Main metrics endpoint, fetches and converts usage data
  - `seats.ts` - Seat analysis data
  - `health.ts`, `ready.ts`, `live.ts` - Health check endpoints (no auth required)
  - `teams.ts` - Team comparison data
  - `github-stats.ts`, `repository-stats.ts` - Additional GitHub statistics
- **middleware/**:
  - `github.ts` - Authentication and scope detection (org/team/enterprise)
  - `log.ts` - Request logging
- **modules/authentication.ts** - GitHub authentication logic (PAT or OAuth)
- **plugins/http-agent.ts** - HTTP proxy support for corporate environments

**Shared (shared/utils/)**
- `metrics-util.ts` - Core metrics fetching logic with caching and mock data support
- `getLocale.ts`, `capitalize.ts`, `getDisplayName.ts` - Utility functions

### Key Architectural Patterns

**Dual API Format Support**: The application handles both old "metrics" and new "usage" API formats. The `/api/metrics` endpoint fetches data as "usage" format and converts to "metrics" format for backward compatibility using `MetricsToUsageConverter`.

**Authentication Flow**:
1. All API routes pass through `server/middleware/github.ts` (except health endpoints)
2. Middleware calls `authenticateAndGetGitHubHeaders()` which supports:
   - Mock mode (no auth required)
   - Personal Access Token (PAT) from `NUXT_GITHUB_TOKEN`
   - OAuth via GitHub App (when `NUXT_PUBLIC_USING_GITHUB_AUTH=true`)
3. Headers are attached to event context and used for GitHub API calls

**Caching Strategy**:
- Implemented in `shared/utils/metrics-util.ts`
- Cache keys are bound to authentication fingerprint + path + query parameters
- 5-minute TTL prevents unauthorized data leakage
- Automatic cache invalidation on errors

**Scope Detection**:
- Supports organization, enterprise, team-organization, and team-enterprise scopes
- Can be set via environment variables or URL route parameters
- Priority: Route params > Query params > Environment variables
- Examples:
  - `/orgs/my-org` → organization scope
  - `/enterprises/my-ent` → enterprise scope
  - `/orgs/my-org/teams/my-team` → team-organization scope

**Mock Data System**:
- Activated via `NUXT_PUBLIC_IS_DATA_MOCKED=true` or `?mock=true` query param
- Dynamic date range adjustment for mock data based on `since`/`until` parameters
- Holiday filtering support with locale-specific holiday detection

**Options Pattern**: The `Options` class (app/model/Options.ts) centralizes:
- Query parameter parsing and serialization
- API URL construction based on scope
- Mock data path resolution
- Date range and locale management

### Important Files

- **nuxt.config.ts** - Nuxt configuration including runtime config, modules (Vuetify, nuxt-auth-utils), SSR settings
- **server/middleware/github.ts** - Authentication entry point, skips health endpoints
- **shared/utils/metrics-util.ts** - Core data fetching, caching, mock data handling
- **app/model/MetricsToUsageConverter.ts** - Converts between API response formats

## Environment Configuration

### Required Variables
```bash
NUXT_SESSION_PASSWORD=<min_32_chars>  # Required for session encryption
NUXT_PUBLIC_SCOPE=organization        # 'organization' | 'enterprise' | 'team-organization' | 'team-enterprise'
```

### GitHub Integration
```bash
# Option 1: Personal Access Token (simpler)
NUXT_GITHUB_TOKEN=<token>  # Scopes: copilot, manage_billing:copilot, manage_billing:enterprise, read:enterprise, read:org

# Option 2: GitHub OAuth App (for multi-user deployments)
NUXT_PUBLIC_USING_GITHUB_AUTH=true
NUXT_OAUTH_GITHUB_CLIENT_ID=<client_id>
NUXT_OAUTH_GITHUB_CLIENT_SECRET=<client_secret>
```

### Scope Configuration
```bash
NUXT_PUBLIC_GITHUB_ORG=<org_name>   # For organization scope
NUXT_PUBLIC_GITHUB_ENT=<ent_name>   # For enterprise scope
NUXT_PUBLIC_GITHUB_TEAM=<team_name> # Optional: filter to specific team
```

### Mock Mode (Development)
```bash
NUXT_PUBLIC_IS_DATA_MOCKED=true  # Use sample data, no GitHub token needed
```

### Advanced
```bash
HTTP_PROXY=<proxy_url>           # Corporate proxy support
CUSTOM_CA_PATH=<cert_path>       # Custom CA certificate
NITRO_PORT=80                    # Server port (default: 80 in Docker)
```

## Testing Scenarios

### Health Endpoints (No Authentication)
```bash
curl http://localhost:3000/api/health  # General health check
curl http://localhost:3000/api/ready   # Readiness probe
curl http://localhost:3000/api/live    # Liveness probe
```

### Mock Data Testing
```bash
npm run dev
# Visit: http://localhost:3000/orgs/mocked-org?mock=true
```

### Route Examples
```bash
http://localhost:3000/orgs/octodemo
http://localhost:3000/enterprises/octo-ent
http://localhost:3000/orgs/octodemo/teams/the-a-team
```

## Known Issues

- **Linting**: 43 ESLint errors (mostly @typescript-eslint/no-explicit-any)
- **Type Checking**: 18 TypeScript errors in existing codebase
- **Font Provider Warnings**: Normal in restricted networks, non-blocking
- **Playwright**: Browser installation may fail in restricted environments

## Development Notes

### When Adding API Endpoints
- Add to health check exclusion list in `server/middleware/github.ts` if auth is not required
- Use authentication via `event.context.headers` (set by middleware)
- Consider cache invalidation strategy

### When Modifying Metrics Processing
- Check both conversion directions in `MetricsToUsageConverter.ts`
- Verify mock data generation in `metrics-util.ts::updateMockDataDates()`
- Update `ensureCopilotMetrics()` for new nested structures

### When Adding UI Components
- Use Vuetify components for consistency
- Charts use Chart.js via vue-chartjs
- Follow existing tooltip pattern for metric descriptions
- Date handling should use `app/utils/dateUtils.ts` utilities

### When Working with Tests
- Unit tests use Vitest with happy-dom environment
- Test setup in `tests/test.setup.ts`
- E2E tests use Playwright with mocked data
- All tests run with `NUXT_PUBLIC_IS_DATA_MOCKED=true`

## Deployment

See DEPLOYMENT.md for:
- Kubernetes deployment with health probes
- Azure deployment via azd
- Docker multi-stage builds
- GitHub App registration for OAuth
- Certificate and proxy configuration
