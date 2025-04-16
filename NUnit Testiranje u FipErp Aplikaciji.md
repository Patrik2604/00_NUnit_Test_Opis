# NUnit Testiranje u FipErp Aplikaciji

## Dodavanje NUnit test projekta u solution

Prije nego što možemo pisati i izvršavati testove, potrebno je dodati test projekt u solution. Evo koraka kako to učiniti:

1. **Preduvjeti**:
   - Instalirani Visual Studio (2019 ili noviji)
   - .NET SDK (verzija koja odgovara aplikaciji koju testiramo)

2. **Dodavanje NUnit test projekta**:
   - Desni klik na solution u Solution Explorer-u
   - Odabir "Add" > "New Project"
   - U dijalogu za odabir predloška projekta, pretraži "NUnit"
   - Odaberi "NUnit Test Project" (.NET Core ili .NET 5+, ovisno o verziji aplikacije)
   - Dodjeli projektu ime (npr. "FipErp.Tests") i klikni "Create"

3. **Instaliranje potrebnih NuGet paketa**:
   - NUnit (obično već uključen u predložak)
   - NUnit3TestAdapter (za izvršavanje testova u Visual Studio-u)
   - Microsoft.NET.Test.Sdk (za izvršavanje testova)
   - Moq (za mockanje zavisnosti) - library koji omogućuje kreiranje "mock" objekata za simulaciju ponašanja stvarnih zavisnosti. Mockanje zavisnosti je tehnika koja omogućava testiranje komponente u izolaciji tako da se stvarne zavisnosti (npr. servisi, repozitoriji, baze podataka) zamijene s kontroliranim simulacijama koje imaju predvidljivo ponašanje. Ovo je korisno jer:
     - Omogućava testiranje komponente bez ovisnosti o vanjskim sustavima
     - Ubrzava testove jer nema stvarnih poziva bazi podataka ili API-jima
     - Omogućuje simulaciju specifičnih scenarija koji bi bili teško izvedivi s pravim objektima
     - Omogućuje verifikaciju interakcija između testirane komponente i njezinih zavisnosti
   - Ostali paketi specifični za projekt (npr. EntityFrameworkCore.InMemory za testiranje s bazom podataka)

   Pakete možete instalirati kroz NuGet Package Manager ili sa Package Manager Console (ako se ne preuzmu automatski):
   ```
   Install-Package NUnit
   Install-Package NUnit3TestAdapter
   Install-Package Microsoft.NET.Test.Sdk
   Install-Package Moq
   ```

4. **Dodavanje reference na projekt koji testiramo (mogu se automatski dodavati prilikom pisanja testova)**:
   - Desni klik na projekt u Solution Explorer-u
   - Odabir "Add" > "Project Reference"
   - Odabir projekta koji sadrži kod koji želimo testirati (npr. "FipErp.API")

5. **Struktura test projekta**:
   - Preporučuje se organizirati testove u strukturi koja odražava strukturu izvornog koda
   - Na primjer, za kontroler `PredmetController` kreiramo test klasu `PredmetControllerTests`
   - Testove možemo grupirati u folder-e prema komponentama (Controllers, Services, Repositories, itd.)

## Uvod u NUnit testiranje

NUnit je jedan od najkorištenijih okvira za testiranje u .NET razvojnom okruženju. U priloženom kodu vidimo primjere unit testova za FipErp aplikaciju. Prije nego što detaljno analiziramo prikazane testove, važno je razumjeti nekoliko ključnih koncepata i atributa koje NUnit koristi:

### Ključni NUnit atributi

- **[TestFixture]** - označava klasu koja sadrži testove
- **[SetUp]** - metoda označena ovim atributom izvršava se prije svakog testa
- **[TearDown]** - metoda označena ovim atributom izvršava se nakon svakog testa
- **[Test]** - označava metodu koja je testni slučaj
- **[TestCase]** - omogućuje izvršavanje istog testa s različitim parametrima

### Moq framework

U primjeru vidimo korištenje Moq frameworka za kreiranje "mock" objekata. "Mockanje" je tehnika gdje stvaramo simulirane verzije zavisnih komponenti koje se ponašaju na predvidljiv način, što nam omogućuje da izoliramo samo onaj dio koda koji želimo testirati.

Moq omogućuje:
1. **Kreiranje mock objekata** - `var mockService = new Mock<IService>()`
2. **Konfiguraciju ponašanja** - `mockService.Setup(s => s.GetData()).Returns(testData)`
3. **Verifikaciju interakcija** - `mockService.Verify(s => s.SaveData(It.IsAny<Data>()), Times.Once)`
4. **Simulaciju izuzetaka** - `mockService.Setup(...).Throws(new Exception())`
5. **Praćenje poziva metoda** - pomoću `Callback` metode

