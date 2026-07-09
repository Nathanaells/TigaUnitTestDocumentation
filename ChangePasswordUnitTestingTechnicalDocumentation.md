# Technical Documentation: Unit Testing Change Password

Dokumen ini menjelaskan unit testing fitur **Change Password** pada dua area:

- **Client-side**: `ChangePasswordCtrl.test.js` menggunakan Jest untuk AngularJS 1.x.
- **Server-side**: `UserManager.Tests.cs` menggunakan NUnit dan NSubstitute untuk logic business layer/backend.

Fokus utama pengujian adalah memastikan flow change password berjalan benar tanpa memanggil dependency eksternal seperti HTTP API, database, hashing asli, validasi password asli, atau runtime CoreWCF penuh.

## 1. Client-Side Unit Test

File test:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML.Tests/Modules/Settings/ChangePasswordCtrl.test.js
```

Helper loader:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML.Tests/TestUtils/loadScript.js
```

Source controller:

```text
FugarClient/Fugar.HTML/Fugar.UI.HTML/Modules/Settings/Controllers/ChangePasswordCtrl.js
```

### 1.1 Ringkasan Mocking

| Komponen/Service yang Di-mock | Mengapa Harus Di-mock? | Perilaku/Behavior Mock |
|---|---|---|
| `securityManagerService.UpdateCurrentUserWithPasswordAsync` | Controller seharusnya diuji secara unit, bukan melakukan request HTTP asli ke endpoint `SecurityManager/UpdateCurrentUserWithPassword`. Mock ini memutus dependency ke `ServiceClientManager`, `$http`, WAMP/Apache, dan backend. | Menggunakan `jest.fn()`. Pada skenario sukses, mock langsung memanggil `successCallback()`. Pada skenario gagal, mock langsung memanggil `errorCallback(serviceError)`. |
| `userCredential` | Controller membutuhkan data current user untuk membangun object `user` yang dikirim ke service. Data ini biasanya berasal dari resolved modal/session state, bukan tanggung jawab test ini. | Object sederhana berisi `Username`, `Fullname`, dan `Email`. Test memverifikasi field-field ini diteruskan ke service bersama password baru. |
| `$scope.$close` | Pada runtime aplikasi, `$scope.$close` berasal dari modal Angular UI Bootstrap. Unit test tidak membuka modal sungguhan, sehingga fungsi close harus diganti agar bisa diverifikasi. | `jest.fn()` dipasang ke `$scope.$close`. Test memeriksa apakah dipanggil dengan `true` pada success callback. |
| `FaultError` | Controller melempar `new FaultError(result)` pada callback gagal. Di browser production, `FaultError` tersedia sebagai global script. Di Jest, global browser script itu tidak otomatis tersedia. | Helper `defineFaultError()` mendefinisikan constructor `FaultError` di VM context yang sama dengan controller, sehingga `toThrow(FaultError)` dapat memvalidasi error path. |
| `loadController(...)` helper | File controller lama menggunakan pola fungsi global, bukan CommonJS/ES Module. Karena tidak ada `module.exports`, Jest tidak bisa cukup memakai `require()`. | Helper membaca file asli dengan `fs.readFileSync`, menjalankannya di VM, lalu mengembalikan function global controller, misalnya `ChangePasswordCtrl`. Ini membuat path source root didefinisikan satu kali, mirip konsep reference folder. |
| AngularJS injector (`angular.injector(["ng", "Tiga.changePasswordCtrlTest"])`) | Test membutuhkan `$controller` dan `$rootScope` asli dari AngularJS agar controller dijalankan mendekati cara AngularJS menginisialisasi controller. Ini bukan full app bootstrap. | Test membuat module kecil khusus test, mendaftarkan controller dan value mock, lalu mengambil `$controller` dan `$rootScope` dari injector. |

### 1.2 Mekanisme Loading Controller

Controller production tidak diubah. Test mengambil controller melalui helper:

