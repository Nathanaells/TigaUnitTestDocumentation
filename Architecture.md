# Tiga Architecture

This repository contains Tiga, a metadata-driven business application framework. The system is built around a generic server that stores application metadata, dynamically maps business entities to database tables, exposes service endpoints, and runs workflow automation written mostly in JavaScript.

At a high level:

```text
Browser / Silverlight client
        |
        | HTTP JSON / WCF-style REST / SOAP / raw upload/download
        v
Fugar.Host ASP.NET Core process
        |
        | CoreWCF service facade + gRPC + MVC controllers
        v
Fugar.Service
        |
        | business managers
        v
Fugar.Lib
        |
        | IDataManager / IFileTransferManager / cache / workflow executor
        v
Fugar.Persistence + workspace FileStorages + Redis + database
```

## Repository Layout

### `FugarServer`

Server-side .NET solution. The main solution is:

```text
FugarServer/FugarServer.sln
```

Important projects:

- `Fugar.Host`: ASP.NET Core executable host. It starts Kestrel, configures CoreWCF endpoints, gRPC, middleware, workspace config, and long-running workers.
- `Fugar.Builder`: dependency graph and infrastructure composition. This is where managers, persistence, workflow executors, Redis cache managers, email, file storage, and service endpoints are wired together.
- `Fugar.Service`: service facade. It exposes `FugarService` partial classes implementing contracts like `IEntityManagerService`, `ISecurityManagerService`, `IWorkflowManagerService`, and others.
- `Fugar.Lib`: business logic layer. Contains managers for entities, security, workflow, reports, packages, preferences, config, connectivity, audit, and other business areas.
- `Fugar.Workflow`: workflow execution engines. Contains the Jint JavaScript executor, Node/gRPC executor, script manager, and supporting Jint integration.
- `Fugar.Persistence`: NHibernate-backed persistence abstraction, unit of work, file storage, dynamic mapping, and low-level database operations.
- `Fugar.Model`: shared domain and metadata model classes.
- `Fugar.Mapping`: NHibernate mapping definitions for system entities.
- `Fugar.Connectivity`: email and IMAP integration.
- `Fugar.Scheduler`: Quartz scheduler wrapper.
- `Fugar.Lib.Common`: common serialization, templating, logging, and utility code.
- `Fugar.CustomException`: custom exception types.
- `Fugar.GrammarParser`: Irony-based parsers used for filters and workflow parameter strings.

### `FugarClient`

Client applications.

- `Fugar.HTML/Fugar.UI.HTML`: AngularJS web client.
- `Fugar.Silverlight/Fugar.UI.Silverlight`: older Silverlight client.

The HTML client bootstraps from:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML/index.html
FugarClient/Fugar.HTML/Fugar.UI.HTML/Bootstrapper.js
FugarClient/Fugar.HTML/Fugar.UI.HTML/BaseHref.js
```

`Bootstrapper.js` declares the AngularJS module named `Tiga`, registers services, managers, directives, filters, and controllers. Client service wrappers live mostly under:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML/Infrastructures/Services
```

### `FugarCommon`

Shared JavaScript controls, nginx config, third-party binaries, and `.proto` definitions used by server/client projects.

### `DB Scripts`

Release and migration SQL scripts.

### `BuildTools`

Build scripts and CI/build configuration.

## Runtime Workspace

The server is workspace-driven. A workspace contains config files, logs, static content, attachments, and workflow task scripts.

Sample workspace:

```text
FugarServer/Fugar.Host/SampleWorkspaces/W0
```

Runtime workspace is usually copied to:

```text
FugarServer/Fugar.Host/bin/Debug/Workspaces/W0
```

Important workspace directories:

- `Configs`: `System.config`, `Business.config`, `hibernate.cfg.xml`, and service-account credentials.
- `FileStorages`: file-backed data used by the app.
- `FileStorages/WorkflowTasks`: JavaScript workflow task files loaded by `WorkflowManager`.
- `FileStorages/BusinessAttachments`: uploaded business attachments.
- `FileStorages/OutgoingEmailAttachments`: generated email attachments.
- `StaticContents`: files served through the `Main` static content endpoint.
- `LogFiles`: server logs.

## Server Startup

The executable starts in:

```text
FugarServer/Fugar.Host/Program.cs
```

Startup flow:

