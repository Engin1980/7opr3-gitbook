---
description: >-
  Tato stránka popisuje implementaci zabezpečení back-end webové aplikace
  vytvořené v ASP.NET Core přes JWT.
---

# Implementace JWT v ASP.NET Core

cTato stránka vysvětluje implementaci zabezpečení přes JWT v back-end aplikaci postavené na ASP.NET Core. V této části se již předpokládá seznámení se základními pojmy, a to nejen z oblasti JWT, ale také obecných pojmů z oblasti back-end webových aplikací, zejména se zaměřením REST API, služeb (services) atp. Předpokládá se také znalost základních operací ve vývojovém prostředí Visual Studio (aktuálně použitá verze je Visual Studio 2022) a znalost práce s balíčkovacím systémem NuGet.

Aplikace je vytvořena jako _minimal working example_, kde se bude vycházet z prázdné ASP.NET Core aplikace a kód nebude využívat ukládání dat do databáze pomocí Entity Frameworku.

## Úvodní konfigurace projektu

Nejdříve vytvoříme ve Visual Studiu 2022 prázdný webový projekt - _ASP.NET Core Empty_. V projektu nebudeme používat Docker ani konfiguraci pro HTTPS. Projekt je postaven nad .NET 6.0.&#x20;

V projektu vytvoříme 3 složky:

* `Services` kde budou definovány služby,
* `Model` kde bude uvedena modelová třída a
* `Controllers` kde budou umístěn kontroler projektu.

### Nastavení konfiguračního souboru

Nejdříve provedeme nastavení konfiguračního souboru `appsettings.json`. Do souboru přidáme novou větev na konfiguraci JWT. Ve větvi můžeme definovat toho, kdo JWT vydal - `issuer`, toho, pro koho je token určen `audience` a finálně, chceme-li, pevný soukromý klíč `key`.

{% code title="appsettings.json" lineNumbers="true" %}
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Jwt": {
    "Issuer": "https://www.osu.cz/",
    "Audience": "https://www.osu.cz/",
    "Key": "This is a sample secret key - please don't use it in production environment.'"
  }
}
```
{% endcode %}

### Přidání podpory JWT v .NET + BCrypt

Podporu JWT do .NET zajistíme přidáním knihovny `Microsoft.AspNetCore.Authentication.JwtBearer` (aktuálně verze 6.0.11, pozor verze typicky musí sedět na verzi .NET Core) přes balíčkovací systém NuGet.

Zároveň budeme hashování hesla uživatele provádět přes BCrypt - opět přes NuGet přidáme podporu `BCrypt.Net-Next`(aktuálně ve verzi 4.0.3).

### Definice skeletonu služeb

Do projektu přidáme zatím prázdné služby, které budeme v kódu využívat. Ve složce `Services` vytvoříme dva soubory:

* Třídu `SecurityService`, která se bude starat o bezpečnost, a
* třídu `AppUserService`, která bude poskytovat přístup a nástroje pro práci s modelovou třídou uživatele `AppUser`(bude vytvořena později).

### Nastavení spouštěcího souboru Program.cs

V projektu se aplikace spouští kódem v souboru `Program.cs`, který využívá tzv. _top-level statements_. Kód se nepíše do funkce, ale jako sekvence příkazů, které se provádějí postupně.

{% hint style="info" %}
Pro více informací o _top-level statements_ viz například [https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/top-level-statements](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/top-level-statements).
{% endhint %}

Kód upravíme/nahradíme do následující formy:

{% code title="Program.cs" lineNumbers="true" %}
```csharp
using JWTCoreDemo.Services;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<IConfiguration>(builder.Configuration);
SecurityService securityService = new(builder.Configuration);
builder.Services.AddSingleton<SecurityService>(securityService);
builder.Services.AddSingleton<AppUserService>();