## Analiza PredmetControllerTests klase

### Priprema testnog okruženja (SetUp)

Klasa `PredmetControllerTests` testira funkcionalnost `PredmetController`-a. Primjećujemo da kontroler ima mnogo dependecy koje se u testu zamjenjuju mock objektima:

```csharp
[SetUp]
public void Setup()
{
    _mockPredmetService = new Mock<IPredmetService>();
    // ... ostali mock objekti ...
    
    _controller = new PredmetController(
        _mockPredmetService.Object,
        // ... ostali objekti ...
    );
}
```

Ovo je primjer "dependency injection" principa gdje se dependecy prosljeđuju kontroleru umjesto da ih on sam stvara. To nam omogućuje da u testu kontroliramo ponašanje tih zavisnosti.

## Pojašnjenje primjera iz danog koda

### Primjer 1: DohvatiPredmeteZaUrudzbeni_ReturnsOkResult

```csharp
[Test]
public void DohvatiPredmeteZaUrudzbeni_ReturnsOkResult()
{
    // Arrange
    int urudzbeniID = 1;
    bool klasaPlan = true;
    var predmeti = new List<Predmet> { new Predmet() };
    _mockUrudzbeniService.Setup(s => s.ReadByID(urudzbeniID)).Returns(new Urudzbeni { PlanKlasifikacija = klasaPlan });
    _mockPredmetService.Setup(s => s.DohvatiPredmeteZaUrudzbeni(urudzbeniID, klasaPlan)).Returns(predmeti);

    // Act
    var result = _controller.DohvatiPredmeteZaUrudzbeni(urudzbeniID) as OkObjectResult;

    // Assert
    Assert.Multiple(() =>
    {
        Assert.That(result, Is.Not.Null);
        if (result != null)
        { 
            Assert.That(result.StatusCode, Is.EqualTo(200));
            Assert.That(result.Value, Is.EqualTo(predmeti));
        }
    });
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):** 
   - Definiramo testne podatke: `urudzbeniID = 1` i `klasaPlan = true`
   - Kreiramo testnu listu predmeta s jednim praznim predmetom
   - Konfiguriramo `_mockUrudzbeniService` da kada se pozove njegova metoda `ReadByID` s parametrom `urudzbeniID`, vrati novi objekt `Urudzbeni` s postavljenom vrijednosti `PlanKlasifikacija = klasaPlan`
   - Konfiguriramo `_mockPredmetService` da kada se pozove njegova metoda `DohvatiPredmeteZaUrudzbeni` s parametrima `urudzbeniID` i `klasaPlan`, vrati našu testnu listu predmeta

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera koju testiramo: `_controller.DohvatiPredmeteZaUrudzbeni(urudzbeniID)`
   - Rezultat castamo u `OkObjectResult` što nam omogućuje pristup statusnom kodu i vrijednosti odgovora

3. **Provjera (Assert):**
   - Koristimo `Assert.Multiple` koji omogućuje izvođenje više provjera u jednom bloku
   - Prvo provjeravamo da rezultat nije null (`Assert.That(result, Is.Not.Null)`)
   - Ako rezultat nije null, provjeravamo:
     - Da je statusni kod 200 (`Assert.That(result.StatusCode, Is.EqualTo(200))`)
     - Da je vraćena vrijednost jednaka našoj testnoj listi predmeta (`Assert.That(result.Value, Is.EqualTo(predmeti))`)

Ovaj test provjerava da kada pozovemo metodu `DohvatiPredmeteZaUrudzbeni` s valjanim ID-om, očekujemo da kontroler:
1. Dohvati odgovarajući urudžbeni objekt putem servisa
2. Dohvati predmete povezane s tim urudžbenim brojem
3. Vrati te predmete u HTTP OK odgovoru (status 200)

### Primjer 2: DohvatiPredmeteZaUrudzbeniIStatus_ReturnsOkResult

```csharp
[Test]
public void DohvatiPredmeteZaUrudzbeniIStatus_ReturnsOkResult()
{
    // Arrange
    int urudzbeniID = 1;
    int statusID = 1;
    var predmeti = new List<Predmet> { new Predmet() };
    _mockPredmetService.Setup(s => s.DohvatiPredmeteZaUrudzbeniIStatus(urudzbeniID, statusID)).Returns(predmeti.AsQueryable());

    // Act
    var result = _controller.DohvatiPredmeteZaUrudzbeniIStatus(urudzbeniID, statusID) as OkObjectResult;

    // Assert
    Assert.Multiple(() =>
    {
        Assert.That(result, Is.Not.Null);
        if (result != null)
        { 
            Assert.That(result.Value, Is.EqualTo(predmeti));
            Assert.That(result.StatusCode, Is.EqualTo(200));
        }
    });
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):**
   - Definiramo testne podatke: `urudzbeniID = 1` i `statusID = 1`
   - Kreiramo testnu listu predmeta s jednim praznim predmetom
   - Konfiguriramo `_mockPredmetService` da kada se pozove njegova metoda `DohvatiPredmeteZaUrudzbeniIStatus` s parametrima `urudzbeniID` i `statusID`, vrati našu testnu listu predmeta kao `IQueryable` kolekciju 
   (`.AsQueryable()` pretvara listu u tip koji omogućuje LINQ upite)

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera koju testiramo: `_controller.DohvatiPredmeteZaUrudzbeniIStatus(urudzbeniID, statusID)`
   - Rezultat castamo u `OkObjectResult`