```js
const { defineFaultError, loadController } = require("../../TestUtils/loadScript");

const FaultError = defineFaultError();
const ChangePasswordCtrl = loadController(
  "Modules/Settings/Controllers/ChangePasswordCtrl.js",
  "ChangePasswordCtrl"
);
```

Alasan teknis:

- `ChangePasswordCtrl.js` mendefinisikan `function ChangePasswordCtrl(...)` di global scope.
- Node/Jest module system membungkus setiap file dalam module scope, sehingga function global lama tidak otomatis bisa diakses via `require`.
- Helper `loadController` membuat satu abstraction agar setiap test tidak perlu menghitung path seperti `../../../Fugar.UI.HTML/...`.
- Untuk controller lain, test cukup mengganti relative path dan nama function:

```js
const GlobalSettingsCtrl = loadController(
  "Modules/Settings/Controllers/GlobalSettingsCtrl.js",
  "GlobalSettingsCtrl"
);
```

### 1.3 Detail Daftar Skenario Test

#### Test 1: `PasswordsNotMatch returns false when passwords match`

**Tujuan Test**

Membuktikan bahwa fungsi validasi `PasswordsNotMatch` mengembalikan `false` ketika `NewPassword` dan `Confirm` sama. Ini penting karena controller menggunakan fungsi ini sebagai `SaveCommandDisabled`; jika return value salah, tombol update bisa salah enable/disable.

**Alur Eksekusi**

1. **Arrange**
   - Test membuat `$scope` baru melalui `$rootScope.$new()`.
   - Controller diinisialisasi melalui `$controller("ChangePasswordCtrl", { $scope })`.
   - Controller membuat `$scope.Model` dengan field `CurrentPassword`, `NewPassword`, `Confirm`, dan `AlertText`.
   - Test mengisi:

     ```js
     $scope.Model.NewPassword = "NewPass1!";
     $scope.Model.Confirm = "NewPass1!";
     ```

2. **Act**
   - Test memanggil:

     ```js
     $scope.PasswordsNotMatch()
     ```

   - Fungsi controller menjalankan perbandingan:

     ```js
     return $scope.Model.NewPassword != $scope.Model.Confirm;
     ```

3. **Assert**
   - Karena kedua password sama, hasil harus `false`.
   - Assertion:

     ```js
     expect($scope.PasswordsNotMatch()).toBe(false);
     ```

#### Test 2: `CloseCommand calls scope close`

**Tujuan Test**

Membuktikan bahwa `CloseCommand` meneruskan hasil operasi ke `$scope.$close`. Ini memastikan controller menutup modal menggunakan API modal yang benar.

**Alur Eksekusi**

1. **Arrange**
   - `$scope.$close` diganti dengan `jest.fn()`.
   - Test memasang spy:

     ```js
     const closeSpy = jest.spyOn($scope, "$close");
     ```

2. **Act**
   - Test memanggil command yang diekspos controller:

     ```js
     $scope.CloseCommand(true);
     ```

   - Di dalam controller, `CloseCommand(result)` menjalankan:

     ```js
     $scope.$close(result);
     ```

3. **Assert**
   - Test memastikan `$scope.$close` dipanggil dengan nilai `true`.

     ```js
     expect(closeSpy).toHaveBeenCalledWith(true);
     ```

#### Test 3: `ChangePasswordCommand calls service and success callback closes dialog`

**Tujuan Test**

Membuktikan flow sukses change password di sisi client:

- Controller membangun object `user` dari `userCredential`.
- Controller mengambil old password dari `$scope.Model.CurrentPassword`.
- Controller mengambil new password dari `$scope.Model.Confirm`.
- Controller memanggil `securityManagerService.UpdateCurrentUserWithPasswordAsync`.
- Success callback `OnUpdateCompleted` menutup modal dengan `true`.

**Alur Eksekusi**

