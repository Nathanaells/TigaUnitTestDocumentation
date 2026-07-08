I analyzed the workflow path. The aggregation mechanism is not `JintTaskExecutor`; it is `WorkflowManager`.

**Key Finding**

The single “Workflow Definition” script is built here:

[FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowManager.cs](/d:/Tiga/FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowManager.cs:937)

`GenerateRunScript(...)` does the aggregation:

1. Reads each active `WorkflowDefinition.Actions`.
2. Loads each referenced `WorkflowTaskName.js` via `RetrieveWorkflowTaskFileBased(...)`.
3. Injects action parameters as JavaScript `var` declarations.
4. Appends task content into one big script.
5. Wraps bulk workflows through `JintTaskScriptManager`.
6. Returns one script for execution by `JintTaskExecutor.RunTask(...)`.

Related files:

- [WorkflowManager.cs](/d:/Tiga/FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowManager.cs:384): `DoConvertToCustomTaskContent`
- [WorkflowManager.cs](/d:/Tiga/FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowManager.cs:956): loads each `WorkflowAction.WorkflowTaskName`
- [WorkflowManager.cs](/d:/Tiga/FugarServer/Fugar.Lib/Business/WorkflowManager/WorkflowManager.cs:1097): reads `{WorkflowTaskName}.js` from file storage
- [WorkflowCacheManager.cs](/d:/Tiga/FugarServer/Fugar.Lib/Business/CacheManager/WorkflowCacheManager.cs:37): caches the generated script
- [JintTaskExecutor.cs](/d:/Tiga/FugarServer/Fugar.Workflow/JintTaskExecutor.cs:55): executes the final script
- [JintTaskScriptManager.cs](/d:/Tiga/FugarServer/Fugar.Workflow/JintTaskScriptManager.cs:34): adds bulk-workflow pre/post wrappers

I did not find the workflow task files in this checkout. The README says they come from a separate repo and should be copied to `Workspaces\W0\FileStorages\WorkflowTasks`:

[README](/d:/Tiga/README:41), [README](/d:/Tiga/README:74)

The only local Change Password controller I found is the Angular client controller:

[FugarClient/Fugar.HTML/Fugar.UI.HTML/Modules/Settings/Controllers/ChangePasswordCtrl.js](/d:/Tiga/FugarClient/Fugar.HTML/Fugar.UI.HTML/Modules/Settings/Controllers/ChangePasswordCtrl.js:1)

It calls:

[ChangePasswordCtrl.js](/d:/Tiga/FugarClient/Fugar.HTML/Fugar.UI.HTML/Modules/Settings/Controllers/ChangePasswordCtrl.js:37)

```js
_securityManagerService.UpdateCurrentUserWithPasswordAsync(...)
```

Server-side password validation/update logic lives in:

[UserManager.cs](/d:/Tiga/FugarServer/Fugar.Lib/Business/SecurityManager/UserManager.cs:487)

**Step-By-Step Unit Test Guide**

1. Create or locate a test project.

   This repository currently does not appear to have a normal xUnit/NUnit/MSTest project. I would add something like:

   ```text
   FugarServer/Fugar.Workflow.Tests/Fugar.Workflow.Tests.csproj
   ```

   Reference:

   ```text
   Fugar.Workflow
   Fugar.Lib
   Fugar.Model
   Fugar.Persistence
   ```

   Add test packages such as `xunit`, `xunit.runner.visualstudio`, and `Moq`.

2. Decide the test boundary.

   For this feature, I recommend two focused tests:

   - Aggregation test: verify `WorkflowManager.ConvertToCustomTaskContent(...)` turns a `WorkflowDefinition` with a Change Password action into one executable JavaScript string.
   - Execution test: pass the generated script into `JintTaskExecutor.RunTask(...)` and verify the expected password-change primitive/function is called.

3. Provide a workflow fixture.

   Because the actual workflow files are external to this checkout, the unit test should provide a fixture script that represents the Change Password workflow task file.

   Example fixture content:

   ```js
   ChangePassword(p_Username, p_CurrentPassword, p_NewPassword);
   ```

   In the real test, replace `ChangePassword` and parameter names with whatever `changepasswordctrl.js` or the workflow task actually calls.

4. Mock workflow task metadata.

   `WorkflowManager.GenerateRunScript(...)` first queries `WorkflowTask` from persistence, then loads `{WorkflowTaskName}.js` from the file store. Your test should mock both:

   - `IDataManager.DbQuery<WorkflowTask>(...)`
   - `IFileTransferManager.FileExists(...)`
   - `IFileTransferManager.FileSize(...)`
   - `IFileTransferManager.Retrieve(...)`

5. Build a `WorkflowDefinition`.

   The important part is:

   ```csharp
   Actions =
   {
     new WorkflowAction
     {
       WorkflowTaskName = "ChangePassword",
       IsActive = true,
       ParameterString = "p_Username=alice,p_CurrentPassword=OldPass1!,p_NewPassword=NewPass1!"
     }
   }
   ```

6. Generate the aggregated script.

   Call:

   ```csharp
   workflowManager.ConvertToCustomTaskContent(unitOfWork.Object, "", workflowDefinition);
   ```

   Assert the output contains:

   - parameter declarations
   - `Running action ChangePassword`
   - your fixture script content

