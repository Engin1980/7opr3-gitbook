# JWT - JSON Web Token

JSON Web Token je otevřený standard ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)) definující sebepopisný mechanismus pro předávání informací mezi dvěma stranami založený na formátu dat [JSON](https://www.w3schools.com/js/js\_json\_intro.asp). Cílem je poskytnout malou a kompaktní strukturu předávanou v rámci HTTP požadavků v HTTP hlavičce. Cílem JWT je poskytnout druhé straně informace o identitě volajícího. Typické použití JWT je k:

* autentizaci - uživatel poskytne svá pověření (např. jméno+heslo) a obdrží JWT token, který popisuje jeho identitu a práva v rámci aplikace,
* autorizaci - uživatel požádá o službu a přiloží k požadavku svůj JWT token; služba na základě JWT rozhodne, zda je uživatel oprávněn ji použít,
* předávání dat - JWT tokeny jsou podepsané; příjemce má tedy jistotu, že data obsažená v JWT tokenu jsou důvěryhodná.

### Formát JWT

JWT token má obsah zapsaný ve formátu JSON. Token sestává ze tří částí, které jsou navzájem odděleny tečkou:

* Header - hlavičky. V hlavičce je uveden typ tokenu a algoritmus použití k zašifrování tokenu. K širování tokenu lze použít symetrické i asymetrické klíče. Typicky se v příkladech symetrických klíčů uvádí HS256 (HMAC + SHA256), u asymetrických pak RS256, nebo ES256 či EdDSA.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

* Payload - obsah/data/claims. V obsahu tokenu je vlastní zpráva. V případě JWT jsou to tzv. prohlášení - _claims_ - uložené ve formátu JSON. Prohlášení udávají informace o tokenu - kdo ho vydal, kdy, do kdy platí, pro koho je určen, a další. Blíže budou uvedeny dále.

```json
{
  "iss": "osu.cz",
  "sub": "marek.vajgl@ou.cz",
  "aud": "tv.osu.cz",
  "iat": 1479203274,
  "exp": 1479205074,
  "roles": "admin,user"
}
```

* Signature - podpis. V poslední části je podpis sestavený na základě algoritmu (viz hlavička) pomocí soukromého klíče.&#x20;

Výsledný token má tedy obecný formát `header.payload.signature`, v reálu může vypadat cca takto (pro přenositelnost jako text přes HTTP se token ještě převádí do kódování [Base64](https://cs.wikipedia.org/wiki/Base64)):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJvc3UuY3oiLCJzdWIiOiJtYXJlay52YWpnbEBvdS5jeiIsImF1ZCI6InR2Lm9zdS5jeiIsImlhdCI6MTQ3OTIwMzI3NCwiZXhwIjoxNDc5MjA1MDc0LCJyb2xlcyI6ImFkbWluLHVzZXIifQ.q24SEgxKvNxTn_hOhFniFeyjiRFcFSA69w07udIOeEY
```

Podpis se používá pro ověření, zda nedošlo ke změně dat v hlavičce a obsahu a token je tedy důvěryhodný. Cílová strana (za použití soukromého klíče v případě symetrické šifry, nebo veřejného klíče v případě asymetrické šifry) sestaví z podpisu hlavičku a obsah. Následně rekonstruované údaje porovná. Pokud se neshodují s informacemi v původním JWT tokenu, nejsou hodnoty důvěryhodné (někdo je pozměnil) a token není platný. Je tedy zjevné, že **změnou hlavičky nebo obsahu se zneplatní jeho podpis.** Token je tedy (bez znalosti soukromého klíče) chráněn před změnou.&#x20;

### Prohlášení - Claims

V rámci těla JWT lze definovat tzv. prohlášení, které vydávají informace o daném subjektu - například jeho jméno, e-mail, ale také informace o vydavateli, příjemci a časové platnosti tokenu. Prohlášení lze použít buď **registrovaná** (předdefinovaná prohlášení kvůli jednodušší spolupráci mezi různými aplikacemi - viz), nebo prohlášení vlastní - libovolná.

Mezi nejčastější předdefinovaná prohlášení patří (plný výpis viz [IANA JSON Web Token Claims Registry](https://www.iana.org/assignments/jwt/jwt.xhtml#claims)):

* `iss` - issuer - kdo token vydal
* `sub` - subject - o kom se vydávají prohlášení (typicky uživatel/účet)
* `aud` - audience - pro koho je token určen
* `iat` - issued at - kdy byl token vydán
* `nbf` - not before - od kdy je token platný (pokud má být platný později než kdy byl vydán)
* `exp` - expiration - kdy vyprší platnost tokenu

{% hint style="info" %}
Claim `iat` (issued at) neslouží primárně k verifikaci časové platnosti tokenu. Pokud je třeba explicitně nastavit začátek časové platnosti tokenu, je nutno nastavit claim `nbf` (not before).
{% endhint %}

Vlastní prohlášení může aplikace používat libovolně. Tvoří se ve formátu klíč-hodnota.&#x20;

### JWT, JWS, JWE

Pojmem JWT se chápe samotný JSON Web Token v jeho čisté podobě. Pro jeho předání se typicky používají dva odvozené formáty:

* JSON Web Signatures (JWS) - podepsaný (signed) JWT token. U tohoto tokenu jsou data (claims) podepsána, ale přístupná. Kdokoliv tedy může dekódovat přiložené informace v tokenu, což je nebezpečné v případě použití citlivých dat v claims. Podpis se však vytváří na základě hlavičky a dat; při ověření tokenu je tedy jistota, že hlavička ani data tokenu se nemohla změnit.
* JSON Web Encryption (JWE) - zašifrovaný JWT token. U tohoto formátu jsou zašifrována i data JWT, takže bez znalosti klíče nelze data (claims) přečíst. Stejně jako u JWS, podpis se vytváří na základě hlavičky i obsahu, takže je při ověření jistota, že hlavička ani data tokenu se nemohla změnit.

### Obecné použití JWT

Při přihlášení  pošle uživatel aplikaci své pověření (jméno+heslo apod.). Aplikace ověří správnost uživatele a jeho účtu, a pokud je uživatel korektně autentizován, vytvoří JWT Token. Do těla tokenu zapíše správná prohlášení, zejména kdo jej vytvořil (iss), kdo je subjekt (sub), případně pro koho je určen (aud). Dále vždy doplní datum vytvoření (iat), datum expirace (exp), případně datum začátku platnosti (nbf). Dále může doplnit vlastní prohlášení (například oprávnění, názvy rolí atp.). Poté se zvolí šifrovací algoritmus, tato informace se zapíše do hlavičky. Finálně se token podepíše, převede do Base64 a vrací uživateli.

Při další komunikaci uživatel při každém požadavku posílá také tento token. Token se používá v HTTP hlavičce `Authorization` ve speciální formátu s prefixem `Bearer` , tedy například `Bearer eXjfelasef.boasefa.yosefeasfe`. Příjemce u HTTP požadavku zjistí, zda hlavička `Authorization` __ začíná na `Bearer`, následně tento prefix z tokenu odstraní. Poté dekóduje token z Base64 do bytového pole a čistý token potom příjemce z podpisu dešifruje. Následně:

1. Porovná dešifrovaná data s původní hlavičkou a obsahem. Pokud se liší, token někdo pozměnil. Takový token je podvržený a nesmí být brán jako platný.
2. Porovná aktuální čas s hodnotami exp a nbf. Pokud aktuální čas nepasuje do platnosti tokenu, tj. token ještě není aktuální nebo expiroval, a nesmí být brán jako platný.
3. Na základě dalších požavků může příjemce dekódat další prohlášení (například názvy rolí) a dle toho rozhodnout, zda má uživatel právo na provedení operace.

<figure><img src="../.gitbook/assets/jwt.png" alt=""><figcaption></figcaption></figure>

<details>

<summary>plantuml</summary>

@startuml

title "JWT princip"

actor User participant App database DB

\== Autentizace ==

User -> App : Pověření (např. jméno + heslo) activate App App -> DB : Ověř uživatele activate DB DB --> App : (informace o uživateli) deactivate DB App -> App : Vytvoř JWT s informacemi o uživateli App -> App : Nastav expiraci (a další parametry) JWT App -> App : Podepiš JWT => JWS App --> User: (JWS) deactivate App

\== Autorizace ==

User -> App : Požadavek na akci + JWS activate App App -> App : Dekóduj JWS na JWT App -> App : Zkontroluj, zda sedí hlavička + obsah App -> App : Zkontroluj, zda je token platný vůči expiraci note right: Pozor, kontroluje se nejen "exp",\nale také "nbf" a "iat"\n(viz claims) App -> App : Zjisti z JWT identitu uživatele a jeho práva note right: Nemusí se komunikovat\ns DB App -> App : Ověř, zda uživatel může operaci provést App --> User : (výsledek nebo zamítnutí akce)

@enduml

</details>

{% hint style="danger" %}
Šifrovací algoritmus může být prázdný (`none`). Takový token však není chráněn proti úpravě, proto je toto použití nedoporučováno.
{% endhint %}

{% hint style="danger" %}
Pozor na vytváření tokenů bez expirace. Takové tokeny jsou platné po nekonečně dlouhou dobu a jejich platnost nelze zrušit (leda změnou soukromého klíče).
{% endhint %}

{% hint style="info" %}
Pozor na hodnoty claims `nbf` a `exp`. Aktuální čas mezi servery se může lišit, proto v případě, že token zpracovává někdo jiný, než kdo jej vydal, je vhodné uvážit drobnou toleranci (max jednotky sekund) při určování platnosti tokenu.
{% endhint %}

### Výhody a nevýhody JWT

Výhody:

* JWT je založeno na JSON. Obsah tokenu je tedy jednoduše dekódovatelný ve všech moderních programovacích prostředcích.
* JWT je samonosný - příjemce tokenu obsahuje všechny informace o subjektu z prohlášeních tokenu a nemusí informace ověřovat vůči vlastní databázi (není-li třeba jinak). Informace lze jednoduše předávat mezi více příjemci.
* JWT expiruje - mechanismus obsahuje automatický expirační mechanismus, díky čemuž se nemusí příjemce starat o zajištění kontroly platnosti tokenu vůči vydavateli.
* JWT je malý - jednoduché kódování s malým množstvím dat umožňuje relativně jednoduché předávání dat v rámci HTTP požadavků. V případě Stateless JWT však toto nemusí platit.
* JWT je důvěryhodný - podepisování JWT způsobí, že prohlášení v JWT jsou důvěryhodná oproti podpisu.

Nevýhody:

* JWT nelze zrušit platnost. JWT je platný do expirace a neexistuje univerzální mechanismus, který by ji zrušil. Proto je odhlašování uživatele (či dokonce blokování uživatele) složitější a typicky platné až po uplynutí posledního tokenu daného uživatele.
* JWS prohlášení jsou veřejná a může hrozit únik citlivých informací. Typická forma JWS nešifruje obsah (ani hlavičku) tokenu. Obsah (prohlášení) tedy může dekódovat kdokoliv a mít přístup k potencionálně citlivým informacím.
* Složité zvolit vhodný čas expirace. U krátké expirace je nutné často obnovovat token s novou expirací, což může být pro uživatele nepříjemné (časté odhlašování). U dlouhé expirace je složité uživatele odhlásit/blokovat (viz první bod).
