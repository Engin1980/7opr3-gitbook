# Vytvoření služby

Služba bude fungovat jako mezivrstva mezi komponentami (které budou zobrazovat data) a službou pro komunikaci (která se bude starat o komunikaci přes HTTP). Jedná se o prostředníka, který bude zajišťovat předávání a transformaci dat mezi formáty, které vyžaduje HTTP a které vyžadují komponenty.

Nejdříve si představíme teoretická východiska řešení některých problémů a následně ukážeme kód třídy. Pokud chcete vidět přímo finální zdrojový kód, přeskočte na následující blok.

## Výchozí myšlenky implementace

Představme si na  začátku obecnou implementaci třídy, která má v sobě proměnnou `http` typu `EventHttpService`.  Díky ní potřebujeme vytvořit metody, které budou obsluhovat vrácení položek. Jednoduchým příkladem bude metoda `get(...)`:

{% code lineNumbers="true" %}
```typescript
public get(eventId: number): Observable<EventDto> {
  return this.http.get(eventId);
}
```
{% endcode %}

V této metodě jen jednoduše voláme `http` s cílem vrátit výsledek jeho metody `get(...)`. Protože `this.http.get(...)` vrací `Observable<EventDto>`, tak pokud není třeba nic navíc (například logování nebo zachycení chyby), stačí vše vrátit příkazem `return`.

### Získávání listu položek

U listu položek může být na první pohled situace identická. Můžeme jednoduše vrátit list všech položek jako:

{% code lineNumbers="true" %}
```typescript
public getAll(): Observable<EventDto[]> {
  return this.http.list();
}
```
{% endcode %}

Někdy však může být výhodné nevrátit voláním celý list najednou (jako jedno vyvolání u `Observable<EventDto[]`), ale využít observable k tomu, aby vracel postupně všechny položky listu (jako "n" vyvolání u `Observable<EventDto>`). Tehdy potřebujeme převést mapování z `Observable<EventDto[]` -> `Observable<EventDto>`. K tomu využijeme `pipe(...)` a mapovací funkci `mergeMap(...)`, která provádí mapování listu `[]` do observablů jednotlivých položek listu:

{% code lineNumbers="true" %}
```typescript
public getAll(): Observable<EventDto> {
  return this.http.list().pipe(
    concatMap(q => q)
  );
}
```
{% endcode %}

### Ukládání položek

Při ukládání položek `http` typicky vrací (jen) ID uložené položky. Někdy ale zpátky do komponenty potřebujeme vrátit celou vytvořenou položku (protože komponenta ji potom následně chce zobrazit). V našem případě by se nám tedy líbilo provést mapování z `eventId : number` -> `event : EventDto`.&#x20;

```typescript
public create(title: string, dateTime: Date): Observable<EventDto> {...}
```

Jsou dva jednoduché způsoby, jak to udělat.

Prvním je konstrukce objektu na straně služby - id jsme získali jako výsledek volání nad `http` a všechny ostatní parametry jsme získali od komponenty při volání funkce. Sestavíme tedy po volání `http` výsledný objekt a ten vrátíme:

{% code lineNumbers="true" %}
```typescript
public createAlternative(title: string, dateTime: Date): Observable<EventDto> {
  return this.http.create(title, dateTime).pipe(
    map(q => {
      const ret: EventDto = {
        eventId: q,
        title: title,
        dateTime: dateTime,
        notes: []
      };
      return ret;
    })
  );
}
```
{% endcode %}

Vidíme, že výsledek volání `this.http.create(...)` si odchytíme pomocí `pipe(...)` a následně zavoláme funkci `map(...)`, která realizuje mapování `X` -> `Y` (v našem případě `number` -> `EventDto`).  V ní si vezmeme parametr `q` (to je vrácené `eventId`) a sestavíme nový objekt dle `EventDto` (řádky 4-9).

Druhým způsobem je po získání `eventId` znovu požádat `http` o vrácení objektu podle id přes `this.http.get(eventId)`. Budeme sice dělat volání navíc, ale získáme prokazatelně obraz objektu z  backendové vrstvy.

{% code lineNumbers="true" %}
```typescript
public create(title: string, dateTime: Date): Observable<EventDto> {
  return this.http.create(title, dateTime).pipe(
    mergeMap(q => this.get(q))
  );
}
```
{% endcode %}

V tomto případě využijeme funkci `mergeMap(...)`, která umí dělat transformaci z `Observable<X>`->`Observable<Y>`, tedy v našem případě z `Observable<number>` -> `Observable<EventDto>`. Mapování provedem voláním přes `this.http.get(q)`, kde v `q` máme vrácené `eventId`.

### Funkce jako lambdy

TypeScript/JavaScript umožňují využívat lambda funkce pro definici chování metod. Mějme triviální metodu, která dělá pouhé jednoduché vrácení hodnoty:

{% code lineNumbers="true" %}
```typescript
public deleteNote (noteId: number): Observable<void> {
  return this.http.deleteNote(noteId);
}
```
{% endcode %}

Toto volání lze zkrátit s využitím `=>` jako:

{% code lineNumbers="true" %}
```typescript
public deleteNote = 
  (noteId: number): Observable<void> => this.http.deleteNote(noteId);
```
{% endcode %}

{% hint style="info" %}
Pokud jste zvyklí na podobnou syntax z Java/C#, povšimněte si důležitého znaku `=` mezi názvem a deklarací signatury metody.

Obdobnou odlišností je nemožnost  zalomit řádek před/za `=>`.
{% endhint %}

## Vlastní implementace služby

Do složky /src/app/components přidáme novou službu `EventService` pomocí příkazové řádky jako:

```powershell
ng g s event
```

Vytvoří se nový soubor `event.service.ts` s třídou `EventSevice`, u které nahradíme obsah:

{% code title="" lineNumbers="true" %}
```typescript
import {Injectable} from '@angular/core';
import {concatMap, map, mergeMap, Observable} from "rxjs";
import {EventDto} from "../model/event-dto";
import {EventHttpService} from "./event-http.service";
import {NoteDto} from "../model/note-dto";

@Injectable({
  providedIn: 'root'
})
export class EventService {

  constructor(
    private http: EventHttpService
  ) {
  }

  public getAll(): Observable<EventDto> {
    return this.http.list().pipe(
      concatMap(q => q)
    );
  }

  public get(eventId: number): Observable<EventDto> {
    return this.http.get(eventId);
  }

  public create(title: string, dateTime: Date): Observable<EventDto> {
    return this.http.create(title, dateTime).pipe(
      mergeMap(q => this.get(q))
    );
  }

  public createNote(eventId: number, text: string): Observable<NoteDto> {
    return this.http.createNote(eventId, text).pipe(
      map(q => ({
        noteId: q,
        text: text
      }))
    );
  }

  public deleteNote =
    (noteId: number): Observable<void> => this.http.deleteNote(noteId);
}
```
{% endcode %}

