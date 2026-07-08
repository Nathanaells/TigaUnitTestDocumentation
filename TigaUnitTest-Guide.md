# Dokumentasi Unit Test — `JintTaskExecutor`

## 1. Latar Belakang

`JintTaskExecutor` adalah class yang menjalankan script JavaScript (workflow task) menggunakan
Jint engine, lalu mengembalikan hasilnya sebagai `object`.

Class ini **self-contained**: satu-satunya dependency eksternal yang di-inject lewat constructor
adalah `IFmlxLogger`. Engine Jint dibuat baru di dalam setiap pemanggilan `RunTask()`
(`InitializeEngine()`), jadi **tidak perlu** setup database, HTTP context, atau service TIGA
lain untuk mengetesnya.

Karena itu, test terhadap class ini bisa dijalankan sebagai **unit test / integration test
ringan yang berdiri sendiri**, tanpa perlu menjalankan aplikasi TIGA secara penuh atau memanggil
endpoint API.

## 2. Yang Perlu Di-mock

| Dependency | Perlu di-mock? | Alasan |
|---|---|---|
| `IFmlxLogger` | Ya | Satu-satunya dependency constructor. Mock supaya test tidak menulis log sungguhan. |
| `Engine` (Jint) | Tidak | Dibuat internal via `new Engine(...)`, bukan di-inject. Biarkan berjalan asli — ini bagian yang justru ingin diverifikasi (apakah script JS benar-benar jalan sesuai harapan). |
| `IServiceCollection` / DI container aplikasi | Tidak | Tidak dipakai oleh class ini sama sekali. |

Karena `Engine` tidak di-mock, test terhadap `RunTask()` secara teknis lebih dekat ke
**integration test ringan** ketimbang unit test murni — tapi ini wajar dan justru lebih
bernilai untuk scripting engine, karena tujuan utama test adalah memastikan *behavior* hasil
eksekusi JS, bukan sekadar memverifikasi urutan pemanggilan method C#.

## 3. Setup Test Project

Gunakan NUnit + NSubstitute (sesuai stack yang sudah dipakai di `JintExecutor`).

```csharp
[TestFixture]
public class JintTaskExecutorTests
{
    private IFmlxLogger _logger;
    private JintTaskExecutor _executor;

    [SetUp]
    public void SetUp()
    {
        _logger = Substitute.For<IFmlxLogger>();
        _executor = new JintTaskExecutor( _logger );
    }
}
```

## 4. Skenario Test

### 4.1 Script sederhana — ekspresi aritmatika

```csharp
[Test]
public void RunTask_SimpleArithmetic_ReturnsCorrectResult()
{
    string script = "1 + 1";
    var parameters = new Dictionary<string, object>();
    var functions = new Dictionary<string, Delegate>();

    object result = _executor.RunTask( script, parameters, functions );

    Assert.That( Convert.ToInt32( result ), Is.EqualTo( 2 ) );
}
```

### 4.2 Script dengan parameter yang dikirim ke engine

```csharp
[Test]
public void RunTask_WithParameter_UsesParameterValue()
{
    string script = "myNumber * 2";
    var parameters = new Dictionary<string, object> { { "myNumber", 5 } };
    var functions = new Dictionary<string, Delegate>();

    object result = _executor.RunTask( script, parameters, functions );

    Assert.That( Convert.ToInt32( result ), Is.EqualTo( 10 ) );
}
```

### 4.3 Script yang memanggil function C# yang di-inject (delegate)

```csharp
[Test]
public void RunTask_WithInjectedFunction_CallsFunctionCorrectly()
{
    string script = "Greet('Nathan')";
    var parameters = new Dictionary<string, object>();
    Func<string, string> greetFunc = name => $"Hello, {name}!";
    var functions = new Dictionary<string, Delegate> { { "Greet", greetFunc } };

    object result = _executor.RunTask( script, parameters, functions );

    Assert.That( result.ToString(), Is.EqualTo( "Hello, Nathan!" ) );
}
```

### 4.4 Script yang throw error biasa

```csharp
[Test]
public void RunTask_ScriptThrowsError_ThrowsTaskExecutorException()
{
    string script = "throw new Error('Something broke')";
    var parameters = new Dictionary<string, object>();
    var functions = new Dictionary<string, Delegate>();

    Assert.Throws<TaskExecutorException>( () =>
        _executor.RunTask( script, parameters, functions ) );
}
```

