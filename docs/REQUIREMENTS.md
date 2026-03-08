# Twenty CRM — Complete Requirements Specification

This document is a reverse-engineered requirements specification for the Twenty codebase. It is intended to allow reimplementation of the system (including with AI) from this spec alone.

---

## 1. Product Overview

### 1.1 Purpose

Twenty is an **open-source CRM** (Customer Relationship Management) application with:

- **Multi-tenant workspaces**: Each workspace has its own subdomain or custom domain, metadata, and data.
- **Metadata-driven data model**: Objects (entities) and fields are defined via metadata; the API and UI are generated from that metadata.
- **Dynamic workspace schema**: Per-workspace GraphQL schema is built at runtime from object/field metadata.
- **Standard + custom objects**: Standard objects (Company, Person, Opportunity, Task, etc.) plus user-defined custom objects.
- **Views**: Table, Kanban, Calendar, and Fields Widget views with filters, sorts, and groups.
- **Record CRUD**: Create, read, update, delete, duplicate detection, merge, restore (soft delete), batch operations.
- **Auth**: Email/password, Google, Microsoft; optional 2FA; invite links; workspace membership and roles.
- **Integrations**: Calendars, messaging (Gmail/Microsoft), workflows, webhooks, AI agents, applications marketplace.

### 1.2 High-Level Architecture

- **Monorepo**: Nx workspace, Yarn 4, multiple packages under `packages/`.
- **Backend**: Single NestJS application (`twenty-server`) exposing GraphQL (two schemas) and REST.
- **Frontend**: Single React SPA (`twenty-front`) with React Router, Jotai, Apollo Client.
- **Shared**: Common types, constants, and utilities in `twenty-shared`; shared UI in `twenty-ui`.
- **Database**: PostgreSQL (primary, TypeORM), Redis (sessions/cache), optional ClickHouse (analytics).

---

## 2. Key Data Models (Entity Diagram)

This section describes the main entities, their attributes, and relationships so you can reason about or draw the entity diagram. All IDs are UUID unless noted. Schema is `core` for core and metadata tables unless stated.

### 2.0 Base Types (Abstract)

- **WorkspaceRelatedEntity** (abstract)  
  - `workspaceId` (UUID, FK → Workspace)  
  - `workspace` (relation)  
  - Every workspace-scoped entity extends this (or a subclass).

- **SyncableEntity** (abstract, extends WorkspaceRelatedEntity)  
  - All from WorkspaceRelatedEntity, plus:  
  - `universalIdentifier` (UUID)  
  - `applicationId` (UUID, FK → Application)  
  - `application` (relation)  
  - Unique on `(workspaceId, universalIdentifier)`.  
  - Used for metadata entities that participate in workspace migrations.

---

### 2.1 Core: Workspace, User, UserWorkspace

**Workspace** (`core.workspace`)

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| displayName | string | nullable |
| logo | string | deprecated |
| logoFileId | UUID | nullable, FK → File |
| inviteHash | string | nullable |
| deletedAt | timestamptz | nullable, soft delete |
| createdAt, updatedAt | timestamptz | |
| allowImpersonation | boolean | default true |
| isPublicInviteLinkEnabled | boolean | default true |
| trashRetentionDays | integer | default 14 |
| eventLogRetentionDays | integer | default 90 |
| activationStatus | enum | e.g. INACTIVE, PENDING_CREATION; check: onboarded ⇒ defaultRoleId set |
| suspendedAt | timestamptz | nullable |
| metadataVersion | integer | default 1 |
| databaseUrl, databaseSchema | string | |
| subdomain | string | unique |
| customDomain | string | nullable, unique |
| isGoogleAuthEnabled, isMicrosoftAuthEnabled, isPasswordAuthEnabled | boolean | + bypass flags |
| isTwoFactorAuthenticationEnforced | boolean | |
| isCustomDomainEnabled | boolean | |
| editableProfileFields | varchar[] | nullable |
| defaultRoleId | UUID | nullable, FK → Role (logical) |
| version | string | nullable |
| fastModel, smartModel, routerModel | string | AI model ids |
| aiAdditionalInstructions | text | nullable |
| autoEnableNewAiModels, useRecommendedModels | boolean | |
| disabledAiModelIds, enabledAiModelIds | varchar[] | |
| workspaceCustomApplicationId | UUID | FK → Application |

Relations: workspaceUsers (UserWorkspace), featureFlags, appTokens, keyValuePairs, applications, agents, webhooks, apiKeys, views, viewFields, viewFilters, viewFilterGroups, viewGroups, viewSorts, approvedAccessDomains, emailingDomains, publicDomains, workspaceSSOIdentityProviders, etc.

---

**User** (`core.user`)

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| firstName, lastName | string | default '' |
| email | string | unique (where deletedAt IS NULL) |
| defaultAvatarUrl | string | nullable |
| isEmailVerified | boolean | default false |
| disabled | boolean | default false |
| passwordHash | string | nullable |
| canImpersonate | boolean | default false |
| canAccessFullAdminPanel | boolean | default false |
| createdAt, updatedAt | timestamptz | |
| deletedAt | timestamptz | nullable |
| locale | varchar | e.g. SOURCE_LOCALE |

Relations: userWorkspaces (UserWorkspace), appTokens, keyValuePairs.

---