1. **Arrange**
   - Mock service disiapkan agar langsung memanggil success callback:

     ```js
     securityManagerService.UpdateCurrentUserWithPasswordAsync
       .mockImplementation(function(user, oldPassword, successCallback) {
         successCallback();
       });
     ```

   - Test mengisi form model:

     ```js
     $scope.Model.CurrentPassword = "OldPass1!";
     $scope.Model.Confirm = "NewPass1!";
     ```

2. **Act**
   - Test menjalankan command:

     ```js
     $scope.SaveCommand();
     ```

   - `SaveCommand` menunjuk ke `ChangePasswordCommand`.
   - Controller membuat object:

     ```js
     user.Username = _userCredential.Username;
     user.Fullname = _userCredential.Fullname;
     user.Email = _userCredential.Email;
     user.Password = $scope.Model.Confirm;
     ```

   - Controller mengambil:

     ```js
     var oldPassword = $scope.Model.CurrentPassword;
     ```

   - Controller memanggil service dengan pola callback, bukan Promise:

     ```js
     _securityManagerService.UpdateCurrentUserWithPasswordAsync(
       user,
       oldPassword,
       OnUpdateCompleted,
       OnServiceCallFailed
     );
     ```

   - Mock service langsung menjalankan `successCallback()`.
   - Callback `OnUpdateCompleted` menjalankan `CloseCommand(true)`.

3. **Assert**
   - Test memverifikasi service dipanggil dengan payload user yang benar:

     ```js
     expect(securityManagerService.UpdateCurrentUserWithPasswordAsync)
       .toHaveBeenCalledWith(
         {
           Username: userCredential.Username,
           Fullname: userCredential.Fullname,
           Email: userCredential.Email,
           Password: "NewPass1!"
         },
         "OldPass1!",
         expect.any(Function),
         expect.any(Function)
       );
     ```

   - Test memverifikasi modal ditutup:

     ```js
     expect($scope.$close).toHaveBeenCalledWith(true);
     ```

#### Test 4: `ChangePasswordCommand calls service and error callback throws FaultError`

**Tujuan Test**

Membuktikan flow gagal di sisi client ketika service memanggil error callback. Controller harus mengubah error service menjadi `FaultError`, sehingga error handling global aplikasi dapat menangani error dengan format yang konsisten.

**Alur Eksekusi**

1. **Arrange**
   - Test menyiapkan error service:

     ```js
     const serviceError = { Message: "Invalid verification data." };
     ```

   - Mock service dibuat agar langsung memanggil `errorCallback(serviceError)`:

     ```js
     securityManagerService.UpdateCurrentUserWithPasswordAsync
       .mockImplementation(function(user, oldPassword, successCallback, errorCallback) {
         errorCallback(serviceError);
       });
     ```

   - Form model diisi:

     ```js
     $scope.Model.CurrentPassword = "WrongPass1!";
     $scope.Model.Confirm = "NewPass1!";
     ```

2. **Act**
   - Test menjalankan:

     ```js
     $scope.SaveCommand();
     ```

   - Controller tetap membangun payload dan memanggil service.
   - Mock service memanggil callback gagal.
   - `OnServiceCallFailed(result)` menjalankan:

     ```js
     throw new FaultError(result);
     ```

3. **Assert**
   - Test memastikan command melempar `FaultError`:

     ```js
     expect(function() {
       $scope.SaveCommand();
     }).toThrow(FaultError);
     ```

   - Test memastikan modal tidak ditutup pada error:

     ```js
     expect($scope.$close).not.toHaveBeenCalled();
     ```

## 2. Server-Side Unit Test

File test:

```text
FugarServer/Fugar.Lib.Tests/Business/SecurityManager/UserManager.Tests.cs
```

Source utama:

```text
FugarServer/Fugar.Service/FugarService.SecurityManager.cs
FugarServer/Fugar.Lib/Business/SecurityManager/UserManager.cs
```

### 2.1 Scope Pengujian Server

