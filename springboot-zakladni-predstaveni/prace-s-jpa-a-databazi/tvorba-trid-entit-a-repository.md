# Tvorba tříd entit a repository

Nyní vytvoříme třídy, které budou pracovat s databází:

* entity - jsou třídy, které reprezentují jednotlivé záznamy (zjednodušeně tabulky) získávané z databáze,
* repository - jsou třídy, které pro programátora zapouzdřují a nabízejí mu operace pro práce s entitami pro jejich načítání a ukládání.

V našem kódu vytvoříme jednoduchou entitu `Event` pro práci s událostí a `EventRepository` pro její správu.

## Vytvoření entity Event

Do projektu vložíme novou třídu `Event`. Abychom věci týkající se db modelu udrželi pohromadě, vložíme vše do balíčku `model`.

{% code title="Event.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.LocalDateTime;

@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Entity
public class Event {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private int eventId;
  @Column(nullable = false)
  private String title;
  private LocalDateTime dateTime;
}
```
{% endcode %}

Obdobně vložíme **rozhraní** `EventRepository`:

```
package cz.osu.kip.eventReminder.model;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface EventRepository extends JpaRepository<Event, Integer> {
}

```