1. `Program.Main` builds and runs the ASP.NET Core host.
2. Kestrel listens on two ports from configuration:
   - `TigaPort`: main HTTP/CoreWCF port.
   - `GRPCPort`: HTTP/2 gRPC port.
3. `Startup.ConfigureServices` resolves the workspace directory.
4. `MainObjectGraphBuilder.BuildObjectGraph` builds the application object graph from workspace config.
5. Core services are registered in dependency injection.
6. NHibernate mappings are built.
7. Database connection is opened.
8. application seed data is initialized.
9. workflow, report, email, tenant refresh, authorization cleanup, audit, and entity retrieval workers are started.
10. `Startup.Configure` registers:
    - gRPC service endpoint,
    - attachment controller pipeline,
    - CoreWCF `FugarService` endpoints.

Main composition file:

```text
FugarServer/Fugar.Builder/MainObjectGraphBuilder.cs
```

This file is the best starting point when asking "where is this dependency created?"

## Configuration

`System.config` and `Business.config` are read through `PropertyFile` / `PropertySource`.

Common `System.config` settings include:

- `Tenant.Workspace.ShortcutID`
- `Host.BaseAddress`
- `Host.LocalAddress`
- `Client.WebAddress`
- `System.ShouldUpdateSchemaDuringStart`
- `Email.Simulated`
- `Email.From`
- `Email.ErrorRecipient`
- `SMTP.*`
- `Imap.IsAllowed`
- `WorkflowManager.RetrieveWorkflowTaskFileBasedMaxRetry`
- `TaskExecutor.AllowDebug`
- `TaskExecutor.MemoryLimit`
- `Node.Grpc.Server`

Common `Business.config` settings include:

- Redis connection settings.
- schedule include/exclude filters.
- Google service account paths.
- business-specific feature flags.
- culture settings.

Database connection details are in:

```text
Workspaces/W0/Configs/hibernate.cfg.xml
```

## Service Endpoints

Endpoint registration is in:

```text
FugarServer/Fugar.Builder/ServiceHostBuilder.cs
```

The host exposes both SOAP-like and REST-like CoreWCF endpoints.

REST-style endpoints use service names such as:

- `Builder`
- `SchemaDesigner`
- `PackageManager`
- `EntityManager`
- `SecurityManager`
- `ReportManager`
- `WorkflowManager`
- `PreferenceManager`
- `TenantWorkspaceManager`
- `ErrorManager`
- `ConnectivityManager`
- `BusinessConfigManager`
- `AuthorizationManager`
- `ClientLogManager`
- `EmailManager`

Special endpoints:

- `Main`: static content retrieval.
- `Download`: raw download service.
- `Raw/PackageManagerRaw`
- `Raw/EntityManagerRaw`
- `policy`

The HTML client calls these endpoints through:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML/Infrastructures/ServiceClientManager.js
```

`ServiceClientManager.CallService` prefixes URLs with `$rootScope.TigaHost`, sends credentials/cookies, unwraps the first property from the wrapped service response, and forwards the result to the caller.

## Client Architecture

The AngularJS app is loaded by `index.html`, which lists a large set of scripts. `Bootstrapper.js` then registers:

- service wrappers, such as `EntityManagerService`, `SecurityManagerService`, `WorkflowManagerService`;
- higher-level client managers, such as `workflowManager`, `workflowExecutorManager`, `packageProfileManager`, `modalManager`, `stateManager`;
- reusable directives and UI helpers;
- controllers for modules under `Modules`.

Typical client request flow:

```text
Controller
  -> Angular service wrapper, for example EntityManagerService
  -> ServiceClientManager.CallService
  -> Fugar.Host CoreWCF REST endpoint
  -> Fugar.Service partial method
  -> Fugar.Lib manager
  -> Fugar.Persistence / FileStorages / Redis / external service
```

Example:

```text
ChangePasswordCtrl.js
  -> SecurityManagerService.UpdateCurrentUserWithPasswordAsync
  -> SecurityManager/UpdateCurrentUserWithPassword
  -> FugarService.SecurityManager.UpdateCurrentUserWithPassword
  -> UserManager.RestrictedUpdate
