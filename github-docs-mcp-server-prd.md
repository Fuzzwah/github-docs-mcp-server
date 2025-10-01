# Product Requirements Document: GitHub Docs MCP Server

**Version:** 1.0  
**Date:** October 2, 2025  
**Status:** Draft  
**Author:** @fuzzwah & GitHub Copilot (Claude Sonnet 4.5)

---

## 1. Executive Summary

### 1.1 Overview
This document outlines the requirements for a Model Context Protocol (MCP) server that enables AI coding agents to interact with a locally cached, scraped copy of the rendered GitHub documentation from docs.github.com. The server will provide efficient access to GitHub's official documentation in its fully rendered form, enabling agents to retrieve accurate, up-to-date information about GitHub features, APIs, and best practices without dealing with Liquid templates or build processes.

### 1.2 Objectives
- Provide coding agents with programmatic access to rendered GitHub documentation
- Store fully rendered HTML/Markdown content without template variables
- Enable fast, indexed search across documentation content
- Support incremental updates by detecting changes on docs.github.com
- Maintain compatibility with the Model Context Protocol standard
- Preserve all rendered content including dynamic variables and computed values
- Return live docs.github.com URLs with all responses for user reference and verification

---

## 2. Background & Context

### 2.1 Problem Statement
Coding agents frequently need to reference GitHub documentation to:
- Answer user questions about GitHub features
- Generate code that uses GitHub APIs correctly
- Provide accurate guidance on GitHub workflows and best practices
- Stay current with GitHub's evolving platform

Currently, agents must either:
1. Make web requests to docs.github.com (slow, rate-limited, requires internet)
2. Include limited documentation in their training data (outdated, incomplete)
3. Rely on general knowledge (potentially inaccurate)

**Challenge with Raw Repository:**
The github/docs repository contains unrendered Liquid templates with variables like `{% data variables.product.company_short %}`, reusable content blocks, and other dynamic content that requires the full build toolchain to render properly. This makes direct repository cloning impractical for consumption by AI agents.

### 2.2 Solution Approach
An MCP server that maintains a locally scraped, rendered copy of docs.github.com and provides:
- Fully rendered HTML content converted to clean Markdown
- Fast local search and retrieval
- Structured access to documentation content
- Intelligent updates by detecting changes on the live site
- Efficient storage through content deduplication and compression

---

## 3. User Personas

### 3.1 Primary Users
**AI Coding Agents**
- Need fast, reliable access to GitHub documentation
- Operate in various environments (cloud, local, enterprise)
- Require structured, parseable documentation format
- May have limited disk space or bandwidth

### 3.2 Secondary Users
**Developers/DevOps Engineers**
- Install and configure the MCP server
- Monitor server health and updates
- Manage storage and update schedules

---

## 4. Functional Requirements

### 4.1 Core Features

#### 4.1.1 Content Scraping & Management
**FR-1.1: Initial Scraping**
- MUST scrape rendered HTML from docs.github.com/en
- MUST follow the documentation sitemap or navigation structure
- MUST respect robots.txt and implement polite scraping (rate limiting)
- MUST store the source URL for each scraped page
- MUST convert HTML to clean Markdown while preserving formatting
- MUST extract and preserve code blocks with language tags
- MUST handle pagination and dynamic content loading
- SHOULD complete initial scrape in < 30 minutes with rate limiting
- SHOULD support resume capability for interrupted scrapes

**FR-1.2: Content Updates**
- MUST detect changes by comparing page checksums/ETags/Last-Modified headers
- MUST support incremental updates (only re-scrape changed pages)
- SHOULD use sitemap.xml to discover new/changed content
- SHOULD support configurable update intervals (hourly, daily, weekly, manual)
- MUST handle update failures gracefully with retry logic
- MUST maintain version history for rollback capability
- SHOULD notify when updates are applied successfully

**FR-1.3: Storage Management**
- MUST store rendered content in structured format (Markdown + metadata)
- MUST support content deduplication for repeated sections
- MUST support automatic cleanup of stale content
- SHOULD provide disk usage reporting
- SHOULD warn when disk space is low
- SHOULD support optional compression for storage efficiency
- Target: < 1GB total storage for content and index

#### 4.1.2 Documentation Search & Retrieval

