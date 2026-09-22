MCP-BASED API DISCOVERY FOR LEGACY MODERNIZATION

# HACKATHON TECHNICAL DESIGN + LOCAL DEMO IMPLEMENTATION GUIDE {#hackathon-technical-design--local-demo-implementation-guide}

## Purpose

This document is intended to be handed to an implementation
agent/developer. It explains the complete idea, production architecture,
local hackathon PoC, technical components, tool contracts, mock
services, Copilot integration, demo flow, production mapping, testing,
and presentation pitch.

The production architecture shown in the architecture diagram is the
target architecture. The hackathon implementation is intentionally
smaller and runs locally with mocked enterprise systems.

## IMPORTANT DESIGN PRINCIPLE

The hackathon does NOT need real Splunk, Confluence, Jira, or Middleware
connections.

Instead:

    Real production system        Local hackathon equivalent
    ---------------------         ---------------------------
    Middleware / API Catalog  ->  Mock Middleware REST service / JSON
    Confluence               ->  Mock Confluence REST service / JSON
    Jira                     ->  Mock Jira REST service / JSON
    Splunk                   ->  Mock Splunk REST service / JSON

The MCP tool interface should remain stable. Only the implementation
behind each tool changes when moving from PoC to production.

# 

# 1. EXECUTIVE IDEA {#1-executive-idea}

A developer working on a legacy application frequently needs to answer:

    "Does an API already exist for this packet / transaction / business use case?"

Today the answer may require:

-   Asking Middleware or another developer.
-   Searching Confluence.
-   Searching Jira.
-   Checking runtime/logging information in Splunk.
-   Verifying whether the API is active and actually used.
-   Comparing several sources before deciding whether to reuse or create
    an API.

The proposed solution introduces an Enterprise MCP Server.

GitHub Copilot remains the developer-facing AI experience.

The MCP Server exposes governed tools such as:

    find_middleware_api
    search_confluence
    search_jira
    search_splunk
    get_api_details
    get_api_usage

Copilot can discover and invoke these tools through MCP.

The MCP server then calls the relevant enterprise APIs/connectors,
combines the evidence, and returns structured information to Copilot.

The developer can therefore ask a natural-language question such as:

    "Check whether packet ABC123 has an existing API and show
     documentation and usage details."

The desired response could be:

    Packet: ABC123
    Existing API: YES
    API: /payments/transaction
    Status: ACTIVE
    Consumers: 14
    Documentation: AVAILABLE
    Recent Usage: ACTIVE
    Related Jira: PROJ-1234

    Conclusion:
    An existing API is available and should be investigated for reuse.

The system should present evidence, not silently make an architectural
decision on behalf of the developer.

#  {#-1}

# 2. PRODUCTION ARCHITECTURE {#2-production-architecture}

TARGET PRODUCTION FLOW

    Developer / RAM
          |
          v
    IntelliJ IDEA
          |
          +---- GitHub Copilot
          |
          v
    Corporate Network
    SSO / VPN / Firewall
          |
          v
    PCF / Tanzu Application Service
          |
          v
    Enterprise MCP Server
    Spring Boot
          |
          +-- MCP Tool Layer
          |
          +-- Orchestration
          |
          +-- Configuration / Connection Management
          |
          +-- Logging / Monitoring
          |
          v
    Integration Layer
    Secure connectors / adapters / approved internal APIs
          |
          +-------------------+------------------+----------------+
          |                   |                  |                |
          v                   v                  v                v
    Middleware/API       Confluence            Jira             Splunk
       Catalog
          |
          v
    Existing enterprise data/services

PRODUCTION SECURITY / PLATFORM CONCERNS

The production system should additionally provide:

-   Authentication.
-   Authorization.
-   Least-privilege access.
-   Secrets management.
-   Audit logging.
-   Monitoring and tracing.
-   High availability.
-   Multiple MCP instances.
-   Load balancing / platform scaling.
-   Configuration management.
-   Rate limiting where appropriate.
-   Tool-level access control.
-   Read-only tools for discovery wherever possible.

The main architecture diagram intentionally keeps these concerns out of
the center of the visual flow so the core idea remains easy to
understand.