```

## Domain Model

Tiga is metadata-driven. Instead of writing a normal C# class and database table for every business table, the app stores table definitions and uses dynamic mapping.

Core metadata models:

- `PackageDefinition`: application/module grouping.
- `EntityDefinition`: business table definition.
- `PropertyDefinition`: business field definition.
- `EntityRelationship`: relationship definition.
- `ViewDefinition`: UI layout definition.
- `WorkflowDefinition`: ordered workflow definition.
- `WorkflowAction`: one action inside a workflow definition.
- `WorkflowTask`: named JavaScript task metadata.
- `CustomActionButtonDefinition`: custom button/action configuration.
- `ScheduleDefinition`: scheduled job configuration.
- `ReportDefinitionCatalog`: report configuration.
- `UserCredential`, `GroupCredential`, `GroupToPackageProfileACL`: security and authorization data.
- `BusinessEntityBase`: generic runtime business record container.

Business records use:

```text
BusinessEntityBase.Properties
BusinessEntityBase.PropertiesWrapper
```

`Fugar.Service` often boxes/unboxes properties between the raw `IDictionary` form and a string-keyed wrapper so WCF/JSON serialization can round-trip records reliably.

## Persistence And Dynamic Mapping

Persistence is built by:

```text
FugarServer/Fugar.Builder/DataManagerBuilder.cs
```

Key abstractions:

- `IDataManager`: application-facing persistence API.
- `IDbManager`: lower-level database abstraction.
- `IUnitOfWork`: transaction boundary.
- `NHibernateWrapper`: NHibernate implementation.
- `IFileTransferManager`: file storage abstraction.

The database uses tenant-prefixed system and business tables:

```text
S_<Tenant>_...  system tables
B_<Tenant>_...  business tables
P_...           system columns
C_...           business columns
```

For workspace `W0`, table names are typically prefixed as:

```text
S_W0_
B_W0_
```

Mapping startup:

1. `MapperBuilder.BuildBaseMap` maps only `UserMap`.
2. `IDataManager.Open` builds an initial NHibernate session factory.
3. `MapperBuilder.BuildAllMap` adds system mapped types and loads dynamic mappings from `UserMap`.
4. `IDataManager.Open` is called again with the full mapping set.

Dynamic entity tables are created from `EntityDefinition` metadata. `NHibernateWrapper.CreateTable` generates mapping XML for both the business table and audit table, stores it in `UserMap`, and rebuilds the session factory.

## Business Entity Flow

The central generic business data service is:

```text
FugarServer/Fugar.Lib/Business/EntityManager/BusinessEntityManager.cs
```

It handles:

- create/update/delete/archive/unarchive;
- related records;
- row-level authorization;
- filters and sorting;
- dynamic SQL retrieval;
- import/export;
- audit hooks;
- workflow event hooks;
- state changes;
- encryption handling;
- retrieval worker threads for multiplexed data loads.

`BusinessEntityManager` raises events such as:

- `PreCreateBusinessEntity`
- `PostCreateBusinessEntity`
- `PreUpdateBusinessEntity`
- `PostUpdateBusinessEntity`
- `PreDeleteBusinessEntity`
- `PostDeleteBusinessEntity`
- `PreLinkBusinessEntity`
- `PostLinkBusinessEntity`
- `CustomAction`
- `PreStateChangedBusinessEntity`
- `PostStateChangedBusinessEntity`
- bulk variants of the above.

These events are the bridge into workflow execution.

## Workflow Architecture

Workflow definitions are stored in the database. Workflow task script files live under:

```text
Workspaces/W0/FileStorages/WorkflowTasks
```

Key classes:

- `WorkflowManager`: manages workflow definitions/tasks and aggregates workflow task files into executable scripts.
- `WorkflowExecutorManager`: subscribes to entity/scheduler events and runs matching workflows.
- `JintTaskExecutor`: executes JavaScript workflows inside .NET using Jint.
- `NodeTaskExecutor`: delegates node workflows to a Node gRPC server.
- `JintTaskScriptManager`: adds bulk workflow wrappers and initializes workflow parameters.
- `WorkflowTaskPrimitivesWrapper`: exposes server-side helper functions to JavaScript workflow scripts.
- `WorkflowCacheManager`: caches generated workflow scripts and workflow definition lookups in Redis.

### Workflow Execution Flow

```text
BusinessEntityManager event or ScheduleManager trigger
  -> WorkflowExecutorManager handler
  -> Retrieve matching WorkflowDefinitionHolder/WorkflowDefinition
  -> Evaluate workflow filter condition
  -> WorkflowManager.ConvertToCustomTaskContent
  -> WorkflowManager.GenerateRunScript
  -> read each active WorkflowAction.WorkflowTaskName + ".js"
  -> inject action parameters
  -> append workflow task content
  -> JintTaskExecutor.RunTask or NodeTaskExecutor.RunTask
