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
