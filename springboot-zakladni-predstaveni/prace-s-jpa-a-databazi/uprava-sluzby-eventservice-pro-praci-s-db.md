# Úprava služby EventService pro práci s DB

Posledním krokem je úprava služby EventService, aby namísto lokální kolekce pracovala s entitou.&#x20;

## Vložka I: Data-Transfer objects

Při práci na backendu typicky pracujeme s entitami - v službě (Service) je pomocí Repository získáváme z DB a dále s nimi pracujeme. Je však nevhodné je předávat v čisté formě dál na REST API - například protože obsahují citlivé nebo zbytečné údaje, které není třeba ven z REST API vystavovat, nebo protože, jelikož se jedná o mapování do DB, mohou obsahovat reference, které při vracení přes REST API a formátování do JSON způsobí cyklické závislosti - například Firma odkazuje na své zaměstnance a ti zase odkazují zpátky na svou firmu... `company.emplyees[0].company.employees[0].company.employees[0]...`.&#x20;

Proto je běžné definovat speciální třídy, které slouží pro předávání dat přes REST API, a do kterého/z kterého se mapují pak konkrétní entity, se kterými pracujeme. V projektu již podobná třída existuje, jmenuje se `EventData` a slouží pro předání parametrů nové instance. Tyto třídy se typicky odkazují jako **Data Transfer Object** a typicky (ne však nutně) mají u svého názvu postfix DTO.

Pro předávání dat a update záznamu si vytvoříme podobnou třídu `EventJTO` (která však již bude obsahovat i `eventId`) a vložíme ji do projektu:

{% code title="EventJTO.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.controllers.jtos;

import jakarta.persistence.Column;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.LocalDateTime;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class EventJTO {
  private int eventId;
  private String title;
  private LocalDateTime dateTime;
}

```
{% endcode %}

Pomocí Lombok je vytvoření velmi jednoduché - definujeme pouze vnitřní proměnné a všechno zabalíme pomocí Lombok anotací (řádky 11-14).

## Vložka II: Mapovací knihovna

safse

```xml
<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>3.1.1</version>
</dependency>
```

fase

## Úprava služby EventService

Uvedeme celý nový kód `EventService` s následným vysvětlením.

{% code title="EventService.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.services;

import cz.osu.kip.eventReminder.controllers.jtos.EventJTO;
import cz.osu.kip.eventReminder.model.Event;
import cz.osu.kip.eventReminder.model.EventRepository;
import lombok.RequiredArgsConstructor;
import org.modelmapper.ModelMapper;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class EventService {
  private final EventRepository eventRepository;

  private static final int EMPTY_ID = -1;

  public int create(String title, LocalDateTime dateTime) {
    Event event = new Event(EMPTY_ID, title, dateTime);
    eventRepository.save(event);
    return event.getEventId();
  }

  public Optional<Event> getById(int eventId) {
    Optional<Event> ret = this.eventRepository.findById(eventId);
    return ret;
  }

  public List<Event> getAll() {
    List<Event> ret = this.eventRepository.findAll(Sort.by("dateTime"));
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
}
```
{% endcode %}

Ze třídy byly odstraněny všechny zmínky o kolekci `List<Event> events`:

* Přidali jsme definici (řádek 18), která automaticky vloží do instance služby instanci repository, kterou JPA generuje automaticky (pamatujme, že `EventRepository` je jenom rozhraní.
* Aby se tato třída automaticky naplnila přes konstruktor, před třídu přidáme Lombok anotaci `@RequiredArgsConstructor`; ta zajistí, že vznikne konstruktor, který automaticky vloží hodnotu do `eventRepository` (viz info okno níže).
* Definujeme konstantu `EMPTY_ID` (řádek 20) pro nastavení ID novým entitám.
* Ve funkci `create()`(řádky 22-26) se vytvoří nová entita, nastaví se jí parametry podle předaných hodnot a jednoduchým voláním funkce `save()` se uloží do DB. Následně se z ní vrátí nově vygenerované ID, které vzniklo při uložení do databáze.
* Ve funkci `getById(...)` se z repository potencionálně (`Optional`) získá instance objektu dle id.
* Ve funkci `getAll()` se vrátí všechny záznamy. Volání `findAll(...)` je navíc zkrášleno seřazením vrácených záznamů podle datumu a času události (řádek 34).
* Funkce `update()` využívá JTO, aby aktualizovala záznam v DB. Nejdříve zjistí odkazový objekt eventu (řádek 39)(objekt nepotřebujeme načítat celý, stačí nám reference), následně jej naplníme hodnotami z JTO (řádek 41) a uložíme (řádek 42).
* Funkce `delete()` vymaže záznam z DB dle ID. Nejdříve se zjistí odkazový objekt (řádek 46) a ten se následně smaže (řádek 47).
