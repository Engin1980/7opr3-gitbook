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

Třída využívá zajímavé anotace:

* v celém projektu budeme hojně využívat projekt Lombok, díky kterému nemusíme psat například konstruktory a get/set metody a nahradit je anotacemi; projekt Lombok pak vygeneruje příslušný kód za nás (viz `@Getter, @Setter, @AllArgsConstructor, @NoArgsConstructor`,
* základním bodem je anotace `@Entity`, podle které JPA pozná, že je daná třída entitní a bude se ukládat/načítat z databáze; neřekneme-li jinak, bude se tabulka v DB jmenovat stejně jako třída,
* ve třídě definujeme požadované soukromé třídní proměnné; JPA poté pro každou proměnnou v tabulce vygeneruje sloupeček s odpovídajícím názvem a datovým typem;
* anotace `@Id` ukazuje, který ze sloupců je primárním klíčem; navíc anotace `@GeneratedValue` říká, že se tento primární klíč bude automaticky generovat v databázi (programátor jej tedy při ukládání instance nebude ukládat),
* anotace `@Column(nullable=false)` říká, že daný sloupec nebude mít hodnotu null; pomocí této anotace lze nastavit i další vlastnosti sloupce, jako použitý datový typ, maximální délku a podobně.

Tím je entita hotova.

## Vytvoření repository EventRepository

Repository jsou třídy poskytující zapouzdřené databázové operace pro práci s entitami. V JPA se nedefinují jako třídy, nýbrž jako rozhraní (interface), které dědí z předpřipraveného rozhraní `JpaRepository`. Tato třída bere dva generické parametry: prvním je datový typ entity, se kterým má repository pracovat, druhým je datový typ primárního klíče entity. JPA pak při spuštění samo vytvoří požadovanou implementaci. Programátor tedy v základu nemusí nic programovat.&#x20;

Samotné JPA nabízí již základní předdefinované metody pro práci s entitami.

<table><thead><tr><th width="268">Metoda</th><th>Popis</th></tr></thead><tbody><tr><td>void save(E entity)</td><td>Uloží entitu do databáze; pokud již záznam s ID existuje, bude přepsán, pokud se jedná o nový záznam do databáze, bude vytvořen.</td></tr><tr><td>Optional&#x3C;E> findById(K id)</td><td>Najde (pokud existuje) entitu podle zadaného ID.</td></tr><tr><td>List&#x3C;E> findAll()</td><td>Vrátí všechny záznamy dané entity  databáze.</td></tr><tr><td>void delete(E entity)</td><td>Vymaže danou entitu z databáze.</td></tr><tr><td>boolean existsById(E entity)</td><td>Vrací příznak, zda entita s daným ID existuje.</td></tr><tr><td>int count()</td><td>Vrátí počet položek entity v DB</td></tr><tr><td>E getReferenceById(K id)</td><td>Vrátí odkaz na danou entitu podle ID. Používá se, pokud nepotřebujeme data se všemi sloupci dané entity, ale pouze jednoduchý odkaz na danou entitu (abychom ji například následně pomocí <code>delete</code> mohli smazat.</td></tr></tbody></table>

Do projektu do balíčkou `model` tedy vložíme **rozhraní** `EventRepository`:

{% code title="EventRepository.java" lineNumbers="true" %}
```java
package cz.osu.kip.eventReminder.model;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface EventRepository extends JpaRepository<Event, Integer> {
}

```
{% endcode %}

Rozhraní je velmi jednoduché. Obsahuje uvedenou anotaci `@Repository`, abychom jej mohli jednoduše využívat dále v Dependency Injection (bude ukázáno dále) a dva generické parametry, `<Event, Integer>` odkazující na typ entity a typ jejího primárního klíče.
