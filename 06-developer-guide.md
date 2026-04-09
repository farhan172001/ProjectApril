# CMWS Developer Setup and Conventions Guide

> **Audience:** External contractors onboarding to the CMWS codebase.
> **Last updated:** 2026-04-06
> **Branch:** `GI_Sprint_Rel_06.12.25_DEV_Helix_to_EDM`

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Project Structure Walkthrough](#2-project-structure-walkthrough)
3. [Build System](#3-build-system)
4. [Importing into the IDE](#4-importing-into-the-ide)
5. [Environment Configuration](#5-environment-configuration)
6. [Coding Conventions](#6-coding-conventions)
7. [Testing Approach](#7-testing-approach)
8. [Common Development Tasks](#8-common-development-tasks)
9. [Git Branching Conventions](#9-git-branching-conventions)
10. [Dependency Management](#10-dependency-management)
11. [Troubleshooting](#11-troubleshooting)
12. [Key Files to Know](#12-key-files-to-know)

---

## 1. Prerequisites

### Required Software

| Tool | Version | Notes |
|------|---------|-------|
| **JDK** | 1.7 (source/target in build.xml), JRE 1.6 in .classpath | Build targets Java 7; the Eclipse project references JavaSE-1.6 and WebSphere 7 JRE. Install **JDK 7** at minimum. |
| **IBM WebSphere Application Server** | v7.0 (or later compatible) | The .classpath references `com.ibm.ws.ast.st.runtime.runtimeTarget.v70/was.base.v7`. A local WAS install or WAS Liberty profile is needed to run/debug. |
| **IBM Rational Application Developer (RAD)** | 7.x or 8.x (Eclipse-based) | The `.project` files use IBM-specific natures (`com.ibm.wtp.web.WebNature`, `com.ibm.etools.struts.StrutsNature`). Standard Eclipse WTP can work but RAD is the intended IDE. |
| **Apache Ant** | 1.9+ | Build orchestration. Bundled Ivy (`apache-ivy-2.5.2`) handles dependency resolution. |
| **Apache Ivy** | 2.5.2 | Included in `CMWSWeb/build/apache-ivy-2.5.2/`. No separate install needed. |
| **Documentum (DCTM) Client Libraries** | DFC 7.2/7.3 | `dfc.jar`, `dfcbase.jar`, `dctm.jar` in `WEB-INF/lib/`. These are legacy libraries from the Helix/Documentum era. Document storage now goes through EDM (REST API) to Filenet. Some legacy code paths may still reference these. |
| **Oracle Database Client** | 11g+ | `ojdbc6.jar` is the JDBC driver. Stored procedures use Oracle PL/SQL with `RAISE_ERROR`. |
| **Git** | 2.x+ | Source control. |

### Network/Access Requirements

- **Artifactory access:** Ivy resolves dependencies from `us-artifactory.prudential.com` (jcenter-remote-cache, pru-ext-release-local, libs-release, libs-snapshot). You need network access or a VPN to this host.
- **EDM REST API:** Document storage endpoint config in `WebContent/config/edm_*.xml` files per environment. This is the current document middleware (EDM -> Filenet). Note: GIAL also uses EDM independently for its batch ingestion pipeline.
- **Documentum Content Server (legacy):** Connection config in `WebContent/config/dmcl.ini.*` files per environment. Being phased out as part of the Helix-to-EDM migration.
- **Oracle database:** JNDI datasources defined in `web.xml` (`jdbc/conn2lcms`, `jdbc/meta2dcms`, `jdbc/conn2linx`, `jdbc/wf_common_oracle_database1`, `jdbc/wf_common_oracle_database2`, `jdbc/meta_common_oracle_database1`, `jdbc/meta_common_oracle_database2`).
- **WebSEAL / Tivoli Access Manager (TAM):** The application reads the `iv-user` HTTP header for authentication. In development mode, the `dev_user` context parameter in `web.xml` is used as a fallback (currently `X173998`).

---

## 2. Project Structure Walkthrough

```
Project_Workspace/
  CMWSWeb/           -- Main web application (WAR) - the core of CMWS
  GIAL/              -- Separate web application (WAR) - batch ingestion pipeline (image/document loading into repositories)
  CMWSEAR/           -- EAR packager only - no application code; its build.xml is the master orchestrator
  CMWSDeploymentTest/ -- Selenium smoke tests (TestNG + Selenium WebDriver, run post-deploy)
  CMWSReports/       -- 347 Crystal Reports (.rpt files), no Java code; deployed separately to BI server
  ImageViewer/       -- 253-file Java applet project for TIFF image viewing (signed, packed JARs)
```

### CMWSWeb (the main application)

```
CMWSWeb/
  src/main/java/com/pru/gi/
    workflow/               -- Core workflow engine
      common/               -- Shared utilities (CMWSUtil, ServiceContext, ResultDataRow, WorkflowException)
      config/               -- Configuration loading (ConfigurationUtility, WorkflowInstance, ServiceEngine, Repository)
      data/                 -- Data access layer (OracleDataAccess, AuthorizationContext, TivoliAccessManagerAccess)
      edm/                  -- EDM (Enterprise Document Management) integration classes -- the CURRENT document middleware (EdmService, EdmServiceImpl, EdmTransformer, EdmTransformationService)
      facade/               -- SOAP web service facades (FacadeBase, WorkflowServiceFacade, MetadataServiceFacade, etc.)
      logging/              -- Logging utilities (LoggingUtility)
      metadata/             -- Metadata search and handling
      process/              -- Business process layer (CMWSWorkflowProcess, CMWSStorageProcess, DocumentumRepositoryProcess, etc.)
      queues/               -- Queue/work item management (QueueManager, WorkflowQueue, WorkItem, InboxManager)
      servlets/             -- HTTP servlets and filters (WEBSEALFilter, CharsetFilter, GetDocument, PutDocument, etc.)
      utility/              -- General utilities
      web/
        actions/            -- Struts Action classes (AbstractCMWSAction + subclasses per feature)
          admin/            -- User administration actions
          caselisting/      -- Case listing CRUD
          cob/              -- Client On-Boarding actions
          epr/              -- EPR (Enrollment Processing) actions
          ipr/              -- IPR (Individual Processing) actions
          lcms/             -- LCMS (Life Claims) actions
          lcnv/             -- Life Conversions actions
          osgli/            -- OSGLI actions
          recordsadmin/     -- Records management
          reports/          -- Reporting actions
          ...
        actionforms/        -- Struts ActionForm beans (e.g., WorkTypeRequestForm, EASRequestForm)
    lockbox/                -- Lockbox domain
    lrk/                    -- LPM/Record Keeping domain
    osgli/                  -- OSGLI domain
    lcms/                   -- Life Claims domain
    cob/                    -- Client On-Boarding domain
    epr/                    -- Enrollment Processing domain
    esc/                    -- Enrollment Support Center domain
    gulgvul/                -- GUL/GVUL domain
    lcnv/                   -- Life Conversions domain
    walmart/                -- Walmart domain
    mu/                     -- Medical Underwriting domain
    wlms/                   -- WLMS client
    verizon/                -- Verizon domain
    smcob/                  -- Small Market COB domain

  src/test/java/            -- Unit tests (JUnit 4 + Suite pattern)
  WebContent/
    WEB-INF/
      web.xml               -- Servlet/filter/JNDI definitions
      struts-config.xml     -- Struts action mappings
      lib/                  -- Runtime JAR dependencies (~70+ JARs)
      classes/              -- Compiled output
    Workflow/               -- JSP pages organized by feature
      admin/                -- Admin JSPs
      caselist/             -- Case listing JSPs
      cob/                  -- COB JSPs
      epr/                  -- EPR JSPs
      cmwsmain.jsp          -- Main entry point / welcome page
      queues.jsp            -- Queue view
      results.jsp           -- Results display
      workflows.jsp         -- Workflow selection
      ...
    config/                 -- Runtime configuration (Spring XML beans, environment-specific files)
    META-INF/               -- MANIFEST.MF
    web/                    -- Static web resources, WSDL files, jarfiles for applet
  build/                    -- Ant build files, Ivy descriptors
  lib/                      -- Additional compile-time JARs (DB drivers, legacy Documentum libs)
  lib_test/                 -- Test-time JARs (JUnit, TestNG, Selenium, JaCoCo)
  tiff_lib/                 -- TIFF processing libraries
```

### GIAL (GI Abstraction Layer -- Batch Ingestion Pipeline)

A separate web module with its own WAR, responsible for batch ingestion of images and documents into document repositories (EDM/Filenet). **GIAL is not called by CMWSWeb at runtime** -- it runs its own scheduled batch loading processes. Package: `com.pru.gial.*` with subpackages: `common`, `config`, `data`, `ImageLoader`, `logging`, `metadata`.

GIAL has its own independent configuration:
- `GIAL_config_*.xml` -- GIAL-specific application config per environment
- `GIAL_repositories_*.xml` -- GIAL repository connection config per environment
- Its own `ConfigurationUtility` instance (separate from CMWSWeb's)
- Uses **PDFBox** for PDF-to-TIFF conversion during ingestion

### CMWSEAR (Enterprise Application -- EAR Packager)

Contains **no application code** -- only the master build orchestrator. Its `build.xml` is the single entry point that imports all sub-project build files and assembles the final `cmws.ear` containing:
- `CMWS.war` (from CMWSWeb)
- `gial.war` (from GIAL)
- Shared libraries and config zips

### ImageViewer (253 files, own build)

A signed Java applet for viewing TIFF images in the browser. Uses **JAI (Java Advanced Imaging)** for TIFF rendering. The build compiles the applet, packs JARs (`.jar.pack.gz` format), and signs them with a keystore (`cmws-keystore-entrust-*.jks`). The packed/signed JARs are then copied into CMWSWeb's `WebContent/web/jarfiles/` directory.

Key subdirectory: `imaging/extension/metadata/` contains domain-specific metadata editors.

### CMWSDeploymentTest (Selenium Smoke Tests)

Selenium smoke tests using TestNG and Selenium WebDriver. Tests are executed against a running CMWS instance post-deployment (not during the standard build). Output: `cmws_deployment_tests.zip`.

---

## 3. Build System

### Overview

The project uses **Apache Ant** with **Apache Ivy** for dependency resolution. There is **no Maven or Gradle**. The master build file is:

```
Project_Workspace/CMWSEAR/build/build.xml   (default target: CMWS-Run-Build)
```

This imports the build files from all sub-projects:

```
CMWSWeb/build/build.xml      (target: CMWSWeb-Run-Build)
GIAL/build/build.xml         (target: GIAL-Run-Build)
ImageViewer/build/build.xml  (target: ImageViewer-Run-Build)
CMWSDeploymentTest/build/build.xml
```

### Build Sequence (CMWS-Run-Build)

The build order matters because of artifact dependencies between sub-projects:

```
1. Clean        -- Delete prior build artifacts
2. Init         -- Create output directories
3. ImageViewer-Run-Build:          ← FIRST (its JARs feed into CMWSWeb)
   a. Clean, Init
   b. Ivy resolve (imageviewer libs)
   c. Compile ImageViewer
   d. JUnit tests (with JaCoCo coverage)
   e. Build ImageViewer.jar
   f. Pack JARs (.jar.pack.gz) and sign with keystore
   g. Copy packed/signed JARs into CMWSWeb/WebContent/web/jarfiles/
4. GIAL-Run-Build:                 ← SECOND
   a. Clean, Init
   b. Ivy resolve (GIAL libs)
   c. Compile GIAL
   d. Build gial.war
   e. Create config/driver zips
5. CMWSWeb-Run-Build:              ← THIRD (depends on ImageViewer JARs)
   a. Clean, Init
   b. Ivy resolve (web, web_test, lib, web_jarfiles, tiff_lib)
   c. Compile CMWSWeb (main + test)
   d. JUnit tests (with JaCoCo coverage)
   e. Build CMWS.war (includes ImageViewer packed JARs in web/jarfiles/)
   f. Create config, web, drivers, sharedlib zips
6. CMWSDeploymentTest              ← Build deployment test ZIP
7. Assemble EAR:
   a. Package cmws.ear from CMWS.war + gial.war + META-INF + properties
   b. Create driver zips
```

### Running the Build

```bash
# From the CMWSEAR/build directory:
cd Project_Workspace/CMWSEAR/build
ant

# Or to build just CMWSWeb:
cd Project_Workspace/CMWSWeb/build
ant

# To run just tests with coverage:
cd Project_Workspace/CMWSWeb/build
ant "Junit Test"
```

### Build Outputs

| Artifact | Location | Description |
|----------|----------|-------------|
| `cmws.ear` | `CMWSEAR/bin/cmws.ear` | Deployable EAR for WebSphere (contains CMWS.war + gial.war) |
| `CMWS.war` | `CMWSWeb/bin/CMWS.war` | Web application |
| `gial.war` | `GIAL/bin/gial.war` | GIAL batch ingestion web module |
| `cmws_deployment_tests.zip` | `CMWSDeploymentTest/bin/` | Selenium smoke test package |
| `drivers.zip` | `CMWSWeb/bin/drivers.zip` | JDBC drivers for WAS (ojdbc6/7, sqljdbc4, db2jcc) |
| `config.zip` | `CMWSWeb/bin/config.zip` | Runtime config files for WAS |
| `sharedlib.zip` | `CMWSWeb/bin/sharedlib.zip` | Shared libraries (legacy Documentum DFC, log4j) |
| `web.zip` | `CMWSWeb/bin/web.zip` | Static web resources |
| JaCoCo reports | `CMWSWeb/gen/jacoco/` | Code coverage HTML reports |

### Ivy Dependency Files

Located in `CMWSWeb/build/`:

| File | Purpose |
|------|---------|
| `ivy_web.xml` | Runtime dependencies -> `WEB-INF/lib/` |
| `ivy_web_test.xml` | Test dependencies -> `lib_test/` |
| `ivy_lib.xml` | Compile-time libs -> `lib/` |
| `ivy_web_jarfiles.xml` | Applet JARs -> `WebContent/web/jarfiles/` |
| `ivysettings.xml` | Artifactory resolver config |

### What Ivy Resolves

Ivy resolves the following key dependencies from Artifactory (not just from checked-in `lib/` JARs):

- **JDBC drivers:** `ojdbc6.jar`, `ojdbc7.jar`, `sqljdbc4.jar`, `db2jcc.jar`
- **WebSphere runtime stubs:** Compile-time stubs for WAS APIs
- **Logging:** `commons-logging.jar`, `log4j-*.jar`
- **Apache SOAP:** SOAP client libraries for web service facades
- **ImageViewer deps:** JAI (Java Advanced Imaging) libraries for TIFF rendering

> **Important:** Dependencies are managed by Ivy, not just by JARs checked into `lib/`. Always check the Ivy files before manually adding JARs.

---

## 4. Importing into the IDE

### Eclipse/RAD Import

1. **File > Import > General > Existing Projects into Workspace**
2. Set root directory to `Project_Workspace/`
3. Eclipse should detect these projects (each has a `.project` file):
   - `CMWSWeb`
   - `GIAL`
   - `CMWSEAR`
   - `CMWSDeploymentTest`
   - `ImageViewer`
4. Import all of them.

### Fixing Classpath Issues

The `.classpath` files contain hard-coded paths that you will need to update:

**CMWSWeb/.classpath** references:
- `C:/Users/x258250/Downloads/HelixGIWrapper-3.0.8 (1).jar` -- Replace with your local path to this JAR
- `C:/Program Files (x86)/IBM/SDP/runtimes/base_v7/plugins/com.ibm.ws.jpa.jar` -- Update to your IBM RAD/WAS install
- `C:/Program Files (x86)/IBM/SDP/runtimes/base_v7/plugins/com.ibm.ws.runtime.jar` -- Same

**Required classpath containers:**
- `org.eclipse.jst.j2ee.internal.web.container` -- J2EE Web module container
- `org.eclipse.jst.j2ee.internal.module.container` -- J2EE module container
- `org.eclipse.jst.server.core.container/com.ibm.ws.ast.st.runtime.runtimeTarget.v70/was.base.v7` -- WAS v7 server runtime

If you do not have RAD, you need a WebSphere runtime installed and configured in Eclipse WTP.

### First Build After Import

1. Run `ant` from `CMWSWeb/build/` to resolve Ivy dependencies (populates `WEB-INF/lib/`, `lib/`, `lib_test/`)
2. Refresh projects in Eclipse (F5)
3. Clean and rebuild all projects

### Source Directories

- Main source: `src/main/java` (output: `WebContent/WEB-INF/classes`)
- Test source: `src/test/java` (output: `gen/test-classes`)

---

## 5. Environment Configuration

### Environment Detection

The application detects its environment via a single JVM property:

```
-Dcom.pru.AppServerEnv=DEV|QA|STAGE|PROD
```

This property is set on the WebSphere JVM and read by `ConfigurationUtility` to load the correct environment-specific config files. Both CMWSWeb and GIAL have their own `ConfigurationUtility` instance.

**Hot-reload:** `ConfigurationUtility` hot-reloads configuration files every **3 seconds**, so config changes take effect without a restart.

### Configuration Architecture

The application uses **Spring XML bean definitions** (not Spring MVC -- just Spring IoC for configuration). Key config files are in `WebContent/config/`:

```
config/
  workflows.xml          -- Defines all workflow instances (IDs, names, engine references)
  engines.xml            -- Imports per-workflow engine configurations
  engines/               -- Individual engine XMLs (engines_lpm.xml, engines_osgli.xml, etc.)
  repositories.xml       -- Legacy document repository definitions (contain Helix hlxrtvapi URLs being migrated away)
  loaders.xml            -- Image loader configurations
  queueviews.xml         -- Queue view definitions
  businessunits.xml      -- Business unit configuration
  emailmessages.xml      -- Email templates
  mail.xml               -- Mail server settings
  log4j.properties       -- Logging configuration
  edm_*.xml              -- EDM (Enterprise Document Management) REST API endpoints -- the current document middleware (EDM -> Filenet)
```

### Environment-Specific Files

The application supports 8 environments. Each has its own config files:

| Environment | Config Suffix | Description |
|-------------|---------------|-------------|
| DEV | `_DEV.xml` | Development |
| DEV2 | `_DEV2.xml` | Secondary development |
| QA | `_QA.xml` | Quality Assurance |
| QA2 | `_QA2.xml` | Secondary QA |
| STAGE | `_STAGE.xml` | Staging/Pre-production |
| STAGE2 | `_STAGE2.xml` | Secondary staging |
| INT | `_INT.xml` | Integration |
| PROD | `_PROD.xml` | Production |

Files that have environment variants:
- `engines_*.xml` (database connections, service URLs)
- `repositories_*.xml` (legacy Helix repository connections with `hlxrtvapi` URLs -- being migrated to EDM)
- `loaders_*.xml` (image loader settings)
- `edm_*.xml` (EDM REST API endpoints and credentials -- the current document middleware)
- `GIAL_config_*.xml` (GIAL-specific config)
- `GIAL_repositories_*.xml`
- `dmcl.ini.*` (legacy Documentum client config: `dmcl.ini.dev`, `dmcl.ini.qa`, `dmcl.ini.stage`, `dmcl.ini.prod` -- being phased out)

### Switching Environments

To target a specific environment:

1. Copy the environment-specific file over the base file. For example, to use DEV:
   ```
   cp engines_DEV.xml engines.xml
   cp repositories_DEV.xml repositories.xml
   cp loaders_DEV.xml loaders.xml
   cp edm_DEV.xml edm.xml
   cp dmcl.ini.dev dmcl.ini
   ```
2. Or, if using a ZooKeeper-based approach (indicated by `*-rename-after-zookeeper-is-ready.tmpl` files), the config would be externalized.

### JNDI Datasources

Defined in `web.xml` and must be configured in WebSphere:

| JNDI Name | Purpose |
|-----------|---------|
| `jdbc/conn2lcms` | LCMS workflow database |
| `jdbc/meta2dcms` | DCMS metadata database |
| `jdbc/conn2linx` | LINX connection |
| `jdbc/wf_common_oracle_database1` | Primary workflow Oracle DB |
| `jdbc/wf_common_oracle_database2` | Secondary workflow Oracle DB |
| `jdbc/meta_common_oracle_database1` | Primary metadata Oracle DB |
| `jdbc/meta_common_oracle_database2` | Secondary metadata Oracle DB |

### Development User Override

In `web.xml`, the `dev_user` context parameter (`X173998`) is used when no WebSEAL `iv-user` header is present. Change this to your own user ID for local development:

```xml
<context-param>
    <param-name>dev_user</param-name>
    <param-value>YOUR_USER_ID</param-value>
</context-param>
```

---

## 6. Coding Conventions

### Package Naming

- **Root package:** `com.pru.gi`
- **Core workflow:** `com.pru.gi.workflow.*`
- **Domain-specific:** `com.pru.gi.<domain>` where domain is one of: `lockbox`, `lrk`, `osgli`, `lcms`, `cob`, `epr`, `esc`, `gulgvul`, `lcnv`, `walmart`, `mu`, `wlms`, `verizon`, `smcob`
- **GIAL:** `com.pru.gial.*`

### Class Naming Patterns

| Pattern | Example | Description |
|---------|---------|-------------|
| `*Action.java` | `QueueViewAction`, `ApproveAction`, `SearchAction` | Struts Action classes (UI layer) |
| `Abstract*Action.java` | `AbstractCMWSAction` | Base Struts Action with shared setup |
| `*ActionHelper.java` | `CMWSActionHelper`, `EASActionHelper` | Helper/utility classes for Actions |
| `*Form.java` | `WorkTypeRequestForm`, `EASRequestForm` | Struts ActionForm beans |
| `*Facade.java` | `WorkflowServiceFacade`, `MetadataServiceFacade` | SOAP web service endpoints (extend `FacadeBase`) |
| `*Facade_SEI.java` | `WorkflowServiceFacade_SEI` | Service Endpoint Interface for SOAP |
| `*Process.java` | `CMWSWorkflowProcess`, `DocumentumRepositoryProcess` (legacy name) | Business process/logic layer |
| `I*Process.java` | `IProcessCaseManagement` | Process interfaces |
| `*DataAccess.java` | `OracleDataAccess` | Database access layer |
| `*Manager.java` | `QueueManager`, `InboxManager` | Singleton/cache managers |
| `*Helper.java` | `WorkflowHelper`, `EmailHelper`, `TransferHelper` | Utility/helper classes |
| `*Utility.java` | `ConfigurationUtility`, `LoggingUtility` | Static utility classes |
| `*Constants.java` | `NotificationEventConstants` | Constants classes |
| `*Cache.java` | `WorkItemCache`, `UserInboxCache` | In-memory cache classes |

### Architectural Layers

```
JSP (view) --> Struts Action --> Facade --> Process --> DataAccess --> Oracle DB
                                   |                      |
                                   v                      v
                            Spring XML config      JNDI DataSource
                                   |
                                   v
                            ConfigurationUtility
```

1. **View (JSP):** Pages in `WebContent/Workflow/`. URL pattern `*.go` maps to Struts actions.
2. **Controller (Struts Actions):** Extend `AbstractCMWSAction`. Override `executeCMWSAction()`.
3. **Facade (SOAP Services):** Extend `FacadeBase` (which implements `ServiceLifecycle`). Exposed as JAX-RPC web services via IBM WebSphere `WebServicesServlet`.
4. **Process:** Business logic layer. Classes like `CMWSWorkflowProcess` orchestrate operations.
5. **Data:** `OracleDataAccess` executes stored procedures and SQL against Oracle via JNDI connections.

### Logging

- **Framework:** Apache Commons Logging (`org.apache.commons.logging.Log`) backed by Log4j 1.x.
- **Pattern:** Each class declares a static logger:
  ```java
  private static Log log = LogFactory.getLog(MyClass.class);
  ```
- **Levels used:** `log.info()`, `log.debug()`, `log.error()`
- **Guard clauses:** Used inconsistently. Best practice observed:
  ```java
  if (log.isDebugEnabled()) {
      log.debug("WEBSEAL USER=" + websealUser);
  }
  ```
- **Anti-pattern present:** Many places use `System.out.println(e)` instead of proper logging. When adding new code, always use the logger.
- **Config:** `WebContent/config/log4j.properties` -- root level is DEBUG, console appender.

### Exception Handling

- **Custom exception:** `com.pru.gi.workflow.common.WorkflowException` (unchecked/runtime exception).
- **Common pattern:** Catch `Exception`, add to an `errorList` ArrayList, set as request attribute:
  ```java
  try {
      // business logic
  } catch (Exception e) {
      System.out.println(e);  // legacy pattern -- use log.error() instead
      errorList.add(e.getMessage());
  } finally {
      request.setAttribute("errorList", errorList);
  }
  ```
- **Facade pattern:** `FacadeBase.destroy()` logs elapsed time. `init()` throws `WorkflowException` for unknown context types.

### Authentication and Authorization

- **Authentication:** WebSEAL reverse proxy sets the `iv-user` HTTP header. The `WEBSEALFilter` servlet filter extracts it and stores it in a `ThreadLocal` on `FacadeBase`. In development, falls back to `dev_user` from `web.xml`.
- **Authorization:** `AbstractCMWSAction.setPrivledges()` calls `wfFacade.getUserPrivileges()` which returns an array of privilege codes. These are set as request attributes (e.g., `canAssign`, `canApprove`, `canSeeAdmin`). Privilege codes include: `CMWSADMIN`, `ADMINUPDT`, `ASGN`, `APRV`, `UNRESERVE`, `UNASGN`, `RPTACCESS`, `RCRDSADMIN`, `RCRDSUPDT`, `SETPRIORTY`, `DASHBOARD`, `SLAREAD`, `SLAUPDT`, `WTREQ`, `WTUPDATE`, `EDITCHKLST`, `CLISTREAD`, `CLISTUPDT`, `NJEAPRIV`, `ASGNCLM`, `QRSAMPUPD`, `UPDTBRCHR`, `MANUALACTV`, `PRORATUPD`.
- **Important:** Privileges are passed between requests via hidden form fields (the `arePrivsSet` / `copyParamsAsAttributes` pattern). This is a core architectural characteristic.

### Threading / Concurrency

- `QueueManager` uses a static `HashMap` with `synchronized` blocks and a mutex object for thread-safe queue caching.
- `WorkflowHelper` uses `ThreadLocal` for the current workflow ID.
- `FacadeBase` uses `ThreadLocal` for the WebSEAL user.

### Constants Style

Constants are defined as `public static` (not always `final`) fields:
```java
public static String EVENT_STATUS_CODE_SUCCESS = "01";
public static final int LPM_WORKFLOW_ID = 1;
```

Workflow IDs are integer constants (1=LPM, 2=Lockbox, 3=OSGLI, 7=Walmart, 8=LCNV, 9=GUL/GVUL, 10=ESC, 11=COB, 13=Verizon, 14=ECM, 101=Medical Underwriting, 102=LCMS-GLCD, 103=Disability Claims, 104=LCMS-OSGLI, 105=LCMS-PEB, 107=LCMS-ILI, 15=OSGLI Archive).

### Code Style Details

- **Indentation:** Tabs (4-space equivalent)
- **Braces:** Opening brace on same line for methods, next line for class declarations (inconsistent)
- **Generics:** Raw types used throughout (e.g., `ArrayList` not `ArrayList<String>`) -- Java 1.4 heritage
- **String comparison:** `equals()` method used (correct); some places use `equalsIgnoreCase()`
- **Null checks:** Manual null checks, no Optional (pre-Java 8 codebase)
- **PVCS/revision history:** Many files have `$Log:` headers with full PVCS revision history in comments
- **Encoding:** Source files use `iso-8859-1` encoding (specified in build.xml)
- **Comments:** Inline comments reference ticket numbers (e.g., `//TT17510`, `//Wavier of Premium Project update - 08.25.2016`)

---

## 7. Testing Approach

### Unit Tests (JUnit 4)

- **Location:** `CMWSWeb/src/test/java/`
- **Master suite:** `com.pru.gi.workflow.TestAll` -- uses `@RunWith(Suite.class)` to aggregate tests
- **Sub-suites:** `com.pru.gi.workflow.facade.AllTests`
- **Coverage:** JaCoCo generates HTML reports in `gen/jacoco/`
- **Execution:** Part of the Ant build (`Junit Test` target)
- **Test pattern:** Suite-based aggregation. Individual test classes follow `*Test.java` naming (e.g., `CMWSUtilTest`, `GulGvulConstantsTest`, `LCMSConstantsTest`, `OsgliConstantsTest`).

### Integration/Deployment Tests (TestNG + Selenium)

- **Location:** `CMWSDeploymentTest/`
- **Config:** `testng.xml` defines test suites
- **Example test class:** `com.pru.gi.cmws.test.deployment.TestECMWorkflow`
- **Dependencies:** Selenium WebDriver, HtmlUnit, PhantomJS driver (in `lib_test/`)
- **Execution:** NOT part of the standard build. Run separately against a deployed instance.
- **Thread count:** Single-threaded (`thread-count="1"`)

### Test Coverage

Coverage is limited. Most tests cover:
- Constants and utility classes (e.g., `CMWSUtilTest`)
- Domain-specific constants (e.g., `OsgliConstantsTest`)
- Facade layer (via `AllTests` suite)

Large portions of the codebase (Actions, Process layer, DataAccess) have minimal or no unit test coverage.

---

## 8. Common Development Tasks

### Adding a New Batch Loader (GIAL)

GIAL handles batch ingestion of documents. To add a new loader:

1. **Create Loader class** in `GIAL/src/main/java/com/pru/gial/ImageLoader/` implementing the loader interface
2. **Register in `loaders.xml`:** Add a new `<bean>` definition for your loader in `WebContent/config/loaders.xml`
3. **Add environment variants:** Create/update `loaders_DEV.xml`, `loaders_QA.xml`, etc. with environment-specific settings
4. GIAL runs its own scheduled processes -- your loader will be picked up by the batch framework

### Adding a Domain to ImageViewer

To add a new metadata domain to the ImageViewer applet:

1. **Create metadata editor** in `ImageViewer/imaging/extension/metadata/` with a new domain package
2. **Register the domain** in the ImageViewer configuration
3. **Rebuild ImageViewer** -- the build will pack (`.jar.pack.gz`) and sign the JARs, then copy them into CMWSWeb's `WebContent/web/jarfiles/`
4. **Rebuild CMWSWeb** to pick up the updated applet JARs

### Adding a New Business Domain

1. **Create package:** `com.pru.gi.<newdomain>/` under `CMWSWeb/src/main/java/`
2. **Create Facade:** `<NewDomain>Facade.java` extending `FacadeBase` + a `<NewDomain>Facade_SEI.java` interface
3. **Register in web.xml:**
   ```xml
   <servlet>
       <servlet-name>com_pru_gi_newdomain_NewDomainFacade</servlet-name>
       <servlet-class>com.pru.gi.newdomain.NewDomainFacade</servlet-class>
       <load-on-startup>1</load-on-startup>
   </servlet>
   <servlet-mapping>
       <servlet-name>com_pru_gi_newdomain_NewDomainFacade</servlet-name>
       <url-pattern>services/NewDomainFacade</url-pattern>
   </servlet-mapping>
   ```
4. **Add WebSEAL filter mapping** in `web.xml`
5. **Add workflow instance** in `WebContent/config/workflows.xml` with a new workflow ID
6. **Add engine configs** in `WebContent/config/engines/engines_newdomain.xml` and import in `engines.xml`
7. **Update `AbstractCMWSAction.setWorkflowName()`** to include the new workflow ID -> name mapping

### Adding a New Struts Action

1. **Create Action class** extending `AbstractCMWSAction`:
   ```java
   public class MyAction extends AbstractCMWSAction {
       public ActionForward executeCMWSAction(ServiceContext context,
               ActionMapping mapping, ActionForm form,
               HttpServletRequest request, HttpServletResponse response,
               ActionErrors errors) throws Exception {
           // Your logic here
           return mapping.findForward("success");
       }
   }
   ```
2. **Register in `struts-config.xml`:**
   ```xml
   <action path="/mydomain/myaction"
           type="com.pru.gi.workflow.web.actions.mydomain.MyAction">
       <forward name="success" path="/Workflow/mydomain/mypage.jsp" />
   </action>
   ```
3. **Create the JSP** at `WebContent/Workflow/mydomain/mypage.jsp`
4. **URL pattern:** The action is accessible at `/mydomain/myaction.go` (the `*.go` pattern maps to Struts)

### Adding a New JSP Page

1. Create the JSP file under `WebContent/Workflow/<feature>/`
2. Reference it from a Struts action forward in `struts-config.xml`
3. The JSP can access request attributes set by the Action class
4. Standard JSP tag libraries: Struts HTML, Bean, Logic tags

### Adding a SOAP Web Service

1. **Create Facade class** extending `FacadeBase`:
   ```java
   public class MyFacade extends FacadeBase {
       public String myOperation(String param) {
           ServiceContext context = new ServiceContext();
           setContextUser(context);
           // Business logic
           return result;
       }
   }
   ```
2. **Create SEI interface** (`MyFacade_SEI.java`) defining the service contract
3. **Register servlet** in `web.xml` (see "Adding a New Business Domain" above)
4. **WSDL** is typically generated by WebSphere or placed in `WebContent/web/wsdl/`

### Adding a Queue Handler/Extension

1. **Create class** in `com.pru.gi.workflow.queues` implementing `IWorkflowExtensionStep` or `ICalculateDisplayColumn`
2. **Register** in the queue view configuration (`WebContent/config/queueviews.xml`)
3. For calculated columns, extend the display column pattern:
   ```java
   public class CalculateMyColumn implements ICalculateDisplayColumn {
       public DisplayColumn calculate(WorkItem item) {
           // calculation logic
       }
   }
   ```

### Adding Environment Configuration

For each environment-specific setting:
1. Add the property to the base config file (e.g., `engines.xml`)
2. Create/update environment variants (`engines_DEV.xml`, `engines_QA.xml`, etc.)
3. For EDM endpoints, update `edm_<ENV>.xml` files

---

## 9. Git Branching Conventions

### Branch Naming Pattern

```
GI_Sprint_Rel_<MM.DD.YY>_<ENV>_<Feature_Description>
```

**Example:** `GI_Sprint_Rel_06.12.25_DEV_Helix_to_EDM`

Components:
- **GI_Sprint_Rel** -- Prefix indicating a GI sprint release branch
- **06.12.25** -- Target release date (June 12, 2025)
- **DEV** -- Target environment
- **Helix_to_EDM** -- Feature/initiative description

### Commit Message Pattern

Based on recent commits:
```
[MS-XXXXX] Brief description of the change
```

Where `MS-XXXXX` is the Jira/tracking ticket number.

### Workflow

- Feature branches are created from the sprint release branch
- Commit messages reference ticket numbers
- Multiple commits per feature are common (incremental development)

---

## 10. Dependency Management

### No Maven/Gradle

This project does **not** use Maven or Gradle. Dependencies are managed via:

1. **Ivy** -- Resolves JARs from Artifactory and places them in the correct directories
2. **Checked-in JARs** -- Many JARs are committed directly to `WEB-INF/lib/`
3. **External JARs** -- Some JARs reference external paths (IBM RAD/WAS install)

### Key Runtime Dependencies (WEB-INF/lib/)

| Library | JAR(s) | Purpose |
|---------|--------|---------|
| Struts 1.x | `struts.jar` | MVC framework |
| Spring 2.x | `spring.jar` | IoC container (XML bean config only) |
| Commons | `commons-beanutils.jar`, `commons-collections.jar`, `commons-digester.jar`, `commons-lang.jar`, `commons-io.jar`, `commons-logging.jar`, `commons-fileupload.jar`, `commons-validator.jar` | Apache Commons utilities |
| Documentum DFC (legacy) | `dfc.jar`, `dfcbase.jar`, `dctm.jar` | Legacy document management client (from Helix/Documentum era; EDM/Filenet is the current document path) |
| Oracle JDBC | `ojdbc6.jar` | Database driver |
| Jackson 2.14 | `jackson-core-2.14.1.jar`, `jackson-databind-2.14.1.jar`, `jackson-annotations-2.14.1.jar`, `jackson-dataformat-xml-2.14.1.jar` | JSON/XML processing |
| Crystal Reports | `CrystalReportsSDK.jar`, `jrcadapter.jar`, `jrcerom.jar`, `cereports.jar`, `cesession.jar`, `cecore.jar`, `celib.jar`, etc. | Report generation |
| iText | `itext.jar` (2.0.6) | PDF generation |
| JExcelAPI | `jxl.jar` | Excel file generation |
| JavaMail | `mail.jar`, `activation.jar` | Email sending |
| Log4j | `log4j-1.2-api.jar`, `logging.jar` | Logging |
| ESAPI | `esapi.jar` | OWASP security utilities |
| Derby | `derby.jar` | Embedded DB (possibly for testing) |
| SAP/BO | `boconfig.jar`, `biarengine.jar`, `biplugins.jar`, etc. | Business Objects integration |

### Sub-Project-Specific Dependencies

| Library | Project | Purpose |
|---------|---------|---------|
| **JAI** (Java Advanced Imaging) | ImageViewer | TIFF rendering in the applet |
| **PDFBox** | GIAL | PDF-to-TIFF conversion during batch ingestion |
| **Apache SOAP** | CMWSWeb | SOAP client for web service facades |

### Test Dependencies (lib_test/)

JUnit 4, TestNG, Selenium WebDriver, JaCoCo, Hamcrest, HtmlUnit, Apache POI, CGLib, Mockito-adjacent libraries.

### Adding a New Dependency

1. Add the dependency to the appropriate Ivy file (`ivy_web.xml`, `ivy_web_test.xml`, or `ivy_lib.xml`)
2. Run the Ivy resolve target: `ant resolve_web` (or `resolve_web_test`, `resolve_lib`)
3. Alternatively, manually place the JAR in `WEB-INF/lib/` and add it to `.classpath`
4. If the dependency needs to be on the WebSphere shared library path, add it to the `sharedlib.zip` target in `build.xml`

---

## 11. Troubleshooting

### Common Issues

#### Build fails: "Ivy resolve error"
- **Cause:** Cannot reach `us-artifactory.prudential.com`
- **Fix:** Ensure VPN is connected. Check `ivysettings.xml` credentials.

#### ClassNotFoundException at runtime
- **Cause:** JAR missing from `WEB-INF/lib/` or not on WebSphere shared library
- **Fix:** Check if the JAR should be in `WEB-INF/lib/` (app classloader) or `sharedlib/` (server classloader). Legacy Documentum JARs (`dfc.jar`, `dctm.jar`) and Log4j must be on the shared library.

#### "Unknown init argument type" WorkflowException
- **Cause:** `FacadeBase.init()` received an unexpected context type
- **Fix:** Ensure the facade is being initialized with either a `ServletEndpointContext` (SOAP) or `ServletContext` (servlet).

#### dev_user / authentication issues
- **Cause:** No WebSEAL `iv-user` header in development
- **Fix:** Update the `dev_user` parameter in `web.xml` to your user ID. Ensure your user has appropriate TAM groups.

#### "Unable to determine queue type" error
- **Cause:** Queue ID not found in the database for the current workflow
- **Fix:** Verify the workflow ID and queue ID exist in the Oracle database. Check `workflows.xml` and engine configurations.

#### Eclipse: "Build path references missing container"
- **Cause:** Missing WebSphere runtime in Eclipse/RAD
- **Fix:** Install the WAS runtime adapter, or edit `.classpath` to reference available runtimes.

#### EDM / document storage connection failures
- **Cause:** `edm_*.xml` pointing to wrong environment or incorrect credentials
- **Fix:** Verify the `edm_<ENV>.xml` file has the correct `documentsUrl`, `basicAuth`, and `edmEnv` values for your target environment. The `EdmServiceImpl` class in `com.pru.gi.workflow.edm` handles EDM REST API communication.

#### Legacy Documentum connection failures (if still using legacy code paths)
- **Cause:** `dmcl.ini` pointing to wrong environment
- **Fix:** Copy the correct `dmcl.ini.<env>` to `dmcl.ini`. Note: Documentum/Helix is being phased out in favor of EDM/Filenet.

#### Encoding issues in compiled classes
- **Cause:** Source files use `iso-8859-1` encoding
- **Fix:** Ensure your IDE and build are configured for `iso-8859-1` (not UTF-8). The Ant build explicitly sets `encoding="iso-8859-1"`.

---

## 12. Key Files to Know

### "Start Here" Files

If you are new to this codebase, read these files first, in this order:

| # | File | Why |
|---|------|-----|
| 1 | `CMWSWeb/WebContent/WEB-INF/web.xml` | Entry point. Shows all servlets, filters, JNDI resources, and the Struts config reference. |
| 2 | `CMWSWeb/WebContent/WEB-INF/struts-config.xml` | Maps every URL to its Action class and JSP. This is your roadmap to the UI. |
| 3 | `CMWSWeb/src/main/java/com/pru/gi/workflow/web/actions/AbstractCMWSAction.java` | Base class for all UI actions. Shows auth, privilege, and context initialization patterns. |
| 4 | `CMWSWeb/src/main/java/com/pru/gi/workflow/facade/FacadeBase.java` | Base class for all SOAP facades. Shows lifecycle, logging, and auth patterns. |
| 5 | `CMWSWeb/WebContent/config/workflows.xml` | Defines all workflow instances and their engine mappings. |
| 6 | `CMWSWeb/WebContent/config/engines.xml` | Master engine configuration -- imports all per-domain engine files. |
| 7 | `CMWSWeb/src/main/java/com/pru/gi/workflow/config/ConfigurationUtility.java` | Loads and caches all configuration. Central to understanding how the app initializes. |
| 8 | `CMWSWeb/src/main/java/com/pru/gi/workflow/data/OracleDataAccess.java` | Data access layer. Shows how stored procedures and queries are executed. |
| 9 | `CMWSWeb/src/main/java/com/pru/gi/workflow/queues/QueueManager.java` | Queue caching and management. Central to the workflow processing model. |
| 10 | `CMWSEAR/build/build.xml` | Master build file. Shows how everything fits together. |
| 11 | `CMWSWeb/WebContent/Workflow/cmwsmain.jsp` | Main entry point JSP -- the welcome page users see. |
| 12 | `CMWSWeb/WebContent/config/log4j.properties` | Logging configuration. |

### Domain-Specific Entry Points

Each business domain has a Facade as its primary entry point:

| Domain | Facade | Web Service URL |
|--------|--------|-----------------|
| Workflow (core) | `WorkflowServiceFacade` | `/services/WorkflowServiceFacade` |
| Metadata | `MetadataServiceFacade` | `/services/MetadataServiceFacade` |
| Document Storage | `DocumentStorageFacade` | `/services/DocumentStorageFacade` |
| Letters | `LetterServiceFacade` | `/services/LetterServiceFacade` |
| Reporting | `WorkflowReportingFacade` | `/services/WorkflowReportingFacade` |
| Case Management | `CaseManagementFacade` | `/services/CaseManagementFacade` |
| Workflow Management | `WorkflowManagementFacade` | `/services/WorkflowManagementFacade` |
| Quality Review | `QualityReviewFacade` | `/services/QualityReviewFacade` |
| QTS | `QTSFacade` | `/services/QTSFacade` |
| Notifications | `NotificationEventFacade` | `/services/NotificationEventFacade` |
| Lockbox | `LockboxFacade` (com.pru.gi.lockbox) | `/services/LockboxFacade` |
| LPM | `LPMFacade` (com.pru.gi.lrk) | `/services/LPMFacade` (via GIAL) |
| OSGLI | `OSGLIFacade` (com.pru.gi.osgli) | `/services/OSGLIFacade` |
| LCNV | `LCNVFacade` (com.pru.gi.lcnv) | `/services/LCNVFacade` |
| Walmart | `WalmartFacade` (com.pru.gi.walmart) | `/services/WalmartFacade` |
| GUL/GVUL | `GULGVULFacade` (com.pru.gi.gulgvul) | `/services/GULGVULFacade` |
| ESC | `ESCFacade` (com.pru.gi.esc) | `/services/ESCFacade` |
| COB | `COBWorkTypeItemFacade` (com.pru.gi.cob) | `/services/COBWorkTypeItemFacade` |
| EPR | `EPRWorkTypeItemFacade` (com.pru.gi.epr) | `/services/EPRWorkTypeItemFacade` |
| LCMS | `LCMSFacade` (com.pru.gi.lcms) | `/services/LCMSFacade` |
| MU | `MUFacade` (com.pru.gi.mu) | `/services/MUFacade` |

---

## Appendix: Quick Reference Card

```
Build:          cd Project_Workspace/CMWSEAR/build && ant
Build CMWSWeb:  cd Project_Workspace/CMWSWeb/build && ant
Run tests:      cd Project_Workspace/CMWSWeb/build && ant "Junit Test"
Coverage:       open CMWSWeb/gen/jacoco/index.html

Main package:   com.pru.gi.workflow
URL pattern:    *.go -> Struts Actions
SOAP services:  /services/<FacadeName>
Config dir:     CMWSWeb/WebContent/config/
JSP dir:        CMWSWeb/WebContent/Workflow/
Welcome page:   cmwsmain.jsp

Auth header:    iv-user (WebSEAL)
Dev fallback:   web.xml -> dev_user parameter
Database:       Oracle (JNDI datasources)
Doc storage:    Filenet via EDM REST API (formerly Helix/Documentum)
Env detection:  -Dcom.pru.AppServerEnv=DEV|QA|STAGE|PROD
Config reload:  ConfigurationUtility hot-reloads every 3 seconds
EAR contents:   CMWS.war + gial.war
Logging:        Commons Logging + Log4j 1.x
Build tool:     Ant + Ivy
IDE:            IBM RAD / Eclipse WTP
App server:     WebSphere 7+
Java version:   1.7 (source/target)
```