**UserWorkspace** (`core.userWorkspace`) — workspace membership  
Extends: WorkspaceRelatedEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| userId | UUID | FK → User; unique with workspaceId (where deletedAt IS NULL) |
| workspaceId | UUID | FK → Workspace (from base) |
| defaultAvatarUrl | string | nullable |
| locale | varchar | |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: user, workspace; twoFactorAuthenticationMethods.

---

### 2.2 Metadata Backbone: DataSource, ObjectMetadata, FieldMetadata

**DataSource** (`dataSource`)  
Extends: WorkspaceRelatedEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| workspaceId | UUID | FK → Workspace (from base) |
| label | string | nullable |
| url | string | nullable |
| schema | string | nullable |
| type | enum | e.g. 'postgres' |
| isRemote | boolean | default false |
| createdAt, updatedAt | timestamptz | |

Relations: objects (ObjectMetadata[]).

---

**ObjectMetadata** (`objectMetadata`)  
Extends: SyncableEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| dataSourceId | UUID | FK → DataSource |
| nameSingular, namePlural | string | unique per (workspaceId) |
| labelSingular, labelPlural | string | |
| description | text | nullable |
| icon | varchar | nullable |
| standardOverrides | jsonb | nullable |
| targetTableName | varchar | deprecated |
| isCustom, isRemote, isActive, isSystem | boolean | |
| isUIReadOnly, isAuditLogged, isSearchable | boolean | |
| duplicateCriteria | jsonb | nullable |
| shortcut | varchar | nullable |
| labelIdentifierFieldMetadataId | UUID | nullable |
| imageIdentifierFieldMetadataId | UUID | nullable |
| isLabelSyncedWithName | boolean | |
| universalIdentifier, applicationId | UUID | (from SyncableEntity) |
| workspaceId | UUID | (from base) |
| createdAt, updatedAt | timestamptz | |

Relations: dataSource; fields (FieldMetadata[]); indexMetadatas; objectPermissions; fieldPermissions; views.  
Uniques: (nameSingular, workspaceId), (namePlural, workspaceId).

---

**FieldMetadata** (`fieldMetadata`)  
Extends: SyncableEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| objectMetadataId | UUID | FK → ObjectMetadata |
| type | varchar | FieldMetadataType enum |
| name | string | unique (name, objectMetadataId, workspaceId) |
| label | string | |
| defaultValue | jsonb | nullable |
| description | text | nullable |
| icon | varchar | nullable |
| standardOverrides, options, settings | jsonb | nullable |
| isCustom, isActive, isSystem, isUIReadOnly | boolean | |
| isNullable, isUnique | boolean | nullable |
| isLabelSyncedWithName | boolean | |
| relationTargetFieldMetadataId | UUID | nullable (for relation/morph) |
| relationTargetObjectMetadataId | UUID | nullable (for relation/morph) |
| morphId | UUID | nullable; required if type = MORPH_RELATION (check) |
| universalIdentifier, applicationId | UUID | (from SyncableEntity) |
| workspaceId | UUID | (from base) |
| createdAt, updatedAt | timestamptz | |

Relations: object; relationTargetFieldMetadata; relationTargetObjectMetadata; indexFieldMetadatas; fieldPermissions; viewFields; viewFilters; viewSorts; kanbanAggregateOperationViews; calendarViews; mainGroupByFieldMetadataViews.

---

### 2.3 Views: View, ViewField, ViewSort, ViewFilter, ViewFilterGroup, ViewGroup, ViewFieldGroup

**View** (`core.view`)  
Extends: SyncableEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| name | text | default '' |
| objectMetadataId | UUID | FK → ObjectMetadata |
| type | enum | ViewType: TABLE, KANBAN, CALENDAR, FIELDS_WIDGET |
| key | enum | ViewKey e.g. INDEX; nullable |
| icon | text | |
| position | double | default 0 |
| isCompact, isCustom | boolean | |
| openRecordIn | enum | ViewOpenRecordIn e.g. SIDE_PANEL |
| kanbanAggregateOperation | enum | AggregateOperations; nullable |
| kanbanAggregateOperationFieldMetadataId | UUID | nullable, FK → FieldMetadata |
| calendarLayout | enum | nullable |
| calendarFieldMetadataId | UUID | nullable, FK → FieldMetadata; required if type=CALENDAR (check) |
| mainGroupByFieldMetadataId | UUID | nullable, FK → FieldMetadata |
| shouldHideEmptyGroups | boolean | |
| anyFieldFilterValue | text | nullable |
| visibility | enum | ViewVisibility e.g. WORKSPACE |
| createdByUserWorkspaceId | UUID | nullable, FK → UserWorkspace |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: objectMetadata; viewFields; viewFieldGroups; viewFilters; viewSorts; viewGroups; viewFilterGroups; kanbanAggregateOperationFieldMetadata; calendarFieldMetadata; mainGroupByFieldMetadata; createdBy.

---

**ViewField** (`core.viewField`)  
Extends: SyncableEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| fieldMetadataId | UUID | FK → FieldMetadata; unique with viewId (where deletedAt IS NULL) |
| viewId | UUID | FK → View |
| viewFieldGroupId | UUID | nullable, FK → ViewFieldGroup |
| isVisible | boolean | default true |
| size | int | default 0 |
| position | double | default 0 |
| aggregateOperation | enum | AggregateOperations; nullable |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: fieldMetadata; view; viewFieldGroup.

---