builder.Services.AddControllers();
builder.Services.AddAuthorization();
builder.Services.AddAuthentication(opt =>
  {
    opt.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    opt.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    opt.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
  })
  .AddJwtBearer(opt =>
  {
    // key from config
    //var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]));
    // or random key each app-start
    var key = new SymmetricSecurityKey(securityService.Key);
    opt.TokenValidationParameters = new TokenValidationParameters()
    {
      IssuerSigningKey = key,
      ValidateIssuer = false,
      ValidateAudience = false,
      ValidateIssuerSigningKey = true,
      ValidateLifetime = true,
      ClockSkew = TimeSpan.Zero
    };
  });

var app = builder.Build();

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```
{% endcode %}

V kódu:

* definujeme konfigurační službu (řádek 8),
* definujeme naše dvě předvytvořené služby. Služba `SecurityService`bude definována jako _singleton_, protože ji chceme jednu pro celou aplikaci. Služba `AppUserService` by v běžném projektu byla definována nejspíše jako _transientní_, protože chceme, aby se vytvářela vždy pro každý požadavek separátně. Protože však v našem projektu bude pro zjednoušení také udržovat data (v reálném projektu by tomu tak pravděpodobně **nikdy** nebylo), zavedeme si ji také jako singleton. Protože instanci služby `SecurityService`budeme ihned dále potřebovat, nenecháme ji vytvořit automaticky (jako na řádku 8), ale uděláme si nejdříve instanci a tu předáme správci služeb (řádky 9-10).
* zavedeme služby pro controllery (řádek 13),
* zavedem služby pro autorizaci (řádek 14) a autentifikaci spolu s definicí JWT (řádky 15-37) - bude vysvětleno podrobněji dále,
* následně se aplikace sestaví (řádek 38) a připojí se vrstvy: vrstva směrování, vrstva autentizace, vrstva autorizace a vrstva mapující end-pointy kontrolerů aplikací (řádky 40-43);
* nakonec se aplikace spustí (řádek 44).

Zajímavá je ještě definice chování JWT tokenu. Řádky 17-19 definují chování schémat (nebude vysvětleno dále, ponecháme viz nastavení). 5ádky 23-34 definují, co vše a jak se bude v JWT validovat:

* zda chceme použít klíč z konfiguračního souboru nebo dynamicky generovaný (řádky 23-26),
* klíč JWT (řádek 29) a zda kontrolujeme validitu klíče (**toto chceme vždy**) - řádek 32),
* zda kontrolujeme validitu vydavate a příjemce JWT (dle vlastní volby; v našem případě nepoužíváme, ale všimněte si, že kód v JWT (dále) i v konfiguračním souboru je na to připraven) - řádky 30-31,
* zda kontrolujeme časová razítka JWT (**toto chceme také vždy**) - řádek 33.

Kvůli možné odlišnosti času vydavatele a ověřovatele JWT knihovna automaticky přidává plovoucí časové okno (ve výchozím stavu 5 minut), ve kterém je token stále považován za platný, přestože jeho platnost vypršela. Toto okno můžeme nastavit sami (v našem případě jej pro snadnější testování rušíme) - řádek 34.

### Vytvoření jednoduché modelové třídy uživatele AppUser

Finálně do složky `Model`vytvoříme třídu `AppUser`, do které doplníme triviální kód třídy uživatele obsahující e-mail, heslo a role:

{% code title="Model\AppUser.cs" lineNumbers="true" %}
```csharp
using System.Reflection.Metadata.Ecma335;

namespace JWTCoreDemo.Model
{
  public class AppUser
  {
    public const string ADMIN_ROLE_NAME = "ADMIN";

    public AppUser(string email, string passwordHash)
    {
      Email = email;
      PasswordHash = passwordHash;
    }