#  {#-2}

# 3. WHAT MCP ACTUALLY DOES {#3-what-mcp-actually-does}

MCP is the contract between an AI client and external tools/context.

The MCP server does NOT need to contain another LLM.

The basic separation is:

    GitHub Copilot
         |
         | MCP
         v
    MCP Server
         |
         +--> Tool: find_middleware_api()
         +--> Tool: search_confluence()
         +--> Tool: search_jira()
         +--> Tool: search_splunk()

Each tool is implemented by normal application code.

For example:

    find_middleware_api("ABC123")
             |
             v
    Middleware API / Catalog integration
             |
             v
    API metadata returned

Likewise:

    search_confluence("ABC123")
             |
             v
    Confluence REST API
             |
             v
    Documentation returned

MCP standardizes how Copilot discovers and invokes those capabilities.

#  {#-3}

# 4. LOCAL HACKATHON ARCHITECTURE {#4-local-hackathon-architecture}

For the hackathon, use two main applications/processes.

## APPLICATION 1: CUSTOMER / LEGACY SERVICE PROJECT {#application-1-customer--legacy-service-project}

This is the existing project opened in IntelliJ.

Example:

    customer-service/
      src/
      pom.xml
      ...

GitHub Copilot is available inside IntelliJ.

This project represents the real developer environment.

## APPLICATION 2: MCP SERVER

Create a separate Spring Boot project.

Example:

    enterprise-mcp-server/
      src/main/java/...
      src/main/resources/
      pom.xml

Run it separately.

Conceptually:

## IntelliJ Window 1

    Customer / Legacy Service
    GitHub Copilot
          |
          | MCP
          v
    localhost MCP server

## IntelliJ Window 2

    Enterprise MCP Server
    Spring Boot

IMPORTANT: The customer service application itself does NOT need to
implement MCP.

Copilot is the MCP client in this scenario.

The separate MCP server provides the tools.

#  {#-4}

# 5. LOCAL DEMO OPTIONS {#5-local-demo-options}

## OPTION A: STDIO

For the fastest local PoC, the MCP server can be launched by Copilot
using a command.

Conceptually:

    Copilot
       |
       | starts MCP process
       v
    java -jar enterprise-mcp-server.jar
       |
       +--> tools
       +--> mocks

Advantages:

-   Very simple local setup.
-   No networking configuration.
-   No localhost port dependency.
-   Easy for a hackathon.

The current MCP Java SDK supports STDIO transport.

## OPTION B: STREAMABLE HTTP

For a demo that visually resembles the production architecture more
closely:

    Copilot
       |
       | HTTP MCP connection
       v
    localhost:8081
       |
       v
    Spring Boot MCP Server

The MCP server then calls:

    localhost:9001/mock/middleware
    localhost:9002/mock/confluence
    localhost:9003/mock/jira
    localhost:9004/mock/splunk

This option is useful if you want to demonstrate that the MCP server is
a separate running service.

RECOMMENDATION FOR THIS HACKATHON: Use Streamable HTTP if the team is
comfortable with the transport. Use STDIO if the priority is getting a
reliable demo running quickly.

The MCP Java SDK currently supports STDIO, SSE, and Streamable HTTP
transports. Spring-specific WebFlux/WebMVC MCP transports are provided
through current Spring AI MCP modules, while the core MCP Java SDK can
be used independently.

#  {#-5}

# 6. TECHNOLOGY STACK {#6-technology-stack}

## LOCAL MCP SERVER

Recommended:

-   Java 17+
-   Spring Boot
-   Maven
-   MCP Java SDK
-   Jackson / JSON
-   Spring Web if mock REST endpoints are implemented inside the same
    application.

No Spring AI is required if the implementation uses the core MCP Java
SDK directly.

Spring AI is an optional alternative if the team wants Spring-native MCP
server/client starters and annotations.

## CUSTOMER / LEGACY PROJECT {#customer--legacy-project}

-   Existing Java/Spring Boot project.
-   IntelliJ IDEA.
-   GitHub Copilot.
-   No MCP implementation required inside the legacy application.

## MOCK SYSTEMS

For the hackathon, they can be:

