# Vytvoření komponenty

Dalším krokem je vytvoření komponenty, která bude zobrazovat data.

## Routing

Nejdříve přizpůsobíme routing. V naší aplikaci budeme mít pouze jednu hlavní zobrazovací komponentu. Proto uděláme jednu univerzální cestu. Do souboru `app-routing.module.ts` přidáme cestu (upravíme proměnnou `routes`):

```typescript
const routes: Routes = [
  { path : "", component : EventListComponent}
];
```

## Pipe

Pro formátování datumu v TypeScriptu lze využít funkci `format` (například viz [https://medium.com/@glasshost/how-to-format-date-time-in-typescript-97ca7580afb1](https://medium.com/@glasshost/how-to-format-date-time-in-typescript-97ca7580afb1)), ale její použití je v základu omezené na anglické nastavení. Pro podporu dalších jazyků se do projektu musí zavést internacionalizace, což je pro projekt naše rozsahu zbytečné.

Proto si vytvoříme vlastní funkci, která bude generovat zobrazování datumu v českém formátu. Do projektu do složky /src/app/pipes přidáme jednoduchý soubor `czech-date.pipe.ts` s následujícím kódem:

{% code title="czech-date.pipe.ts" lineNumbers="true" %}
```typescript
import {Pipe, PipeTransform} from "@angular/core";
import {formatDate} from "@angular/common";

@Pipe({name: 'czechDatePipe'})
export class CzechDatePipe implements PipeTransform {
  transform(value: Date): string {
    return formatDate(value, "d. MMMM yyyy H:mm", "en-US")
      .replace(      "January", "ledna")
      .replace(      "February", "února")
      .replace(      "March", "března")
      .replace(      "April", "dubna")
      .replace(      "May", "května")
      .replace(      "June", "června")
      .replace(      "July", "července")
      .replace(      "August", "srpna")
      .replace(      "September", "září")
      .replace(      "October", "října")
      .replace(      "November", "listopadu")
      .replace(      "December", "prosince");
  }
}

```
{% endcode %}

Třída obsahuje funkci `transform(...)`. Ve funkci provedeme základní převedení datumu na anglický a posléze jen nahradíme text měsíce českým ekvivalentem.

Povšimněte si řádku 4, který definuje, že funkce se bude chovat jako "pipe" - její použití uvidíme v komponentě.

## Tvorba komponent

V projektu použijeme 3 komponenty:

* **event-list** jako hlavní komponentu, která bude zobrazovat seznam událostí, a interně bude v sobě zobrazovat další komponenty,
* **event-create** jako komponentu pro pro vytváření nové události; bude zobrazena uvnitř hlavní komponenty,
* **event-note** jako komponentu zobrazující poznámky u události a jejich přidávání a mazání; bude zobrazena uvnitř hlavní komponenty.

Protože se hned na komponenty budeme odkazovat, vytvoříme je všechny najednou, do složky /src/app/components:

```powershell
ng g c event-create
ng g c event-list
ng g c event-note
```

## Tvorba komponenty event-list

asfe

## Tvorba komponenty event-create

asef

## Tvorba komponenty event-note

asef