# Neo4j Knowledge Management Skill

This skill manages the ERP codebase knowledge base stored in Neo4j.

## Purpose

Query and store codebase documentation, patterns, modules, and technologies in Neo4j graph database for easy retrieval and context-aware code assistance.

## Prerequisites

- Neo4j container must be running: `docker ps | grep neo4j`
- Container name: `erp-neo4j`
- Neo4j Browser: http://localhost:7474 (neo4j/password)
- Scripts directory: `scripts/` with npm packages installed

## Capabilities

### 1. Query Knowledge from Neo4j

Query existing documentation using the provided scripts or direct Cypher queries.

**Using Query Script:**
```bash
cd scripts && npm run query-docs
```

**Direct Cypher Queries:**
```bash
# Get all codebases
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (c:Codebase) RETURN c.name, c.type, c.framework, c.language, c.description ORDER BY c.name"

# Get all modules
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (c:Codebase)-[:HAS_MODULE]->(m:Module) RETURN c.name as Codebase, m.name as Module, m.path as Path, m.description as Description ORDER BY c.name, m.name"

# Get all patterns
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (c:Codebase)-[:USES_PATTERN]->(p:Pattern) RETURN c.name as Codebase, p.name as Pattern, p.category as Category, p.description as Description, p.example as Example ORDER BY c.name, p.category, p.name"

# Get all technologies
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (c:Codebase)-[:USES_TECHNOLOGY]->(t:Technology) RETURN c.name as Codebase, t.name as Technology, t.version as Version, t.purpose as Purpose ORDER BY c.name, t.name"

# Search patterns by category
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (p:Pattern {category: 'Architecture'}) RETURN p.name, p.description, p.example"

# Get graph statistics
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (n) RETURN labels(n) as NodeType, count(*) as Count ORDER BY NodeType"
```

### 2. Store New Knowledge in Neo4j

When you discover new patterns, modules, or architectural decisions, you can add them to the knowledge base.

**Store All Documentation:**
```bash
cd scripts && npm run store-docs
```

**Add Individual Nodes (Cypher):**

```bash
# Add a new module
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "
MATCH (c:Codebase {name: 'ERP Backend'})
CREATE (m:Module {
  name: 'New Module Name',
  path: 'src/modules/new-module',
  description: 'Module description'
})
CREATE (c)-[:HAS_MODULE]->(m)
"

# Add a new pattern
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "
MATCH (c:Codebase {name: 'ERP Frontend'})
MERGE (p:Pattern {name: 'New Pattern'})
ON CREATE SET 
  p.category = 'Category Name',
  p.description = 'Pattern description',
  p.example = 'Example code or usage'
CREATE (c)-[:USES_PATTERN]->(p)
"

# Add a new technology
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "
MATCH (c:Codebase {name: 'ERP Backend'})
MERGE (t:Technology {name: 'New Library'})
ON CREATE SET 
  t.version = '1.0.0',
  t.purpose = 'Purpose description'
CREATE (c)-[:USES_TECHNOLOGY]->(t)
"

# Add a new component with relationships
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "
CREATE (comp:Component {
  name: 'ComponentName',
  path: 'libs/ui-components/src/components/ComponentName'
})
WITH comp
MATCH (page:Page {name: 'PageName'})
CREATE (page)-[:USES]->(comp)
"
```

## Graph Structure

### Node Types
- **Codebase**: Main codebase (Backend/Frontend)
  - Properties: name, type, framework, language, description
- **Module**: Feature modules or pages
  - Properties: name, path, description
- **Pattern**: Coding patterns and best practices
  - Properties: name, category, description, example
- **Technology**: Libraries and frameworks
  - Properties: name, version, purpose
- **Component**: UI components
  - Properties: name, path, description
- **Page**: Application pages
  - Properties: name, path, description
- **Dashboard**: Dashboard sections
  - Properties: name, path
- **Tab**: Tab components
  - Properties: name, path

