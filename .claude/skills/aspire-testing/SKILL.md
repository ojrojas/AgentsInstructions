---
name: aspire-testing
description: Helps create, analyze, maintain, and troubleshoot xUnit tests for .NET Aspire applications. Uses Aspire.Hosting.Testing and  DistributedApplicationTestingBuilder to run functional and integration tests against the AppHost and its resources. Use this skill when the user asks to create tests for an Aspire project, test endpoints or services deployed by the AppHost, verify communication between resources, validate databases or containers, manage the AppHost lifecycle in tests, or troubleshoot Aspire testing issues.  Aspire Testing with xUnit Purpose This skill helps generate and maintain tests for .NET Aspire applications following the official Aspire testing practices.

The primary goal is to test the distributed application as a whole by using
Aspire.Hosting.Testing.

Important: Aspire Testing is primarily intended for functional and
integration testing. It is not a replacement for traditional unit testing.

Use traditional unit tests when testing isolated classes or business logic.
Use Aspire Testing when the test needs to run the AppHost and interact with
real application resources.

Official Documentation
When working with Aspire Testing, prioritize the official Aspire
documentation:

https://aspire.dev/testing/overview/
https://aspire.dev/testing/write-your-first-test/
https://aspire.dev/testing/manage-app-host/
https://aspire.dev/testing/accessing-resources/
https://aspire.dev/testing/advanced-scenarios/
https://aspire.dev/testing/ci-cd/
Do not assume that an Aspire API exists in the version being used.

Always inspect the project's Aspire version before introducing or modifying
Aspire APIs.

Check:

.csproj
Directory.Packages.props
Directory.Build.props
global.json
NuGet package references
Existing test projects
If the project uses a specific Aspire version, prefer APIs documented for that
version.

Unit Tests vs Aspire Integration Tests
Before creating tests, determine which type of test is appropriate.

Traditional Unit Tests
Use traditional unit tests when:

Testing a single class.
Testing business logic in isolation.
Mocking dependencies.
Testing pure functions.
Testing validation logic.
The AppHost does not need to run.
Containers and external resources are unnecessary.
Do not use DistributedApplicationTestingBuilder for these scenarios.

Aspire Functional / Integration Tests
Use Aspire Testing when:

The AppHost needs to run.
An application needs to be tested through HTTP.
Multiple services need to communicate.
A service depends on another Aspire resource.
PostgreSQL, SQL Server, Redis, RabbitMQ, or other resources need to be
tested.
Service discovery needs to be validated.
Containerized infrastructure needs to participate in the test.
The behavior of the distributed application needs to be verified.
These tests execute the application resources as separate processes or
containers. The test should therefore interact with the application through
its public interfaces rather than directly accessing internal application
services.

Initial Project Analysis
Before modifying code:

Locate the AppHost project.
Locate the test project.
Identify the testing framework.
Determine the .NET version.
Determine the Aspire version.
Determine the xUnit version.
Inspect NuGet package references.
Inspect the resources defined in the AppHost.
Identify relevant endpoints.
Identify existing test infrastructure.
Determine whether the requested test is a unit or integration test.
Reuse existing test infrastructure whenever possible.
Do not reorganize the solution unnecessarily.

Creating a Test Project
If the solution does not have an xUnit Aspire test project and the user wants
to create one, prefer the official Aspire template:

dotnet new aspire-xunit -o xUnit.Tests

Then add the AppHost project reference:

dotnet reference add ../<AppHost>/<AppHost>.csproj --project xUnit.Tests.csproj

Use the actual project names discovered in the solution.

Do not invent project names.

Required Testing Package
Aspire integration tests use:

<PackageReference Include="Aspire.Hosting.Testing" Version="<compatible-version>" />

Do not blindly insert the latest package version.

Determine the version compatible with the project's Aspire version.

The test project also requires the appropriate xUnit and Microsoft test SDK
packages for the project's configuration.

If those dependencies already exist, reuse them.

Creating the AppHost for Testing
The primary API for creating an Aspire test AppHost is:

var appHost =
    await DistributedApplicationTestingBuilder
        .CreateAsync<Projects.MyAppHost>();

Then build the application:

var app = await appHost.BuildAsync();

Do not assume that building the AppHost means every resource is immediately
ready.

When a test requires a specific resource to be available, explicitly wait for
the resource to reach the required state using the Aspire APIs available in
the installed version.

AppHost Lifecycle
When multiple tests use the same AppHost, prefer creating the AppHost once
per test class or shared fixture instead of starting a new AppHost for every
test.