3. **Provjera (Assert):**
   - Koristimo `Assert.Multiple` za više provjera
   - Prvo provjeravamo da rezultat nije null
   - Ako rezultat nije null, provjeravamo:
     - Da je vraćena vrijednost jednaka našoj testnoj listi predmeta
     - Da je statusni kod 200

Ovaj test provjerava da metoda `DohvatiPredmeteZaUrudzbeniIStatus` ispravno dohvaća predmete filtrirane po urudžbenom broju i statusu, te ih vraća u HTTP OK odgovoru.

### Primjer 3: DohvatiPredmeteZaSearchParam_ReturnsOkResult

```csharp
[Test]
public void DohvatiPredmeteZaSearchParam_ReturnsOkResult()
{
    // Arrange
    string upit = "test";
    var predmeti = new List<Predmet> { new Predmet { Naziv = "test" } };
    var spotlightObjects = new List<SpotlightObject>
    {
        new SpotlightObject { predmetID = predmeti.First().ID, name = "test - ", icon = "object-group" }
    };

    _mockPredmetService.Setup(s => s.ReadAllActive(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<List<object>>(), It.IsAny<int>())).Returns(predmeti.AsQueryable());

    // Act
    var result = _controller.DohvatiPredmeteZaSearchParam(upit) as OkObjectResult;

    // Assert
    Assert.Multiple(() =>
    {
        Assert.That(result, Is.Not.Null);
        if (result != null)
        {
            Assert.That(result.StatusCode, Is.EqualTo(200));
            //Assert.That((List<SpotlightObject>)result.Value, Has.Count.EqualTo(spotlightObjects.Count));
        }
    });
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):**
   - Definiramo testni upit: `upit = "test"`
   - Kreiramo testnu listu predmeta s jednim predmetom koji ima naziv "test"
   - Kreiramo testnu listu `spotlightObjects` koja predstavlja očekivani format rezultata
   - Konfiguriramo `_mockPredmetService` da kada se pozove njegova metoda `ReadAllActive` s bilo kojim parametrima (`It.IsAny<...>()`), vrati našu testnu listu predmeta

   Zanimljivo je uočiti korištenje `It.IsAny<type>()` - to je Moq funkcionalnost koja omogućuje da specificirano da nas ne zanima točna vrijednost parametra, već samo želimo da mock reagira na bilo koju vrijednost tog tipa.

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera koju testiramo: `_controller.DohvatiPredmeteZaSearchParam(upit)`

3. **Provjera (Assert):**
   - Provjeravamo da rezultat nije null
   - Provjeravamo da je statusni kod 200
   - Vidimo da postoji zakomentirana linija koja bi provjerila broj elemenata u rezultatu (`Has.Count.EqualTo(spotlightObjects.Count)`)

Ovaj test provjerava da metoda `DohvatiPredmeteZaSearchParam` ispravno pretražuje predmete prema zadanom upitu i vraća rezultate u očekivanom formatu.


## Analiza WorkflowConfigControllerTests klase

Ova klasa pokazuje kako testirati controller koji upravlja konfiguracijama workflow-a.

### Priprema testnog okruženja (SetUp)

```csharp
[SetUp]
public void Setup()
{
    _mockWorkflowConfigService = new Mock<IWorkflowConfigService>();
    _mockWorkflowConfigRepository = new Mock<IWorkflowConfigRepository>();
    _mockKorisnikService = new Mock<IKorisnikService>();
    _mockHostingEnvironment = new Mock<IWebHostEnvironment>();
    _mockLogger = new Mock<ICustomLoggerService>();
    _mockCurrentHttpRequestContext = new Mock<CurrentHttpRequestContext>();

    _testKorisnik = new Korisnik
    {
        ID = 1,
        KorisnickoIme = "testUser",
        KompanijaID = 1,
    };

    typeof(CurrentHttpRequestContext)
        .GetProperty(nameof(CurrentHttpRequestContext.PrijavljeniKorisnik))
        .SetValue(_mockCurrentHttpRequestContext.Object, _testKorisnik);

    _controller = new WorkflowConfigController(
        _mockWorkflowConfigService.Object,
        _mockWorkflowConfigRepository.Object,
        _mockKorisnikService.Object,
        _mockHostingEnvironment.Object,
        _mockLogger.Object,
        _mockCurrentHttpRequestContext.Object
    );
}
```

Setup metoda stvara mock objekte za sve dependecy kontrolera. Zanimljivo je primijetiti kako testovi simuliraju prijavljenog korisnika:
1. Stvara se testni korisnik `_testKorisnik`
2. Koristi se refleksija (`typeof(CurrentHttpRequestContext).GetProperty...`) da se postavi svojstvo prijavljenog korisnika na mock objektu
3. Ovime osiguravamo da kontroler ima pristup kontekstu prijavljenog korisnika kao u stvarnoj aplikaciji

### Čišćenje resursa (TearDown i IDisposable)

Klasa implementira `IDisposable` sučelje i ima `TearDown` metodu, što pokazuje pravilno upravljanje resursima u testovima:

```csharp
[TearDown]
public void TearDown()
{
    _controller?.Dispose();
}