**FR-2.1: Content Indexing**
- MUST index all Markdown documentation files
- MUST support full-text search across documentation
- MUST index metadata (titles, categories, frontmatter)
- SHOULD update index automatically after repository updates
- SHOULD support incremental index updates (not full rebuild)

**FR-2.2: Search Capabilities**
- MUST support keyword/phrase search
- MUST support semantic/natural language queries
- MUST rank results by relevance
- SHOULD support filtering by:
  - Documentation category (Actions, API, CLI, etc.)
  - Product/service
  - Content type (guide, reference, tutorial)
- SHOULD return results in < 200ms for typical queries

**FR-2.3: Content Retrieval**
- MUST return documentation in Markdown format
- MUST preserve code blocks, formatting, and structure
- MUST include frontmatter metadata
- MUST include the live docs.github.com URL for every document
- MUST map local paths to canonical URLs (e.g., local:actions/intro.md → https://docs.github.com/en/actions/intro)
- SHOULD support retrieving specific sections/headers
- SHOULD support retrieving related/linked documents

**FR-2.4: Content Processing**
- MUST convert HTML to clean Markdown with GitHub Flavored Markdown syntax
- MUST preserve all rendered content (no template variables)
- MUST store the source URL with each scraped document
- MUST maintain bidirectional mapping between local paths and live URLs
- MUST handle HTML elements correctly:
  - Code blocks with proper language tags and syntax highlighting hints
  - Tables and formatted data
  - Admonitions/callouts (notes, warnings, tips)
  - Nested lists and complex formatting
- SHOULD extract and expose:
  - Document title and description (from HTML meta tags)
  - Breadcrumb navigation
  - Table of contents/headings (from rendered HTML)
  - Code examples with language tags
  - Internal and external links
  - Version/deprecation notices
  - Related articles links

#### 4.1.3 MCP Protocol Implementation

**FR-3.1: MCP Server Interface**
- MUST implement MCP protocol specification v1.0+
- MUST support standard MCP transport mechanisms (stdio, SSE)
- MUST provide proper capability advertisement
- MUST handle connection lifecycle correctly

**FR-3.2: Tools/Resources**
- MUST expose the following MCP tools:
  1. `search_docs` - Search documentation by query
  2. `get_document` - Retrieve specific document by URL/path
  3. `list_categories` - List available documentation categories
  4. `get_related_docs` - Find related documentation
  5. `trigger_update` - Trigger incremental content update/scrape
  6. `get_server_status` - Get server health and statistics

**FR-3.3: Resources**
- MUST expose documentation as MCP resources
- SHOULD support resource URIs like: `github-docs://path/to/doc.md`
- SHOULD support resource templates for dynamic access

### 4.2 Configuration & Management

**FR-4.1: Configuration**
- MUST support configuration file (JSON/YAML)
- MUST allow configuration of:
  - Repository URL and branch
  - Clone depth
  - Update schedule
  - Storage location
  - Index settings
  - Cache settings
- SHOULD support environment variable overrides

**FR-4.2: Logging & Monitoring**
- MUST provide structured logging (JSON format preferred)
- MUST log all operations with appropriate levels
- SHOULD expose metrics:
  - Query count and latency
  - Update success/failure rate
  - Cache hit rate
  - Disk usage
  - Memory usage

**FR-4.3: Health & Status**
- MUST provide health check endpoint/tool
- MUST report repository status (last update, size, health)
- MUST report index status (document count, last rebuild)
- SHOULD provide diagnostics for troubleshooting

---

## 5. Non-Functional Requirements

### 5.1 Performance
- **NFR-1.1:** Search queries MUST return results in < 200ms (p95)
- **NFR-1.2:** Document retrieval MUST complete in < 50ms (p95)
- **NFR-1.3:** Initial repository clone SHOULD complete in < 5 minutes
- **NFR-1.4:** Repository updates SHOULD complete in < 2 minutes
- **NFR-1.5:** Server startup MUST complete in < 10 seconds

### 5.2 Scalability
- **NFR-2.1:** MUST handle 100+ concurrent requests
- **NFR-2.2:** MUST support repositories with 10,000+ documentation files
- **NFR-2.3:** SHOULD maintain performance with index size up to 1GB

### 5.3 Reliability
- **NFR-3.1:** Server uptime target: 99.9%
- **NFR-3.2:** MUST recover automatically from transient failures
- **NFR-3.3:** MUST not corrupt repository or index on crash
- **NFR-3.4:** MUST handle network failures during updates gracefully

### 5.4 Security
- **NFR-4.1:** MUST validate all file paths to prevent directory traversal
- **NFR-4.2:** MUST sanitize git operations to prevent command injection
- **NFR-4.3:** SHOULD support authentication for MCP connections (if protocol supports)
- **NFR-4.4:** MUST not expose sensitive system information in logs/responses

### 5.5 Compatibility
- **NFR-5.1:** MUST support Linux, macOS, and Windows
- **NFR-5.2:** MUST support Node.js 18+ or Python 3.9+
- **NFR-5.3:** MUST work with git 2.25+
- **NFR-5.4:** MUST be compatible with MCP clients (Copilot in VS Code, Claude Desktop, Cline, etc.)

### 5.6 Usability
- **NFR-6.1:** Installation SHOULD be single-command
- **NFR-6.2:** MUST provide clear error messages
- **NFR-6.3:** SHOULD include comprehensive README
- **NFR-6.4:** SHOULD include example configurations

### 5.7 Maintainability
- **NFR-7.1:** Code MUST be well-documented
- **NFR-7.2:** MUST include unit tests (>80% coverage)
- **NFR-7.3:** SHOULD include integration tests
- **NFR-7.4:** MUST use standard dependencies (minimize exotic libraries)

---

## 6. Technical Architecture

### 6.1 Technology Stack (Recommendations)

**Option A: Python Implementation**
- Language: Python 3.9+
- MCP SDK: `mcp` Python package
- Web Scraping: `httpx` (async HTTP client), `beautifulsoup4` or `lxml` (HTML parsing)
- HTML to Markdown: `html2text` or `markdownify`
- Search/Index: `whoosh` or `tantivy-py` (for lightweight indexing)
- Markdown: `markdown-it-py` or `mistune`
- Sitemap parsing: `usp` or `advertools`

**Option B: TypeScript/Node.js Implementation**
- Language: TypeScript 5+
- MCP SDK: `@modelcontextprotocol/sdk`
- Web Scraping: `axios` or `node-fetch`, `cheerio` or `jsdom` (HTML parsing)
- HTML to Markdown: `turndown`
- Search/Index: `lunr` or `flexsearch`
- Markdown: `markdown-it`
- Sitemap parsing: `sitemap` or custom XML parser

### 6.2 Component Architecture

```
┌─────────────────────────────────────────────┐
│         MCP Server Interface                │
│  (Protocol Handler, Transport Layer)        │
└─────────────────┬───────────────────────────┘
                  │
    ┌─────────────┴─────────────┐
    │                           │
┌───▼────────────┐    ┌────────▼─────────┐
│  Tool Handler  │    │ Resource Handler │
│                │    │                  │
│ - search_docs  │    │ - URI resolver   │
│ - get_document │    │ - Content loader │
│ - list_*       │    │                  │
└───┬────────────┘    └────────┬─────────┘
    │                           │
    └─────────────┬─────────────┘
                  │
    ┌─────────────▼──────────────────┐
    │    Documentation Service       │
    │                                │
    │  - Query Processing            │
    │  - Content Retrieval           │
    │  - Metadata Extraction         │
    └─────────────┬──────────────────┘
                  │
    ┌─────────────┼──────────────┐
    │             │              │
┌───▼──────┐  ┌──▼────────┐  ┌─▼──────────┐
│  Index   │  │  Scraper  │  │   Cache    │
│  Engine  │  │  Manager  │  │  Manager   │
│          │  │           │  │            │
│ - Search │  │ - Crawl   │  │ - LRU      │
│ - Rank   │  │ - Parse   │  │ - Rendered │
│ - Filter │  │ - Convert │  │   Content  │
└──────────┘  │ - Detect  │  └────────────┘
              │   Changes │
              └───────────┘
```

### 6.3 Data Flow

1. **Initialization:**
   - Load configuration
   - Initialize scraper manager
   - Check if local cache exists
   - If empty, trigger initial scrape
   - Build/load search index
   - Start MCP server

2. **Initial Scraping (first run):**
   - Fetch sitemap.xml from docs.github.com
   - Parse URLs and prioritize by section
   - Crawl pages with rate limiting
   - Extract HTML content
   - Convert HTML to Markdown
   - Extract metadata and structure
   - Store source URL with each document
   - Create URL mapping (local path ↔ live URL)
   - Store in local database/filesystem
   - Build search index

3. **Search Request:**
   - MCP client → search_docs tool
   - Parse query and filters
   - Query search index
   - Rank and filter results
   - Return metadata and snippets

4. **Document Retrieval:**
   - MCP client → get_document tool
   - Check cache
   - Load from local storage
   - Return formatted Markdown content
   - Return metadata and structure

5. **Update Cycle:**
   - Timer triggers or manual request
   - Fetch sitemap.xml for change detection
   - Check Last-Modified headers or checksums
   - Identify new/changed pages
   - Re-scrape only changed content
   - Update search index incrementally
   - Clear relevant caches
   - Log update status

---

## 7. Data Models

### 7.1 Document Model
```typescript
interface Document {
  path: string;              // Relative path in local storage
  url: string;               // Live docs.github.com URL (REQUIRED)
  title: string;             // Document title
  description?: string;      // Short description
  category: string;          // Category (e.g., "actions", "api")
  subcategory?: string;      // Subcategory if applicable
  version?: string;          // Version info
  lastModified: Date;        // Last scraped/modified date
  content: string;           // Full Markdown content
  frontmatter: Record<string, any>; // YAML frontmatter
  headings: Heading[];       // Table of contents
  codeBlocks: CodeBlock[];   // Extracted code examples
  links: Link[];             // Internal and external links
}

interface Heading {
  level: number;             // 1-6
  text: string;
  id: string;                // Anchor ID
}

interface CodeBlock {
  language: string;
  code: string;
  lineStart: number;
}

interface Link {
  text: string;
  url: string;
  isInternal: boolean;
}
```

### 7.2 Search Result Model
```typescript
interface SearchResult {
  path: string;
  url: string;               // Live docs.github.com URL (REQUIRED)
  title: string;
  description?: string;
  category: string;
  score: number;             // Relevance score 0-1
  snippet: string;           // Context snippet with highlights
  matchedTerms: string[];
}
```

### 7.3 Server Status Model
```typescript
interface ServerStatus {
  status: "healthy" | "degraded" | "unhealthy";
  scraper: {
    localPath: string;
    lastUpdate: Date;
    lastScrapeDuration: string;
    totalPages: number;
    sizeBytes: number;
  };
  index: {
    documentCount: number;
    lastRebuild: Date;
    sizeBytes: number;
  };
  metrics: {
    queryCount: number;
    averageQueryTime: number;
    cacheHitRate: number;
    uptime: number;
  };
}
```

---

## 8. MCP Tools Specification

### 8.1 search_docs
**Description:** Search GitHub documentation by keyword or natural language query

**Input Schema:**
```json
{
  "query": {
    "type": "string",
    "description": "Search query (keywords or natural language)",
    "required": true
  },
  "category": {
    "type": "string",
    "description": "Filter by category (actions, api, cli, etc.)",
    "required": false
  },
  "limit": {
    "type": "number",
    "description": "Maximum results to return (1-50)",
    "default": 10,
    "required": false
  },
  "includeContent": {
    "type": "boolean",
    "description": "Include full content in results",
    "default": false,
    "required": false
  }
}
```

**Output:** Array of SearchResult objects

### 8.2 get_document
**Description:** Retrieve a specific documentation file by path or URL

**Input Schema:**
```json
{
  "path": {
    "type": "string",
    "description": "Relative path or full docs.github.com URL (e.g., 'actions/learn-github-actions/introduction' or 'https://docs.github.com/en/actions/learn-github-actions/introduction')",
    "required": true
  },
  "format": {
    "type": "string",
    "enum": ["markdown", "parsed", "html"],
    "description": "Output format",
    "default": "markdown",
    "required": false
  }
}
```

**Output:** Document object

### 8.3 list_categories
**Description:** List all available documentation categories

**Input Schema:**
```json
{
  "includeCount": {
    "type": "boolean",
    "description": "Include document count per category",
    "default": false,
    "required": false
  }
}
```

**Output:**
```json
{
  "categories": [
    {
      "name": "actions",
      "displayName": "GitHub Actions",
      "documentCount": 150
    }
  ]
}
```

### 8.4 get_related_docs
**Description:** Find documentation related to a given document

**Input Schema:**
```json
{
  "path": {
    "type": "string",
    "description": "Path to source document",
    "required": true
  },
  "limit": {
    "type": "number",
    "description": "Maximum related docs to return",
    "default": 5,
    "required": false
  }
}
```

**Output:** Array of SearchResult objects

### 8.5 trigger_update
**Description:** Trigger a manual content update/scrape

**Input Schema:**
```json
{
  "force": {
    "type": "boolean",
    "description": "Force update even if recently updated",
    "default": false,
    "required": false
  }
}
```

**Output:**
```json
{
  "success": true,
  "message": "Content updated successfully",
  "pagesChecked": 1250,
  "pagesUpdated": 12,
  "duration": "8m 23s"
}
```

### 8.6 get_server_status
**Description:** Get server health, statistics, and status

**Input Schema:** None

**Output:** ServerStatus object

---

## 9. Configuration Example

```yaml
# github-docs-mcp-config.yaml
server:
  name: "github-docs-mcp"
  version: "1.0.0"
  transport: "stdio"  # or "sse"
  logLevel: "info"    # debug, info, warn, error

scraper:
  baseUrl: "https://docs.github.com/en"
  sitemapUrl: "https://docs.github.com/sitemap.xml"
  userAgent: "GitHubDocsMCP/1.0 (Documentation Mirror; +https://github.com/your-org/github-docs-mcp)"
  rateLimit:
    requestsPerSecond: 5
    concurrent: 3
    respectRetryAfter: true
  timeout: 30  # seconds
  maxRetries: 3
  localPath: "./data/github-docs"
  
update:
  enabled: true
  schedule: "0 */6 * * *"  # Every 6 hours (cron syntax)
  autoUpdateOnStartup: false  # Don't scrape on every startup
  retryAttempts: 3
  retryDelay: 300  # seconds
  incrementalOnly: true  # Only scrape changed pages

index:
  engine: "lunr"  # or "whoosh", "flexsearch"
  path: "./data/index"
  rebuildOnStartup: false
  incrementalUpdates: true
  fields:
    - title: { weight: 3 }
    - description: { weight: 2 }
    - content: { weight: 1 }
    - headings: { weight: 2 }

cache:
  enabled: true
  maxSizeMB: 100
  ttl: 3600  # seconds
  strategy: "lru"

performance:
  maxConcurrentQueries: 100
  queryTimeout: 5000  # ms
  workerThreads: 4

security:
  validatePaths: true
  allowedCategories: []  # empty = all allowed
  authentication:
    enabled: false
    method: "token"  # if enabled
```

---

## 10. Implementation Phases

### Phase 1: MVP
- Basic web scraping infrastructure
- HTML to Markdown conversion
- Simple keyword search
- Core MCP tools: search_docs, get_document, get_server_status
- Basic configuration
- README and setup docs

**Success Criteria:**
- Can scrape and store docs.github.com content
- Content is fully rendered (no template variables)
- Can search and retrieve documents
- Works with at least one MCP client
- < 1GB disk usage
- Respects rate limits and robots.txt

### Phase 2: Enhanced Search
- Advanced indexing with ranking
- Category filtering
- Related documents
- Search result snippets and highlights
- Performance optimization
- Comprehensive logging

**Success Criteria:**
- Sub-200ms search response time
- Accurate relevance ranking
- Category filtering works correctly

### Phase 3: Auto-Updates & Polish
- Scheduled scraping updates
- Change detection via sitemap/headers
- Incremental index updates
- Cache implementation
- Error handling improvements
- Monitoring and metrics
- Unit and integration tests

**Success Criteria:**
- Incremental updates work reliably (only fetch changed pages)
- All tests passing (>80% coverage)
- Handles scraping failures gracefully
- Respects rate limits and doesn't overload docs.github.com

---

## 11. Success Metrics

### 11.1 Technical Metrics
- **Search Performance:** p95 latency < 200ms
- **Availability:** > 99% uptime
- **Storage Efficiency:** < 500MB total disk usage
- **Update Reliability:** > 95% successful updates
- **Test Coverage:** > 80%

### 11.2 User Metrics
- **Query Success Rate:** > 90% of queries return relevant results
- **Installation Success:** Users can install in < 5 minutes
- **Error Rate:** < 1% of requests result in errors

### 11.3 Business Metrics
- Adoption by AI coding agents
- Reduction in direct web requests to docs.github.com
- Developer satisfaction scores

---

## 12. Risks & Mitigations

### 12.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Website structure changes | High | Medium | Flexible HTML parsing, multiple selector fallbacks, regular testing |
| Rate limiting/blocking by GitHub | High | Medium | Polite scraping, user agent identification, respect rate limits, backoff strategy |
| HTML parsing failures | Medium | Medium | Robust error handling, fallback parsers, validation |
| Search index corruption | Medium | Low | Regular validation, atomic updates, backup strategy |
| Performance degradation with large sites | Medium | Medium | Pagination, query optimization, caching, incremental updates |
| MCP protocol changes | Low | Low | Pin to stable version, regular SDK updates |

### 12.2 Operational Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Disk space exhaustion | High | Low | Monitoring, automatic cleanup, configurable limits |
| Network connectivity issues | Medium | Medium | Graceful degradation, offline mode, cached data |
| Breaking changes in docs repo | Medium | Low | Version pinning option, regression testing |

---

## 13. Dependencies

### 13.1 External Dependencies
- docs.github.com availability and accessibility
- MCP SDK (Python or Node.js)
- HTTP client library (httpx, axios, etc.)
- HTML parser (BeautifulSoup, cheerio, etc.)
- HTML to Markdown converter
- Search indexing library
- Markdown parser

### 13.2 System Requirements
- **Disk Space:** 2-3 GB free (scraped content + index + cache)
- **Memory:** 512 MB minimum, 1 GB recommended (for concurrent scraping)
- **CPU:** 1 core minimum, 2+ cores recommended
- **Network:** Broadband for initial scraping (will take longer than git clone due to rate limiting)

---

## 16. Appendices

### Appendix A: GitHub Docs Site Structure
```
https://docs.github.com/en/
├── actions/                    # GitHub Actions documentation
├── authentication/             # Authentication & security
├── code-security/              # Code security features
├── copilot/                    # GitHub Copilot
├── rest/                       # REST API reference
├── graphql/                    # GraphQL API reference
├── get-started/                # Getting started guides
├── repositories/               # Repository management
├── pull-requests/              # Pull requests
├── issues/                     # Issues
├── projects/                   # Projects
└── ...

Each page contains:
- Rendered HTML with full content (no Liquid variables)
- Breadcrumb navigation
- Table of contents
- Code examples with syntax highlighting
- Related articles
- Metadata (last updated, versions, etc.)
```

### Appendix B: Example MCP Client Usage
```typescript
// Example: Using the MCP server from a coding agent
const results = await mcp.callTool('search_docs', {
  query: 'how to create a GitHub Action workflow',
  category: 'actions',
  limit: 5
});

// Each result includes the live URL
console.log(results[0].url);
// => https://docs.github.com/en/actions/learn-github-actions/introduction-to-github-actions

const doc = await mcp.callTool('get_document', {
  path: results[0].path,
  format: 'parsed'
});

// Document includes the URL for user reference
console.log(doc.url);
// The agent can include this URL in responses:
// "According to the GitHub Actions documentation 
//  (https://docs.github.com/en/actions/...), you should..."
```

### Appendix C: Performance Benchmarks (Target)
- Initial scrape: ~30-60 minutes (with rate limiting, ~3000-5000 pages)
- Index build: ~30 seconds (10,000 documents)
- Simple search: ~50ms
- Complex search with filters: ~150ms
- Document retrieval: ~10ms (cached), ~30ms (uncached)
- Incremental update: ~5-10 minutes (for ~50 changed pages)
- HTML to Markdown conversion: ~50ms per page

### Appendix D: Related Projects & References
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [GitHub Documentation](https://docs.github.com)
- [GitHub Docs Repository](https://github.com/github/docs) - Source code (for reference)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Robots.txt for docs.github.com](https://docs.github.com/robots.txt)
- [Sitemap for docs.github.com](https://docs.github.com/sitemap.xml)

---

## 17. Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-02 | @fuzzwah | Initial draft |

---

**End of Document**
