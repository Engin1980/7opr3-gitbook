# Práce s JPA a databází

Cílem této části je rozšířit vytvářený úvodní projekt o práci s databází. Rozšíření bude založeno na JPA a Apache Derby.

Řešení bude z pohledu zdrojových kódů rozděleno do dvou větví.&#x20;

V první větvi je základní uvození entity, připojení DB a vytvoření služby a kontroleru.

{% embed url="https://github.com/Engin1980/7opr3-springboot-introduction/tree/Persistence" %}
Implementace Event,  základní připojení k DB a vracení+úprava dat přes REST API
{% endembed %}

V druhé větvi bude přidání vztahu mezi dvěma entitami, intenzivnější využití DTO a odpovídající úprava služeb a kontroleru.

{% embed url="https://github.com/Engin1980/7opr3-springboot-introduction/tree/PersistenceWithRelationAndJTO" %}
Implementace Event+Notes, relace mezi entitami, DTO
{% endembed %}
