# Připojení knihoven a vytvoření DB

V úvodní části připravíme projekt na práci s DB. Vložíme do projektu požadované knihovny a spustíme a vytvoříme databázi.

## Připojení knihoven

Knihovny používané v projektu jsou definované přes Maven v souboru `pom.xml`. Do tohoto souboru do sekce `dependencies` potřebujeme připojit tři závislosti:

```xml
<dependency>
    <groupId>org.apache.derby</groupId>
    <artifactId>derby</artifactId>
    <version>10.16.1.1</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.apache.derby</groupId>
    <artifactId>derbytools</artifactId>
    <version>10.16.1.1</version>
    <scope>runtime</scope>
</dependency>
<!-- https://mvnrepository.com/artifact/org.apache.derby/derbyclient -->
<dependency>
    <groupId>org.apache.derby</groupId>
    <artifactId>derbyclient</artifactId>
    <version>10.16.1.1</version>
</dependency
```

Pokud jsme již při vytváření projektu přidali do dependencies podporu pro databázi Apache Derby, první dvě závislosti již v souboru `pom.xml` budou. Třetí závislost je implementace databázového ovladače, který použijeme, a nalezneme ji přes maven repository ([https://mvnrepository.com/artifact/org.apache.derby/derbyclient](https://mvnrepository.com/artifact/org.apache.derby/derbyclient)). Po vložení/úpravě souboru `pom.xml` je třeba obnovit maven závislosti v prostředí IDEA.

{% hint style="info" %}
Je třeba dávat pozor, ať u všech tří závislostí je zadána stejná odkazovaná verze.
{% endhint %}