```

The aggregation mechanism is:

```text
FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowManager.cs
```

`GenerateRunScript` loops through active workflow actions, loads each task file, injects parameters, adds log calls, and concatenates the result into one script. This is the "compiled" workflow definition executed by Jint.

### Jint Runtime

`JintTaskExecutor.RunTask`:

- prepends `var FugarModel = importNamespace('Fugar.Model');`;
- creates a fresh Jint `Engine`;
- enables CLR access for configured assemblies;
- registers parameters like `p_BusinessEntities`, `p_EventTriggerType`, etc.;
- registers helper functions from `WorkflowExecutorManager`;
- evaluates the final script;
- wraps workflow-thrown errors in workflow-specific exceptions.

Common JavaScript helper functions exposed to workflow scripts include:

- `Log`
- `ThrowError`
- `CreateBusinessEntity`
- `UpdateBusinessEntity`
- `DeleteBusinessEntity`
- `RetrieveExtendedBusinessEntity`
- `RetrieveRelatedBusinessEntity`
- `SendEmail`
- `SendHtmlEmail`
- `CreateAttachment`
- `RetrieveAttachmentFile`
- `GetUserCredential`
- `HasGroupCredential`
- `CallExternalService`
- `RetrieveBusinessConfig`
- `UpdateBusinessConfigMap`
- many bulk and report helpers.

### Node Workflows

Some workflows can run through the Node executor. `WorkflowDefinition.NodeExecutor` decides the execution path. For node workflows, active workflow task names are passed to `NodeTaskExecutor`, which communicates with the configured Node gRPC server.

Node workflow files are also present under:

```text
Workspaces/W0/FileStorages/WorkflowTasks/node_workflows
```

## Scheduling

Scheduling is built around Quartz through:

```text
FugarServer/Fugar.Scheduler
FugarServer/Fugar.Lib/Business/ScheduleManager
```

`Startup` starts the scheduler in the background after core services and workflow/report managers have started.

For workflow schedules:

```text
ScheduleManager.Trigger
  -> WorkflowExecutorManager.HandleTrigger
  -> WorkflowDefinition by ID
  -> workflow runs as #AdminSession#
```

## Security And Authorization

Main classes:

- `AuthorizationManager`
- `UserManager`
- `GroupManager`
- `ACLManager`
- `TenantWorkspaceManager`
- `UserTokenAuthorizationManager`
- `SessionTokenAuthorizationManager`

Authentication creates a `UserSession` and returns cookies through `CookieManager`.

Important service facade:

```text
FugarServer/Fugar.Service/FugarService.SecurityManager.cs
```

Authorization influences:

- service endpoint access;
- row-level business entity access;
- package/profile/table visibility;
- group and ACL checks;
- session cleanup;
- Google sign-in;
- OTP/PIN flows.

## Static Content And Files

Static content is served by `StaticContentManager` from:

```text
Workspaces/W0/StaticContents
```

Attachment and file storage uses `IFileTransferManager`, rooted at:

```text
Workspaces/W0/FileStorages
```

Common subdirectories:

- `WorkflowTasks`: workflow script files.
- `BusinessAttachments`: business record attachments.
- `OutgoingEmailAttachments`: generated email attachment files.

## Caching

Redis is configured in `Business.config`. `MainObjectGraphBuilder` creates one or more `RedisCacheManager` instances and registers cache wrappers for:

- package profiles;
- authorization;
- preferences;
- reports;
- workflows;
- user sessions.

Workflow script cache keys include session, entity, workflow definition ID, and event trigger type.

## Email, IMAP, External Services

Email is wired in `MainObjectGraphBuilder`.

Depending on `Email.Simulated`, the app uses:

- `SimulatedEmailManager`; or
- `EmailManager`.

Workers:

- sender worker;
- optional retry worker.

IMAP can be enabled through `Imap.IsAllowed`.

External HTTP calls are handled through `ExternalServiceManager` and workflow helper wrappers.

Google integrations are handled through `GoogleServiceManager`.

## Reports

Reports are managed by:

```text
FugarServer/Fugar.Lib/Business/ReportManager
```

`ReportManager.Start()` is called during startup. Reports can retrieve business data, generate files, email results, and integrate with Google spreadsheets/Drive.

## Audit

Audit manager subscribes to `BusinessEntityManager` post events and records history. It is started in `Startup.ConfigureServices` before most workers are started.

Business entities may have paired audit tables generated by dynamic mapping, usually named with an `H_` prefix inside the dynamic mapping logic.

## Change Password Flow

The settings Change Password feature does not appear to use the workflow task system. Its flow is:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML/Modules/Settings/Controllers/ChangePasswordCtrl.js
  -> securityManagerService.UpdateCurrentUserWithPasswordAsync
  -> SecurityManager/UpdateCurrentUserWithPassword
  -> FugarService.SecurityManager.UpdateCurrentUserWithPassword
  -> UserManager.RestrictedUpdate
```