Endpoint service layer yang relevan adalah:

```csharp
public UserCredential UpdateCurrentUserWithPassword(UserCredential user, string oldPassword)
{
  user.ID = Guid.Parse(GetUserID());
  user = _userManager.RestrictedUpdate(user, oldPassword, true);
  RemovePasswordInformation(user);
  FixRestResponseResult(user);
  FixRestResponseResult(user);

  return user;
}
```

Logic inti change password ada di:

```csharp
UserManager.RestrictedUpdate(user, oldPassword, true)
```

yang masuk ke private method:

```csharp
DoRestrictedUpdate(IUnitOfWork unitOfWork, UserCredential user, string oldPassword, bool hasNewPassword)
```

Unit test saat ini menguji `UserManager.RestrictedUpdate(...)` karena di sanalah behavior security utama berada:

- validasi old password,
- validasi kekuatan password baru,
- assignment password baru,
- hashing password melalui `DoUpdate`,
- update database,
- commit/rollback transaction.

`FugarService.SecurityManager.UpdateCurrentUserWithPassword` adalah facade CoreWCF yang juga bergantung pada `OperationContext`, cookie `UserID`, dan REST response shaping. Dependency tersebut lebih cocok untuk integration test/service-host test. Unit test ini menjaga fokus pada business logic di `UserManager.RestrictedUpdate` yang dipanggil oleh facade tersebut.

### 2.2 Ringkasan Mocking

| Komponen/Service yang Di-mock | Mengapa Harus Di-mock? | Perilaku/Behavior Mock |
|---|---|---|
| `IDataManager` | Menghindari akses database/NHibernate asli. `UserManager` menggunakan `IDataManager` untuk membuat unit of work, query user saat ini, dan update user. Unit test harus mengontrol data user yang ditemukan dan memastikan update terjadi atau tidak terjadi. | `CreateUnitOfWork()` mengembalikan mock `IUnitOfWork`. `DbQuery<UserCredential>(...)` mengembalikan current user sesuai `userID`. `DbUpdate(...)` mengembalikan object user yang diterima agar mudah diverifikasi. |
| `IUnitOfWork` | Menghindari transaction database sungguhan dan memungkinkan verifikasi transaction boundary. Ini penting karena success harus commit, sedangkan failure harus rollback. | Pada sukses: `BeginTransaction()` dan `Commit()` harus dipanggil, `Rollback()` tidak dipanggil. Pada gagal: `BeginTransaction()` dan `Rollback()` harus dipanggil, `Commit()` tidak dipanggil. `Dispose()` selalu dipanggil. |
| `IHashGenerator` | Hashing asli bersifat detail teknis cryptographic dan tidak perlu diuji di test `UserManager`. Yang perlu diuji adalah kapan verify/hash dipanggil dan bagaimana hasil hash digunakan. | `VerifyString(oldPassword, CurrentPasswordHash)` return `true` untuk old password valid, return `false` untuk mismatch. `StringToHash(NewPassword)` return `NewPasswordHash` pada skenario sukses. |
| `IPasswordValidator` | Validasi password strength sudah domain tersendiri. Test ini hanya perlu memastikan `UserManager` memanggil validator ketika `hasNewPassword = true` dan menolak password lemah. | `IsStrong(NewPassword)` return `true` untuk success. `IsStrong("weak")` return `false` untuk weak password scenario. Pada old password mismatch, validator tidak boleh dipanggil. |
| `IEmailManager`, `IEmailFactory`, `IFmlxLogger` | Tidak relevan untuk `RestrictedUpdate` change password. Dependency tersebut digunakan oleh flow create/reset password, bukan inti restricted update yang sedang diuji. | Diberikan sebagai `null` saat membuat `UserManager`. |
| `UserCredential currentUser` | Mengganti record user dari database. Tanpa mock ini, test harus membaca database asli hanya untuk mendapatkan password hash lama. | `CreateCurrentUser(userID)` membuat user database simulasi dengan `Password = CurrentPasswordHash`. |
| `UserCredential requestedUser` | Mengganti input dari service facade/client. Object ini merepresentasikan data profile dan password baru yang diminta user. | `CreateRequestedUser(userID, newPassword)` membuat user request dengan username/fullname/email baru dan password baru. |