public void Dispose()
{
    _controller?.Dispose();
}
```

Ovo garantira da će se kontroler pravilno osloboditi bez obzira na način završetka testa.

### Primjer 1: UcitajWorkflowConfigZaGrid_ReturnsCorrectData

```csharp
[Test]
public void UcitajWorkflowConfigZaGrid_ReturnsCorrectData()
{
    // Arrange
    var expectedConfigs = new List<WorkflowConfig>
    {
        new WorkflowConfig { ID = 1, Naziv = "Config 1" },
        new WorkflowConfig { ID = 2, Naziv = "Config 2" }
    };

    _mockWorkflowConfigService.Setup(s => s.UcitajWorkflowConfigZaGrid()).Returns(expectedConfigs);

    // Act
    var result = _controller.UcitajWorkflowConfigZaGrid();

    // Assert
    Assert.That(result, Is.Not.Null);
    Assert.That(result.Count, Is.EqualTo(2));
    Assert.That(result, Is.EqualTo(expectedConfigs));
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):** 
   - Kreiramo listu s dvije konfiguracije workflow-a
   - Konfiguriramo `_mockWorkflowConfigService` tako da kada se pozove metoda `UcitajWorkflowConfigZaGrid`, vrati našu očekivanu listu konfiguracija

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera koju testiramo: `_controller.UcitajWorkflowConfigZaGrid()`

3. **Provjera (Assert):**
   - Potvrđujemo da rezultat nije null
   - Provjeravamo da rezultat sadrži očekivani broj elemenata (2)
   - Provjeravamo da je vraćeni rezultat jednak očekivanoj listi konfiguracija

Ovaj test verificira da metoda kontrolera `UcitajWorkflowConfigZaGrid` ispravno dohvaća konfiguracije workflow-a od servisa i vraća ih klijentu.

### Primjer 2: DodajWorkflowConfig_WithValidData_ReturnsOkResult

