# Vytvoření komponent

Dalším krokem je vytvoření komponent, které budou zobrazovat data.

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

### Kód na pozadí - ts

Kód na pozadí komponenty je poměrně jednoduchý.

{% code title="event-list.component.ts" lineNumbers="true" %}
```typescript
import {Component} from '@angular/core';
import {EventDto} from "../../model/event-dto";
import {EventService} from "../../services/event.service";
import {EventHttpError} from "../../services/event-http.service";

@Component({
  selector: 'app-event-list',
  templateUrl: './event-list.component.html',
  styleUrls: ['./event-list.component.css']
})
export class EventListComponent {

  events: EventDto[] = [];

  constructor(
    private eventService: EventService
  ) {
  }

  ngOnInit() {
    this.eventService.getAll()
      .subscribe({
        next: q => this.events.push(q),
        error: (e: EventHttpError) => console.log(e.message)
      })
  }

  protected onEventCreated(event: EventDto): void {
    this.events.push(event);
  }
}

```
{% endcode %}

V konstruktoru (řádek 16) připojíme do komponenty službu událostí `eventService`. Na řádku 13 definujeme `events` jako seznam událostí, které se budou zobrazovat. Tato proměnná se ihned incializuje jako prázdné pole `[]`, do kterého se při inicializaci komponenty (`ngOnInit()`, řádek 20) přes `eventService.getAll()` načtou všechny události a postupně se plní do seznamu událostí (řádek 23).

Jako parametru funkce `subscribe(...)` (řádek 23-24) je struktura se dvěma prvky:

* `next` říká, co se má dít s nově příchozím prvkem;
* `error` říká, co se má stát v případě chyby.

Na konci kódu je bonusová funkce `onEventCreated(...)`, která se volá, když vnitřní komponenta vytvoří novou událost. Nově vytvořená událost se pošle do této funkce jako parametr a funkce nový objekt přidá do seznamu objektů.

### Kód popředí - html

Kód na popředí vložíme do souboru `event-list.component.html`:

{% code title="event-list.component.html" lineNumbers="true" %}
```html
<h3>Seznam událostí</h3>

<div *ngIf="events.length == 0">
  ... zatím žádné události
</div>

<div *ngIf="events.length != 0">
  <div class="row">
    <div *ngFor="let event of events" class="col-sm-12 col-md-6 col-lg-4  mb-2">
      <div class="card">
        <div class="card-header d-flex">
          <div>
            {{event.title}}
          </div>
          <div class="ms-auto">
            <div class="badge ml-5"
            [class.bg-primary]="event.notes && event.notes.length > 0"
            [class.bg-secondary]="!event.notes || event.notes.length == 0">
              {{event.notes.length}}
            </div>
          </div>
        </div>
        <div class="card-body">
          {{event.dateTime | czechDatePipe}}
          <hr />
          <app-event-note [event]="event"></app-event-note>
        </div>
      </div>
    </div>
  </div>
</div>

<div>
  <h3 class="mt-3">
    Vytvořit novou událost
  </h3>
  <app-event-create (onCreated)="onEventCreated($event)"></app-event-create>
</div>
```
{% endcode %}

Blok řádků 3-5 se zobrazí pouze v případě, kdy je kolekce událostí prázdná.&#x20;

Naopak, blok řádků 7-31 se zobrazuje, pokud je kolekce událostí neprázdná. V tom případě se každá položka (iterace řádek 9) kreslí do tzv. "karty" BootStrapu ([https://getbootstrap.com/docs/5.0/components/card/](https://getbootstrap.com/docs/5.0/components/card/)). Záhlaví karty obsahuje název události (řádek 13) a badge ([https://getbootstrap.com/docs/5.0/components/badge/](https://getbootstrap.com/docs/5.0/components/badge/)) s počtem poznámek (řádky 16-20, povšimněte si podmíněného přiřazení stylů podle počtu poznámek u události). V samotném těle karty se zobrazí datum události formátovaný námi dříve vytvořenou pipe `CzechDatePipe` (řádek 24) a vloženou komponentou pro poznámky (řádek 26). Konečně, na konci stránky se zobrazuje formulář pro přidání nových událostí s vloženou komponentou (řádek 37).

## Tvorba komponenty event-create

### Kód na pozadí - ts

Kód na pozadí sestaví zejména podkladové prvky pro angular-reactový formulář a obsluhující metodu:

{% code title="event-create.component.ts" lineNumbers="true" %}
```typescript
import {Component, EventEmitter, Output} from '@angular/core';
import {EventService} from "../../services/event.service";
import {FormBuilder, Validators} from "@angular/forms";
import {EventDto} from "../../model/event-dto";

@Component({
  selector: 'app-event-create',
  templateUrl: './event-create.component.html',
  styleUrls: ['./event-create.component.css']
})
export class EventCreateComponent {

  protected form = this.fb.group({
    text: ['', Validators.required],
    dateTime: ['', Validators.required]
  });
  @Output() onCreated = new EventEmitter<EventDto>();

  constructor(
    private eventService: EventService,
    private fb: FormBuilder) {
  }

  protected submitNewEvent() {
    const text = this.form.get("text")?.value!;
    const dateTimeS = this.form.get("dateTime")?.value!;
    const dateTime = new Date(dateTimeS);

    this.eventService.create(text, dateTime)
      .subscribe(q => this.onCreated.emit(q));
  }
}

```
{% endcode %}



### Kód na popředí - html

asef

{% code title="event-create.component.html" lineNumbers="true" %}
```html
<form [formGroup]="form" (ngSubmit)="submitNewEvent()">
  <div class="form-group mt-1">
    <label for="title">Název události</label>
    <input type="text" name="title" id="title" class="form-control" formControlName="text" />
  </div>
  <div class="form-group mt-1">
    <label for="dateTime">Datum a čas události</label>
    <input type="datetime-local" name="dateTime" id="dateTime" class="form-control" formControlName="dateTime" />
  </div>
  <div class="form-group mt-1">
    <button type="submit" class="btn btn-primary" [disabled]="!form.valid">Vytvořit</button>
  </div>
</form>

```
{% endcode %}



## Tvorba komponenty event-note

asef

TODO: Posunout routing pod tvorbu komponent