### 2.3 Detail Daftar Skenario Test

#### Test 1: `RestrictedUpdate_WithValidCurrentPasswordAndStrongNewPassword_ShouldCommitAndUpdatePassword`

**Tujuan Test**

Membuktikan flow sukses change password pada `UserManager.RestrictedUpdate`:

- old password cocok dengan hash saat ini,
- password baru kuat,
- password baru di-hash sebelum disimpan,
- data profile user ikut diperbarui,
- database update dilakukan,
- transaction commit,
- rollback tidak terjadi.

**Alur Eksekusi**

1. **Arrange: Data user**
   - Test membuat `userID` baru:

     ```csharp
     Guid userID = Guid.NewGuid();
     ```

   - `currentUser` merepresentasikan data user yang sudah ada di database:

     ```csharp
     Password = CurrentPasswordHash
     ```

   - `requestedUser` merepresentasikan data request update:

     ```csharp
     Password = NewPassword
     Username = "updated.user"
     Fullname = "Updated User"
     Email = "updated.user@formulatrix.com"
     ```

2. **Arrange: Fixture dan mock**
   - `ChangePasswordFixture` membuat mock:

     - `IDataManager`,
     - `IUnitOfWork`,
     - `IHashGenerator`,
     - `IPasswordValidator`.

   - `SetupCurrentUser(userID, currentUser)` mengatur query:

     ```csharp
     DbQuery<UserCredential>(
       UnitOfWork,
       "from UserCredential where ID = :userID",
       parameters with userID
     )
     ```

     agar mengembalikan current user.

   - Hash verification dibuat sukses:

     ```csharp
     VerifyString(CurrentPassword, CurrentPasswordHash).Returns(true);
     ```

   - Password strength dibuat sukses:

     ```csharp
     IsStrong(NewPassword).Returns(true);
     ```

   - Hash password baru dibuat deterministic:

     ```csharp
     StringToHash(NewPassword).Returns(NewPasswordHash);
     ```

3. **Act**
   - Test memanggil:

     ```csharp
     UserCredential result =
       fixture.UserManager.RestrictedUpdate(requestedUser, CurrentPassword, true);
     ```

   - Public method `RestrictedUpdate`:

     - membuat unit of work,
     - memulai transaction,
     - memanggil `DoRestrictedUpdate`,
     - commit jika sukses,
     - dispose unit of work.

   - `DoRestrictedUpdate`:

     - memvalidasi username tidak kosong,
     - mengambil `currentUser` dari database simulasi,
     - memverifikasi old password,
     - menyalin `Username`, `Fullname`, `Email`,
     - karena `hasNewPassword = true`, memanggil `_passwordValidator.IsStrong(user.Password)`,
     - menaruh password baru sementara di `currentUser.Password`.

   - `DoUpdate`:

     - memvalidasi user,
     - karena `excludePassword = false`, memanggil `_hashGenerator.StringToHash(...)`,
     - mengganti `currentUser.Password` menjadi hash,
     - memanggil `_dataManager.DbUpdate(...)`.

4. **Assert**
   - Result harus memuat data user yang diperbarui:

     ```csharp
     Assert.That(result.Password, Is.EqualTo(NewPasswordHash));
     Assert.That(result.Username, Is.EqualTo("updated.user"));
     ```

   - `DbUpdate` harus dipanggil sekali dengan user yang password-nya sudah hash:

     ```csharp
     fixture.DataManager.Received(1).DbUpdate(
       fixture.UnitOfWork,
       Arg.Is<UserCredential>(user => user.Password == NewPasswordHash)
     );
     ```

   - Transaction boundary:

     ```csharp
     BeginTransaction() dipanggil sekali
     Commit() dipanggil sekali
     Rollback() tidak dipanggil
     Dispose() dipanggil sekali
     ```

