# Vytvoření služby

Služba bude fungovat jako mezivrstva mezi komponentami (které budou zobrazovat data) a službou pro komunikaci (která se bude starat o komunikaci přes HTTP). Jedná se o prostředníka, který bude zajišťovat předávání a transformaci dat mezi formáty, které vyžaduje HTTP a které vyžadují komponenty.

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

  public create(title: string, dateTime: Date): Observable<EventDto> {
    return this.http.create(title, dateTime).pipe(
      mergeMap(q => this.get(q))
    );
  }

  public get(eventId: number): Observable<EventDto> {
    return this.http.get(eventId);
  }

  public createNote(eventId: number, text: string): Observable<NoteDto> {
    return this.http.createNote(eventId, text).pipe(
      map(q => ({
          noteId: q,
          text: text
      }))
    );
  }

  public deleteNote = (noteId: number): Observable<void> 
    => this.http.deleteNote(noteId);
}

```
{% endcode %}

