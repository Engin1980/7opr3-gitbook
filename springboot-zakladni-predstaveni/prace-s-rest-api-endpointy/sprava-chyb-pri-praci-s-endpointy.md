# Správa chyb při práci s endpointy

Nedílnou součástí práce s endpointy je práce s chybami. Chyby můžou vzniknout hned na několika místech - počínaje nevhodnými daty předanými na endpoint, konče selháním služeb, které endpointy využívají ke splnění požadavku. V takových případech typicky endpoint není schopen splnit požadovanou činnost a musí vrátit neúspěch. Jako neúspěch běžně vracíme odpovídající stavový kód HTTP a případně vysvětlující informaci.

{% hint style="info" %}
Pro tuto sekci předpokládáme dobrou seznámenost čtenáře s prací s výjimkami v jazyce Java (například [https://www.baeldung.com/java-exceptions](https://www.baeldung.com/java-exceptions)) a také s návratovými HTTP stavovými kódy (například [https://cs.wikipedia.org/wiki/Stavov%C3%A9\_k%C3%B3dy\_HTTP](https://cs.wikipedia.org/wiki/Stavov%C3%A9\_k%C3%B3dy\_HTTP)).
{% endhint %}

Obecně, chyby na výstupu endpointu (tedy v metodě, která jej obsluhuje), mohou vzniknout dvěma způsoby - buď chybu chce vyvolat sám programátor (obdoba cíleného vyvolání nové výjimky) nebo výjimka vznikne sama někde uvnitř kódu obsluhujícího endpoint - v takovém případě musí programátor problémový blok kódu ošetřit pomocí sekvence `try-catch`. Po  zachycení výjimky je pokračování v obou případech shodné.

V hotových implementacích jsme si ukázali, že programátor z endpointu při úspěšném provedení vrací nějaká data (připadně nic - hodnotu `void`) a spolu s vrácením objektu se vrací stavový kód HTTP 200 - OK, například v dříve implementované funkci:

{% code lineNumbers="true" %}
```java
  @GetMapping
  public Event getById(@RequestParam int eventId) {
    Event ret = this.eventService.getById(eventId).orElse(null);
    return ret;
  }
```
{% endcode %}

Řekněme, že v případě, že událost podle daného ID nebyla nalezena, chceme vrátit chybu.

## Vrácení chyby přes ResponseEntity<>

První variantou je využití datového typu `ResponseEntity<>` jako návratového typu z funkce. Tento typ je zapouzdření vracejících dat, kdy mu můžeme definovat i vlastní HTTP stavový kód, tělo, či přistoupit do hlaviček požadavku.&#x20;

{% hint style="info" %}
Pro bližší informace o možnostech typu `ResponseEntity<>` viz například [https://www.baeldung.com/spring-response-entity](https://www.baeldung.com/spring-response-entity).
{% endhint %}

Upravený kód tedy bude:

{% code lineNumbers="true" %}
```java
  @GetMapping
  public ResponseEntity<?> getById(@RequestParam int eventId) {
    Optional<Event> event = this.eventService.getById(eventId);
    ResponseEntity<?> ret;
    
    if (event.isPresent())
      ret = ResponseEntity.ok(event);
    else
      ret = new ResponseEntity<>(
        "Event with eventId=" + eventId + " not found.", 
        HttpStatus.NOT_FOUND);

    return ret;
  }
```
{% endcode %}

Co se změnilo?

* Na řádku 2 se  změnil návratový typ metody na `ResponseEntity<?>`. Nemůžeme použít `ResponseEntity<Event>`, protože potřebujeme vracet buď `Event` (v případě úspěchu) nebo objekt bez udaného typu (v případě chyby).&#x20;
* Řádky 3, 4 obsahují deklarace a vrácení hodnoty objektu `Event`, pokud existuje.
* Pokud event existuje (řádek 6), vytvoříme novou instanci `ResponseEntity` jak `ok(...)` stav se stavovým kódem HTTP OK - 200 a hodnotou `event` v těle odpovědi (řádek 7).
* Pokud event neexistuje (řádek 9-11), vytvoříme novou instanci `ResponseEntity` obsahující tělo zprávy (řetězec, řádek 10) a požadovaný kód HTTP NOT FOUND - 404.

Nyní lze celou funkcionalitu vyzkoušet přes Postman. V případě úspěchu se vrací objekt jako dříve, v případě neúspěchu se vrátí chyba 404 - not found a tělo obsahující prostý text chyby.

{% hint style="info" %}
Ohledně vracení chyby lze samozřejmě vracet nejen prostý text (string), ale i komplexní objekt obsahující nějaké další informace. Parametr `body` konstruktoru `ResponseEntity` je datového typu `Object` a v případě potřeby se sám převede na JSON.
{% endhint %}

## Vrácení chyby přes výjimku ResponseStatusException

Další variantou je nevracet přímo vlastní vytvořený `ResponseEntity`objekt, ale vyhodit přímo k tomuto chování určenou výjimku `ResponseEntityStatus`.

{% code lineNumbers="true" %}
```java
  @GetMapping
  public Event getById(@RequestParam int eventId) {
    Optional<Event> event = this.eventService.getById(eventId);
    
    if (event.isEmpty())
      throw new ResponseStatusException(
        HttpStatus.NOT_FOUND, 
        "Event with eventId=" + eventId + " not found.");
    
    return event.get();
  }
```
{% endcode %}

Změny oproti předchozí variantě:

* Návratový typ endpointu nyní bude nyní zpět `Event` (případně `ResponseEvent<Entity>`, pokud chce programátor využít tuto implementaci).
* V kódu se zkontroluje (řádek 5), zda byla událost nalezena; pokud ne, vyhodí se nová instance výjimky `ResponseStatusException` (řádek 6), která se jako parametr předá opět status chyby (404 - NOT FOUND, řádek 7) a chybová zpráva/objekt (řádek 8).

## Globální zachytávání chyb ze všech kontrolerů - RestControlerAdvice

Protože realizace zachytávání všech chyb ve všech endpointech všech kontrolerů by bylo náročné, existuje univerzální mechanismus umožňující zachytit chyby vzniklé ve vybraném/všech kontrolerech. Tento mechanismus je založen na tzv _Rest-Controller-Advice_ mechanismu.

Nejdříve do projektu uděláme novou třídu `ControllerExceptionHandler`, která bude zachytávat výjimky v kontrolerech:

{% code title="ControllerExceptionHandler.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.controllers;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;

import java.util.NoSuchElementException;

@RestControllerAdvice
public class ControllerExceptionHandler {

  @ExceptionHandler(NoSuchElementException.class)
  public ResponseEntity<String> noSuchElementExceptionHandler(
    NoSuchElementException ex, 
    WebRequest request){
    return new ResponseEntity<>("Element not found", HttpStatus.NOT_FOUND);
  }

}
```
{% endcode %}

Zajímavosti:

* Třída má u sebe anotaci `@RestControllerAdvice`, která udává, že třída bude zachytávat výjimky vzniklé v kontrolerech
* Metoda `noSuchElementException(...)` (řádek 15) má anotaci `@ExceptionHandler(NoSuchElementException.class)` (řádek 14) udávající, jaký typ výjimky tato metoda zachytává - v našem případě budeme vyvolávat a zachytávat výjimku typu `NoSuchElementException`. První parametr metody (řádek 16) je pak typu této výjimky, druhý parametr umožňuje přístup na informace daného webového požadavku, při kterém výjimka vznikla.
* V kódu výjimky prostě vrátíme instanci již známé třídy `ResponseEntity<>`.

V třídě `ControllerExceptionHandler` takových metod (pro různé výjimky) můžeme samozřejmě definovat libovolný počet. Mechanismus SpringBoot sám vybírá nejvíce vhodnou metodu, která vzniklou výjimku zachytí.

Následuje upravený kód funkce `getById(...)`, která se velmi zjednodušila:

{% code lineNumbers="true" %}
```java
  @GetMapping
  public Event getById(@RequestParam int eventId) {
    return this.eventService
            .getById(eventId)
            .orElseThrow(() -> new NoSuchElementException(
                    "Event with eventId=" + eventId + " not found."));
  }
```
{% endcode %}

Funkce už nyní vrací zpátky jen datový typ `Event`, a v jejím kódu se pokusíme získat událost podle id (řádek 4), pokud se to nezdaří (řádek 5) vyhodíme novou výjimku typu `NoSuchElementException`se zprávou (řádky 5-6).
