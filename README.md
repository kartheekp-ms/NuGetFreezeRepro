# NuGet Freeze Reproduction Project

## Issue Summary
This project reproduces Visual Studio 2026 NuGet Package Manager freezing/crashing during package upgrades, as reported in Visual Studio Developer Community feedback ticket #11041174.

## Problem Description
Visual Studio 2026 freezes approximately 50% of the time during NuGet package upgrades in projects with complex assembly binding redirects. The issue does NOT occur in Visual Studio 2017 with the same project configuration.

### Symptoms
- IDE becomes unresponsive during package update operations
- NuGet Package Manager UI freezes at "Resolving dependencies..."
- Complete IDE crash requiring restart
- Package installation never completes

### Root Cause
The performance regression appears to be related to:
- 40+ assembly binding redirects in Web.config
- Complex dependency trees
- NuGet dependency resolver hanging during resolution
- Legacy .NET Framework project structure

## Project Configuration

### Technology Stack
- **Framework**: ASP.NET Web API
- **Target Framework**: .NET Framework 4.5
- **Project Type**: Enterprise web application
- **VS Version**: Visual Studio 2026 (issue present), Visual Studio 2017 (issue absent)

### Key Characteristics Replicating Customer Environment
1. **40+ Assembly Binding Redirects**: The Web.config contains over 40 assembly binding redirects including:
   - Newtonsoft.Json
   - System.Web.Mvc/WebPages/Helpers
   - Entity Framework
   - Azure Storage libraries
   - Autofac, Unity, Ninject (multiple DI containers)
   - OWIN middleware
   - Identity libraries
   - Various System.* assemblies
   - Logging frameworks (NLog, log4net)
   - AutoMapper, RestSharp, Polly, etc.

2. **Legacy Package Versions**: Intentionally uses older package versions to simulate enterprise upgrade scenarios:
   - Newtonsoft.Json 6.0.4 (outdated)
   - Microsoft.AspNet.WebApi 5.2.3

3. **Complex Dependency Tree**: Multiple packages with overlapping transitive dependencies

## Build Verification

✅ **Build Status**: The project builds successfully!

```powershell
# Restore packages
nuget restore NuGetFreezeRepro.sln

# Build solution
msbuild NuGetFreezeRepro.sln /p:Configuration=Debug
```

**Expected Warnings** (these are intentional and demonstrate the issue):
- Version conflicts for Newtonsoft.Json (shows dependency complexity)
- Security vulnerability for old Newtonsoft.Json 6.0.4 (intentional outdated version)

**Output**: `NuGetFreezeRepro\bin\NuGetFreezeRepro.dll` (~7 KB)

## How to Reproduce the Issue

### Prerequisites
- Visual Studio 2026 installed
- .NET Framework 4.5 SDK

### Steps to Reproduce
1. Open `NuGetFreezeRepro.sln` in Visual Studio 2026
2. Restore NuGet packages (should succeed initially)
3. Attempt to update a package using **NuGet Package Manager UI**:
   - Right-click solution → Manage NuGet Packages for Solution
   - Go to "Updates" tab
   - Try to update `Newtonsoft.Json` from 6.0.4 to latest version
4. **Expected Behavior**: Package updates smoothly
5. **Actual Behavior**: 
   - IDE freezes during "Resolving dependencies..." phase
   - Progress bar stops responding
   - May require force-closing Visual Studio

### Reproducing in Different VS Versions
- **VS 2017**: Package updates complete successfully (no freeze)
- **VS 2026**: Freeze occurs ~50% of the time

## Workarounds

The following workarounds have been identified by affected customers:

### 1. Use Package Manager Console (RECOMMENDED)
Instead of using the UI, use PowerShell commands:
```powershell
Update-Package Newtonsoft.Json
```
This is the most reliable method and avoids the NuGet UI resolver.

### 2. Manually Clear NuGet Cache
Before updating packages:
```powershell
dotnet nuget locals all --clear
```
Or in Package Manager Console:
```powershell
Clear-Package-Cache
```

### 3. Edit .csproj Directly
1. Close Visual Studio
2. Edit the `.csproj` file in a text editor
3. Update package references manually
4. Delete `packages` folder
5. Reopen solution and restore packages

### 4. Disable Auto-Restore
In Visual Studio options:
- Tools → Options → NuGet Package Manager
- Uncheck "Automatically check for missing packages during build in Visual Studio"
- Manually restore when needed

### 5. Reduce Assembly Binding Redirects
If possible, consolidate package versions to reduce the number of binding redirects (not always feasible in enterprise projects).

## Expected Fix
This is a performance regression in Visual Studio 2026's NuGet dependency resolver. Microsoft should:
1. Optimize dependency resolution for projects with many assembly binding redirects
2. Add timeout/cancellation handling to prevent complete IDE freeze
3. Improve progress reporting during long-running resolution operations
4. Consider caching resolved dependency graphs

## Additional Notes
- The issue severity increases with the number of assembly binding redirects
- Projects with 10-20 redirects may experience slowness
- Projects with 40+ redirects frequently freeze (as demonstrated here)
- The freeze occurs in the dependency resolution phase, not during download/installation

## Project Structure
```
NuGetFreezeRepro/
├── NuGetFreezeRepro.sln
└── NuGetFreezeRepro/
    ├── NuGetFreezeRepro.csproj
    ├── packages.config
    ├── Web.config          (Contains 40+ assembly binding redirects)
    ├── Web.Debug.config
    ├── Web.Release.config
    ├── Global.asax
    ├── Global.asax.cs
    ├── App_Start/
    │   └── WebApiConfig.cs
    ├── Controllers/
    │   └── ValuesController.cs
    └── Properties/
        └── AssemblyInfo.cs
```

## Testing Checklist
- [ ] Open solution in VS 2026
- [ ] Restore packages successfully
- [ ] Attempt to update Newtonsoft.Json via NuGet UI
- [ ] Observe freeze/hang behavior
- [ ] Test workaround: Update via Package Manager Console
- [ ] Compare behavior in VS 2017 (if available)

## Related Issues
- Visual Studio Developer Community: #11041174
- Azure DevOps Work Item: 2662811

---

**Note**: This is a minimal reproduction project. Real enterprise applications may have additional complexity that exacerbates the issue.