-   JSON files.
-   In-memory Java collections.
-   Mock REST controllers.
-   WireMock.
-   Separate tiny Spring Boot mock services.

For the simplest implementation, use JSON or in-memory data.

## PRODUCTION

-   MCP server deployed to PCF / Tanzu Application Service.
-   Enterprise identity.
-   Secrets management.
-   Approved APIs/connectors.
-   Real Middleware/API Catalog.
-   Real Confluence.
-   Real Jira.
-   Real Splunk.
-   Enterprise observability.

#  {#-6}

# 7. MCP TOOL DESIGN {#7-mcp-tool-design}

Keep the tools small and explicit.

## TOOL 1: find_middleware_api

Purpose: Find whether an API exists for a packet/business identifier.

Input:

    {
      "packetId": "ABC123"
    }

Output:

    {
      "packetId": "ABC123",
      "exists": true,
      "apiName": "Transaction API",
      "endpoint": "/payments/transaction",
      "status": "ACTIVE"
    }

## TOOL 2: get_api_details

Input:

    {
      "apiName": "Transaction API"
    }

Output:

    {
      "apiName": "Transaction API",
      "endpoint": "/payments/transaction",
      "version": "v2",
      "status": "ACTIVE",
      "consumers": 14,
      "owner": "Payments Middleware"
    }

## TOOL 3: search_confluence

Input:

    {
      "query": "ABC123 transaction API"
    }

Output:

    {
      "found": true,
      "documents": [
        {
          "title": "Transaction API Design",
          "summary": "API supporting transaction processing",
          "documentId": "CONF-123"
        }
      ]
    }

## TOOL 4: search_jira

Input:

    {
      "query": "ABC123"
    }

Output:

    {
      "tickets": [
        {
          "key": "PROJ-1234",
          "summary": "Migrate ABC123 transaction flow",
          "status": "Done"
        }
      ]
    }

## TOOL 5: search_splunk

Input:

    {
      "query": "ABC123",
      "timeRange": "30d"
    }

Output:

    {
      "found": true,
      "requestCount": 18432,
      "recentActivity": true,
      "errorRate": 0.4
    }

## TOOL 6: get_api_usage

Input:

    {
      "apiName": "Transaction API"
    }

Output:

    {
      "apiName": "Transaction API",
      "active": true,
      "requestCount30d": 18432,
      "consumerCount": 14
    }

#  {#-7}

# 8. MOCK DATA {#8-mock-data}

Create consistent data across all mocks.

Example packet:

    ABC123

Mock Middleware:

    packetId: ABC123
    apiName: Transaction API
    endpoint: /payments/transaction
    status: ACTIVE
    consumers: 14

Mock Confluence:

    packetId: ABC123
    document: Transaction API Design
    documentId: CONF-123

Mock Jira:

    packetId: ABC123
    ticket: PROJ-1234
    status: Done

Mock Splunk:

    packetId: ABC123
    requests30d: 18432
    recentActivity: true
    errorRate: 0.4

Also create a negative example.

Example:

    XYZ999

Mock Middleware:

    exists: false

Mock Confluence:

    found: false

Mock Jira:

    no related ticket

Mock Splunk:

    no API traffic

This lets the demo show both:

1.  Existing API discovery.
2.  No existing API discovery.

#  {#-8}

# 9. SUGGESTED PROJECT STRUCTURE {#9-suggested-project-structure}

enterprise-mcp-server/ \| +\-- pom.xml \| +\--
src/main/java/com/company/mcp/ \| \| \| +\--
EnterpriseMcpApplication.java \| \| \| +\-- mcp/ \| \| +\--
MiddlewareApiTool.java \| \| +\-- ConfluenceTool.java \| \| +\--
JiraTool.java \| \| +\-- SplunkTool.java \| \| +\-- ApiUsageTool.java \|
\| \| +\-- service/ \| \| +\-- MiddlewareService.java \| \| +\--
ConfluenceService.java \| \| +\-- JiraService.java \| \| +\--
SplunkService.java \| \| \| +\-- mock/ \| \| +\--
MockMiddlewareController.java \| \| +\-- MockConfluenceController.java
\| \| +\-- MockJiraController.java \| \| +\-- MockSplunkController.java
\| \| \| +\-- model/ \| +\-- ApiDetails.java \| +\--
ConfluenceResult.java \| +\-- JiraResult.java \| +\-- SplunkResult.java
\| +\-- src/main/resources/ +\-- application.yml +\-- mock-data/ +\--
middleware.json +\-- confluence.json +\-- jira.json +\-- splunk.json

