# Vytváření API Endpointů

Vytvoření pomocných tříd

Prvním krokem bude vytvoření jednoduché třídy `Event`, která bude uchovávat data o uložených událostech. Třídu umístíme do balíčku `model` pod naším projektem. Bude obsahovat položky pro id, název a datum události.

{% code title="Event.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.model;

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