**ViewSort** (`core.viewSort`)  
Extends: SyncableEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| fieldMetadataId | UUID | FK → FieldMetadata; unique with viewId (where deletedAt IS NULL) |
| viewId | UUID | FK → View |
| direction | enum | ViewSortDirection: ASC, DESC |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: fieldMetadata; view.

---

**ViewFilter** (`core.viewFilter`)  
Extends: SyncableEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| fieldMetadataId | UUID | FK → FieldMetadata |
| viewId | UUID | FK → View |
| viewFilterGroupId | UUID | nullable, FK → ViewFilterGroup |
| operand | enum | ViewFilterOperand e.g. CONTAINS |
| value | jsonb | ViewFilterValue |
| positionInViewFilterGroup | double | nullable |
| subFieldName | text | nullable |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: fieldMetadata; view; viewFilterGroup.

---

**ViewFilterGroup** (`core.viewFilterGroup`)  
Extends: SyncableEntity. Supports nested filter groups (AND/OR).

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| viewId | UUID | FK → View |
| parentViewFilterGroupId | UUID | nullable, FK → ViewFilterGroup (self) |
| logicalOperator | enum | ViewFilterGroupLogicalOperator: AND, OR |
| positionInViewFilterGroup | double | nullable |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: view; viewFilters (ViewFilter[]); parentViewFilterGroup; childViewFilterGroups.

---

**ViewGroup** (`core.viewGroup`)  
Extends: SyncableEntity. One row per distinct group value (e.g. Kanban column).

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| viewId | UUID | FK → View |
| isVisible | boolean | default true |
| fieldValue | text | group value |
| position | double | default 0 |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: view.

---

**ViewFieldGroup** (`core.viewFieldGroup`)  
Extends: SyncableEntity. Groups of columns in a view (e.g. “Contact” section).

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| viewId | UUID | FK → View |
| name | text | |
| position | double | default 0 |
| isVisible | boolean | default true |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: view; viewFields (ViewField[]).

---

### 2.4 Permissions: Role, RoleTarget, ObjectPermission, FieldPermission

**Role** (`role`)  
Extends: SyncableEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| label | string | unique (label, workspaceId) |
| canUpdateAllSettings | boolean | default false |
| canAccessAllTools | boolean | default false |
| canReadAllObjectRecords | boolean | default false |
| canUpdateAllObjectRecords | boolean | default false |
| canSoftDeleteAllObjectRecords | boolean | default false |
| canDestroyAllObjectRecords | boolean | default false |
| description | text | nullable |
| icon | varchar | nullable |
| isEditable | boolean | default true |
| canBeAssignedToUsers | boolean | default true |
| canBeAssignedToAgents | boolean | default true |
| canBeAssignedToApiKeys | boolean | default true |
| createdAt, updatedAt | timestamptz | |

Relations: roleTargets; objectPermissions; fieldPermissions; permissionFlags; rowLevelPermissionPredicates; rowLevelPermissionPredicateGroups.

---

**RoleTarget** (`roleTarget`)  
Extends: SyncableEntity. Assigns a role to a user-workspace, agent, or API key (exactly one).

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| roleId | UUID | FK → Role |
| userWorkspaceId | UUID | nullable, FK → UserWorkspace |
| agentId | UUID | nullable |
| apiKeyId | UUID | nullable, FK → ApiKey |
| createdAt, updatedAt | timestamptz | |

Check: exactly one of (userWorkspaceId, agentId, apiKeyId) is non-null.  
Uniques: (workspaceId, userWorkspaceId), (workspaceId, agentId), (workspaceId, apiKeyId).

Relations: role; (one of) userWorkspace / agent / apiKey.

---

**ObjectPermission** (`objectPermission`)  
Extends: WorkspaceRelatedEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| roleId | UUID | FK → Role; unique with objectMetadataId |
| objectMetadataId | UUID | FK → ObjectMetadata |
| canReadObjectRecords | boolean | nullable |
| canUpdateObjectRecords | boolean | nullable |
| canSoftDeleteObjectRecords | boolean | nullable |
| canDestroyObjectRecords | boolean | nullable |
| createdAt, updatedAt | timestamptz | |

Relations: role; objectMetadata.

---

**FieldPermission** (`fieldPermission`)  
Extends: WorkspaceRelatedEntity.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| roleId | UUID | FK → Role; unique with fieldMetadataId |
| objectMetadataId | UUID | FK → ObjectMetadata |
| fieldMetadataId | UUID | FK → FieldMetadata |
| canReadFieldValue | boolean | nullable |
| canUpdateFieldValue | boolean | nullable (false or null only) |
| createdAt, updatedAt | timestamptz | |

Relations: role; objectMetadata; fieldMetadata.

---

### 2.5 Application (Syncable / Workspace Scope)

**Application** (`core.application`)  
Extends: WorkspaceRelatedEntity. Represents an app (e.g. Twenty standard) that can own syncable metadata.

| Attribute | Type | Notes |
|----------|------|--------|
| id | UUID | PK |
| workspaceId | UUID | FK → Workspace |
| universalIdentifier | UUID | unique with workspaceId (where deletedAt IS NULL) |
| name | text | |
| description | text | nullable |
| version | text | nullable |
| sourceType | text | e.g. LOCAL |
| sourcePath | text | |
| packageJsonChecksum, yarnLockChecksum | text | nullable |
| packageJsonFileId, yarnLockFileId | UUID | nullable, FK → File |
| createdAt, updatedAt, deletedAt | timestamptz | |