```csharp
[Test]
public void DodajWorkflowConfig_WithValidData_ReturnsOkResult()
{
    // Arrange
    var koordinate = new List<WorkflowConfigPotpisiKoordinate>
    {
        new WorkflowConfigPotpisiKoordinate { ID = 1, X = 10, Y = 20 }
    };

    var workflowConfig = new WorkflowConfig
    {
        ID = 0,
        Naziv = "Novi Config",
        PotpisiKoordinate = koordinate
    };

    var createdConfig = new WorkflowConfig
    {
        ID = 1,
        Naziv = "Novi Config"
    };

    _mockWorkflowConfigService.Setup(s => s.CreateID(It.IsAny<Korisnik>(), It.IsAny<WorkflowConfig>()))
        .Returns(createdConfig);

    _mockWorkflowConfigService.Setup(s => s.KreirajAzurirajKoordinate(It.IsAny<List<WorkflowConfigPotpisiKoordinate>>(), It.IsAny<WorkflowConfig>()))
        .Returns(koordinate);

    // Act
    var actionResult = _controller.DodajWorkflowConfig(workflowConfig);
    var result = actionResult as OkObjectResult;
    var returnedConfig = result?.Value as WorkflowConfig;

    // Assert
    Assert.That(result, Is.Not.Null);
    Assert.That(result.StatusCode, Is.EqualTo(200));
    Assert.That(returnedConfig, Is.Not.Null);
    Assert.That(returnedConfig.ID, Is.EqualTo(1));
    Assert.That(returnedConfig.Naziv, Is.EqualTo("Novi Config"));
    Assert.That(returnedConfig.PotpisiKoordinate, Is.EqualTo(koordinate));

    _mockWorkflowConfigService.Verify(s => s.CreateID(It.IsAny<Korisnik>(), It.IsAny<WorkflowConfig>()), Times.Once);
    _mockWorkflowConfigService.Verify(s => s.KreirajAzurirajKoordinate(It.IsAny<List<WorkflowConfigPotpisiKoordinate>>(), It.IsAny<WorkflowConfig>()), Times.Once);
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):** 
   - Kreiramo listu koordinata za potpise u radnom tijeku
   - Pripremamo objekt konfiguracije workflow-a (`workflowConfig`) s ID=0 što označava novu konfiguraciju
   - Pripremamo objekt `createdConfig` koji predstavlja konfiguraciju nakon što je spremljena u bazu (s dodijeljenim ID=1)
   - Konfiguriramo mock servisa:
     - `CreateID` metoda će vratiti objekt s dodijeljenim ID-om
     - `KreirajAzurirajKoordinate` metoda će vratiti našu listu koordinata

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera koja dodaje novu konfiguraciju: `_controller.DodajWorkflowConfig(workflowConfig)`
   - Castamo rezultat u `OkObjectResult` da bismo mogli provjeriti statusni kod i vraćenu vrijednost
   - Izvlačimo vraćenu konfiguraciju iz rezultata

3. **Provjera (Assert):**
   - Provjeravamo da je rezultat tipa `OkObjectResult` i da nije null
   - Provjeravamo da je statusni kod 200 (OK)
   - Provjeravamo da vraćena konfiguracija ima očekivani ID (1) i naziv
   - Provjeravamo da su vraćene koordinate jednake onima koje smo poslali
   - Pomoću `Verify` metode provjeravamo da su se metode servisa pozvale točno jednom (`Times.Once`)

Ovaj test provjerava cjelokupni tok dodavanja nove konfiguracije workflow-a, od kreiranja objekta, preko validacije, do poziva odgovarajućih servisa i vraćanja ispravnog HTTP odgovora.

### Primjer 3: DodajWorkflowConfig_WithInvalidData_ReturnsBadRequest

```csharp
[Test]
public void DodajWorkflowConfig_WithInvalidData_ReturnsBadRequest()
{
    // Arrange
    WorkflowConfig nullConfig = null;
    _controller.ModelState.AddModelError("Error", "Model error");

    // Act
    var result = _controller.DodajWorkflowConfig(nullConfig) as BadRequestObjectResult;

    // Assert
    Assert.That(result, Is.Not.Null);
    Assert.That(result.StatusCode, Is.EqualTo(400));
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):** 
   - Postavimo `nullConfig` na null vrijednost da simuliramo nedostatak podataka
   - Ručno dodajemo grešku u `ModelState` kontrolera, što simulira grešku validacije

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera s neispravnim podacima: `_controller.DodajWorkflowConfig(nullConfig)`
   - Castamo rezultat u `BadRequestObjectResult`

3. **Provjera (Assert):**
   - Provjeravamo da je rezultat tipa `BadRequestObjectResult` i da nije null
   - Provjeravamo da je statusni kod 400 (Bad Request)

Ovaj test provjerava "negativni scenarij" kada kontroler dobije neispravne podatke. To je važan aspekt testiranja jer osigurava da API ispravno odbija neispravne zahtjeve s odgovarajućim statusnim kodom.

### Primjer 4: KopirajWorkflowConfig_WithValidData_ReturnsOkResult

