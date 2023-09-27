# Vytváření API Endpointů

## Vytvoření pomocných tříd

Prvním krokem bude vytvoření jednoduché třídy `Event`, která bude uchovávat data o uložených událostech. Třídu umístíme do balíčku `model` pod naším projektem. Bude obsahovat položky pro id, název a datum události.

{% code title="Entity.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.model;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;

import java.time.LocalDateTime;

@Getter
@Setter
@AllArgsConstructor
public class Event {
  private int eventId;
  private String title;
  private LocalDateTime dateTime;
}
```
{% endcode %}

Povšimněme si anotací před názvem třídy - řádky 8 a 9: `@Getter @Setter`. Jedná se o anotace projektu _Lombok_, které zajistí, že do kódu nemusíme sami generovat get/set metody pro soukromé proměnné, ale vytvoří se automaticky. Obdobně anotace na řádku 10 zajistí, že se automaticky vytvoří konstruktor pro všechny proměnné třídy. Kód třídy vypadá přehledněji.

Dalším krokem bude vytvoření třídy realizující úložiště pro naše data. Protože neukládáme data do databáze, vystačíme si s velkým zjednodušením s pomocí třídy `EventService`.

{% code title="EventService.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.services;

import cz.osu.kip.eventReminder.model.Event;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@Service
public class EventService {
  private final List<Event> events = new ArrayList<>();

  public int create(String title, LocalDateTime dateTime) {
    int id = 1 + this.events.stream()
            .mapToInt(q -> q.getEventId())
            .max()
            .orElse(0);
    Event event = new Event(id, title, dateTime);
    this.events.add(event);
    return id;
  }

  public Optional<Event> getById(int eventId) {
    Optional<Event> ret = this.events.stream()
            .filter(q -> q.getEventId() == eventId)
            .findFirst();
    return ret;
  }

  public List<Event> getAll() {
    List<Event> ret = new ArrayList<>(this.events);
    return ret;
  }
}
```
{% endcode %}

Metoda `create(...)`na řádcích 15-23 vytvoří z nových parametrů novou instanci třídy `Event`. Pro získání jejího ID využije vyčtení maximální hodnoty z aktuálně uložených objektů událostí v seznamu událostí `events`. Vytvořený event se vloží do tohoto seznamu.

Metody `getById(...)` a `getAll()` pak vracejí jednu událost podle ID, respektive všechny události z listu.

{% hint style="info" %}
V kódu se využívá funkcionality třídy `Optional`, viz například [https://www.baeldung.com/java-optional](https://www.baeldung.com/java-optional).
{% endhint %}

{% hint style="warning" %}
Třída má nahoře (řádek 11) uvedenou anotaci `@Service`. Tato anotace způsobí, že se třída bude chovat jako služba a lze u ní využívat vkládání závislostí - dependency injection. Bude ukázáno dále.
{% endhint %}

## Vytvoření endpointů pro práci s událostmi

### Definice očekávaných endpointů

Před vytvořením endpointů pro práci s událostmi si nejdříve načrtneme, jak by dané endpointy mohly vypadat:

<table><thead><tr><th width="168">End-point</th><th width="144">Http metoda</th><th width="227">Parametry</th><th>Výstup</th></tr></thead><tbody><tr><td>/event</td><td>POST</td><td>title, dateTime </td><td>ID</td></tr><tr><td>/event</td><td>GET</td><td>eventId</td><td>Event</td></tr><tr><td>/event/list</td><td>GET</td><td>-</td><td>Event[]</td></tr></tbody></table>

Předávané parametry i návraty jsou vždy v textové podobě, bude ukázána varianta předání jako (zjednodušeně) prostého textu, nebo ve formátu JSON.

První end-point vytváří nový objekt (proto metoda POST), očekává dva parametry - titulek a datum+čas události; jako výstup vrací ID vytvořené události. Druhý end-point očekává na vstupu ID události a vrací danou událost ve formátu JSON. Poslední endpoint vrací všechny známé události jako pole ve formátu JSON. Podle prefixu `event` je vidět, že všechny endpointy patří do stejného kontroleru.

### Vytvoření kontroleru pro endpointy

Kontroler je v základu běžná třída. Proto do projektu (do správného balíčku, viz níže) přidáme nový kontroler-třídu `EventController`. Že to je REST kontroler naznačíme anotací `@RestController`.

{% code title="EventController.java (kostra)" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.controllers;

import cz.osu.kip.eventReminder.services.EventService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/event")
public class EventController {
  
  @Autowired private EventService eventService;
  
}

@Getter
@Setter
class EventData {
  private String title;
  private LocalDateTime dateTime;
}
```
{% endcode %}

Už v tomto kódu si lze povšimnout dvou základních zajímavostí:

* Anotace `@RestController` - tato anotace způsobí, že SpringBoot bude třídu chápat jako REST kontroler. Projde její metody a ty z nich, které budou vhodně označené (uděláme dále) vystaví ven z aplikace jako vstupní REST API.
* Anotace `@RequestMapping("/event")` specifikuje pro celý kontroler (a jeho endpointy) výchozí mapovací cestu. Pro server `www.test.cz` proto tento kontroler a jeho endpointy budou dostupné na cestě `www.test.cz/event`.
* Proměnná třídy `eventService` s anotací `@Autowired` - tato anotace je navázána na tzv. Dependency Injection (DI). DI je technika, kdy programátor sám nevytváří instance tříd, ale tyto instance jsou mu samy vkládány do připravených proměnných k použití. Výše jsme vytvořili novou třídu `EventService`, ale nikde z ní nevytváříme instanci (nikde nemáme a ani nebudeme mít kód typu `new EventService()`). Místo toho jsme proměnnou označili anotací `@Autowired`, a to způsobí, že SpringBoot sám vytvoří instance požadované třídy a vloží ji do námi připravené proměnné. My tuto instanci pak už jen budeme používat.
* Nakonec si připravíme pomocnou třídu, pomocí které budeme předávat data při vytváření nové událost do endpointu - nazveme ji `EventData`.

{% hint style="info" %}
Problematika DI ve SpringBoot je komplexnější. Zájemce o bližší studium odkazujeme například na [https://www.baeldung.com/spring-dependency-injection](https://www.baeldung.com/spring-dependency-injection).
{% endhint %}

{% hint style="info" %}
DI vkládá instance služeb s ohledem na jejich tzv. _scope_. Ten říká, zda má SpringBoot při požadavku na DI vkládat stejnou instanci, jakou využil už dříve, nebo zda má vytvářet instance nové. Ve výchozím nastavení jsou všechny služby (i dále uvedené typy vkládaných DI objektů) chápány jako Singletony - vytvoří se jednou a používají se pro běh celé aplikace. Toto chování lze ale v případě potřeby změnit. Pro bližší zájemce viz [https://www.baeldung.com/spring-bean-scopes](https://www.baeldung.com/spring-bean-scopes).
{% endhint %}

{% hint style="info" %}
Třídám, která předávají data mezi endpointem a jejich volajícími, se typicky říká _Data Transfer Object_ a nezřídka mají u svého názvu postfix `DTO`.&#x20;
{% endhint %}

#### Endpoint pro vytvoření nové události

Nyní vytvoříme endpoint pro vytvoření nové události:

```java
  @PostMapping
  public int create(@RequestBody EventData data){
    int ret = this.eventService.create(data.getTitle(), data.getDateTime());
    return ret;
  }
```

Tento jednoduchý endpoint bude díky `@GetMapping` dostupný přes HTTP metodu POST (vkládáme data pro nový záznam), na adrese `.../event` a bude očekávat dva parametry. Definice `@GetMapping` je vlastně zkrácenou verzí anotace `@RequestMapping(method = RequestMethod.POST)`, která už byla použita u kontroleru. Anotace `@RequestBody` říká, že SpringBoot má při volání endpointu hledat hodnoty parametrů v těle požadavku.

{% hint style="info" %}
Je více způsobů, jak předat data do požadavku; nejběžnější jsou `@RequestBody`,  `@RequestParam`, či `@PathVariable`. Další varianty mohou být například `@RequestHeader`. Zájemce o bližší studiu odkážeme na internet.
{% endhint %}

Pokud budeme chtít na takový endpoint poslat data, můžeme využít například format JSON a parametry předat v těle požadavku:

```json
{
  "title" : "Moje událost",
  "dateTime" : "2023-12-24T18:30:20"
}
```

{% hint style="warning" %}
Pozor, názvy parametrů v datech musí přesně sedět na názvy parametrů endpointu. V opačném případě se namapování objektu nezdaří a SpringBoot vrátí (možná trochu překvapivě) chybu 404. Pokud bude mapování názvů parametrů správné, ale nebudou sedět datové typy hodnot, zobrazí SpringBoot tuto informaci na konzoli.
{% endhint %}

#### Endpoint pro získání události podle ID

Pro získání informace vložíme do kontroleru novou funkci:

```java
  @GetMapping
  public Event getById(@RequestParam int eventId){
    Event ret = this.eventService.getById(eventId).orElse(null);
    return ret;
  }
```

Tentokrát se jedná o mapování přes HTTP-GET - `@GetMapping`, a očekává se jako vstup id požadované události. Pro zajímavost jsme zde zvolili, že data se nebudou předávat přes JSON v těle požadavku, ale jako parametr požadavku (viz dále v testování požadavku).

{% hint style="info" %}
V kódu neřešíme chybový stav, kdy objekt dle ID nebyl nazelen - zatím vracíme jednoduše `null`. V sekci věnované práci s chybami v rámci endpointů bude tento problém vyřešen.
{% endhint %}

#### Endpoint pro získání všech událostí

Pro získání všech událostí vložíme do kontroleru novou funkci:

```java
  @GetMapping("/list")
  public List<Event> list(){
    List<Event> ret = this.eventService.getAll();
    return ret;
  }
```

I tato funkce je přímočará. Jedinou zajímavostí je uvedení složitější varianty anotace `@GetMapping("/list")`. Už výše uvedená funkce `getById(...)` používá HTTP-GET na adrese kontroleru `/event`. Kdybychom definovaly druhou funkci jako endpoint na HTTP-GET, SpringBoot by při požadavku na `.../event` nevěděl, kterou z nich má vybrat. Proto musíme druhý HTTP-GET endpoint odlišit jinou url (viz výše tabulka endpointů) na `/event/list`. Část `/event` je již uvedena jako výchozí cesta u celého kontroleru (viz anotaci u názvu třídy), část `/list` je uvedena u samotné metody.

#### Shrnutí kódu kontroleru

{% code title="EventController.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.controllers;

import cz.osu.kip.eventReminder.model.Event;
import cz.osu.kip.eventReminder.services.EventService;
import lombok.Getter;
import lombok.Setter;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;
import java.util.List;

@RestController
@RequestMapping("/event")
public class EventController {

  @Autowired
  private EventService eventService;

  @PostMapping
  public int create(@RequestBody EventData data) {
    int ret = this.eventService.create(data.getTitle(), data.getDateTime());
    return ret;
  }

  @GetMapping
  public Event getById(@RequestParam int eventId) {
    Event ret = this.eventService.getById(eventId).orElse(null);
    return ret;
  }

  @GetMapping("/list")
  public List<Event> list() {
    List<Event> ret = this.eventService.getAll();
    return ret;
  }
}

@Getter
@Setter
class EventData {
  private String title;
  private LocalDateTime dateTime;
}

```
{% endcode %}