For a very small hackathon, the mock controllers can be skipped and the
service classes can read JSON directly.

#  {#-9}

# 10. INTERNAL MCP FLOW {#10-internal-mcp-flow}

The important separation is:

    MCP TOOL
       |
       v
    APPLICATION SERVICE
       |
       v
    ENTERPRISE CONNECTOR
       |
       v
    EXTERNAL SYSTEM

Do NOT make the MCP tool contain all business logic.

For example:

    find_middleware_api()
             |
             v
    MiddlewareService
             |
             v
    MiddlewareClient
             |
             v
    Middleware API

Today:

    MiddlewareClient -> Mock Middleware

Production:

    MiddlewareClient -> Real Middleware API

This makes the PoC easy to migrate to production.

#  {#-10}

# 11. COPILOT CONNECTION {#11-copilot-connection}

GitHub Copilot supports MCP servers in its IDE experience.

For JetBrains IDEs, Copilot MCP configuration is available through the
Copilot Chat MCP configuration flow.

Conceptually the local configuration is:

    {
      "servers": {
        "enterprise-mcp": {
          "...": "local MCP server configuration"
        }
      }
    }

For a local process, the configuration can launch the MCP server
command.

For a separately running HTTP MCP server, configure the appropriate
remote/ HTTP MCP connection supported by the installed Copilot/IDE
version.

DO NOT hard-code credentials in the configuration.

For the hackathon:

-   No production credentials.
-   No real enterprise tokens.
-   Only localhost/mock data.

When configured, Copilot can discover the MCP tools and expose them in
its tool interface. GitHub documents the MCP configuration flow for
Copilot Chat and JetBrains IDEs.

#  {#-11}

# 12. WHAT HAPPENS WHEN THE USER TYPES THE PROMPT {#12-what-happens-when-the-user-types-the-prompt}

User enters:

    "Check if packet ABC123 has an existing API.
     Show the documentation and current usage."

Step 1: Copilot understands that this requires external enterprise
information.

Step 2: Copilot sees the available MCP tools.

Step 3: Copilot selects relevant tools, for example:

    find_middleware_api
    search_confluence
    get_api_usage
    search_jira

Step 4: MCP receives the tool calls.

Step 5: Each MCP tool calls its underlying mock service.

Step 6: The MCP server returns structured results.

Step 7: Copilot combines those results into a readable answer.

Example:

    Existing API found: YES

    API:
    Transaction API
    /payments/transaction
    Status: ACTIVE

    Documentation:
    Transaction API Design
    CONF-123

    Usage:
    18,432 requests in last 30 days
    14 consumers

    Jira:
    PROJ-1234 - Done

    Summary:
    An existing active API is available for this use case.
    Review the existing API before creating a new integration.

#  {#-12}

# 13. DEMO SCRIPT {#13-demo-script}

## DEMO SETUP

Open two IntelliJ windows.

Window 1: Customer / Legacy Service Project GitHub Copilot enabled.

Window 2: Enterprise MCP Server Spring Boot application running locally.

Start the MCP server.

Example:

    localhost:8081

Start the mock enterprise services if they are separate.

Example:

    Mock Middleware  -> localhost:9001
    Mock Confluence  -> localhost:9002
    Mock Jira         -> localhost:9003
    Mock Splunk       -> localhost:9004

## DEMO STEP 1

Show the legacy code.

Say:

    "This is our existing legacy/customer service application.
     The developer is trying to understand whether a particular packet
     already has an API."

## DEMO STEP 2

Open GitHub Copilot Agent mode.

Enter:

    "Check whether packet ABC123 has an existing API.
     If it exists, show the API details, documentation and recent usage."

## DEMO STEP 3

Show the MCP tool calls.