```csharp
[Test]
public void KopirajWorkflowConfig_WithValidData_ReturnsOkResult()
{
    // Arrange
    var koordinate = new List<WorkflowConfigPotpisiKoordinate>
    {
        new WorkflowConfigPotpisiKoordinate { ID = 1, X = 10, Y = 20 }
    };

    var tocke = new List<WorkflowConfigTocka>
    {
        new WorkflowConfigTocka { ID = 1, Naziv = "Točka 1", RedniBroj = 1 }
    };

    var workflowConfig = new WorkflowConfig
    {
        ID = 0,
        Naziv = "Kopirani Config",
        PotpisiKoordinate = koordinate
    };

    var copyClass = new WorkflowConfigController.WorkflowCopyClass
    {
        workflowConfig = workflowConfig,
        tockeZaKopiranje = tocke
    };

    var createdConfig = new WorkflowConfig
    {
        ID = 2,
        Naziv = "Kopirani Config"
    };

    _mockWorkflowConfigService.Setup(s => s.CreateID(It.IsAny<Korisnik>(), It.IsAny<WorkflowConfig>()))
        .Returns(createdConfig);

    _mockWorkflowConfigService.Setup(s => s.KreirajAzurirajKoordinate(It.IsAny<List<WorkflowConfigPotpisiKoordinate>>(), It.IsAny<WorkflowConfig>()))
        .Returns(koordinate);

    // Act
    var actionResult = _controller.KopirajWorkflowConfig(copyClass);
    var result = actionResult as OkObjectResult;
    var returnedConfig = result?.Value as WorkflowConfig;

    // Assert
    Assert.That(result, Is.Not.Null);
    Assert.That(result.StatusCode, Is.EqualTo(200));
    Assert.That(returnedConfig, Is.Not.Null);
    Assert.That(returnedConfig.ID, Is.EqualTo(2));

    _mockWorkflowConfigService.Verify(s => s.CreateID(It.IsAny<Korisnik>(), It.IsAny<WorkflowConfig>()), Times.Once);
    _mockWorkflowConfigService.Verify(s => s.KreirajAzurirajKoordinate(It.IsAny<List<WorkflowConfigPotpisiKoordinate>>(), It.IsAny<WorkflowConfig>()), Times.Once);
    _mockWorkflowConfigService.Verify(s => s.KopirajWorkflowTocke(returnedConfig.ID, tocke), Times.Once);
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):** 
   - Kreiramo liste koordinata i točaka workflow-a
   - Pripremamo objekt konfiguracije workflow-a (`workflowConfig`) s ID=0
   - Stvaramo poseban objekt `copyClass` koji sadrži konfiguraciju i točke za kopiranje
   - Konfiguriramo mock servisa za stvaranje nove konfiguracije i ažuriranje koordinata

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera koja kopira konfiguraciju: `_controller.KopirajWorkflowConfig(copyClass)`
   - Castamo rezultat da dobijemo statusni kod i vraćene podatke

3. **Provjera (Assert):**
   - Provjeravamo da je rezultat uspješan s kodom 200
   - Provjeravamo da vraćena konfiguracija ima očekivani ID (2)
   - Pomoću `Verify` metode provjeravamo da su se metode servisa pozvale točno jednom:
     - `CreateID` - za stvaranje nove konfiguracije
     - `KreirajAzurirajKoordinate` - za kopiranje koordinata
     - `KopirajWorkflowTocke` - specifično za kopiranje točaka workflow-a

Ovaj test je posebno zanimljiv jer testira kompleksniju funkcionalnost kopiranja cijele konfiguracije workflow-a s pripadajućim koordinatama i točkama.

### Primjer 5: AzurirajWorkflowConfig_WithValidData_ReturnsOkResult

```csharp
[Test]
public void AzurirajWorkflowConfig_WithValidData_ReturnsOkResult()
{
    // Arrange
    var koordinate = new List<WorkflowConfigPotpisiKoordinate>
    {
        new WorkflowConfigPotpisiKoordinate { ID = 1, X = 10, Y = 20 }
    };

    var datoteke = new List<WorkflowConfigDatoteke>
    {
        new WorkflowConfigDatoteke { ID = 1, Naziv = "Datoteka 1" }
    };

    var workflowConfig = new WorkflowConfig
    {
        ID = 1,
        Naziv = "Ažurirani Config",
        PotpisiKoordinate = koordinate,
        WorkflowConfigDatoteke = datoteke
    };

    _mockWorkflowConfigService.Setup(s => s.UpdateID(It.IsAny<WorkflowConfig>()))
        .Returns(workflowConfig);

    _mockWorkflowConfigService.Setup(s => s.KreirajAzurirajKoordinate(It.IsAny<List<WorkflowConfigPotpisiKoordinate>>(), It.IsAny<WorkflowConfig>()))
        .Returns(koordinate);

    _mockWorkflowConfigService.Setup(s => s.KreirajAzurirajDatoteke(It.IsAny<List<WorkflowConfigDatoteke>>(), It.IsAny<WorkflowConfig>()))
        .Returns(datoteke);

    // Act
    var result = _controller.AzurirajWorkflowConfig(workflowConfig) as OkResult;

    // Assert
    Assert.That(result, Is.Not.Null);
    Assert.That(result.StatusCode, Is.EqualTo(200));

    _mockWorkflowConfigService.Verify(s => s.UpdateID(It.IsAny<WorkflowConfig>()), Times.Once);
    _mockWorkflowConfigService.Verify(s => s.KreirajAzurirajKoordinate(It.IsAny<List<WorkflowConfigPotpisiKoordinate>>(), It.IsAny<WorkflowConfig>()), Times.Once);
    _mockWorkflowConfigService.Verify(s => s.KreirajAzurirajDatoteke(It.IsAny<List<WorkflowConfigDatoteke>>(), It.IsAny<WorkflowConfig>()), Times.Once);
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):** 
   - Kreiramo liste koordinata i datoteka
   - Pripremamo objekt konfiguracije workflow-a (`workflowConfig`) s ID=1, što označava postojeću konfiguraciju
   - Konfiguriramo mock servisa za ažuriranje konfiguracije, koordinata i datoteka

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera koja ažurira konfiguraciju: `_controller.AzurirajWorkflowConfig(workflowConfig)`
   - Castamo rezultat u `OkResult`