    public string Email { get; private set; }
    public string PasswordHash { get; private set; }
    public List<string> Roles { get; private set; } = new();
  }
}
```
{% endcode %}

{% hint style="info" %}
Ve třídě uživatele není zmíněna _sůl_ použitá při hashování hesla. Je tomu tak proto, že pro hashování budeme využívat BCrypt, který sůl k heslu automaticky a ukládá sám. Pokud by se použil jiný přístup hashování, musela by třída uživatele obsahovat i tuto informaci.
{% endhint %}

## Implementace služeb

Nejdříve implementujeme služby.

### Pomocná třída výjimky - ServiceException

Na začátku implementujeme jednoduchou pomocnou třídu pro správu výjimek vyhazovaných ze služeb. Do složky `Services` přidáme třídu `ServiceException`.

{% code title="Services \ ServiceException.cs" lineNumbers="true" %}
```csharp
namespace JWTCoreDemo.Services
{
  public class ServiceException : Exception
  {
    public Type Service { get; private set; }
    public ServiceException(Type service, string message, Exception? cause = null)
      : base(message, cause)
    {
      this.Service = service ?? throw new ArgumentNullException();
    }
  }
}
```
{% endcode %}

Jednoduchá výjimka bere do konstruktoru informaci o službě, který ji vhodila, zprávě a případné vnitřní výjimce.

{% hint style="info" %}
První parametr konstruktoru odkazuje na obecný typ `Type`, kam však lze poslat libovolnou třídu a ne jen "službu". Celý mechanismus v projektu lze jednoduše vylepšti tím, že řekneme, že každá naše služba bude dědit z nějakého rozhraní (například `IAppService` a první parametr omezíme na instanci této třídy. Vyzkoušejte.
{% endhint %}

### Služba uživatele - AppUserService

Protože kód služeb bude trošku složitější, bude představen po částech/metodách.

Do projektu do složky `Services` nejdříve vytvoříme třídu `AppUserService`. Do třídy definujeme nejdříve dvě proměnné - jedna drží informaci o všech uživatelích a v našem případě nahrazuje chování perzistence (například databáze), druhá bude obsahovat injektovaný (přes Dependency Inejction) odkaz na naši druhou službu `SecurityService`- její instance se do objektu služby dostane přes konstruktor:

```csharp
private readonly List<AppUser> inner = new();
private readonly SecurityService securityService;

public AppUserService([FromServices] SecurityService securityService)
{
  this.securityService = securityService;
}
```

Do kódu přidáme dvě pomocné soukromé funkce, které budeme využívat dále - jedna kontroluje validitu parametru, druhá je pomocná funkce pro vytvoření výjimky:

{% code lineNumbers="true" %}
```csharp
private static void EnsureNotNull(string value, string parameterName)
{
  if (value == null)
    throw CreateException($"Parameter {parameterName} cannot be null.");
}

private static ServiceException CreateException(
  string message, Exception? innerException = null) =>
  new(typeof(AppUserService), message, innerException);