The important tools should appear:

    find_middleware_api
    search_confluence
    get_api_usage
    search_jira

Explain:

    "Copilot does not need to know how Middleware or Confluence works.
     MCP exposes those capabilities as standardized tools."

## DEMO STEP 4

Show the mock service logs or console.

For example:

    POST /mock/middleware/search
    packetId=ABC123

    GET /mock/confluence/search?q=ABC123

    GET /mock/splunk/usage?packetId=ABC123

Say:

    "For the hackathon these are mocked enterprise APIs.
     In production these connectors would call the actual approved
     enterprise APIs."

## DEMO STEP 5

Show the final Copilot response.

Expected:

    Existing API found.

    API: Transaction API
    Endpoint: /payments/transaction
    Status: ACTIVE

    Documentation: Transaction API Design
    Consumers: 14
    Recent usage: ACTIVE

    Related Jira: PROJ-1234

## DEMO STEP 6

Run the negative case.

Prompt:

    "Check whether packet XYZ999 has an existing API."

Expected:

    No existing API was found in the available sources.

Then explain:

    "The same tool interface handles both cases.
     In production, the data sources would simply be real."

#  {#-13}

# 14. WHY THE MOCKS ARE VALID FOR THE HACKATHON {#14-why-the-mocks-are-valid-for-the-hackathon}

Do not present the mocks as the production implementation.

Present them as a functional PoC of the architecture.

The key thing being demonstrated is:

    Copilot
       ->
    MCP
       ->
    standardized tools
       ->
    enterprise data sources
       ->
    consolidated result

The mock systems prove that the MCP tool contracts and orchestration
work.

The production migration changes:

    MockMiddlewareClient
             ->
    RealMiddlewareClient

    MockConfluenceClient
             ->
    RealConfluenceClient

    MockJiraClient
             ->
    RealJiraClient

    MockSplunkClient
             ->
    RealSplunkClient

The MCP tool names and contracts can remain stable.

#  {#-14}

# 15. PRODUCTION CONNECTOR DESIGN {#15-production-connector-design}

For production, do not make the MCP server directly connect to random
databases.

Prefer:

    MCP Tool
       ->
    Internal Connector / Adapter
       ->
    Approved Enterprise API
       ->
    Enterprise System

Examples:

    Middleware Tool
       ->
    Middleware API / API Catalog

    Confluence Tool
       ->
    Confluence REST API

    Jira Tool
       ->
    Jira REST API

    Splunk Tool
       ->
    Splunk search/query API

Benefits:

-   Centralized authentication.
-   Better auditing.
-   Clear ownership.
-   Easier testing.
-   Easier replacement.
-   Reduced coupling.
-   Better security.

#  {#-15}

# 16. TOOL SECURITY {#16-tool-security}

The initial hackathon tools should be read-only.

Examples:

    find_middleware_api       READ
    search_confluence         READ
    search_jira               READ
    search_splunk             READ
    get_api_usage             READ

Do not create write tools for the hackathon.

In production, apply:

-   Authentication.
-   Authorization.
-   Tool allowlists.
-   Least privilege.
-   Audit logging.
-   Input validation.
-   Output filtering.
-   Secret management.
-   Rate limiting.
-   Timeouts.
-   Retry policies.
-   Circuit breakers where appropriate.

Avoid exposing unrestricted enterprise capabilities through MCP.

#  {#-16}

# 17. ERROR HANDLING {#17-error-handling}

Every connector should handle:

-   Timeout.
-   401/403.
-   404. 
-   429. 
-   5xx.
-   Invalid input.
-   Empty result.
-   Partial result.

Example:

    Splunk unavailable.

The final response should NOT claim:

    "There is no API."

Instead:

    "Middleware reports an active API, but Splunk was unavailable,
     so runtime usage could not be verified."

This distinction is important because absence of evidence is not always
evidence of absence.

#  {#-17}

# 18. RESPONSE / ORCHESTRATION DESIGN {#18-response--orchestration-design}

For the first PoC, keep orchestration simple.