> Catatan: kalau ingin menguji jalur `WorkflowTaskThrowErrorException` (yang dipicu lewat
> marker `__WORKFLOW_TASK_THROW_ERROR__:`), perlu diketahui dulu function apa di
> `functionsMap` yang biasanya menulis marker tersebut (kemungkinan besar function
> `ThrowError(...)` yang di-inject dari luar) — cek pemanggil `RunTask` untuk melihat isi
> `functionsMap` yang biasa dikirim.

### 4.5 Memory limit terlampaui

```csharp
[Test]
public void RunTask_MemoryLimitExceeded_ThrowsWorkflowTaskThrowErrorException()
{
    _executor.EngineMemoryLimit = 1000; // sengaja dibuat kecil
    string script = "var arr = []; while(true) { arr.push('x'.repeat(10000)); }";
    var parameters = new Dictionary<string, object>();
    var functions = new Dictionary<string, Delegate>();

    Assert.Throws<WorkflowTaskThrowErrorException>( () =>
        _executor.RunTask( script, parameters, functions ) );
}
```

### 4.6 Verifikasi logging (opsional, hanya jika `AllowDebug = true`)

```csharp
[Test]
public void RunTask_WithDebugEnabled_LogsExecutionSteps()
{
    _executor.AllowDebug = true;
    string script = "1 + 1";
    var parameters = new Dictionary<string, object>();
    var functions = new Dictionary<string, Delegate>();

    _executor.RunTask( script, parameters, functions );

    _logger.Received().LogInfo( Arg.Any<string>() );
}
```

## 5. Hal yang Sebaiknya Dihindari

- **Jangan** mencoba mock `Engine` (Jint) — tidak di-inject lewat constructor, dan mem-verifikasi
  urutan pemanggilan internal (`InitializeEngine`, `ExtendScript`, dll) membuat test jadi
  *fragile* (gampang rusak walau behavior sebenarnya tidak berubah).
- **Fokus pada input → output/exception**, bukan pada "bagaimana caranya" — sesuai prinsip
  black-box testing.
- Jangan lupa `EngineMemoryLimit` default-nya `0` (`long` default), yang berarti tidak ada
  limit — set eksplisit di test kalau ingin menguji skenario limit terlampaui.

## 6. Cara Menjalankan

Tidak perlu menjalankan endpoint/API TIGA sama sekali. Cukup:

1. Buat/tambahkan test project (NUnit) yang mereferensikan project `Fugar.Workflow`
   (tempat `JintTaskExecutor` berada) dan package `NSubstitute`.
2. Jalankan lewat `dotnet test`, atau lewat Test Explorer di Visual Studio/VS Code.

```bash
dotnet test --filter "FullyQualifiedName~JintTaskExecutorTests"
```

Kalau ingin melihat di mana `RunTask()` sebenarnya dipanggil dari endpoint (untuk konteks,
bukan untuk keperluan testing), gunakan **Find All References** (`Shift+F12`) pada method
`RunTask` di IDE — jauh lebih cepat daripada menyusuri satu per satu di antara puluhan file.

---

## 7. Kasus Lanjutan — Testing Workflow Script yang Bergantung pada `SystemEntityManager`

### 7.1 Temuan Penting

Workflow script TIGA (misalnya `MissingWeeksReport.js`) sering memanggil:

```javascript
_systemEntityManager = new SystemEntityManager();
_systemCLRTypeCreator = new SystemCLRTypeCreator();
```

**`SystemEntityManager` dan `SystemCLRTypeCreator` ini BUKAN class C#** — mereka adalah
**file JavaScript terpisah** (`SystemEntityManager.js`, dst) yang tersimpan sebagai data workflow
di database (kolom `S_W0_WorkflowTask.P_Content`, ditandai lewat nama task
`System_SystemEntityManager` / `System_SystemCLRTypeCreator` di `S_W0_WorkflowAction`).

Di dalam aplikasi TIGA, saat sebuah `WorkflowDefinition` dijalankan, sistem **menggabungkan**
beberapa `WorkflowAction` sesuai urutan `P_Actions_Index` menjadi **satu script gabungan**
sebelum dikirim ke `JintTaskExecutor.RunTask()`. Task bernama `System_*` menyisipkan file JS
"library" (seperti `SystemEntityManager.js`), sedangkan task biasa (`Custom`) menyisipkan script
bisnis yang sebenarnya (misalnya `MissingWeeksReport.js`).

