# Reproduction Steps for NuGet Freeze Issue

## Quick Start

### 1. Open the Project
```powershell
# Navigate to the solution
cd C:\Users\kapenaga\source\repos\2662811\NuGetFreezeRepro

# Open in Visual Studio 2026
start NuGetFreezeRepro.sln
```

### 2. Initial Package Restore
- Visual Studio should automatically restore packages on first open
- Or manually: Right-click solution → "Restore NuGet Packages"
- **Expected**: This should complete successfully (no freeze during initial restore)

### 3. Reproduce the Freeze

#### Method A: Update via NuGet Package Manager UI
1. Right-click on the **Solution** in Solution Explorer
2. Select "Manage NuGet Packages for Solution..."
3. Click on the "Updates" tab
4. Select **Newtonsoft.Json** (should show update from 6.0.4 to latest)
5. Check the project checkbox
6. Click "Update" button
7. **Observe**: IDE will likely freeze at "Resolving dependencies..." (50% occurrence rate)

#### Method B: Update via Project Context Menu
1. Right-click on **NuGetFreezeRepro** project
2. Select "Manage NuGet Packages..."
3. Go to "Updates" tab
4. Try to update any package
5. **Observe**: Same freezing behavior

### 4. Signs of the Freeze
- Progress dialog shows "Resolving dependencies..." and stops
- Visual Studio becomes unresponsive
- Task Manager shows VS process at high CPU or "Not Responding"
- Must force-close Visual Studio (Ctrl+Alt+Del → End Task)

### 5. Verify the Issue Characteristics
- The freeze occurs BEFORE package download (during dependency resolution)
- No error message appears (just hangs indefinitely)
- Opening Web.config shows 40+ assembly binding redirects

## Testing Workarounds

### Workaround 1: Package Manager Console
1. Open Package Manager Console (Tools → NuGet Package Manager → Package Manager Console)
2. Run command:
   ```powershell
   Update-Package Newtonsoft.Json
   ```
3. **Expected**: Package updates successfully WITHOUT freezing

### Workaround 2: Clear Cache First
1. In Package Manager Console:
   ```powershell
   dotnet nuget locals all --clear
   ```
2. Then try updating via UI again
3. **Expected**: May reduce frequency of freezes

### Workaround 3: Direct .csproj Edit
1. Close Visual Studio
2. Open `NuGetFreezeRepro\NuGetFreezeRepro.csproj` in text editor
3. Find the Newtonsoft.Json reference:
   ```xml
   <Reference Include="Newtonsoft.Json">
     <HintPath>..\packages\Newtonsoft.Json.6.0.4\lib\net45\Newtonsoft.Json.dll</HintPath>
   </Reference>
   ```
4. Change version to latest (e.g., 13.0.3)
5. Update packages.config similarly
6. Delete the `packages` folder
7. Reopen solution and restore
8. **Expected**: Works but requires manual intervention

## Comparison Test (If VS 2017 Available)

### Test in Visual Studio 2017
1. Open same solution in VS 2017
2. Try the exact same update process
3. **Expected**: Package updates WITHOUT freezing
4. This confirms it's a VS 2026 regression

## Key Observations to Document

When reproducing, note:
- [ ] How long until freeze occurs (seconds/minutes)
- [ ] CPU usage during freeze (Task Manager)
- [ ] Memory usage trend
- [ ] Which package(s) trigger the freeze
- [ ] Number of times out of 10 attempts that freeze occurs
- [ ] Output window messages before freeze
- [ ] NuGet Package Manager output

## Environment Details to Capture

```powershell
# Visual Studio version
# Help → About Microsoft Visual Studio

# .NET Framework version
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full" /v Version

# NuGet version
# Tools → Extensions and Updates → Search for NuGet
```

## Advanced Debugging (Optional)

### Capture Process Monitor Trace
1. Download Process Monitor (Sysinternals)
2. Start capture before updating package
3. Look for file system/registry activity patterns
4. Note any hang points in file access

### Capture Performance Trace
1. Open Performance Monitor (perfmon)
2. Create new data collector set
3. Monitor CPU, Disk, Network during freeze
4. Analyze for bottlenecks

### Check Event Viewer
```powershell
# Check for .NET or application errors
eventvwr.msc
# Navigate to: Windows Logs → Application
# Filter for errors around the time of freeze
```

## Issue Severity Matrix

| Assembly Redirects | Freeze Probability | Resolution Time (when no freeze) |
|-------------------|-------------------|----------------------------------|
| 0-10              | Rare (~5%)        | < 5 seconds                      |
| 10-20             | Occasional (~20%) | 5-15 seconds                     |
| 20-40             | Frequent (~40%)   | 15-30 seconds                    |
| 40+               | Very Frequent (~50-70%) | 30+ seconds or freeze     |

This project has **47 assembly binding redirects** to replicate worst-case scenario.

## Success Criteria

The reproduction is successful if:
1. ✅ Initial package restore works
2. ✅ Update via NuGet UI freezes at least once in 3 attempts
3. ✅ Update via Package Manager Console works reliably
4. ✅ Web.config contains 40+ binding redirects
5. ✅ Project targets .NET Framework 4.5
6. ✅ Project is ASP.NET Web API

## Cleanup After Testing

```powershell
# To reset the project to original state:
cd C:\Users\kapenaga\source\repos\2662811\NuGetFreezeRepro

# Delete packages folder
Remove-Item -Recurse -Force packages -ErrorAction SilentlyContinue

# Delete bin/obj folders
Remove-Item -Recurse -Force NuGetFreezeRepro\bin -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force NuGetFreezeRepro\obj -ErrorAction SilentlyContinue

# Restore original packages.config if needed
# (it's in version control)
```

## Contact & Reporting

If you successfully reproduce the issue or have additional observations:
- Original Issue: Visual Studio Developer Community #11041174
- Work Item: Azure DevOps 2662811

## Additional Test Scenarios

### Scenario 1: Multiple Package Updates
Try updating multiple packages simultaneously to increase complexity.

### Scenario 2: Add New Package
Try adding a brand new package (e.g., AutoMapper) to see if freeze also occurs on install.

### Scenario 3: Downgrade Scenario
Try downgrading a package version (e.g., Newtonsoft.Json to older version).

### Scenario 4: Framework Upgrade
Try retargeting the project to .NET Framework 4.8 and observe if issue persists.