The server validates the old password, validates password strength, hashes the new password, updates the user, and strips password data before returning the response.

## Development Setup

The README describes the usual setup:

1. Build `FugarServer/FugarServer.sln`.
2. Copy `Fugar.Host/SampleWorkspaces/W0` to the runtime `Workspaces/W0`.
3. Configure `Workspaces/W0/Configs/System.config`.
4. Configure database credentials in `Workspaces/W0/Configs/hibernate.cfg.xml`.
5. Copy HTML static files into `Workspaces/W0/StaticContents` when serving through the server.
6. Copy workflow files into `Workspaces/W0/FileStorages/WorkflowTasks`.
7. Run `Fugar.Host`.
8. Browse to the configured host, typically `/Main/index.html`.

## Where To Look

For server startup and service wiring:

```text
FugarServer/Fugar.Host/Program.cs
FugarServer/Fugar.Host/Startup.cs
FugarServer/Fugar.Builder/MainObjectGraphBuilder.cs
FugarServer/Fugar.Builder/ServiceHostBuilder.cs
```

For client bootstrapping:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML/index.html
FugarClient/Fugar.HTML/Fugar.UI.HTML/Bootstrapper.js
FugarClient/Fugar.HTML/Fugar.UI.HTML/Infrastructures/ServiceClientManager.js
```

For generic business records:

```text
FugarServer/Fugar.Lib/Business/EntityManager/BusinessEntityManager.cs
FugarServer/Fugar.Service/FugarService.EntityManager.cs
FugarServer/Fugar.Model/BusinessEntityBase.cs
```

For metadata/schema:

```text
FugarServer/Fugar.Model/EntityDefinition.cs
FugarServer/Fugar.Model/PropertyDefinition.cs
FugarServer/Fugar.Model/EntityRelationship.cs
FugarServer/Fugar.Lib/Business/SchemaDesigner
```

For persistence/dynamic mapping:

```text
FugarServer/Fugar.Builder/DataManagerBuilder.cs
FugarServer/Fugar.Builder/MapperBuilder.cs
FugarServer/Fugar.Persistence/NHibernateWrapper.cs
FugarServer/Fugar.Mapping
```

For workflows:

```text
FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowManager.cs
FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowExecutorManager.cs
FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowTaskPrimitivesWrapper.cs
FugarServer/Fugar.Workflow/JintTaskExecutor.cs
FugarServer/Fugar.Workflow/NodeTaskExecutor.cs
FugarServer/Fugar.Workflow/JintTaskScriptManager.cs
FugarServer/Fugar.Host/SampleWorkspaces/W0/FileStorages/WorkflowTasks
```

For security:

```text
FugarServer/Fugar.Service/FugarService.SecurityManager.cs
FugarServer/Fugar.Lib/Business/SecurityManager
```

For files/static content:

```text
FugarServer/Fugar.Persistence/FileTransferManager.cs
FugarServer/Fugar.Lib/Business/StaticContentManager
FugarServer/Fugar.Host/SampleWorkspaces/W0/FileStorages
FugarServer/Fugar.Host/SampleWorkspaces/W0/StaticContents
```

## Mental Model

Tiga is best understood as a generic business-app engine:

1. Metadata describes applications, tables, fields, relationships, views, workflow definitions, reports, and schedules.
2. NHibernate dynamic mapping turns that metadata into runtime database mappings.
3. The AngularJS client retrieves metadata and renders generic grids/forms/actions.
4. User actions call generic service endpoints.
5. `BusinessEntityManager` performs the requested data operation and raises lifecycle events.
6. `WorkflowExecutorManager` reacts to those events and executes configured workflow scripts.
7. Workflow scripts call back into server primitives to create/update/query records, send email, generate reports, and call external services.

That is the heart of the repo: metadata in the database, JavaScript workflow in file storage, and a .NET host that binds them together.