Example logic:

    1. Find API in Middleware.
    2. If API exists:
         - Get API details.
         - Search documentation.
         - Get usage.
         - Search related Jira.
    3. If API does not exist:
         - Search Confluence.
         - Search Jira.
         - Search Splunk for related traffic.
    4. Consolidate evidence.
    5. Return structured result to Copilot.

The orchestration can eventually become more intelligent, but the
hackathon does not need a complex agent framework inside the MCP server.

#  {#-18}

# 19. DATA CONTRACT FOR CONSOLIDATED RESULT {#19-data-contract-for-consolidated-result}

Recommended internal result:

    {
      "packetId": "ABC123",
      "api": {
        "exists": true,
        "name": "Transaction API",
        "endpoint": "/payments/transaction",
        "status": "ACTIVE"
      },
      "documentation": {
        "found": true,
        "title": "Transaction API Design",
        "id": "CONF-123"
      },
      "usage": {
        "verified": true,
        "requests30d": 18432,
        "consumers": 14
      },
      "jira": {
        "found": true,
        "tickets": ["PROJ-1234"]
      },
      "sources": [
        "middleware",
        "confluence",
        "splunk",
        "jira"
      ]
    }

This gives Copilot structured evidence rather than a large unstructured
text blob.

#  {#-19}

# 20. TEST PLAN {#20-test-plan}

## UNIT TESTS

Test every tool independently.

Examples:

    find_middleware_api("ABC123")
        -> exists=true

    find_middleware_api("XYZ999")
        -> exists=false

    search_confluence("ABC123")
        -> documentation found

    search_jira("ABC123")
        -> PROJ-1234 returned

    search_splunk("ABC123")
        -> usage returned

## INTEGRATION TEST

Start the mock services.

Call the MCP tools.

Verify:

    Copilot -> MCP -> mock API -> MCP -> Copilot

## NEGATIVE TESTS

-   Middleware unavailable.
-   Confluence unavailable.
-   Empty Jira result.
-   Splunk timeout.
-   Unknown packet.
-   Invalid packet ID.

## DEMO ACCEPTANCE CRITERIA

1.  MCP server starts successfully.
2.  Copilot can discover the MCP tools.
3.  Prompt causes at least two MCP tools to execute.
4.  Mock enterprise services receive requests.
5.  Copilot receives structured results.
6.  Positive API-discovery case works.
7.  Negative case works.
8.  No production credentials are used.

#  {#-20}

# 21. IMPLEMENTATION ORDER {#21-implementation-order}

## PHASE 1 --- CREATE MCP PROJECT {#phase-1--create-mcp-project}

Create:

    enterprise-mcp-server

Use:

-   Java 17+
-   Spring Boot
-   Maven
-   MCP Java SDK

Get a minimal MCP server running.

Verify that Copilot can see at least one test tool.

Example test tool:

    ping()

Response:

    "MCP server is running."

## PHASE 2 --- ADD FIRST REALISTIC TOOL {#phase-2--add-first-realistic-tool}

Implement:

    find_middleware_api(packetId)

Initially return hard-coded/mock data.

Verify from Copilot.

Prompt:

    "Find the API for packet ABC123."

## PHASE 3 --- ADD OTHER TOOLS {#phase-3--add-other-tools}

Implement:

    search_confluence
    search_jira
    search_splunk
    get_api_details
    get_api_usage

## PHASE 4 --- ADD MOCK DATA {#phase-4--add-mock-data}

Create positive and negative scenarios.

ABC123: API exists.

XYZ999: API does not exist.

## PHASE 5 --- ADD CONNECTOR LAYER {#phase-5--add-connector-layer}

Refactor:

    MCP Tool
       ->
    Service
       ->
    Connector
       ->
    Mock API

This makes production migration easy.

## PHASE 6 --- CONNECT COPILOT {#phase-6--connect-copilot}

Configure the MCP server in Copilot\'s MCP configuration.

Verify that the tools appear.

Run the end-to-end prompt.

## PHASE 7 --- POLISH DEMO {#phase-7--polish-demo}

Add:

-   Clear tool names.
-   Structured output.
-   Logging.
-   Simple mock data.
-   One positive scenario.
-   One negative scenario.
-   Clean architecture diagram.
-   Production architecture slide.