Relations: workspace; packageJsonFile; yarnLockFile; registrations; etc.

---

### 2.6 Entity Relationship Summary (for Diagram)

- **Workspace** is the root: 1 — * UserWorkspace, DataSource, Application, View, FeatureFlag, ApiKey, etc.
- **User** — * UserWorkspace (many-to-many with Workspace).
- **DataSource** — * ObjectMetadata (each object belongs to one data source).
- **ObjectMetadata** — * FieldMetadata; * View; * ObjectPermission; * FieldPermission (via FieldMetadata). ObjectMetadata belongs to one DataSource and one Workspace (and one Application for syncable).
- **FieldMetadata** — * ViewField, * ViewFilter, * ViewSort; optional relationTargetFieldMetadata / relationTargetObjectMetadata (self/object); optional kanban/calendar/mainGroupBy links from View.
- **View** — * ViewField, * ViewSort, * ViewFilter, * ViewGroup, * ViewFilterGroup, * ViewFieldGroup; belongs to ObjectMetadata; optional createdBy (UserWorkspace); optional kanban/calendar/mainGroupBy FieldMetadata.
- **ViewFilter** — optional ViewFilterGroup (parent); **ViewFilterGroup** can have parent ViewFilterGroup (tree) and * ViewFilter.
- **ViewField** — optional ViewFieldGroup.
- **Role** — * RoleTarget (user/agent/apiKey); * ObjectPermission; * FieldPermission. Roles are workspace-scoped (SyncableEntity ⇒ workspaceId + applicationId).
- **ObjectPermission** — one Role, one ObjectMetadata. **FieldPermission** — one Role, one ObjectMetadata, one FieldMetadata.

Use this section together with the entity files under `packages/twenty-server/src/engine/core-modules` and `packages/twenty-server/src/engine/metadata-modules` to draw or generate the full entity diagram.

---

## 3. Technology Stack

### 3.1 Mandatory Versions and Conventions

- **Node**: ^24.5.0
- **Package manager**: Yarn >= 4.0.2 (workspace root; no npm for install)
- **TypeScript**: 5.9.2 (strict; no `any`)
- **Formatting**: Prettier (2 spaces, single quotes, trailing commas, semicolons, ~80 print width)
- **Linting**: Oxlint + Prettier; prefer lint-diff vs main for speed

### 3.2 Backend (twenty-server)

- **Runtime**: Node.js
- **Framework**: NestJS
- **ORM**: TypeORM
- **Database**: PostgreSQL (schema `core` for core + metadata tables)
- **Cache/Session**: Redis (or in-memory for dev)
- **API**: GraphQL (GraphQL Yoga driver), REST (Express)
- **File upload**: `graphql-upload` on `/graphql` and `/metadata`
- **Validation**: class-validator (with Nest container)
- **Session**: express-session (storage from config, e.g. Redis)

### 3.3 Frontend (twenty-front)

- **Framework**: React 18
- **Language**: TypeScript (strict)
- **Build**: Vite
- **Routing**: React Router 6 (createBrowserRouter, lazy-loaded pages)
- **State**: Jotai (global/UI), Apollo Client (server/GraphQL)
- **Styling**: Linaria (zero-runtime CSS-in-JS, styled-components-like API)
- **Icons**: Tabler Icons (React)
- **i18n**: Lingui
- **GraphQL**: Apollo Client; generated types/operations via GraphQL Codegen

### 3.4 Shared and UI Library

- **twenty-shared**: Types, enums, constants, composite-type definitions (e.g. FieldMetadataType, ViewType, AppPath, CoreObjectNameSingular), utils (e.g. isDefined), workflow/AI/workspace types. No React. Consumed by twenty-front and twenty-server.
- **twenty-ui**: React components (inputs, feedback, layout, navigation, theme, display, accessibility), Linaria, Jotai, icons. Depends on twenty-shared. Consumed only by twenty-front.

### 3.5 Code Conventions (Apply Everywhere)

- **Exports**: Named exports only; no default exports.
- **Components**: Functional only; no class components.
- **Types**: Prefer `type` over `interface` except when extending third-party.
- **Enums**: String literals preferred except for GraphQL enums.
- **Naming**: camelCase (vars/functions), SCREAMING_SNAKE_CASE (constants), PascalCase (types/classes), kebab-case (files/dirs). No abbreviations in variable names.
- **Comments**: Short `//` comments; explain WHY, not WHAT; no JSDoc blocks.
- **Helpers**: Use shared guards/utils (e.g. isDefined, isNonEmptyString) instead of ad-hoc checks.

---

## 4. Workspace and Multi-Tenancy

### 4.1 Workspace Entity (Core)

- **Schema**: `core.workspace`
- **Identity**: UUID primary key; unique `subdomain`; optional unique `customDomain`.
- **Display**: displayName, logo (deprecated), logoFileId (FK to file).
- **Invites**: inviteHash; isPublicInviteLinkEnabled.
- **Lifecycle**: deletedAt (soft delete), activationStatus (e.g. INACTIVE, PENDING_CREATION, ONGOING_CREATION); suspendedAt; onboarded workspaces must have defaultRoleId.
- **Retention**: trashRetentionDays, eventLogRetentionDays.
- **Auth**: allowImpersonation; isGoogleAuthEnabled, isMicrosoftAuthEnabled, isPasswordAuthEnabled; bypass flags; isTwoFactorAuthenticationEnforced; editableProfileFields (array).
- **Domains**: isCustomDomainEnabled; relation to ApprovedAccessDomain, EmailingDomain, PublicDomain.
- **Defaults**: defaultRoleId (FK to role); workspaceCustomApplicationId (FK to application).
- **AI**: fastModel, smartModel, routerModel, aiAdditionalInstructions, autoEnableNewAiModels, disabledAiModelIds, enabledAiModelIds, useRecommendedModels.
- **Version**: version (varchar), metadataVersion (integer).
- **Storage**: databaseUrl, databaseSchema (for workspace-specific data).
- **Relations**: workspaceUsers (UserWorkspace), featureFlags, appTokens, keyValuePairs, agents, webhooks, apiKeys, applications, views/viewFields/viewFilters/viewFilterGroups/viewGroups/viewSorts (denormalized or via metadata).