```
{% endcode %}

Následně vytvoříme metodu `Create`, která vytvoří nového uživatele podle parametrů:

{% code lineNumbers="true" %}
```csharp
public AppUser Create(string email, string password, bool isAdmin)
{
  EnsureNotNull(email, nameof(email));
  EnsureNotNull(password, nameof(password));

  email = email.ToLower();

  if (inner.Any(q => q.Email == email))
    throw CreateException($"Email {email} already exists.", null);

  string hash = this.securityService.HashPassword(password);

  AppUser ret = new(email, hash);
  if (isAdmin) ret.Roles.Add(AppUser.ADMIN_ROLE_NAME);
  this.inner.Add(ret);

  return ret;
}
```
{% endcode %}

Funkce nejdříve zkontroluje parametry (řádky 3-4), následně upraví e-mail (řádek 6). Potom zkusí zjistit, zda již uživatel s e-mailem existuje, případně vyhodí výjimku (řádky 8-9). Dále se s využitím bezpečnostní služby provede hash hesla (řádek 11), vtvoří se nový uživatel, přidá se do kolekce a vloží se mu role admina, je-li třeba (řádky 12-15). Finálně se uživatel vrátí (řádek 18).

Další primitivní funkce pro vrácení všech uživatelů nepotřebuje komentář:

```csharp
public List<AppUser> GetUsers()
{
  return this.inner.ToList();
}
```

Následuje funkce vracející uživatele podle přihlašovacích údajů:

{% code lineNumbers="true" %}
```csharp
public AppUser GetUserByCredentials(string email, string password)
{
  EnsureNotNull(email, nameof(email));
  EnsureNotNull(password, nameof(password));

  AppUser appUser = inner.FirstOrDefault(q => q.Email == email.ToLower())
    ?? throw CreateException($"Email {email} does not exist.");

  if (!this.securityService.VerifyPassword(password, appUser.PasswordHash))
    throw CreateException($"Credentials are not valid.");

  return appUser;

```
{% endcode %}

Funkce na začátku nejdříve zkontroluje parametry (řádky 3-4). Následně se pokusí získat uživatele podle e-mailu, jinak vrací výjimku (řádky 6-7). Dále se pokusí uživatele autentizovat podle jména a hesla a s využitím bezpečnostní služby (řádky 9-10) a v případě úspěchu jej vrátí (řádek 12).

Nakonec kompletní kód třídy:

{% code title="" lineNumbers="true" %}
```csharp
using JWTCoreDemo.Model;
using Microsoft.AspNetCore.Mvc;

namespace JWTCoreDemo.Services
{
  public class AppUserService
  {
    private readonly List<AppUser> inner = new();
    private readonly SecurityService securityService;

    public AppUserService([FromServices] SecurityService securityService)
    {
      this.securityService = securityService;
    }

    public AppUser Create(string email, string password, bool isAdmin)
    {
      EnsureNotNull(email, nameof(email));
      EnsureNotNull(password, nameof(password));

      email = email.ToLower();

      if (inner.Any(q => q.Email == email))
        throw CreateException($"Email {email} already exists.", null);

      string hash = this.securityService.HashPassword(password);

      AppUser ret = new(email, hash);
      if (isAdmin) ret.Roles.Add(AppUser.ADMIN_ROLE_NAME);
      this.inner.Add(ret);

      return ret;
    }

    public List<AppUser> GetUsers()
    {
      return this.inner.ToList();
    }

    public AppUser GetUserByCredentials(string email, string password)
    {
      EnsureNotNull(email, nameof(email));
      EnsureNotNull(password, nameof(password));

      AppUser appUser = inner.FirstOrDefault(q => q.Email == email.ToLower())
        ?? throw CreateException($"Email {email} does not exist.");

      if (!this.securityService.VerifyPassword(password, appUser.PasswordHash))
        throw CreateException($"Credentials are not valid.");

      return appUser;
    }

    private static void EnsureNotNull(string value, string parameterName)
    {
      if (value == null)
        throw CreateException($"Parameter {parameterName} cannot be null.");
    }

    private static ServiceException CreateException(string message, Exception? innerException = null) =>
      new(typeof(AppUserService), message, innerException);
  }
}
```
{% endcode %}

### Bezpečnostní služba - SecurityService

Bezpečnosti služba se stará o správu hashování, hesla a vytvoření JWT.

Do projektu do složky `Services`vložíme třídu `SecurityService` a opět její kód představíme po částech.

Nejdříve úvodní deklarace a konstruktor:

{% code lineNumbers="true" %}
```csharp
private const int TOKEN_EXPIRATION_IN_SECONDS = 30;
private readonly IConfiguration configuration;
public byte[] Key = { get; } = RandomNumberGenerator.GetBytes(128);

public SecurityService([FromServices] IConfiguration configuration)
{
  this.configuration = configuration;
}
```
{% endcode %}

Zde si povšimněte zejména délky platnosti tokenu (v sekundách) a také dynamického klíče pro JWT - můžeme používat klíč z konfiguračního souboru, nebo tento, který se resetuje a nastaví vždy při restartu aplikace.

Dále vložíme funkce hashující a ověřující heslo - s využitím knihovny jsou samopopisné:

```csharp
public string HashPassword(string password)
{
  string ret = BCrypt.Net.BCrypt.EnhancedHashPassword(password);
  return ret;
}

public bool VerifyPassword(string password, string passwordHash)
{
  bool ret;
  ret = BCrypt.Net.BCrypt.EnhancedVerify(password, passwordHash);
  return ret;
}
```

Nyní vložíme funkci pro vytvoření JWT tokenu:

{% code lineNumbers="true" %}
```csharp
public string BuildJwtToken(AppUser appUser)
{
  // key from configuration:
  // var key = Encoding.ASCII.GetBytes(configuration["Jwt:Key"]);
  // ... or unique key per app start
  var key = this.Key;

  Dictionary<string, object> roleClaims = appUser.Roles
    .ToDictionary(
      q => ClaimTypes.Role,
      q => (object)q.ToUpper());

  var tokenDescriptor = new SecurityTokenDescriptor
  {
    Subject = new ClaimsIdentity(new[]
      {
        new Claim(JwtRegisteredClaimNames.Sub, appUser.Email),
        new Claim(JwtRegisteredClaimNames.Email, appUser.Email),
        new Claim(JwtRegisteredClaimNames.Aud, configuration["Jwt:Audience"]),
        new Claim(JwtRegisteredClaimNames.Iss, configuration["Jwt:Issuer"]),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
      }),
    Expires = DateTime.UtcNow.AddSeconds(TOKEN_EXPIRATION_IN_SECONDS),
    SigningCredentials = new SigningCredentials
      (new SymmetricSecurityKey(key),
      SecurityAlgorithms.HmacSha512Signature),
    Claims = roleClaims
  };

  var tokenHandler = new JwtSecurityTokenHandler();
  var token = tokenHandler.CreateToken(tokenDescriptor);
  var ret = tokenHandler.WriteToken(token);
  return ret;
}
```
{% endcode %}

Nejdříve je zvolen klíč - buď z konfiguračního souboru, nebo dynamický (viz výše) - řádky 3-6. Následně vytvoříme pomocný slovník oprávnění uživatele obsahující role - řádky 8-11. Dále vytvoříme popisovač tokenu - řádky 13-28. Na řádcích 15-23 se definuje subjekt tokenu - jednotilvé položky oprávnění - řádky 17-21 - by měli být jasné. Na řádku 23 nastavíme expiraci tokenu, na řádku 24 nastavíme způsob podepsání JWT; finálně na řáku 27 přidáme do tokenu oprávnění rolí (viz řádek 8).

Na konci token vytvoříme (řádky 30-31), a převedeme na řetězcovou reprezentaci (řádek 32), kterou vracíme jako výsledek.

Následuje úplný kód třídy `SecurityService`:

{% code title="Services \ SecurityService" lineNumbers="true" %}
```csharp
using JWTCoreDemo.Model;
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;
using System.Buffers.Text;
using System.IdentityModel.Tokens.Jwt;
using System.IO.IsolatedStorage;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;

namespace JWTCoreDemo.Services
{
  public class SecurityService
  {
    private const int TOKEN_EXPIRATION_IN_SECONDS = 600;
    private readonly IConfiguration configuration;
    public byte[] Key { get; } = RandomNumberGenerator.GetBytes(128);

    public SecurityService([FromServices] IConfiguration configuration)
    {
      this.configuration = configuration;
    }

    public string BuildJwtToken(AppUser appUser)
    {
      // key from configuration:
      //var key = Encoding.ASCII.GetBytes(configuration["Jwt:Key"]);
      // ... or unique key per app start
      var key = this.Key;

      Dictionary<string, object> roleClaims = appUser.Roles
        .ToDictionary(
          q => ClaimTypes.Role,
          q => (object)q.ToUpper());

      var tokenDescriptor = new SecurityTokenDescriptor
      {
        Subject = new ClaimsIdentity(new[]
          {
        new Claim(JwtRegisteredClaimNames.Sub, appUser.Email),
        new Claim(JwtRegisteredClaimNames.Email, appUser.Email),
        new Claim(JwtRegisteredClaimNames.Aud, configuration["Jwt:Audience"]),
        new Claim(JwtRegisteredClaimNames.Iss, configuration["Jwt:Issuer"]),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
      }),
        Expires = DateTime.UtcNow.AddSeconds(TOKEN_EXPIRATION_IN_SECONDS),
        SigningCredentials = new SigningCredentials
          (new SymmetricSecurityKey(key),
          SecurityAlgorithms.HmacSha512Signature),
        Claims = roleClaims
      };

      var tokenHandler = new JwtSecurityTokenHandler();
      var token = tokenHandler.CreateToken(tokenDescriptor);
      var ret = tokenHandler.WriteToken(token);
      return ret;
    }

    public bool VerifyPassword(string password, string passwordHash)
    {
      bool ret;
      ret = BCrypt.Net.BCrypt.EnhancedVerify(password, passwordHash);
      return ret;
    }

    public string HashPassword(string password)
    {
      string ret = BCrypt.Net.BCrypt.EnhancedHashPassword(password);
      return ret;
    }

    // not required here, but an example how to generate salt using
    // safe random number generator
    //internal string GenerateSalt()
    //{
    //  var bytes = RandomNumberGenerator.GetBytes(SALT_LENGTH);
    //  string ret = System.Convert.ToBase64String(bytes);
    //  return ret;
    //}
  }
}
```
{% endcode %}

Na konci kódu je přidána zakomentovaná nevyužitá funkce generující sůl pro hashování hesla; kód je pouze pro demonstraci, v našem projektu se nepoužije.

## Implementace kontroleru - UserController

Finálně provedeme implementac kontroleru.

Čístý REST API kontroler v ASP.NET Core vytvoříme jako potomka třídy `ControllerBase`a přidáním anotace `[ApiController]`. Kontroler necháme ihned chránit autorizaci a nastavíme mu základní cestu na `/api/user`. Do kontroleru také necháme vložit naše služby.

{% code title="Controllers \ UserController.cs (skeleton)" lineNumbers="true" %}
```csharp
using JWTCoreDemo.Model;
using JWTCoreDemo.Services;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace JWTCoreDemo.Controllers
{
  [Route("api/user")]
  [Authorize]
  [ApiController]
  public class UserController : ControllerBase
  {
    private readonly AppUserService appUserService;
    private readonly SecurityService securityService;

    public UserController(
      [FromServices] AppUserService appUserService,
      [FromServices] SecurityService securityService)
    {
      this.appUserService = appUserService;
      this.securityService = securityService;
    }    
  }
}

```
{% endcode %}

Nyní si definujme end-pointy kontroleru:

| Metoda | Cesta            | Chráněný | Akce                            |
| ------ | ---------------- | -------- | ------------------------------- |
| GET    | /api/user        | Ne       | Vrátí všechna uživatelská jména |
| PUT    | /api/user        | Ne       | Vytvoří nového uživatele        |
| POST   | /api/user        | Ne       | Přihlásí uživatele              |
| GET    | /api/user/emails | Ano      | Vrátí všechny e-maily           |
| GET    | /api/user/all    | Ano+Role | Vrátí všechno                   |

Začneme vrácením všech uživatelů. End-point není chráněný a komukoliv vrátí jmena (začátky e-mailových adres):

{% code lineNumbers="true" %}
```csharp
[HttpGet]
[AllowAnonymous]
public IActionResult GetUserNames()
{
  List<string> ret = appUserService.GetUsers()
    .Select(q => q.Email[..q.Email.IndexOf("@")])
    .ToList();
  return Ok(ret);
}
```
{% endcode %}

Přidáme získání emailů. End-point už je chráněný (viz atribut) a vrátí všechny e-mailové adresy:

{% code lineNumbers="true" %}
```csharp
[HttpGet("emails")]
public IActionResult GetUserEmails()
{
  List<string> ret = appUserService.GetUsers()
    .Select(q => q.Email)
    .ToList();
  return Ok(ret);
}
```
{% endcode %}

Konečně přidáme získání celého infa o uživatelích; end-point je chráněný i rolí:

{% code lineNumbers="true" %}
```csharp
[HttpGet("all")]
[Authorize(Roles = AppUser.ADMIN_ROLE_NAME)]
public IActionResult GetAll()
{
  List<AppUser> ret = appUserService.GetUsers();
  return Ok(ret);
}
```
{% endcode %}

Jednoduchým end-pointem je také vytvoření uživatele:

{% code lineNumbers="true" %}
```csharp
[HttpPut]
[AllowAnonymous]
public IActionResult Create(string email, string password, bool isAdmin)
{
  try
  {
    this.appUserService.Create(email, password, isAdmin);
  }
  catch (Exception ex)
  {
    return BadRequest(ex.Message);
  }
  return Ok();
}
```
{% endcode %}

Posledním end-pointem je přihlášení uživatele. S využitím služeb je kód také velmi jednoduchý:

{% code lineNumbers="true" %}
```csharp
[HttpPost]
[AllowAnonymous]
public IActionResult Login(string email, string password)
{
  AppUser appUser;
  try
  {
    appUser = this.appUserService.GetUserByCredentials(email, password);
  }
  catch (Exception ex)
  {
    return BadRequest(ex.Message);
  }

  string token = securityService.BuildJwtToken(appUser);
  return Ok(token);
}
```
{% endcode %}

Nakonec shrnutí celé třídy kontroleru:

{% code title="Controllers \ UserController.cs" lineNumbers="true" %}
```csharp
using JWTCoreDemo.Model;
using JWTCoreDemo.Services;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace JWTCoreDemo.Controllers
{
  [Route("api/user")]
  [Authorize]
  [ApiController]
  public class UserController : ControllerBase
  {
    private readonly AppUserService appUserService;
    private readonly SecurityService securityService;

    public UserController(
      [FromServices] AppUserService appUserService,
      [FromServices] SecurityService securityService)
    {
      this.appUserService = appUserService;
      this.securityService = securityService;
    }

    [HttpPut]
    [AllowAnonymous]
    public IActionResult Create(string email, string password, bool isAdmin)
    {
      try
      {
        this.appUserService.Create(email, password, isAdmin);
      }
      catch (Exception ex)
      {
        return BadRequest(ex.Message);
      }
      return Ok();
    }

    [HttpGet]
    [AllowAnonymous]
    public IActionResult GetUserNames()
    {
      List<string> ret = appUserService.GetUsers()
        .Select(q => q.Email[..q.Email.IndexOf("@")])
        .ToList();
      return Ok(ret);
    }

    [HttpGet("emails")]
    public IActionResult GetUserEmails()
    {
      List<string> ret = appUserService.GetUsers()
        .Select(q => q.Email)
        .ToList();
      return Ok(ret);
    }

    [HttpGet("all")]
    [Authorize(Roles = AppUser.ADMIN_ROLE_NAME)]
    public IActionResult GetAll()
    {
      List<AppUser> ret = appUserService.GetUsers();
      return Ok(ret);
    }

    [HttpPost]
    [AllowAnonymous]
    public IActionResult Login(string email, string password)
    {
      AppUser appUser;
      try
      {
        appUser = this.appUserService.GetUserByCredentials(email, password);
      }
      catch (Exception ex)
      {
        return BadRequest(ex.Message);
      }

      string token = securityService.BuildJwtToken(appUser);
      return Ok(token);
    }
  }
}

```
{% endcode %}

## Shrnutí

Tím je řešení hotovo. Můžeme jej vyzkoušet přes klienta PostMan, či libovolnou jinou aplikaci.