3. **Provjera (Assert):**
   - Provjeravamo da je rezultat uspješan s kodom 200
   - Pomoću `Verify` metode provjeravamo da su se metode servisa pozvale točno jednom:
     - `UpdateID` - za ažuriranje glavnog objekta konfiguracije
     - `KreirajAzurirajKoordinate` - za ažuriranje koordinata
     - `KreirajAzurirajDatoteke` - za ažuriranje povezanih datoteka

Ovaj test pokazuje postupak ažuriranja postojeće konfiguracije workflow-a i njezinih povezanih entiteta.

### Primjer 6: UcitajZaCBO_ReturnsCorrectData

```csharp
[Test]
public void UcitajZaCBO_ReturnsCorrectData()
{
    // Arrange
    var workflowFetch = new WorkflowFetch();
    var expectedConfigs = new List<WorkflowConfig>
    {
        new WorkflowConfig { ID = 1, Naziv = "Config 1" },
        new WorkflowConfig { ID = 2, Naziv = "Config 2" }
    };

    _mockWorkflowConfigService.Setup(s => s.UcitajZaCBO(workflowFetch)).Returns(expectedConfigs);

    // Act
    var result = _controller.UcitajZaCBO(workflowFetch);

    // Assert
    Assert.That(result, Is.Not.Null);
    Assert.That(result.Count, Is.EqualTo(2));
    Assert.That(result, Is.EqualTo(expectedConfigs));
}
```

**Detaljno objašnjenje:**
1. **Prepare (Arrange):** 
   - Kreiramo objekt `workflowFetch` koji sadrži parametre za dohvaćanje podataka
   - Pripremamo očekivanu listu konfiguracija
   - Konfiguriramo mock servisa da vrati očekivane podatke

2. **Izvršavanje (Act):**
   - Pozivamo metodu kontrolera: `_controller.UcitajZaCBO(workflowFetch)`

3. **Provjera (Assert):**
   - Provjeravamo da rezultat nije null
   - Provjeravamo da rezultat sadrži očekivani broj elemenata (2)
   - Provjeravamo da je rezultat jednak očekivanoj listi

Ovaj test verificira funkcionalnost dohvaćanja konfiguracija workflow-a specifično za CBO (ComboBox) komponentu korisničkog sučelja.

## Usporedba WorkflowConfigControllerTests i PredmetControllerTests

Promatrajući ove dvije testne klase, možemo uočiti nekoliko važnih obrazaca u testiranju web API kontrolera:

1. **Struktura testova**: Obje klase koriste `SetUp` za inicijalizaciju mock objekata i kontrolera, imaju jasno razdvojene faze Prepare-Execute-Assert, te koriste odgovarajuće NUnit atribute za označavanje testova.

2. **Testiranje HTTP odgovora**: Testovi provjeravaju statusne kodove odgovora (200 OK, 400 Bad Request) i strukture vraćenih podataka.

3. **Verifikacija poziva servisa**: Kroz `Verify` metode provjerava se da su odgovarajuće metode servisa pozvane s očekivanim parametrima i očekivani broj puta.

4. **Testiranje pozitivnih i negativnih scenarija**: Oba primjera pokazuju testiranje "sretnog puta" kada sve funkcionira kako treba, kao i testiranje scenarija kada podaci nisu ispravni ili dođe do drugih problema.

## Analiza PredmetFrontendTests klase - E2E test

