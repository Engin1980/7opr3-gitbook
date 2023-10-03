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

Nyní implementujeme základ všech dalších metod, které využijeme v našem projektu pro práci s rozhraním - vytvoření události, získání události podle id, vytvoření a smazání poznámky. Před demonstrací však ještě ukážeme, jakým způsobem rozšíříme metodu získávání seznamu všech událostí o zachycení chyby.

Nejdříve si vytvoříme pomocnou třídu pro informaci o vzniklé chybě:

```typescript
export class EventHttpError {
  public readonly message: string;
  public readonly inner: any;

  constructor(message: string, inner?: any) {
    this.message = message;
    this.inner = inner;
  }
}
```

A nyní se podíváme na rozšíření chyby. Původní metoda se rozšíří o řádky 5-8 plné tajemných lambda výrazů/arrow-function-výrazů:

{% code lineNumbers="true" %}
```typescript
public list(): Observable<EventDto[]> {
  const url = this.url + "/list";
  const ret = this.http.get<EventDto[]>(url).pipe(
    tap(q => this.printResult("list", q)),
    catchError(e =>
      throwError(() =>
        new EventHttpError("Failed to get list of events", e)))
  );
  return ret;
}
```
{% endcode %}

Za příkaz `tap(...)` (řádek 4) za čárku přidáme další příkaz do sekvence pipe - `catchError(...)`. Tato funkce se volá vždy, když při zpracování vznikne nějaká chyba. Ta do funkce vstoupí jako parametr `e`, který zobrazíme do volání funkce `throwError(...)`. Tato funkce zase umí vyhodit novou výjimku. Naším cílem je zabalit původní výjimku/objekt `e` do nové výjimky/objektu `EventHttpError`. Nechceme ale volat pouhé `new EventHttpError(...)` , ale složitější `() => new EventHttpError(...)`. Rozdíl je v tom, že u prvního volání se předává přímo vytvořený objekt chyby (který se proto musí vytvořit vždycky, i když se funkce `catchError(...)` nezavolá, kdežto druhá varianta předává funkci, která umí chybový objekt zkonstruovat. Samotná konstrukce však proběhne až pouze bude-li to třeba.

{% hint style="warning" %}
Přečtěte si předchozí odstavec ještě jednou. Tato technika (předání lambdy namísto přímo vytvořeného objektu) je velmi důležitá a používá se často. Její pochopení je klíčové.
{% endhint %}

{% hint style="info" %}
Lambda výrazům se technicky v Javascriptu/Typescriptu říká Arrow-Function-Expressions. Více například viz  [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow\_functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow\_functions).
{% endhint %}

Toto zachycení přidáme do každé metody. Výsledný kód třídy/souboru `event-http.service.ts` tedy je:

{% code title="event-http.service.ts" lineNumbers="true" %}
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
      tap(q => this.printResult("list", q)),
      catchError(e =>
        throwError(() =>
          new EventHttpError("Failed to get list of events", e)))
    );
    return ret;
  }

  private printResult(method: string, data: any): void {
    console.log("METHOD " + method);
    console.log(JSON.stringify(data));
  }

  public create(title: string, dateTime: Date): Observable<number> {
    const url = this.url;
    const body = {
      title: title,
      dateTime: dateTime.toISOString()
    };
    const ret = this.http.post<number>(url, body).pipe(
      catchError(e=>
        throwError(() =>
          new EventHttpError(`Failed to crate object title={title} and dateTime={dateTime}`, e)))
    );
    return ret;
  }

  public get(eventId: number) {
    const url = this.url
    const ret = this.http.get<EventDto>(url).pipe(
      tap(q => this.printResult("list", q)),
      catchError(e =>
        throwError(() =>
          new EventHttpError(`Failed to get event by eventId = {eventId}`, e)))
    );
    return ret;
  }

  public createNote(eventId: number, text: string): Observable<number> {
    const url = this.url + "/note";
    const formData = new FormData()
    formData.append("eventId", eventId.toString());
    formData.append("noteText", text);
    const ret = this.http.post<number>(url, formData).pipe(
      catchError(e =>
        throwError(() =>
          new EventHttpError(`Failed to create note with eventId={eventId} and text={text}`, e)))
    );
    return ret;
  }

  public deleteNote(noteId: number): Observable<void> {
    const url = this.url + "/note";
    const httpParams = new HttpParams()
      .set("noteId", noteId);
    const ret = this.http.delete<void>(url, {params: httpParams}).pipe(
      catchError(e =>
        throwError(() =>
          new EventHttpError(`Failed to delete note with noteId={noteId}`, e)))
    );
    return ret;
  }
}

export class EventHttpError {
  public readonly message: string;
  public readonly inner: any;

  constructor(message: string, inner?: any) {
    this.message = message;
    this.inner = inner;
  }
}
```
{% endcode %}

Volitelně můžete do zdrojového kódu vložit i pomocné výpisy odpovědí jak je uvedeno u funkce `list(...)` (řádek 20).
