# Služba pro komunikaci s REST API

Nyní vytvoříme službu, která bude komunikovat s backendovým REST API.

{% hint style="info" %}
Nezapomenout, že pro to, aby zde vytvářené řešení fungovalo, musí být spuštěna back-end aplikace a  databázový server.
{% endhint %}

## Tvorba DTO objektů

Nejdříve vytvoříme DTO objekty. Jejich obsah bude shodný s požadovanými předávanými daty z/na backend. Předává se objekt `EventDTO`, jenž obsahuje odkaz na list `NoteDTO` objektů. Do složky src/app/model tedy vytvoříme dva soubory a vložíme jejich obsah:

{% code title="note-dto.ts" lineNumbers="true" %}
```typescript
export interface NoteDto {
  noteId : number;
  text : string;
}
```
{% endcode %}

{% code title="event-dto.ts" lineNumbers="true" %}
```typescript
import {NoteDto} from "./note-dto";

export interface EventDto {
  eventId : number;
  title : string;
  dateTime : Date
  notes : NoteDto[]
}
```
{% endcode %}

DTO "třídy" zde vytváříme jako rozhraní. HTTP služba při vracení hodnot by totiž neuměla vracet naše třídy, ale vracela by jen objekty, které by vypadaly jako naše třídy. Obsahovaly by tedy předdefinované atributy, ale pokud bychom například do `EventDTO` přidali nějakou metodu, objekt vrácený z HTTP služby by tuto metodu neobsahoval. Abychom neměli tendenci do DTO objektů přidávat funkcionalitu navíc, vytvoříme je jako rozhraní, které žádné další operace obsahovat nebudou.

## Tvorba základu HTTP služby

Dalším bodem bude vytvoření http služby. Do složky /app/src/services přidáme novou službu \`EventHttpService'.

```powershell
ng g s event-http
```

Vytvoří se soubor `event-http.service.ts`, ve kterém nahradíme kód:

{% code title="event-http.service.ts - návrh I" lineNumbers="true" %}
```typescript
import {catchError, Observable, tap, throwError} from "rxjs";
import {EventDto} from "../model/event-dto";
import {HttpClient, HttpErrorResponse, HttpParams} from "@angular/common/http";
import {Injectable} from "@angular/core";

@Injectable({
  providedIn: 'root'
})
export class EventHttpService {
  private url = "http://localhost:8080/event"

  constructor(
    private http: HttpClient
  ) {
  }

  public list(): Observable<EventDto[]> {
    const url = this.url + "/list";
    const ret = this.http.get<EventDto[]>(url).pipe(
      tap(q => this.printResult("list", q))
    );
    return ret;
  }

  private printResult(method: string, data: any): void {
    console.log("METHOD " + method);
    console.log(JSON.stringify(data));
  }  
}

```
{% endcode %}

Nyní si obsah souboru rozebereme:

* řádek 6-8 definují, že tato služba je injektovatelná přes Dependency Injection do dalších služeb a komponent v rámci Angular,
* řádek 10 definuje vlastní proměnnou odkazující na (protokol a) adresu, kde běží back-end,
* řádek 12-15 je konstruktor vytvářející novou instanci; je vhodné si povšimnout injektace objektu `HttpClient` na řádku 13, který se stává novou proměnnou `http` třídy,
* řádek 17 definuje funkci vracející načtený seznam všech událostí `EventDTO[]`. Nejdříve se seskládá dotazovací Url (řádek 18, `http://localhost:8080/event/list`, srovnej s implementaci API end-pointu v `EventController.java` ve SpringBoot), a následně se provede HTTP-GET požadavek (řádek 19) přes objekt http: `this.http.get<Event[]>(...)`. Generický parametr ve složených závorkách udává očekávaný návratový typ výsledku po GET operaci. Parametru se předá pouze URL.&#x20;
* Čistě pro ladicí účely jsme provedli ještě drobný rozbor výsledk. Na řádku 19 jsme funkci `pipe(...)` řekli, že před vrácením výsledku s ním budeme chtít ještě pracovat,
* na řádku 20 si na něj (výsledek) sáhneme funkcí `tap(...)`, kdy daný vrácený výsledek - objekt `q` - předáme na vytištění funkci `printResult(...)` - řádek 20.
* Samotná funkce `printResult(...)` jen vytiskne informace o použité metodě, následně převede hodnotu do formát JSON a vypíše jej. Oba výpisy nalezneme na konzoli prohlížeče.

Toto je základní implementace vrácení všech dat událostí z backendu. Protože v aplikaci budeme finálně potřebovat i další operace, budeme v naší implementaci ještě pokračovat. Nyní zde ale můžete skončit a přeskočit na čtení o implementaci zobrazení všech událostí v komponentě (další sekce); pro plnou funkcionalitu je třeba se sem později vrátit a doplnit i ostatní funkce do třídy `EventHttpService`.

## Přidání dalších metod pro práci s rozhraním

fasef

## Přidání zachytávání chyb

asef