#  {#-21}

# 22. WHAT NOT TO BUILD FOR THE HACKATHON {#22-what-not-to-build-for-the-hackathon}

Do NOT spend time building:

-   Real Splunk integration.
-   Real Confluence integration.
-   Real Jira integration.
-   Real Middleware integration.
-   PCF deployment.
-   Enterprise SSO.
-   Enterprise secrets management.
-   Complex RAG.
-   A second LLM inside MCP.
-   Complex autonomous orchestration.
-   Write operations.

The hackathon objective is to prove the architecture and user
experience.

#  {#-22}

# 23. PRODUCTION EVOLUTION {#23-production-evolution}

HACKATHON:

    Copilot
       |
       v
    Local MCP Server
       |
       +--> Mock Middleware
       +--> Mock Confluence
       +--> Mock Jira
       +--> Mock Splunk

PRODUCTION:

    Developers / RAMs
       |
       v
    IntelliJ + GitHub Copilot
       |
       v
    Corporate Network
       |
       v
    PCF / Tanzu
       |
       v
    Enterprise MCP Server
       |
       v
    Secure Integration Layer
       |
       +--> Middleware / API Catalog
       +--> Confluence
       +--> Jira
       +--> Splunk

The architecture remains conceptually the same.

#  {#-23}

# 24. 60-SECOND JUDGE PITCH {#24-60-second-judge-pitch}

\"Today, when a developer is modernizing a legacy application, a very
common question is: do we already have an API for this packet or
business flow?

The problem is that the answer is distributed across Middleware,
Confluence, Jira, Splunk and sometimes people\'s knowledge.

We are proposing an Enterprise MCP Server that sits between GitHub
Copilot and these enterprise systems.

The developer simply asks Copilot a natural-language question.

Copilot can discover our MCP tools, such as finding an existing
Middleware API, searching documentation, checking related Jira tickets
and validating runtime usage in Splunk.

The MCP server orchestrates these calls and returns a consolidated,
evidence-based answer.

For the hackathon, we run the MCP server locally and mock the enterprise
systems through local APIs.

The production architecture is the same concept deployed on PCF, where
the MCP server connects to the real enterprise systems through approved,
authenticated APIs and connectors.

So the PoC proves the developer experience and MCP integration today,
while the architecture provides a clear path to production.\"

#  {#-24}

# 25. 3-MINUTE TECHNICAL PITCH {#25-3-minute-technical-pitch}

\"The solution has two major sides.

On the left is the developer environment: IntelliJ, GitHub Copilot and
the legacy application.

On the right are the enterprise systems that already contain the
information we need --- Middleware, Confluence, Jira and Splunk.

In the middle we introduce an Enterprise MCP Server.

MCP provides the standardized tool interface between Copilot and these
systems.

For example, Copilot can invoke find_middleware_api with a packet ID.
The MCP server passes that request to our Middleware connector.

It can then search Confluence for documentation, Jira for related work
and Splunk for runtime usage.

The important design decision is that these are separate tools and
connectors. The MCP server does not directly contain all the enterprise
business logic.

For the hackathon, each connector points to a mocked API or local
dataset.

For example, packet ABC123 exists in our mock Middleware catalog and
maps to a Transaction API. The mock Confluence service contains the API
design document. The mock Splunk service contains usage information.

When we type the prompt into Copilot, multiple tools are invoked and the
results are consolidated.

In production, those mock clients are replaced with approved enterprise
connectors. The MCP server is deployed on PCF and protected by the
organization\'s authentication, authorization, secrets, logging and
monitoring mechanisms.

This gives us a small, demonstrable PoC without pretending that the
hackathon has production access to enterprise systems.\"

#  {#-25}

# 26. IMPORTANT ARCHITECTURAL CLARIFICATION {#26-important-architectural-clarification}

Do not say:

    "MCP connects directly to all databases."

Say:

    "The MCP server exposes enterprise capabilities through tools.
     Each tool uses an approved connector or API to access the relevant
     enterprise system."

Do not say:

    "Copilot directly calls Splunk."

Say:

    "Copilot invokes the MCP tool; the MCP tool calls the Splunk connector."