Use xUnit's asynchronous lifecycle mechanisms.

Example:

public class WebTests : IAsyncLifetime
{
    private DistributedApplication _app = default!;

    public async Task InitializeAsync()
    {
        var appHost =
            await DistributedApplicationTestingBuilder
                .CreateAsync<Projects.MyAppHost>();

        _app = await appHost.BuildAsync();
    }

    public async Task DisposeAsync()
    {
        await _app.DisposeAsync();
    }

    [Fact]
    public async Task Example()
    {
        // Test implementation
    }
}

Always dispose the DistributedApplication instance.

The goal is to ensure containers, processes, and other Aspire resources are
properly released after testing.

HTTP Testing
When testing an HTTP resource, use the HTTP client provided by Aspire.

Example:

var client = _app.CreateHttpClient("webfrontend");

var response = await client.GetAsync("/");

response.EnsureSuccessStatusCode();

The resource name must match the actual resource defined in the AppHost.

Do not invent resource names.

If the resource exposes multiple endpoints, specify the appropriate
endpointName when required.

Prefer Aspire service discovery and CreateHttpClient() over hardcoded ports.

Example HTTP Test
[Fact]
public async Task GetProducts_ReturnsOk()
{
    var client = _app.CreateHttpClient("api");

    var response = await client.GetAsync("/api/products");

    response.EnsureSuccessStatusCode();
}

Testing JSON APIs
[Fact]
public async Task GetProducts_ReturnsProducts()
{
    var client = _app.CreateHttpClient("api");

    var products =
        await client.GetFromJsonAsync<List<ProductDto>>(
            "/api/products");

    Assert.NotNull(products);
}

Adapt the assertions to the xUnit version already used by the project.

Resource Readiness
Never assume that a resource is ready immediately after the AppHost starts.

Before interacting with a resource:

Identify the resource.
Wait for the required resource state.
Execute the test operation.
Avoid arbitrary delays such as:

await Task.Delay(5000);

Do not use sleeps as a substitute for proper readiness checks.

Prefer the official Aspire resource lifecycle APIs available in the installed
version.

Databases and Infrastructure Resources
When testing a database or infrastructure resource:

Start the real AppHost.
Use Aspire's resource configuration and connection information.
Connect using the appropriate database or resource client.
Execute the operation.
Assert the observable result.
Clean up any test data when necessary.
Do not assume container names, ports, hostnames, or implementation details.

For example, do not hardcode:

localhost:5432

if Aspire can provide the connection information dynamically.

The exact implementation must depend on the Aspire integration package and
version used by the project.

AppHost Test Configuration
The AppHost can receive custom arguments during testing.

Example:

var appHost =
    await DistributedApplicationTestingBuilder
        .CreateAsync<Projects.MyAppHost>(
            [
                "--environment=Testing"
            ]);

Use this mechanism when the AppHost needs test-specific configuration.

For example, an AppHost might contain:

if (builder.Configuration.GetValue("UseVolumes", true))
{
    postgres.WithDataVolume();
}

The test can disable the volume:

var appHost =
    await DistributedApplicationTestingBuilder
        .CreateAsync<Projects.MyAppHost>(
            [
                "UseVolumes=false"
            ]);

Prefer configuration-based customization over modifying production behavior
specifically for tests.

Port Randomization
Aspire randomizes proxied ports by default to allow multiple application
instances to run without port conflicts.

Do not disable port randomization unless there is a concrete requirement.

If disabling it is necessary:

var appHost =
    await DistributedApplicationTestingBuilder
        .CreateAsync<Projects.MyAppHost>(
            [
                "DcpPublisher:RandomizePorts=false"
            ]);

Prefer service discovery and Aspire-generated HttpClient instances instead
of relying on fixed ports.

Dashboard
The Aspire dashboard is disabled by default when using Aspire Testing.

Do not enable it in tests unless it is required for a specific scenario or
diagnostic purpose.

If required, configure it using the APIs supported by the installed Aspire
version.

Example:

var appHost =
    await DistributedApplicationTestingBuilder
        .CreateAsync<Projects.MyAppHost>(
            args: [],
            configureBuilder: (appOptions, hostSettings) =>
            {
                appOptions.DisableDashboard = false;
            });

Verify that this API is supported by the project's Aspire version before using
it.

DistributedApplicationFactory
Use DistributedApplicationFactory when
DistributedApplicationTestingBuilder does not provide sufficient control
over the AppHost lifecycle.

