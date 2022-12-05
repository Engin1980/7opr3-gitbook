---
description: >-
  Tato stránka demonstruje použití Argon2di (implementace
  Konscious.Security.Cryptography.Argon2) v prostředí .NET
---

# Demonstrace použití Argon2di v .NET Core

Vytvoříme prázdný konzolový .NET Core projekt.

Do projekt přes NuGet připojíme balíček obsahující implementaci Argon2 - `Konscious.Security.Cryptography.Argon2` (v době psaní ve verzi 1.3.0).

Následující kód demonstruje použití při vytvoření jednoduchého hesla a soli, hashe a jeho ověření.

{% code lineNumbers="true" %}
```csharp
using Konscious.Security.Cryptography;
using System.Security.Cryptography;

// following constants defines complexity
var LEN = 1024;
var ITERS = 128_000;
var PARS = 64;
var MEMSIZE = PARS * 4;

Console.WriteLine("Hello, World!");

var startTime = DateTime.Now;

var password = RandomNumberGenerator.GetBytes(LEN);
var salt = RandomNumberGenerator.GetBytes(LEN);
var arg = new Argon2id(password)
{
  Salt = salt,
  Iterations = ITERS,
  DegreeOfParallelism = PARS,
  MemorySize = MEMSIZE
};
var hash = arg.GetBytes(LEN);


var otherHash = new Argon2id(password)
{
  Salt = salt,
  Iterations = ITERS,
  DegreeOfParallelism = PARS,
  MemorySize = MEMSIZE
}.GetBytes(LEN);

var endTime = DateTime.Now;
var duration = endTime - startTime;

Console.WriteLine($"Pass {Encode(password)}");
Console.WriteLine($"Salt {Encode(salt)}");
Console.WriteLine($"Hash {Encode(hash)}");
Console.WriteLine($"Hhsh {Encode(otherHash)}");
Console.WriteLine($"Same {hash.SequenceEqual(otherHash)}");
Console.WriteLine($"Duration {duration}");

static string Encode(byte[] data)
{
  return Convert.ToBase64String(data);
}
```
{% endcode %}