7. Execute with Jint.

   Instantiate `JintTaskExecutor`, pass the generated script, and provide the functions map expected by workflow scripts:

   - `Log`
   - `ChangePassword`
   - any actual primitives used by the real workflow

8. Assert behavior.

   For a true unit test, assert the primitive was called with the right username/current password/new password. Avoid touching the database or real file system.

**Example Test Snippet**

```csharp
using System;
using System.Collections.Generic;
using System.Text;
using Fugar.Lib.Business.EntityManager;
using Fugar.Lib.Business.WorkflowManager;
using Fugar.Model;
using Fugar.Persistence;
using Fugar.Workflow;
using Moq;
using Xunit;

public class ChangePasswordWorkflowTests
{
  [Fact]
  public void ChangePasswordWorkflow_AggregatesAndExecutesExpectedTask()
  {
    var unitOfWork = new Mock<IUnitOfWork>();

    var dataManager = new Mock<IDataManager>();
    dataManager
      .Setup(x => x.DbQuery<WorkflowTask>(
        unitOfWork.Object,
        It.IsAny<string>(),
        It.IsAny<IDictionary<string, object>>()))
      .Returns(new List<WorkflowTask>
      {
        new WorkflowTask
        {
          Name = "ChangePassword",
          ParameterNameString = "p_Username,p_CurrentPassword,p_NewPassword"
        }
      });

    var taskScript = "ChangePassword(p_Username, p_CurrentPassword, p_NewPassword);";
    var taskBytes = Encoding.Default.GetBytes(taskScript);

    var fileTransfer = new Mock<IFileTransferManager>();
    fileTransfer.Setup(x => x.FileExists("ChangePassword.js", "WorkflowTasks")).Returns(true);
    fileTransfer.Setup(x => x.FileSize("ChangePassword.js", "WorkflowTasks")).Returns(taskBytes.Length);
    fileTransfer
      .Setup(x => x.Retrieve("ChangePassword.js", 0, taskBytes.Length, "WorkflowTasks"))
      .Returns(taskBytes);

    var converter = new Mock<IWorkflowParameterStringConverter>();
    converter
      .Setup(x => x.ConvertWorkflowParameterString(
        unitOfWork.Object,
        "UserCredential",
        WorkflowEventTriggerTypes.ClientPostCustomAction,
        It.IsAny<string>(),
        It.IsAny<IDictionary<string, string>>(),
        "p_BusinessEntities"))
      .Returns<IUnitOfWork, string, WorkflowEventTriggerTypes, string, IDictionary<string, string>, string>(
        (_, _, _, value, _, _) => value);

    var workflowManager = new WorkflowManager(
      dataManager.Object,
      scheduleManager: null,
      workflowParameterStringConverter: converter.Object,
      authorizationManager: null,
      logger: null,
      fileTransferManager: fileTransfer.Object,
      workflowTaskBaseDirectory: "WorkflowTasks",
      retrieveWorkflowTaskFileBasedMaxRetry: 1,
      retrieveWorkflowTaskFileBasedInterval: 0,
      taskScriptManager: new JintTaskScriptManager());

    var definition = new WorkflowDefinition
    {
      Name = "Change Password",
      IsActive = true,
      EntityDefinition = new EntityDefinition { Name = "UserCredential" },
      EventTriggerType = WorkflowEventTriggerTypes.ClientPostCustomAction,
      Actions =
      {
        new WorkflowAction
        {
          WorkflowTaskName = "ChangePassword",
          IsActive = true,
          ParameterString = "p_Username=alice,p_CurrentPassword=OldPass1!,p_NewPassword=NewPass1!"
        }
      }
    };

    var script = workflowManager.ConvertToCustomTaskContent(unitOfWork.Object, "", definition);

    Assert.Contains("var p_Username = \"alice\";", script);
    Assert.Contains("Running action ChangePassword", script);
    Assert.Contains("ChangePassword(p_Username, p_CurrentPassword, p_NewPassword);", script);

    string capturedUsername = null;
    string capturedOldPassword = null;
    string capturedNewPassword = null;

    var executor = new JintTaskExecutor(logger: null)
    {
      Assemblies = typeof(BusinessEntityBase).Assembly
    };

    executor.RunTask(
      script,
      new Dictionary<string, object>
      {
        ["p_EventTriggerType"] = "ClientPostCustomAction",
        ["p_BusinessEntities"] = new List<BusinessEntityBase>()
      },
      new Dictionary<string, Delegate>
      {
        ["Log"] = new Action<string>(_ => { }),
        ["ChangePassword"] = new Action<string, string, string>((username, oldPassword, newPassword) =>
        {
          capturedUsername = username;
          capturedOldPassword = oldPassword;
          capturedNewPassword = newPassword;
        })
      });

    Assert.Equal("alice", capturedUsername);
    Assert.Equal("OldPass1!", capturedOldPassword);
    Assert.Equal("NewPass1!", capturedNewPassword);
  }
}
```

The main adjustment you’ll need is replacing the fixture function `ChangePassword(...)` with the real primitive/function used by the actual workflow task once the external `WorkflowTasks` files are copied into this repository.