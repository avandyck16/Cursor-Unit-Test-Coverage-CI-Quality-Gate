# Confidentiality Notice

> This repository contains an anonymized portfolio case study based on a real-world project.
>
> Company names, client information, URLs, credentials, proprietary business logic, and sensitive implementation details have been removed or modified to protect confidentiality.
>
> Any code snippets included are examples only and do not represent any protected production codebase.

---

# Cursor-Unit-Test-Coverage-CI-Quality-Gate


# Alset.Api — Unit Test Coverage & CI Quality Gate

PowerShell · Azure DevOps · Cursor

## 1. Project Overview

Expanded the unit test suite for a multi-tenant ASP.NET Core API and implemented an **85% application line-coverage gate** in the Azure DevOps CI/CD pipeline.

The work focused on increasing meaningful test coverage, validating business logic and tenant isolation, and ensuring that insufficient coverage or failed tests block the deployment stages.

## 2. QA Strategy and Automation Scope

The test suite covers:

- Business rules, repository and service behavior, and edge cases.
- Input validation, error handling, and expected failure scenarios.
- Authorization and tenant-isolation behavior.
- HTTP-level authentication and middleware behavior where appropriate.
- Application line coverage, excluding Entity Framework migration files.

Most tests exercise services and repositories directly. HTTP integration tests use `WebApplicationFactory` for scenarios that require the ASP.NET Core request pipeline.

## 3. Test Architecture

| Component | Implementation |
|---|---|
| Test framework | xUnit |
| Mocking | Moq |
| Database isolation | EF Core InMemory |
| API integration testing | `Microsoft.AspNetCore.Mvc.Testing` |
| Coverage collection | Coverlet |
| Coverage enforcement | Custom PowerShell script |
| CI/CD | Azure DevOps Pipelines |

Tests follow the repository conventions defined in `.cursor/rules/tests.mdc`:

- Test classes use the `{Subject}Tests` naming convention.
- Test methods use `Method_Condition_ExpectedResult`.
- Each test receives a uniquely named InMemory database.
- Fixed GUIDs and UTC dates are used where appropriate.
- External integrations, including vendor services, are mocked to keep tests isolated and repeatable.

## 4. Test Execution

Run the complete test project with coverage collection using:

```bash
dotnet test Alset.Api.Tests/Alset.Api.Tests.csproj --framework net8.0 --configuration Release --settings Alset.Api.Tests/coverlet.runsettings --collect:"XPlat Code Coverage"
```

Coverage configuration in `Alset.Api.Tests/coverlet.runsettings`:

- Includes the `Alset.API` assembly.
- Excludes Entity Framework migration files.
- Excludes the test assembly from application coverage.

Coverlet collects the coverage data; the custom `Alset.Api.Tests/coverage-gate.ps1` script enforces the minimum threshold.

## 5. AI-Assisted Test Development

Cursor was used as an implementation assistant, with work organized into small, component-specific test blocks.

The workflow included:

1. Inspecting the target component and existing test patterns.
2. Defining a focused test scope.
3. Implementing tests and running the relevant test subset.
4. Reviewing test results, changed files, and the source-control diff.
5. Committing changes after review.

The engineer retained responsibility for test scope, expected behavior, tenant-isolation requirements, coverage targets, and final review. Cursor accelerated implementation without replacing QA analysis or verification.

## 6. Technical Challenges and Solutions

### Application host initialization

Host-based tests initially encountered an EF Core / SQL client compatibility issue involving `SqlGuidCaster` during application startup. The repository's `Microsoft.Data.SqlClient` dependency was pinned to version `5.2.3`, after which host-based tests could start successfully.

### Parallel integration-test interference

Concurrent `WebApplicationFactory` instances caused intermittent authentication failures. The affected host-test classes were placed in a shared xUnit collection named `WebApplicationFactory`, serializing those tests without disabling parallel execution for the entire test assembly.

```csharp
[Collection("WebApplicationFactory")]
public class CompaniesControllerComplianceDeleteAuthTests : IDisposable
{
    // Authentication and authorization tests
}
```

### Enforcing the coverage threshold

Coverlet's collector produces coverage reports but does not enforce the required threshold through this configuration. A custom PowerShell gate was introduced to calculate application line coverage and fail the pipeline when coverage falls below 85%.

```powershell
# 85% is 17/20. Avoid floating-point comparison errors.
if ([int64] $covered * 20 -lt [int64] $applicable * 17) {
    Fail "Line coverage $percent% is below 85%."
}
```

The calculation counts applicable source lines once and excludes migration directories.

### Coverage report discovery in CI

The pipeline was adjusted to locate the generated `coverage.cobertura.xml` report in the agent's temporary directory, identify a single report, and pass its path to the coverage gate and report-publishing task. Duplicate results-directory configuration was removed to avoid conflicts.

### Preventing deployment after a failed build

The deployment stages explicitly require a successful preceding stage and the intended source branch:

- Development: `dev`
- Production: `main`

This prevents the deployment stages from running when the build or coverage gate fails.

## 7. CI/CD Pipeline

The main `azure-pipelines.yml` defines the following flow:

```text
Build ASP.NET Core API
        |
        v
Run Unit Tests + Collect Coverage
        |
        v
Locate Cobertura Report
        |
        v
Enforce 85% Coverage Gate
        |
        v
Publish Coverage Report
        |
        v
Publish Application Artifact
        |
        +---- dev  ----> Development Deployment
        |
        +---- main ----> Production Deployment
```

The pipeline includes:

- .NET 8 SDK setup and Release build.
- Automated test execution and coverage collection.
- A PowerShell coverage gate that fails the build below 85%.
- Coverage report publication.
- Application artifact publication.
- Branch-specific deployment stages with success conditions.

The pipeline's database migration tasks are explicitly disabled. Database schema updates therefore remain outside the automated deployment steps defined in this YAML.

A separate `.azdo/api-deploy.yml` also defines a Docker-based deployment path to Azure Container Apps. Its activation as a separate Azure DevOps pipeline is not established by the YAML file alone.

## 8. Results

| Metric | Result |
|---|---:|
| Total tests | 951 |
| Passed | 950 |
| Failed | 0 |
| Skipped | 1 |
| Application lines covered | 16,316 / 19,166 |
| Line coverage | **85.13%** |
| Coverage gate | Implemented |

The local Release run on October 9, 2026, reached 85.13% application line coverage. A subsequent Azure DevOps run was reported with the same test totals and a passing coverage gate.

The work was merged into `dev` in commit `ae30621` on October 12, 2026.

<img width="75%" alt="image" src="https://github.com/user-attachments/assets/34b566c7-309c-4552-8673-c1157e8e6250" />


## 9. Limitations and Takeaways

- One `Users.CreateUser` test remains skipped because it calls a shared development site rather than using an isolated test environment.
- Host-based tests depend on development configuration and the SQL client compatibility adjustment.
- The audit register still contains an outdated statement that the main pipeline does not run tests.
- A separate Container Apps deployment YAML exists; whether it is active as an independent pipeline should be distinguished from the behavior of the main pipeline.

This project strengthened automated regression coverage and introduced a measurable quality gate into CI/CD. It demonstrates practical work across test architecture, isolated unit testing, API integration testing, coverage analysis, pipeline troubleshooting, and deployment safeguards.

---

<div style="text-align:center; color:#888; font-size:10px;">
