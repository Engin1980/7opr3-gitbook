---
description: Tato stránka demonstruje použití BCrypt.Net-Next (4.0.3) v prostředí .NET
---

# Demonstrace použití BCrypt v .NET Core

Vytvoříme prázdný konzolový .NET Core projekt.

Do projekt přes NuGet připojíme balíček obsahující implementaci BCrypt - `BCrypt.Net-Next` (v době psaní ve verzi 4.0.3).

Následující kód demonstruje použití při vytvoření jednoduchého hesla, hashe a jeho ověření.

{% hint style="info" %}
Je důležité si uvědomit, že BCrypt při hashování hesla sám automaticky přídává sůl, která je ve výsledném hashi obsažená, takže ji kód explicitně nezmiňuje. Sůl se tedy používá automaticky.
{% endhint %}

{% code lineNumbers="true" %}
```csharp
using BCR = BCrypt.Net.BCrypt;

int workFactor = 4; // defines complexity

string password = "a";

// salt is included in BCrypt by default
// so no salt is provided aside into the algorithm

DateTime startTime = DateTime.Now;

string hash = BCR.HashPassword(password, workFactor);
bool verified = BCR.Verify(password, hash);

DateTime endTime = DateTime.Now;
TimeSpan delay = endTime - startTime;

Console.WriteLine("Password: " + password);
Console.WriteLine("Hash: " + hash);
Console.WriteLine("Verify: " + verified);
Console.WriteLine("Total time: " + delay.ToString());
```
{% endcode %}
