# Přidání vztahu s vazbou a další úpravy

Poslední demonstrovanou funkcionalitu na back-endu bude přidání vazební tabulky do databáze a úprava zbylých částí kódu. Protože se jedná o relativně triviální změny, budou komentovány pouze důležité  změny v kódu.

Základní myšlenkou je přidání k události (`event`) nějakého poznámky (`note`).

## Vytvoření entity Note a provázání tabulek

Do projektu přidáme entitu `Note`:

```java
package cz.osu.kip.eventReminder.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import org.hibernate.annotations.Comment;

@Entity
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class Note {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private int tagId;

  @Column(length = 128, nullable = false)
  private String text;

  @ManyToOne
  @JoinColumn(name = "event_id")
  private Event event;
}
```

Entita je standardní bez zajímavých oblastí vyjma pole `event`, které reprezentuje vazbu do nadřazené tabulky. Před jeho vysvětlením ještě ukážeme výsek třídy `Event`, kde bude přidána druhá strana vztahu:

```java
// ...
public class Event {
  // ...
  @OneToMany(mappedBy = "event", fetch = FetchType.EAGER)
  private Set<Note> notes = new HashSet<>();
}
```

Jedná se o vazbu 1:N - jedna událost (event) bude mít několik poznámek (note). Proto událost má poznámky jako `Set<Note>`,  zatímco každá poznámka má pouze svou "nadřazenou" událost `Event`.  Na základě toho lze odvodit, že v DB schématu tabulka `Event` se nijak nezmění, ale tabulka `Note` bude mít navíc k atributům id a textu cizí klíč, který nazveme `event_id` - a přesně to říkají anotace `@ManyToOne` (spousta poznámek k jedné události) a `@JoinColumn(name="event_id")` definující mapující cizí klíč. Oproti tomu třída události má pouze zaznačenou informaci `@OneToMany` říkající, že jedna událost má několik poznámek (proto musí být proměnná typu kolekce), propojí se přes proměnnou `Note.event` a konečně poslední (nepovinný) atribut udává, zda se mají hodnoty v relaci načítat ihned (`EAGER`) nebo až při jejich potřebě (`LAZY`).

## Přidání NoteRepository

Zde se pouze přidá rozhraní.

{% code title="NoteRepository.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.model;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface NoteRepository extends JpaRepository<Note, Integer> {
}
```
{% endcode %}

## Úprava služby

### Vytvoření DTO objektů

Nejdříve vytvoříme jednoduchý transportní objekt pro Note - `NoteJTO`:

{% code title="NoteJTO.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.controllers.jtos;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class NoteJTO {
  private int tagId;
  private String text;
}
```
{% endcode %}

Následně upravíme transportní objekt `EventJTO` (pouze výsek kódu):

<pre class="language-java"><code class="lang-java"><strong>// ...
</strong>public class EventJTO {
  // ...
  private Set&#x3C;NoteJTO> notes;
}
</code></pre>

Všimněme si, že `NoteJTO`nemá odkaz na `EventJTO` - taková informace by způsobila zacyklení mezi vlastnostmi `NoteJTO.Event`a `EventJTO.Notest` a objekt by nešel uložit do JSON a vrátit jako výsledek. Právě proto nepoužíváme jako návratové hodnoty REST API přímo entity, ale transportní objekty.

### Upravení služby

Pro `Note` nebudeme vytvářet vlastní službu - její funkcionalita spadá pod události a jsou velmi triviální, proto její správu necháme u služby `EventService`.&#x20;

Tentokráte budou úpravy ve třídě `EventService` rozsáhlejší - musíme nahradit všechny návratové typy endpointů z typu `Event` na typ `EventJTO`, spolu s tím realizovat i související mapování `Event` -> `EventJTO` a přidat metody pro přidání a smazání poznámky. Navíc, pro práci s poznámkami musíme do služby přidat i `NoteRepository`.

{% code title="EventService.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.services;

import cz.osu.kip.eventReminder.controllers.jtos.EventJTO;
import cz.osu.kip.eventReminder.model.Event;
import cz.osu.kip.eventReminder.model.EventRepository;
import cz.osu.kip.eventReminder.model.Note;
import cz.osu.kip.eventReminder.model.NoteRepository;
import lombok.RequiredArgsConstructor;
import org.modelmapper.ModelMapper;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.List;
import java.util.NoSuchElementException;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class EventService {
  private final EventRepository eventRepository;
  private final NoteRepository noteRepository;
  private final ModelMapper mm = new ModelMapper();

  private static final int EMPTY_ID = -1;

  public int create(String title, LocalDateTime dateTime) {
    Event event = new Event(EMPTY_ID, title, dateTime, new HashSet<>());
    eventRepository.save(event);
    return event.getEventId();
  }

  public EventJTO getById(int eventId) {
    EventJTO ret = this.eventRepository
            .findById(eventId)
            .map(q->mm.map(q, EventJTO.class))
            .orElseThrow();
    return ret;
  }

  public List<EventJTO> getAll() {
    List<EventJTO> ret = this.eventRepository
            .findAll(Sort.by("dateTime"))
            .stream()
            .map(q->mm.map(q, EventJTO.class))
            .collect(Collectors.toList());
    return ret;
  }

  public void update(EventJTO event) {
    Event original = this.eventRepository.getReferenceById(event.getEventId());
    ModelMapper modelMapper = new ModelMapper();
    modelMapper.map(event, original);
    this.eventRepository.save(original);
  }

  public void delete(int eventId){
    Event event = this.eventRepository.getReferenceById(eventId);
    this.eventRepository.delete(event);
  }

  public void addNote(int eventId, String noteText){
    if (!this.eventRepository.existsById(eventId))
      throw new NoSuchElementException("Event id=" + eventId + " not found.");
    Event event = this.eventRepository.getReferenceById(eventId);
    Note note = new Note(EMPTY_ID, noteText, event);
    this.noteRepository.save(note);
  }
  
  public void deleteNote(int noteId){
    Note note = this.noteRepository.getReferenceById(noteId);
    this.noteRepository.delete(note);
  }
}
```
{% endcode %}

Změněné či nové funkce:

* `getById()` vrací EventJTO, kterou získá pomocí model-mapperu (řádek 24, 37); pokud záznam nebyl nalezen, vrací výjimku,
* `getAll()` vrací list událostí seřazených podle data a také přemapovaných do `EventJTO` (řádky 44-47),
* `addNote(...)` přidává poznámku k události - k tomu potřebuje `eventId` a `noteText`. Nejdříve zkontroluje, zda odpovídají event dle ID existuje (řádek 64-65), následně získá referenci na objekt (řádek 66), vytvoří dle jeho ID novou poznámku (řádek 67) a uloží jej,
* `deleteNote(...)` smaže poznámku dle id.