Example:

public class TestingAppHost()
    : DistributedApplicationFactory(typeof(Projects.MyAppHost))
{
}

Possible lifecycle extension points include:

OnBuilderCreating
OnBuilderCreated
OnBuilding
OnBuilt
Do not introduce DistributedApplicationFactory unless the test has a clear
requirement for customization.

Prefer the simpler DistributedApplicationTestingBuilder approach whenever
possible.

Dependency Injection and Mocking
Do not treat Aspire Testing as a traditional dependency injection mocking
framework.

The AppHost launches applications in separate processes. The test therefore
does not automatically have direct access to the application's internal
Dependency Injection container.

If the goal is to mock:

a repository;
an application service;
an external API;
a database abstraction;
another internal dependency;
prefer a traditional unit test.

For ASP.NET Core application-level testing where dependency replacement is
required, consider WebApplicationFactory when appropriate.

Use Aspire Testing when the purpose is to validate the real distributed
application behavior.

Shared xUnit Fixtures
If multiple test classes need the same AppHost, create a shared xUnit
fixture rather than duplicating AppHost initialization.

Example:

public class AspireFixture : IAsyncLifetime
{
    public DistributedApplication App { get; private set; } = default!;

    public async Task InitializeAsync()
    {
        var builder =
            await DistributedApplicationTestingBuilder
                .CreateAsync<Projects.MyAppHost>();

        App = await builder.BuildAsync();
    }

    public async Task DisposeAsync()
    {
        await App.DisposeAsync();
    }
}

Use the fixture mechanisms supported by the xUnit version installed in the
project.

Do not assume that xUnit v2 and xUnit v3 expose identical fixture APIs.

Inspect the project before generating fixture infrastructure.

Recommended Test Structure
A possible structure for an Aspire test project is:

src/
├── MyApp.AppHost/
│   └── Program.cs
├── MyApp.Api/
│   └── ...
├── MyApp.Web/
│   └── ...
└── MyApp.Tests/
    ├── AppHost/
    │   └── AspireTestFixture.cs
    ├── Api/
    │   └── ProductsTests.cs
    ├── Integration/
    │   └── DatabaseTests.cs
    └── ...

Adapt the structure to the existing solution.

Do not move projects or files unless necessary.

Test Design Principles
When generating tests:

Follow Arrange / Act / Assert.
Use descriptive test names.
Test observable behavior.
Avoid testing implementation details.
Avoid hardcoded ports.
Avoid hardcoded container addresses.
Avoid arbitrary Task.Delay.
Avoid unnecessary container creation.
Reuse the AppHost when appropriate.
Dispose resources correctly.
Keep tests independent.
Avoid shared mutable state between tests.
Use deterministic test data.
Use reasonable timeouts.
Keep each test focused.
Reuse existing helpers and fixtures.
Prefer the official Aspire APIs over custom infrastructure.
Naming Tests
Prefer behavior-oriented names.

Good:

GetProducts_ReturnsOk
CreateOrder_WithValidRequest_ReturnsCreated
Checkout_WhenInventoryIsUnavailable_ReturnsConflict
DatabaseResource_IsReachable

Avoid vague names:

Test1
Works
ApiTest
AspireTest

The name should communicate the scenario and expected behavior.

Test Data
Prefer deterministic test data.

Avoid depending on:

current time;
external public APIs;
developer-specific environment variables;
local machine configuration;
hardcoded localhost ports;
manually running containers.
When external infrastructure is required, let Aspire manage it whenever
possible.

Troubleshooting
When an Aspire test fails, diagnose the failure systematically.

AppHost Does Not Start
Check:

Compilation errors.
.NET SDK version.
Aspire version.
Aspire workload requirements.
Docker/container runtime.
Environment variables.
AppHost configuration.
Resource configuration.
Test arguments.
Existing processes using required resources.
HTTP 404
Check:

AppHost resource name.
Endpoint name.
HTTP route.
Application routing configuration.
Whether the correct resource was selected.
Whether the request is being sent to the correct endpoint.
Do not assume a 404 means the resource failed to start.

Connection Refused
Check:

Whether the resource started successfully.
Whether the endpoint exists.
Whether the test waited for resource readiness.
Whether CreateHttpClient() is being used correctly.
Whether a fixed port was accidentally configured.
Whether the application itself is listening on the expected protocol.
Test Hangs
Check:

