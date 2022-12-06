---
description: >-
  Tato stránka popisuje implementaci zabezpečení back-end webové aplikace
  vytvořené v ASP.NET Core přes JWT.
---

# Implementace JWT v ASP.NET Core

Tato stránka vysvětluje implementaci zabezpečení přes JWT v back-end aplikaci postavené na ASP.NET Core. V této části se již předpokládá seznámení se základními pojmy, a to nejen z oblasti JWT, ale také obecných pojmů z oblasti back-end webových aplikací, zejména se zaměřením REST API, služeb (services) atp. Předpokládá se také znalost základních operací ve vývojovém prostředí Visual Studio (aktuálně použitá verze je Visual Studio 2022) a znalost práce s balíčkovacím systémem NuGet.

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

### Definice skeletonu služeb I

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
builder.Services.AddSingleton<SecurityService>();
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
    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]));
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
* definujeme naše dvě předvytvořené služby. Služba `SecurityService`bude definována jako _singleton_, protože ji chceme jednu pro celou aplikaci. Služba `AppUserService` by v běžném projektu byla definována nejspíše jako _transientní_, protože chceme, aby se vytvářela vždy pro každý požadavek separátně. Protože však v našem projektu bude pro zjednoušení také udržovat data (v reálném projektu by tomu tak pravděpodobně **nikdy** nebylo), zavedeme si ji také jako singleton.
* zavedeme služby pro controllery (řádek 12),
* zavedem služby pro autorizaci (řádek 13) a autentifikaci spolu s definicí JWT (řádky 14-32) - bude vysvětleno podrobněji dále,
* následně se aplikace sestaví (řádek 34) a připojí se vrstvy: vrstva směrování, vrstva autentizace, vrstva autorizace a vrstva mapující end-pointy kontrolerů aplikací (řádky 36-39);
* nakonec se aplikace spustí (řádek 40).

Zajímavá je ještě definice chování JWT tokenu. Řádky 16-18 definují chování schémat (nebude vysvětleno dále, ponecháme viz nastavení). 5ádky 25-30 definují, co vše a jak se bude v JWT validovat:

* klíč JWT (řádek 25) a zda kontrolujeme validitu klíče (**toto chceme vždy**) - řádek 28,
* zda kontrolujeme validitu vydavate a příjemce JWT (dle vlastní volby; v našem případě nepoužíváme, ale všimněte si, že kód v JWT (dále) i v konfiguračním souboru je na to připraven) - řádky 26-27,
* zda kontrolujeme časová razítka JWT (**toto chceme také vždy**) - řádek 29.

Kvůli možné odlišnosti času vydavatele a ověřovatele JWT knihovna automaticky přidává plovoucí časové okno (ve výchozím stavu 5 minut), ve kterém je token stále považován za platný, přestože jeho platnost vypršela. Toto okno můžeme nastavit sami (v našem případě jej pro snadnější testování rušíme) - řádek 30.