### Relationship Types
- **HAS_MODULE**: Codebase → Module
- **USES_PATTERN**: Codebase → Pattern
- **USES_TECHNOLOGY**: Codebase → Technology
- **USES**: Component/Tab/Page → Component
- **CONTAINS**: Dashboard/Page → Page/Tab

## Workflow for AI Agents

### Before Implementing a Feature

1. **Query relevant patterns:**
```bash
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (p:Pattern {category: 'Security'}) RETURN p.name, p.description, p.example"
```

2. **Check existing modules:**
```bash
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (m:Module) WHERE m.name CONTAINS 'Auth' RETURN m.name, m.path, m.description"
```

3. **Review technologies:**
```bash
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (t:Technology) WHERE t.name = 'React Query' RETURN t.version, t.purpose"
```

### After Implementing a Feature

If you've introduced new patterns, components, or architectural decisions:

1. **Document new patterns** in the knowledge base
2. **Update module information** if structure changed
3. **Add new components** to the graph
4. **Create relationships** between new and existing nodes

## Common Query Patterns

### Find Related Patterns
```cypher
MATCH (c:Codebase {name: 'ERP Frontend'})-[:USES_PATTERN]->(p:Pattern)
WHERE p.category IN ['State Management', 'Forms']
RETURN p.name, p.example
```

### Find All Components Used by a Page
```cypher
MATCH (page:Page {name: 'ReportPage'})-[:USES*]->(comp:Component)
RETURN comp.name, comp.path
```

### Get Complete Module Structure
```cypher
MATCH (c:Codebase {name: 'ERP Backend'})-[:HAS_MODULE]->(m:Module)
OPTIONAL MATCH (c)-[:USES_PATTERN]->(p:Pattern)
OPTIONAL MATCH (c)-[:USES_TECHNOLOGY]->(t:Technology)
RETURN m.name, collect(DISTINCT p.name) as patterns, collect(DISTINCT t.name) as technologies
```

## Key Coding Conventions from Knowledge Base

### Backend (NestJS)
- **Path Aliases**: `@common/*`, `@database/*`, `@entities/*`, `@modules/*`
- **Structure**: `module.ts` → `controller.ts` → `service.ts` → `repository.ts`
- **Validation**: Joi schemas in `*.validator.ts`
- **i18n**: `this.i18n.t('common.user.notFound')`
- **Auth**: `@UseGuards(AuthenticatedGuard)`

### Frontend (React)
- **Path Aliases**: `ui-components`, `@ui/*`, `@/services/*`
- **Components**: Functional with hooks, early returns for loading/error
- **Services**: Class-based extending `ApiService`
- **State**: `useQueryWithAuth` for server state
- **Forms**: `useForm({ resolver: zodResolver(schema) })`
- **Styling**: `className="tw:p-6 tw:max-w-full"`
- **i18n**: `const { t } = useTranslation()`

## Reference Files

- **Context Guide**: `/Users/lamlv/Desktop/context/CONTEXT_GUIDE.md`
- **Store Script**: `/Users/lamlv/Desktop/context/scripts/store-docs-in-neo4j.ts`
- **Query Script**: `/Users/lamlv/Desktop/context/scripts/query-docs.ts`
- **Backend Docs**: `/Users/lamlv/Desktop/context/codebase-docs/backend-documentation.md`
- **Frontend Docs**: `/Users/lamlv/Desktop/context/codebase-docs/frontend-documentation.md`

## Tips

1. Always query patterns before implementing new features
2. Keep the knowledge base updated with new discoveries
3. Use descriptive examples in pattern nodes
4. Link related components with appropriate relationships
5. Query by category to find similar patterns across codebases
6. Use graph traversals to understand component dependencies

## Troubleshooting

### Neo4j not running
```bash
cd /Users/lamlv/Desktop/context && docker-compose up -d neo4j
```

### Clear and rebuild knowledge base
```bash
cd scripts && npm run store-docs
```

### Check connection
```bash
docker exec -i erp-neo4j cypher-shell -u neo4j -p password "MATCH (n) RETURN count(n) as total_nodes"
```