Do not say:

    "The MCP server is the AI."

Say:

    "Copilot is the AI client/agent experience. MCP provides the standardized
     tool interface and the server implements the enterprise capabilities."

#  {#-26}

# 27. FINAL DEMO ARCHITECTURE {#27-final-demo-architecture}

LOCAL:

    +-----------------------------+
    | Customer / Legacy Project   |
    | IntelliJ + GitHub Copilot  |
    +--------------+--------------+
                   |
                   | MCP
                   v
    +-----------------------------+
    | Enterprise MCP Server      |
    | Spring Boot                |
    | localhost                  |
    |                             |
    | findMiddlewareApi()        |
    | searchConfluence()         |
    | searchJira()               |
    | searchSplunk()              |
    | getApiUsage()              |
    +--------------+--------------+
                   |
          +--------+--------+--------+--------+
          |                 |        |        |
          v                 v        v        v
       Mock              Mock     Mock     Mock
    Middleware        Confluence  Jira    Splunk

PRODUCTION:

    +-----------------------------+
    | Developers / RAMs           |
    | IntelliJ + GitHub Copilot  |
    +--------------+--------------+
                   |
             Corporate Network
                   |
                   v
    +--------------------------------------+
    | PCF / Tanzu                          |
    |                                      |
    | Enterprise MCP Server                |
    | Spring Boot                          |
    |                                      |
    | MCP Tools                            |
    | Orchestration                        |
    | Configuration                        |
    | Logging / Monitoring                 |
    +------------------+-------------------+
                       |
                Secure Integration
                       |
        +--------------+--------------+
        |              |              |
        v              v              v              v
    Middleware     Confluence       Jira          Splunk
    / API Catalog

#  {#-27}

# 28. IMPLEMENTATION AGENT CHECKLIST {#28-implementation-agent-checklist}

An implementation agent should complete the following in order:

\[ \] Create Spring Boot MCP server. \[ \] Add MCP Java SDK. \[ \]
Implement minimal ping tool. \[ \] Configure local MCP transport. \[ \]
Connect Copilot to local MCP server. \[ \] Verify tool appears in
Copilot. \[ \] Implement find_middleware_api. \[ \] Implement
get_api_details. \[ \] Implement search_confluence. \[ \] Implement
search_jira. \[ \] Implement search_splunk. \[ \] Implement
get_api_usage. \[ \] Create mock data. \[ \] Create mock
connectors/services. \[ \] Connect MCP tools to mock connectors. \[ \]
Test positive scenario ABC123. \[ \] Test negative scenario XYZ999. \[
\] Add error handling. \[ \] Add structured responses. \[ \] Add logs
useful for demo. \[ \] Prepare production architecture diagram. \[ \]
Prepare 3-minute demo. \[ \] Explain mock-to-production mapping.

#  {#-28}

# 29. REFERENCES / CURRENT TECHNICAL NOTES {#29-references--current-technical-notes}

The current MCP Java SDK documentation states that the Java SDK supports
MCP client/server implementations and transports including STDIO, SSE
and Streamable HTTP. The current SDK documentation also notes that the
core convenience module is:

    io.modelcontextprotocol.sdk:mcp

The current SDK documentation lists Java 17+ as a prerequisite.

GitHub\'s current Copilot documentation describes connecting MCP servers
to Copilot Chat and configuring MCP servers in JetBrains IDEs using the
MCP configuration flow.

Useful official references:

MCP Java SDK server documentation:
<https://java.sdk.modelcontextprotocol.io/latest/server/>

MCP Java SDK quickstart:
<https://java.sdk.modelcontextprotocol.io/latest/quickstart/>

MCP Java SDK client documentation:
<https://java.sdk.modelcontextprotocol.io/latest/client/>

GitHub Copilot --- MCP in IDE:
<https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide/extend-copilot-chat-with-mcp>

GitHub Copilot --- MCP overview:
<https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide>

#  {#-29}

# END

Core message to remember:

    "The hackathon mocks the enterprise systems, not the MCP concept.

     We are proving the complete developer-to-MCP-to-tool flow locally,
     while keeping the tool contracts and architecture aligned with the
     production design."