### 4.2 Workspace Resolution

- Request is associated with a workspace by subdomain, custom domain, or context (e.g. JWT, invite hash).
- Middleware: `WorkspaceAuthContextMiddleware` runs on `/graphql`, `/metadata`, and REST `rest/*` to set `workspace` (and user/application) on the request.
- GraphQL workspace schema is built per workspace (and optionally per application) and returned from `conditionalSchema`; if no workspace, return empty schema.

### 4.3 Data Source and Metadata Scope

- Each workspace has one or more **data sources**. Metadata (objects, fields, views, etc.) is scoped by workspace and optionally by application.
- **Syncable entities**: Metadata entities that participate in workspace migrations have `universalIdentifier`, `applicationId` (FK to application), and unique constraint on (workspaceId, universalIdentifier). Base: `SyncableEntity` extending `WorkspaceRelatedEntity`. Each syncable entity type is registered in central constants (e.g. ALL_ENTITY_PROPERTIES_CONFIGURATION_BY_METADATA_NAME, relation/FK registries). Reimplementation of new syncable entities requires: types/constants, validators (no throw, no mutate), cache/transform (entity↔flat), migration action builders, runner/action handlers, module wiring, and integration tests (see .cursor/skills for syncable-entity-*).

---

## 5. Backend Structure

### 5.1 Entry and Bootstrap

- **Entry**: `main.ts` — NestFactory.create(AppModule), CORS, body size limits, session, class-validator container, global exception filter, GraphQL upload on `/graphql` and `/metadata`, then listen on NODE_PORT.
- **Config**: Environment variables via TwentyConfigService; generate front config and inject into served frontend if applicable.

### 5.2 Module Layout

- **AppModule**: Imports GraphQL (async config), TwentyORMModule, GlobalWorkspaceDataSourceModule, ClickHouseModule, CoreEngineModule, ModulesModule, WorkspaceCacheStorageModule, CoreGraphQLApiModule, MetadataGraphQLApiModule, RestApiModule, McpModule, DataSourceModule, MiddlewareModule, WorkspaceMetadataVersionModule, I18nModule; conditionally ServeStaticModule for built front. Middleware: GraphQL and REST routes get GraphQLHydrateRequestFromTokenMiddleware and WorkspaceAuthContextMiddleware; REST also RestCoreMiddleware.
- **Core engine** (`engine/core-modules/`): Auth, user, workspace, billing, file, feature-flag, api-key, application, etc.
- **Metadata modules** (`engine/metadata-modules/`): object-metadata, field-metadata, view, view-field, view-filter, view-sort, view-group, view-filter-group, data-source, role, permission, row-level-permission-predicate, page-layout, page-layout-tab, page-layout-widget, navigation-menu-item, command-menu-item, webhook, AI (agent, chat, execution, etc.), logic-function, front-component, index-metadata, search-field-metadata, etc. Many have “flat” counterparts (flat-object-metadata, flat-field-metadata, …) for workspace-migration and cache layers.
- **Business modules** (`src/modules/`): Calendar, ConnectedAccount, Workflow, FavoriteFolder, Favorite, WorkspaceMember, Messaging, etc., aggregated in ModulesModule.

### 5.3 GraphQL APIs

- **Two endpoints**: Same server, different paths and schema sources.
  - **Core/Workspace** (`/graphql`): Schema is **dynamic per workspace**. Built by WorkspaceSchemaFactory from workspace’s object/field metadata; types and resolvers generated from flat object/field maps. Used for CRUD on workspace records (companies, people, etc.).
  - **Metadata** (`/metadata`): Fixed schema for metadata API (objects, fields, views, viewFields, filters, sorts, roles, etc.). Used to configure the data model and views.
- **Workspace schema build**: Load data sources for workspace; get or recompute flat entity maps (flatObjectMetadataMaps, flatFieldMetadataMaps, flatIndexMaps, flatApplicationMaps); filter by application if needed; build GraphQL types and resolvers (WorkspaceGraphQLSchemaGenerator, WorkspaceResolverFactory); return executable schema.
- **Context**: GraphQL context includes user, workspace, and optionally application (from request).
- **Guards/Hooks**: JWT/session auth; complexity limits (max fields, max root resolvers); optional Sentry tracing; disable introspection for unauthenticated in production; i18n and exception handling in error hook.

### 5.4 REST API

