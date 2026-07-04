# Cross-cutting Concepts

This chapter documents concepts and solutions that span multiple parts of koalixcrm, including
security, access control, configuration management, data models, test coverage, and UI
architecture.

## Documents in this Chapter

### Security

| Document | Description |
|---|---|
| [QQ_SD_Security_Report.md](QQ_SD_Security_Report.md) | Security analysis and findings: authentication mechanisms, credential handling, input validation, and identified vulnerabilities with severity ratings |

### Access Control

| Document | Description |
|---|---|
| [QQ_SD_AccessControl.md](QQ_SD_AccessControl.md) | Authentication mechanisms (OIDC, Bearer JWT, M2M), workspace-scoped RBAC (seven roles, permission mapping), and Django permission integration |

### Configuration

| Document | Description |
|---|---|
| [QQ_SD_Configuration.md](QQ_SD_Configuration.md) | Runtime and deploy-time configuration: environment variables, Django settings overlay pattern, and secrets management |
| [QQ_SD_Settings.md](QQ_SD_Settings.md) | User-specific preferences and personalisation options: timezone, language code, and workspace selection |
| [QQ_SD_Parameterization.md](QQ_SD_Parameterization.md) | Developer-controlled constants, hardcoded defaults, and build-time parameters |

### Data Models

| Document | Description |
|---|---|
| [QQ_SD_EntityRelationDiagram.md](QQ_SD_EntityRelationDiagram.md) | Physical data model: Mermaid ER diagrams for all eight Django apps, database table names, and foreign-key relationships |

### Testing

| Document | Description |
|---|---|
| [QQ_SD_Unit_Test_Coverage.md](QQ_SD_Unit_Test_Coverage.md) | Unit and integration test coverage metrics, test strategy, coverage gaps, and CI reporting configuration |

### UI Architecture

| Document | Description |
|---|---|
| [QQ_SD_UIArchitecture.md](QQ_SD_UIArchitecture.md) | Overall UI architecture: Django Admin (Grappelli) as the primary UI, workspace selector, template-based forms, and REST API consumer patterns |
| [QQ_SD_UIIdentification.md](QQ_SD_UIIdentification.md) | UI framework detection results, project boundaries, and mapping of folder structure to UI abstractions |
| [QQ_UI_Doc_CoreAdminScreens.md](QQ_UI_Doc_CoreAdminScreens.md) | Documentation of the core admin screens: workspace management, role assignments, PDF export status, and workspace-selector widget |
| [QQ_UI_Doc_DomainAdminScreens.md](QQ_UI_Doc_DomainAdminScreens.md) | Documentation of the domain admin screens: contacts, contracts, products, accounting, reporting, subscriptions, and user extension screens |