#### Test 2: `RestrictedUpdate_WithInvalidCurrentPassword_ShouldThrowAndRollback`

**Tujuan Test**

Membuktikan old password mismatch menghentikan proses sebelum validasi password baru, hashing, dan update database. Ini adalah security gate utama: user tidak boleh mengganti password jika tidak bisa membuktikan password lama.

**Alur Eksekusi**

1. **Arrange: Data user**
   - `currentUser` dibuat dengan `Password = CurrentPasswordHash`.
   - `requestedUser` membawa `NewPassword`, tetapi password baru ini tidak boleh diproses karena old password salah.

2. **Arrange: Mock verification gagal**
   - Mock hash generator diatur:

     ```csharp
     VerifyString("WrongPass1!", CurrentPasswordHash).Returns(false);
     ```

3. **Act**
   - Test memanggil:

     ```csharp
     fixture.UserManager.RestrictedUpdate(requestedUser, "WrongPass1!", true)
     ```

   - `RestrictedUpdate` memulai transaction.
   - `DoRestrictedUpdate` mengambil current user.
   - `_hashGenerator.VerifyString(...)` return `false`.
   - Method langsung melempar:

     ```csharp
     ApplicationException("Invalid verification data. User profile is not updated.")
     ```

   - Catch block pada `RestrictedUpdate` melakukan rollback dan rethrow.

4. **Assert**
   - Exception message harus tepat:

     ```csharp
     Assert.That(exception.Message, Is.EqualTo(
       "Invalid verification data. User profile is not updated."
     ));
     ```

   - Password validator tidak boleh dipanggil:

     ```csharp
     fixture.PasswordValidator.DidNotReceive().IsStrong(Arg.Any<string>());
     ```

   - Hash password baru tidak boleh dipanggil:

     ```csharp
     fixture.HashGenerator.DidNotReceive().StringToHash(Arg.Any<string>());
     ```

   - Database update tidak boleh terjadi:

     ```csharp
     fixture.DataManager.DidNotReceive().DbUpdate(...);
     ```

   - Transaction boundary:

     ```csharp
     BeginTransaction() dipanggil sekali
     Commit() tidak dipanggil
     Rollback() dipanggil sekali
     Dispose() dipanggil sekali
     ```

#### Test 3: `RestrictedUpdate_WithWeakNewPassword_ShouldThrowAndRollback`

**Tujuan Test**

Membuktikan kondisi khusus dalam `DoRestrictedUpdate` saat `hasNewPassword = true`:

```csharp
if( hasNewPassword )
{
  if( !_passwordValidator.IsStrong( user.Password ) )
    throw new ApplicationException(...);

  currentUser.Password = user.Password;
}
```

Skenario ini memastikan old password sudah benar, tetapi password baru ditolak karena tidak memenuhi aturan strength.

**Alur Eksekusi**

1. **Arrange: Password baru lemah**
   - Test menggunakan:

     ```csharp
     const string weakPassword = "weak";
     ```

   - `requestedUser.Password` diisi `weakPassword`.

2. **Arrange: Old password valid**
   - Mock hash generator dibuat sukses:

     ```csharp
     VerifyString(CurrentPassword, CurrentPasswordHash).Returns(true);
     ```

   - Ini penting karena test ingin masuk ke branch validasi password baru, bukan berhenti di old password mismatch.

3. **Arrange: Password strength gagal**
   - Mock password validator dibuat gagal:

     ```csharp
     IsStrong(weakPassword).Returns(false);
     ```