- **Base path**: `/rest` (e.g. `/rest/companies`, `/rest/people`). All under `rest/*path` with catch-all.
- **Auth**: JwtAuthGuard, WorkspaceAuthGuard, CustomPermissionGuard; RestApiExceptionFilter.
- **Core record API** (RestApiCoreController):
  - GET `rest/*path` — list/find (with filter, sort, pagination).
  - GET `rest/*path/groupBy` — group by.
  - POST `rest/*path` — create one.
  - POST `rest/batch/*path` — create many.
  - POST `rest/*path/duplicates` — find duplicates.
  - PATCH `rest/*path` — update.
  - PUT `rest/*path` — replace.
  - PATCH `rest/restore/*path` — restore soft-deleted.
  - PATCH `rest/*path/merge` — merge records.
  - DELETE `rest/*path` — delete (soft or hard per config).
- **Metadata REST** (RestApiMetadataController): CRUD for metadata resources (e.g. views, viewFilters) under a metadata path; GET/DELETE/POST/PATCH/PUT.
- **Request**: AuthenticatedRequest carries user, workspaceId, path (parsed to object name, optional id, relation paths). Service layer parses path, loads metadata, performs operations via TwentyORM or equivalent.

### 5.5 Database and Migrations

- **Primary DB**: PostgreSQL. Core and metadata tables in schema `core` (or as configured).
- **TypeORM**: Core datasource from `src/database/typeorm/core/core.datasource.ts`; migrations in `src/database/typeorm/core/migrations/common/` (and optional billing subfolder). Table `_typeorm_migrations` for tracking.
- **Migrations**: Generate with Nx target typeorm migration:generate; name kebab-case; never delete or rewrite committed migrations; include up/down.
- **ClickHouse** (optional): Analytics; migrations in `src/database/clickHouse/migrations/` (SQL files + run script).
- **Workspace tables**: Per-workspace record tables (e.g. company, person) are created/managed via workspace migrations and metadata; names come from object metadata (e.g. targetTableName).

### 5.6 Key Entities (Summary)

- **Core**: Workspace, User, UserWorkspace (workspace membership), Application, ApiKey, AppToken, File, FeatureFlag, ClientConfig, KeyValuePair, PostgresCredentials, PublicDomain, EmailingDomain, ApprovedAccessDomain, WorkspaceSSOIdentityProvider, Billing*, TwoFactorAuthentication, etc.
- **Metadata (syncable where noted)**: DataSource; ObjectMetadata (syncable); FieldMetadata (syncable); View, ViewField, ViewFilter, ViewSort, ViewGroup, ViewFilterGroup, ViewFieldGroup (syncable); IndexMetadata, IndexFieldMetadata; ObjectPermission, FieldPermission; Role, RoleTarget; RowLevelPermissionPredicate; PageLayout, PageLayoutTab, PageLayoutWidget (syncable); NavigationMenuItem, CommandMenuItem (syncable); Webhook; Agent, AgentChatThread, AgentMessage, AgentTurn, AgentTurnEvaluation; LogicFunction, LogicFunctionLayer; FrontComponent (syncable); SearchFieldMetadata; etc.
- **Workspace-related base**: WorkspaceRelatedEntity (workspaceId); SyncableEntity adds universalIdentifier, applicationId.

### 5.7 Object Metadata Entity

- **Table**: objectMetadata (metadata schema).
- **Keys**: id (UUID); dataSourceId; nameSingular, namePlural, labelSingular, labelPlural; description; icon; standardOverrides (JSONB); targetTableName (deprecated but present); isCustom, isRemote, isActive, isSystem, isUIReadOnly, isAuditLogged, isSearchable; duplicateCriteria (JSONB); shortcut; labelIdentifierFieldMetadataId, imageIdentifierFieldMetadataId; isLabelSyncedWithName.
- **Relations**: dataSource; fields (FieldMetadata); indexMetadatas; objectPermissions; fieldPermissions; views.
- **Uniques**: (nameSingular, workspaceId), (namePlural, workspaceId).

### 5.8 Field Metadata Entity

- **Table**: fieldMetadata.
- **Keys**: id; objectMetadataId; type (FieldMetadataType); name; label; defaultValue, description, icon; standardOverrides, options, settings (JSONB); isCustom, isActive, isSystem, isUIReadOnly; isNullable, isUnique; isLabelSyncedWithName; relationTargetFieldMetadataId, relationTargetObjectMetadataId; morphId (for MORPH_RELATION).
- **Check**: If type is MORPH_RELATION then morphId must be set.
- **Uniques**: (name, objectMetadataId, workspaceId).
- **Relations**: object; relationTargetFieldMetadata; relationTargetObjectMetadata; indexFieldMetadatas; fieldPermissions; viewFields; viewFilters; viewSorts; kanbanAggregateOperationViews; calendarViews; mainGroupByFieldMetadataViews.

### 5.9 Field Metadata Types (twenty-shared)

- **Enum FieldMetadataType**: TEXT, NUMBER, BOOLEAN, DATE, DATE_TIME, CURRENCY, FULL_NAME, EMAILS, PHONES, LINKS, ADDRESS, RICH_TEXT, RICH_TEXT_V2, SELECT, MULTI_SELECT, RATING, POSITION, UUID, RAW_JSON, ARRAY, ACTOR, FILES, RELATION, MORPH_RELATION, NUMERIC, TS_VECTOR.
- **Composite types**: Address, Currency, Emails, Phones, Links, FullName, Actor, RichTextV2 have subfield definitions in twenty-shared; used for options/defaultValue/settings typing.

### 5.10 View Entity