```csharp
[Test]
[TestCase("patrik", "1", "http://localhost:4200")]
[TestCase("1", "1", "http://192.168.11.42:11111")]
public async Task UrudzbeniPregled_Test(string username, string password, string ipLokacija)
{
    await Shared.PerformLogin(Page, username, password, ipLokacija + "/login");
    // Sada navigirajte na zaštićenu stranicu ako je potrebno
    await Page.GotoAsync(ipLokacija + "/urudzbeni-file-tree");
    await Page.WaitForNavigationAsync();
}

public static async Task PerformLogin(IPage page, string username, string password, string ipLokacija)
{
    // Navigirajte na stranicu za prijavu
    await page.GotoAsync(ipLokacija);
    // Pričekajte da polje za korisničko ime bude dostupno s povećanim vremenom čekanja
    await page.WaitForSelectorAsync("[formControlName='username']");
    // Ispunite obrazac za prijavu
    await page.FillAsync("[formControlName='username']", username);
    await page.FillAsync("[formControlName='password']", password);
    // Provjerite postoji li gumb za prijavu
    var loginButtonExists = await page.QuerySelectorAsync("button:has-text('prijava')");
    Assert.IsNotNull(loginButtonExists, "Gumb za prijavu ne postoji.");
    // Provjerite je li gumb za prijavu onemogućen
    var isLoginButtonDisabled = await page.EvalOnSelectorAsync<bool>("button:has-text('prijava')", "button => button.disabled");
    Assert.IsFalse(isLoginButtonDisabled, "Gumb za prijavu je onemogućen.");
    // Zabilježite sve konzolne pogreške
    page.Console += (_, msg) => Console.WriteLine($"Poruka: {msg.Text}");
    // Pričekajte da gumb bude omogućen i kliknite ga
    await page.WaitForSelectorAsync("button:has-text('prijava'):not([disabled])");
    await page.ClickAsync("button:has-text('prijava'):not([disabled])");
    // Pričekajte navigaciju na glavnu stranicu nakon prijave
    await page.WaitForNavigationAsync();
}
```

**Detaljno objašnjenje:**
1. **Priprema i parametrizacija:**
   - Test je označen s dva `[TestCase]` atributa, što znači da će se izvršiti dva puta s različitim parametrima
   - Prvi slučaj: korisničko ime "patrik", lozinka "1", lokalna adresa "http://localhost:4200"
   - Drugi slučaj: korisničko ime "1", lozinka "1", adresa na internoj mreži "http://192.168.11.42:11111"

2. **Izvršavanje testa:**
   - Prvo se poziva `Shared.PerformLogin` metoda koja obavlja prijavu korisnika
   - Nakon prijave, test navigira na stranicu za pregled urudžbenih zapisa (`/urudzbeni-file-tree`)
   - Test čeka da se navigacija završi (`await Page.WaitForNavigationAsync()`)

3. **Implementacija PerformLogin metode:**
   - Metoda navigira na URL prijave
   - Čeka da se pojavi polje za unos korisničkog imena
   - Ispunjava obrazac s korisničkim imenom i lozinkom
   - Provjerava postoji li gumb za prijavu i je li omogućen
   - Registrira handler za konzolne poruke radi boljeg debugiranja
   - Čeka da gumb bude omogućen i klikne na njega
   - Čeka završetak navigacije nakon prijave

Ovaj test provjerava da:
1. Korisnik se može uspješno prijaviti u sustav s različitim korisničkim podacima
2. Nakon prijave, korisnik može pristupiti zaštićenoj stranici za pregled urudžbenih zapisa
3. Sustav uspješno navigira na traženu stranicu

Za razliku od unit testova, ovaj E2E test simulira stvarnu korisničku interakciju kroz preglednik, provjeravajući cjelokupno korisničko iskustvo od prijave do pristupa funkcionalnosti.


## Usporedba Unit i E2E testova

**Unit testovi** (prvi primjer):
- Testiraju izolirane komponente (kontroler)
- Koriste mock objekte za simuliranje zavisnosti
- Brzi su za izvršavanje i lokaliziraju probleme
- Idealni za testiranje poslovne logike

**End-to-End testovi** (drugi primjer):
- Testiraju cijeli sustav zajedno
- Koriste stvarni preglednik i simuliraju korisničke interakcije
- Sporiji su, ali verificiraju integraciju komponenti
- Idealni za potvrđivanje korisničkih scenarija

## Ključne razlike između prikazanih primjera

1. **Razina testiranja:**
   - Unit testovi (`PredmetControllerTests`) testiraju izolirane komponente (kontroler)
   - E2E test (`PredmetFrontendTests`) testira cjelokupni sustav kroz korisničko sučelje

2. **Ovisnosti:**
   - Unit testovi koriste mock objekte za simulaciju ponašanja zavisnih komponenti
   - E2E test koristi stvarne komponente i njihove interakcije

3. **Način izvršavanja:**
   - Unit testovi izvršavaju metode direktno i provjeravaju njihov rezultat
   - E2E test simulira korisničke akcije (prijava, navigacija) kroz preglednik

4. **Složenost i brzina:**
   - Unit testovi su jednostavniji i brži za izvršavanje
   - E2E test je složeniji i sporiji, ali pruža bolju potvrdu ispravnosti cijelog sustava

## Zaključak