AppHost lifecycle.
Resource readiness.
Deadlocks.
Containers waiting indefinitely.
External dependencies.
Missing cancellation/timeouts.
Tests waiting for a resource that never reaches the expected state.
Avoid increasing timeouts blindly.

Find the underlying readiness or lifecycle problem first.

Tests Are Slow
Check whether:

The AppHost is created for every test.
The same containers are repeatedly recreated.
Unnecessary resources are included.
Arbitrary sleeps are being used.
Shared fixtures could be used.
The test starts more infrastructure than required.
Continuous Integration
When tests are intended to run in CI:

Verify that the CI environment supports the required container runtime.
Verify the .NET SDK version.
Verify the Aspire version.
Ensure required environment variables are configured.
Avoid dependencies on developer-specific machine configuration.
Ensure containers and processes are properly disposed.
Keep test data deterministic.
Avoid fixed ports unless required.
Make failures diagnosable through test output and logs.
Follow the official Aspire CI/CD guidance for the project's Aspire version.

Agent Workflow
When the user asks:

"Create tests for my Aspire application."

Follow this workflow.

Step 1 — Inspect
Inspect:

Solution files.
AppHost project.
Application projects.
Existing test projects.
.csproj files.
Directory.Packages.props.
Directory.Build.props.
global.json.
AppHost Program.cs.
Existing test fixtures.
Existing test utilities.
Step 2 — Identify Test Type
Determine whether the requested test is:

Unit test.
ASP.NET Core integration test.
Aspire functional/integration test.
If the test does not require the AppHost, do not introduce Aspire Testing.

Step 3 — Identify Aspire Resources
Map the AppHost resources:

AppHost
 ├── API
 ├── Web
 ├── PostgreSQL
 ├── Redis
 ├── RabbitMQ
 └── Other resources

Use the actual resource names from the project.

Step 4 — Identify Test Boundaries
Determine how the behavior should be tested.

For example:

HTTP client
    ↓
API
    ↓
Application service
    ↓
PostgreSQL

If the purpose is to validate this complete flow, use Aspire Testing.

Step 5 — Implement
Use:

DistributedApplicationTestingBuilder

and:

CreateHttpClient()

when appropriate.

Use fixtures when multiple tests can share the AppHost.

Step 6 — Run
Run the smallest relevant test set first.

For example:

dotnet test --filter FullyQualifiedName~ProductsTests

Then run the complete test project:

dotnet test

Step 7 — Diagnose
If tests fail:

Read the complete error.
Identify whether the failure is test code, application code,
configuration, resource startup, or environment-related.
Inspect logs.
Verify resource readiness.
Verify endpoint configuration.
Fix the smallest necessary change.
Run the failing test again.
Run the full test suite afterward.
Step 8 — Report
After implementation, report:

Tests added.
Resources exercised.
Type of testing used.
Important configuration changes.
Commands used to run the tests.
Test results.
Any limitations or environment requirements.
Version Awareness
Aspire APIs evolve.

Before using an API that is not already present in the project:

Determine the Aspire version.
Check the official Aspire documentation.
Verify the API exists for that version.
Prefer APIs already used by the project.
Avoid upgrading Aspire merely to make a test easier.
Do not mix APIs from different Aspire versions.

Safety Rules for Code Changes
Never:

Invent AppHost resource names.
Invent endpoints.
Invent package versions.
Assume containers are ready immediately.
Hardcode ports without a reason.
Hardcode container addresses.
Access internal DI services from the test process as if they were in-process.
Call an integration test a unit test.
Add arbitrary sleeps.
Upgrade Aspire without user intent.
Rewrite the AppHost unnecessarily.
Add testing infrastructure that duplicates existing infrastructure.
Always:

Inspect the existing project first.
Respect the project's Aspire and .NET versions.
Prefer official Aspire APIs.
Prefer service discovery.
Reuse fixtures where appropriate.
Dispose the AppHost correctly.
Keep tests deterministic.
Keep changes minimal.
Run the tests after making changes.
Clearly distinguish unit tests from integration tests.
Definition of Doone
An Aspire testing task is complete when:

The correct type of test has been selected.
The existing AppHost is reused.
The Aspire version has been verified.
The appropriate testing packages are present.
The AppHost lifecycle is correctly managed.
Required resources are awaited before use.
No unnecessary fixed ports are introduced.
Tests verify observable behavior.
Resources are disposed correctly.
The relevant tests pass.
The complete test project passes when practical.
The implementation follows the official Aspire testing model.