- **Table**: view (core schema).
- **Keys**: id; name; objectMetadataId; type (ViewType: TABLE, KANBAN, CALENDAR, FIELDS_WIDGET); key (ViewKey, e.g. INDEX); icon; position; isCompact; isCustom; openRecordIn (ViewOpenRecordIn: SIDE_PANEL, etc.); kanbanAggregateOperation, kanbanAggregateOperationFieldMetadataId; calendarLayout, calendarFieldMetadataId; mainGroupByFieldMetadataId; shouldHideEmptyGroups; anyFieldFilterValue; visibility (ViewVisibility); createdByUserWorkspaceId; createdAt, updatedAt, deletedAt.
- **Relations**: objectMetadata; viewFields; viewFilterGroups; viewFilters; viewSorts; viewGroups; viewFieldGroups; kanbanAggregateOperationFieldMetadata; calendarFieldMetadata; mainGroupByFieldMetadata; createdBy (UserWorkspace).
- **Check**: If type is CALENDAR then calendarLayout and calendarFieldMetadataId must be set.

### 5.11 Auth and Guards

- **Auth module**: JWT (access, refresh, login token), session, password reset, OAuth (Google, Microsoft), 2FA, impersonation, invite-by-hash.
- **Guards**: JwtAuthGuard, WorkspaceAuthGuard, CustomPermissionGuard; optional CaptchaGuard; IsApplicationAuthContextGuard.
- **Middleware**: GraphQLHydrateRequestFromTokenMiddleware (resolve user/workspace/application from token); WorkspaceAuthContextMiddleware.
- **REST**: Same guards on RestApiCoreController and RestApiMetadataController.

---

## 6. Frontend Structure

### 6.1 Entry and Routing

- **Entry**: `index.tsx` mounts `App` into `#root`; App wraps with JotaiProvider (jotaiStore) and styles.
- **Router**: createBrowserRouter(createRoutesFromElements(...)); root element AppRouterProviders (which provides ApolloProvider and core Apollo provider). DefaultLayout for main app; BlankLayout for e.g. Authorize.
- **Paths** (AppPath in twenty-shared): Verify, VerifyEmail, SignInUp, Invite (with workspaceInviteHash), ResetPassword (with passwordResetToken); CreateWorkspace, CreateProfile, SyncEmails, InviteTeam, PlanRequired, PlanRequiredSuccess, BookCallDecision, BookCall; Index; RecordIndexPage (`/objects/:objectNamePlural`), RecordShowPage (`/object/:objectNameSingular/:objectRecordId`); Settings (SettingsCatchAll `settings/*`); Authorize; NotFoundWildcard.
- **Lazy-loaded pages**: RecordIndexPage, RecordShowPage, SignInUp, PasswordReset, Authorize, CreateWorkspace, CreateProfile, SyncEmails, InviteTeam, ChooseYourPlan, PaymentSuccess, BookCallDecision, BookCall, NotFound. Settings via SettingsRoutes (nested).

### 6.2 State and Data

- **Jotai**: Global UI and domain state (e.g. auth step, dropdown state, dialog manager, side panel, context store).
- **Apollo**: Primary client for metadata GraphQL (base URL `${REACT_APP_SERVER_BASE_URL}/metadata`); optional ApolloCoreProvider for core/workspace GraphQL. Generated types and operations in `src/generated/` and `src/generated-metadata/`.
- **Metadata store**: Frontend loads or subscribes to object/field metadata and exposes it to record and settings UIs (e.g. MetadataProviderInitialEffects).

### 6.3 Feature Modules (Frontend)

- **Location**: `src/modules/` — e.g. object-record, object-metadata, views, navigation, page-layout, auth, apollo, settings, workflow, ai, favorites, activities, spreadsheet-import, side-panel, action-menu, blocknote-editor, advanced-text-editor, front-components, metadata-store, navigation-menu-item, billing, context-store, error-handler, etc.
- **Pages**: `src/pages/` — auth (SignInUp, PasswordReset, Authorize), object-record (RecordIndexPage, RecordShowPage), onboarding (CreateWorkspace, CreateProfile, SyncEmails, InviteTeam, etc.), settings (many sub-routes), not-found (NotFound).
- **Record UI**: Record index (table/kanban/calendar) and record show (detail); field widgets per FieldMetadataType; create/edit/delete, bulk actions, duplicate find, merge.

### 6.4 UI and Theming

- **twenty-ui**: Buttons, inputs, modals, dropdowns, layout (DefaultLayout, BlankLayout), navigation, theme tokens, accessibility utilities.
- **Linaria**: Component-level styles; no runtime CSS-in-JS bundle from styled-components.

---

## 7. Configuration and Environment

### 7.1 Server (.env)

- **Required**: NODE_ENV, PG_DATABASE_URL, REDIS_URL, APP_SECRET, FRONTEND_URL.
- **Optional**: PORT (NODE_PORT), ACCESS_TOKEN_EXPIRES_IN, LOGIN_TOKEN_EXPIRES_IN, REFRESH_TOKEN_EXPIRES_IN, FILE_TOKEN_EXPIRES_IN; AUTH_GOOGLE_*, AUTH_MICROSOFT_*; IS_BILLING_ENABLED; STORAGE_TYPE, STORAGE_LOCAL_PATH; MESSAGE_QUEUE_*; LOGGER_*, SENTRY_*; EMAIL_*; CAPTCHA_*; GRAPHQL_MAX_FIELDS, GRAPHQL_MAX_ROOT_RESOLVERS; MUTATION_MAXIMUM_AFFECTED_RECORDS; SIGN_IN_PREFILLED; IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS; etc.
- **Feature flags**: e.g. AUTH_PASSWORD_ENABLED, IS_MULTIWORKSPACE_ENABLED, CALENDAR_PROVIDER_GOOGLE_ENABLED, etc.