**Lapisan sebenarnya:**

```
MissingWeeksReport.js  (script bisnis / Custom task)
      ↓ memanggil
SystemEntityManager.js (JS "library", disisipkan dari task System_SystemEntityManager)
      ↓ method-methodnya (RetrieveData, CreateBusinessData, dll) memanggil
Fungsi level-rendah: NewRetrieveExtendedBusinessEntity, CreateBusinessEntity,
DeleteExtendedBusinessEntity, DbBulkCreateBusinessEntity, RetrievePropertyDefinitionMap, dll
      ↓ ini BARU benar-benar delegate C#
di-inject lewat functionsMap (IDictionary<string, Delegate>) ke JintTaskExecutor.RunTask()
```

Jadi ada **dua lapis JS** sebelum sampai ke C# asli: script bisnis → `SystemEntityManager.js`
(wrapper) → delegate C# level rendah.

### 7.2 Implikasi untuk Strategi Test

Karena `SystemEntityManager` cuma JS wrapper (bukan class C#), **jangan** coba mock dia lewat
NSubstitute atau interface apapun — tidak ada yang bisa di-mock di situ. Yang perlu di-*fake*
adalah **fungsi level rendah** yang dipanggil di dalam `SystemEntityManager.js`, karena itulah
titik di mana JS benar-benar menyentuh dunia C#/database.

Tidak perlu Node.js. Semua tetap jalan di Jint (C#) — hanya perlu:
1. Menggabungkan script `SystemCLRTypeCreator.js` + `SystemEntityManager.js` + script target
   jadi satu string sebelum dieksekusi (meniru apa yang dilakukan engine TIGA saat runtime).
2. Menyediakan `functionsMap` berisi fake C# lambda untuk fungsi level rendah yang benar-benar
   dipanggil pada jalur eksekusi yang diuji — tidak perlu fake semua method, cukup yang dipakai
   skenario test tersebut.

### 7.3 Fungsi Level-Rendah yang Umum Perlu Di-fake

Berdasarkan isi `SystemEntityManager.js`, method-methodnya memanggil fungsi C# berikut
(nama bisa berbeda tergantung skenario spesifik yang diuji — fake hanya yang benar-benar
dipanggil di jalur eksekusi):

| Fungsi level-rendah | Dipanggil oleh method `SystemEntityManager` |
|---|---|
| `NewRetrieveExtendedBusinessEntity` | `RetrieveData` |
| `NewRetrieveRelatedBusinessEntity` | `GetAllRelatedData` |
| `CreateBusinessEntity` | `CreateBusinessData` |
| `UpdateBusinessEntity` | `UpdateBusinessData` |
| `DeleteExtendedBusinessEntity` | `DeleteBusinessData` |
| `DbBulkCreateBusinessEntity` / `...Entities` | `DbBulkCreateBusinessData` |
| `DbBulkUpdateBusinessEntity` / `...Entities` | `DbBulkUpdateBusinessData` / `...Datas` |
| `DbBulkDeleteBusinessEntity` | `DbBulkDeleteBusinessData` |
| `DbBulkArchiveBusinessEntity` / `...Entities` | `BulkArchiveBusinessData` |
| `RetrievePropertyDefinitionMap` | `GetPropertyDefinitionMap` |
| `SumByCurrencyBusinessEntity` | `RetrieveSumByCurrencyData` |
| `RetrieveExtendedBusinessEntityByRawSQL` | `RetrieveDataByRawSQL` |
| `SetBulkSequenceValue` | `SetBusinessDataBulkSequenceValue` |
| `UpdateSystemDropdownStaticItems` | `UpdateDropdownStaticItems` |
| `CreateDeleteRelationshipEntity` | `LinkRelatedData` / `UnlinkRelatedData` |
| `ParseGuid`, `ParseEnum` | konversi tipe di berbagai method |

### 7.4 Contoh Struktur Test

```csharp
[Test]
public void MissingWeeksReport_Run_ReturnsExpectedMissingWeeks()
{
    // 1. Baca isi file JS "library" + script bisnis target
    string systemCLRTypeCreatorJs = File.ReadAllText( "path/to/SystemCLRTypeCreator.js" );
    string systemEntityManagerJs = File.ReadAllText( "path/to/SystemEntityManager.js" );
    string missingWeeksReportJs = File.ReadAllText( "path/to/MissingWeeksReport.js" );

    // 2. Gabungkan sesuai urutan Actions_Index workflow aslinya, lalu panggil entry point
    string combinedScript = systemCLRTypeCreatorJs + "\n"
                           + systemEntityManagerJs + "\n"
                           + missingWeeksReportJs + "\n"
                           + "var report = new MissingWeeksReport(); report.Run();";

    // 3. Parameter yang dibaca lewat p_JsonString (lihat: _customObject = eval('(' + p_JsonString + ')'))
    var parameters = new Dictionary<string, object>
    {
        ["p_JsonString"] = "{\"office\":\"Indonesia\",\"endDate\":\"2025-01-06\"}"
    };

    // 4. Fake hanya fungsi level rendah yang benar-benar dipakai jalur ini
    var functions = new Dictionary<string, Delegate>
    {
        ["Log"] = (Action<string>)( msg => { } ),
        ["ThrowError"] = (Action<string>)( msg => throw new Exception( msg ) ),
        ["ListOfType"] = (Func<Type, IList>)( type =>
        {
            var listType = typeof( List<> ).MakeGenericType( type );
            return (IList) Activator.CreateInstance( listType );
        } ),
        ["NewRetrieveExtendedBusinessEntity"] = (Func<string, bool, string, object, int, int, object, bool, bool, Tuple<long, IList<object>>>)(
            ( entityName, countOnly, filter, propMap, pageIdx, pageSize, sort, active, firstRow ) =>
                Tuple.Create( 0L, (IList<object>) BuildFakeEmployees() )
        ),
        ["RetrievePropertyDefinitionMap"] = (Func<string, object, object, bool, bool, bool, object>)(
            ( entityName, relId, viewType, extend, relabel, stringFormat ) => new Dictionary<string, string>()
        ),
    };

    // 5. Jalankan
    var executor = new JintTaskExecutor( Substitute.For<IFmlxLogger>() );
    object result = executor.RunTask( combinedScript, parameters, functions );

    // 6. Assert (result adalah JSON string, hasil dari JSON.stringify di akhir Run())
    Assert.That( result.ToString(), Does.Contain( "Weeks" ) );
}
```

> **Catatan penting:** signature `Func<...>` di atas cuma contoh kasar. Bentuk parameter dan
> return type sebenarnya harus dicocokkan dengan signature C# asli dari fungsi-fungsi tersebut
> (misalnya `NewRetrieveExtendedBusinessEntity`, `BusinessEntityBase`, dst) — perlu ditelusuri
> dulu lewat **Find All References** ke tempat fungsi-fungsi ini di-`engine.SetValue(...)`-kan
> di kode C#, supaya fake yang dibuat cocok tipenya dan tidak error saat Jint memanggilnya.

### 7.5 Cara Mencari Tempat Fungsi-fungsi Ini Didaftarkan ke Jint

Karena fungsi seperti `NewRetrieveExtendedBusinessEntity` tidak ada di file JS manapun (dia
levelnya sudah C#), cara menemukan signature aslinya:

1. Find in Files (`Ctrl+Shift+F`), scope `*.cs`, cari literal string
   `"NewRetrieveExtendedBusinessEntity"` — kemungkinan besar muncul sebagai key di pemanggilan
   `engine.SetValue( "NewRetrieveExtendedBusinessEntity", ... )` atau di dalam `functionsMap`
   yang disiapkan sebelum `RunTask` dipanggil.
2. Dari situ, cek delegate/method C# yang menjadi value-nya — itulah signature asli yang perlu
   ditiru bentuknya saat membuat fake untuk test.
3. Lakukan hal yang sama untuk setiap fungsi level rendah lain yang muncul di jalur eksekusi
   skenario yang sedang diuji (lihat tabel 7.3).

### 7.6 Temuan Lanjutan — `SystemCLRTypeCreator` Beda Kategori dari `SystemEntityManager`

Isi `SystemCLRTypeCreator.js` ternyata **bukan** memanggil delegate C# lewat `functionsMap`,
melainkan mengakses **CLR/.NET langsung** lewat fitur `AllowClr` Jint:

```javascript
var FugarModel = importNamespace('Fugar.Model');
...
return new FugarModel.BusinessEntityBase();
return System.Guid.NewGuid();
return ListOfType(System.Guid);
currencyDataType.Amount = System.Decimal.Parse( ... );
```

`importNamespace(...)`, `System.Guid`, `System.Decimal`, dll ini ditembus langsung oleh Jint ke
assembly .NET asli, berkat konfigurasi yang sudah kita lihat di `InitializeEngine()`:

```csharp
options.AllowClr( Assemblies ).CatchClrExceptions();
```

Ini **kategori dependency yang berbeda** dari fungsi level-rendah seperti
`NewRetrieveExtendedBusinessEntity` (yang di-inject lewat `functionsMap`/`engine.SetValue`):

| Kategori | Contoh | Cara tersedia di JS | Strategi test |
|---|---|---|---|
| **Real CLR langsung (via `AllowClr`)** | `System.Guid`, `System.Decimal`, `FugarModel.BusinessEntityBase`, `FugarModel.CurrencyDataType` | `importNamespace(...)`, akses tipe .NET langsung | **Tidak perlu di-fake.** Cukup pastikan `Engine.Assemblies` yang di-pass ke `JintTaskExecutor` mencakup assembly yang berisi `Fugar.Model`. Objek yang ter-construct adalah instance asli (kemungkinan besar POCO ringan tanpa dependency database), aman dipakai apa adanya di test. |
| **Bridge ke database (via `functionsMap`)** | `NewRetrieveExtendedBusinessEntity`, `CreateBusinessEntity`, `DeleteExtendedBusinessEntity`, dll | `engine.SetValue( "NamaFungsi", delegate )` sebelum script dijalankan | **Wajib di-fake** — kalau tidak, akan mencoba benar-benar menyambung ke database/NHibernate saat test berjalan. |

Implikasinya: `SystemCLRTypeCreator.js` bisa **ikut digabung apa adanya** ke `combinedScript`
tanpa perlu fake tambahan sama sekali — cukup pastikan property `Assemblies` di
`JintTaskExecutor` diarahkan ke assembly yang memuat namespace `Fugar.Model`:

```csharp
var executor = new JintTaskExecutor( Substitute.For<IFmlxLogger>() )
{
    Assemblies = typeof( Fugar.Model.BusinessEntityBase ).Assembly  // pastikan namespace Fugar.Model bisa diakses
};
```

Kalau `Assemblies` tidak di-set dengan benar, script akan gagal di baris
`importNamespace('Fugar.Model')` atau saat construct `new FugarModel.BusinessEntityBase()`,
bukan gagal karena kurang fake — jadi kalau ketemu error semacam ini di test, cek dulu konfigurasi
`Assemblies`, bukan buru-buru menambah entry baru di `functionsMap`.

### 7.7 Ringkasan Strategi

- **Tidak perlu Node.js** — seluruh eksekusi tetap di Jint (C#), sesuai arsitektur aslinya.
- **Tidak perlu NSubstitute** untuk lapisan `functionsMap` — tidak ada interface C# yang
  di-inject di sini, hanya `IDictionary<string, Delegate>`. Fake cukup berupa lambda C# manual.
- **Pisahkan dua kategori dependency dengan jelas:**
  - `SystemCLRTypeCreator.js` → akses CLR **langsung** (`AllowClr`/`importNamespace`) →
    **tidak perlu di-fake**, cukup pastikan `Assemblies` mengarah ke assembly `Fugar.Model`.
  - `SystemEntityManager.js` → memanggil fungsi level-rendah lewat `functionsMap` yang
    menyentuh database → **wajib di-fake** agar test tidak benar-benar nyambung ke database.
- **Fokus fake hanya pada fungsi level-rendah bridge-database**, bukan pada
  `SystemEntityManager`/`SystemCLRTypeCreator` itu sendiri — keduanya cuma JS biasa yang bisa
  dijalankan apa adanya asal dependency di bawahnya sudah tersedia (baik lewat `functionsMap`
  maupun `Assemblies`).
- Gabungkan script JS "library" (`SystemCLRTypeCreator.js` + `SystemEntityManager.js`) + script
  bisnis target jadi satu string sebelum eksekusi, meniru proses penggabungan `WorkflowAction`
  yang dilakukan sistem TIGA saat runtime.