4. **Act**
   - Test memanggil:

     ```csharp
     fixture.UserManager.RestrictedUpdate(requestedUser, CurrentPassword, true)
     ```

   - `DoRestrictedUpdate`:

     - mengambil current user,
     - memverifikasi old password berhasil,
     - masuk ke block `if (hasNewPassword)`,
     - memanggil `_passwordValidator.IsStrong(weakPassword)`,
     - menerima `false`,
     - melempar `ApplicationException` dengan pesan aturan password.

   - Karena exception terjadi sebelum `currentUser.Password = user.Password` diproses lanjut ke `DoUpdate`, tidak ada hashing dan tidak ada update database.

5. **Assert**
   - Exception message harus sama dengan `WeakPasswordMessage`.
   - Validator harus dipanggil sekali dengan `weakPassword`:

     ```csharp
     fixture.PasswordValidator.Received(1).IsStrong(weakPassword);
     ```

   - Hash password baru tidak boleh dipanggil:

     ```csharp
     fixture.HashGenerator.DidNotReceive().StringToHash(Arg.Any<string>());
     ```

   - Database update tidak boleh terjadi:

     ```csharp
     fixture.DataManager.DidNotReceive().DbUpdate(...);
     ```

   - Transaction boundary:

     ```csharp
     BeginTransaction() dipanggil sekali
     Commit() tidak dipanggil
     Rollback() dipanggil sekali
     Dispose() dipanggil sekali
     ```

### 2.4 Catatan Teknis Tentang `DoRestrictedUpdate`

`DoRestrictedUpdate` adalah private method, sehingga test tidak memanggilnya langsung. Test memanggil public method:

```csharp
RestrictedUpdate(user, oldPassword, true)
```

Pendekatan ini lebih baik daripada reflection karena:

- menjaga test tetap menguji public contract class,
- tetap mencakup branch private method secara natural,
- memverifikasi transaction wrapper di public method,
- tidak membuat test rapuh terhadap refactor internal selama behavior publik tetap sama.

Branch penting yang tercakup:

| Branch | Dicakup Oleh |
|---|---|
| Old password valid | `RestrictedUpdate_WithValidCurrentPasswordAndStrongNewPassword_ShouldCommitAndUpdatePassword`, `RestrictedUpdate_WithWeakNewPassword_ShouldThrowAndRollback` |
| Old password invalid | `RestrictedUpdate_WithInvalidCurrentPassword_ShouldThrowAndRollback` |
| `hasNewPassword = true` | Semua server test change password |
| Password baru strong | `RestrictedUpdate_WithValidCurrentPasswordAndStrongNewPassword_ShouldCommitAndUpdatePassword` |
| Password baru lemah | `RestrictedUpdate_WithWeakNewPassword_ShouldThrowAndRollback` |
| Hash password baru sebelum update | `RestrictedUpdate_WithValidCurrentPasswordAndStrongNewPassword_ShouldCommitAndUpdatePassword` |
| Rollback saat gagal | Mismatch dan weak password tests |

## 3. Cara Menjalankan Test

### Client

```powershell
npm.cmd test
```

Jika PowerShell mengizinkan script execution, perintah berikut juga bisa digunakan:

```powershell
npm test
```

### Server

```powershell
dotnet test FugarServer\Fugar.Lib.Tests\Fugar.Lib.Tests.csproj --no-restore
```

Gunakan tanpa `--no-restore` jika dependency belum pernah di-restore:

```powershell
dotnet test FugarServer\Fugar.Lib.Tests\Fugar.Lib.Tests.csproj
```

## 4. Kesimpulan

Unit test client memastikan controller AngularJS menjalankan validasi lokal, membangun payload service dengan benar, menutup modal pada success callback, dan melempar `FaultError` pada error callback.

Unit test server memastikan logic security utama di `UserManager.RestrictedUpdate` berjalan benar: old password diverifikasi, password baru divalidasi, password baru di-hash sebelum update, database hanya diupdate pada jalur sukses, dan transaction commit/rollback berjalan sesuai hasil operasi.