### 7.2 Frontend (.env)

- **Required**: REACT_APP_SERVER_BASE_URL (e.g. http://localhost:3000).
- **Optional**: REACT_APP_PORT.

### 7.3 Build and Nx

- **Build order**: twenty-shared first; then twenty-ui, twenty-front, twenty-server as needed.
- **Targets**: build, start, test, lint, lint:diff-with-main, typecheck, fmt; database:reset, database:init:prod, database:migrate:prod; graphql:generate (and graphql:generate --configuration=metadata); typeorm migration:generate; workspace:sync-metadata.
- **CI**: Run `packages/twenty-utils/setup-dev-env.sh` before tests/builds/DB operations if not pre-configured.

---

## 8. Key Flows (Reimplementation Checklist)

### 8.1 Workspace and Schema

1. Resolve workspace from request (subdomain/custom domain/JWT).
2. Load data sources and flat entity maps for workspace (with cache).
3. Build workspace GraphQL schema from object/field metadata (types + resolvers).
4. Serve workspace schema on `/graphql` and metadata schema on `/metadata`.

### 8.2 Record CRUD

1. REST: Parse path to object name and optional id/relations; resolve object metadata; check permissions; execute findMany/findOne/createOne/createMany/updateOne/updateMany/delete/restore/merge/groupBy/duplicates via single or batch.
2. GraphQL (workspace): Resolve dynamic types and resolvers from workspace schema; delegate to same record layer (TwentyORM or equivalent).

### 8.3 Metadata CRUD

1. Metadata API (GraphQL or REST): Validate input; apply syncable-entity rules (universalIdentifier, applicationId); run validators and migration actions if using workspace-migration pattern; persist to metadata tables; invalidate workspace cache so schema and flat maps recompute.

### 8.4 Auth

1. Sign-in: Credentials or OAuth; issue access + refresh tokens; optional 2FA step.
2. Invite: Accept invite by hash; add user to workspace with role.
3. Every request: Hydrate user/workspace from JWT/session; attach to context; guards enforce workspace and permission.

### 8.5 Frontend

1. Bootstrap: Load app; verify token (VerifyLoginTokenEffect); load metadata (MetadataProviderInitialEffects).
2. Navigation: Resolve object name from route (`/objects/:objectNamePlural`, `/object/:objectNameSingular/:objectRecordId`); load object/field metadata; render index (table/kanban/calendar) or show (detail) with correct field widgets.
3. Settings: Nested routes for data model (objects, fields), roles, API keys, webhooks, AI, billing, profile, security, etc.; forms call metadata or core GraphQL/REST.

---

## 9. Testing Requirements

- **Unit**: Jest; test behavior not implementation; descriptive names (“should … when …”); prefer single-file runs for speed.
- **Integration**: twenty-server integration tests with DB reset (test:integration:with-db-reset).
- **E2E**: Playwright in twenty-e2e-testing; use “Continue with Email” and prefilled credentials when testing UI.
- **Storybook**: Build and test (storybook:build, storybook:test) for twenty-front.
- **Syncable entities**: Integration tests mandatory for metadata entities (validators, input transpilation, CRUD); see syncable-entity-testing skill.

---

## 10. Security and Data

- **CSV export**: Sanitize before formatting (sanitize then formatValueForCSV).
- **Input**: Validate and sanitize before processing.
- **Errors**: Do not expose internals; use exception handler and consistent codes (e.g. UNAUTHENTICATED).

---

## 11. Standard Objects (Reference)

- **CoreObjectNameSingular** (twenty-shared): activity, activityTarget, apiKey, attachment, blocklist, calendarChannel, calendarEvent, comment, company, connectedAccount, dashboard, timelineActivity, favorite, favoriteFolder, message, messageChannel, messageParticipant, messageFolder, messageThread, note, noteTarget, opportunity, person, task, taskTarget, webhook, workspaceMember, messageThreadSubscriber, workflow, messageChannelMessageAssociation, workflowVersion, workflowRun.
- Standard objects have metadata seeded or created per workspace; custom objects are created via metadata API and get a target table and workspace schema type.

---

## 12. File and Naming Conventions

- **Files**: kebab-case; suffixes: `.component.tsx`, `.styles.ts`, `.test.tsx`, `.service.ts`, `.entity.ts`, `.dto.ts`, `.module.ts`.
- **Barrel**: index.ts with named exports only.
- **Import order**: External libs first, then internal (`@/` or workspace packages), then relative.
- **Size**: Components &lt; 300 lines; services &lt; 500 lines; extract to hooks/utils when larger.

---

## 13. Out of Scope for Minimal Reimplementation

- Full billing (Stripe) and feature flags.
- ClickHouse analytics.
- MCP (Model Context Protocol) server.
- Companion/Electron app, Zapier, SDK, create-twenty-app, Docker/Helm (reference only).
- All optional integrations (Gmail, Microsoft, etc.) — can be stubbed or feature-flagged.
- Exact replication of every standard object and relation; the spec above is enough to implement a minimal metadata-driven CRM with workspace, objects, fields, views, record CRUD, and auth.

---

*End of requirements specification.*
