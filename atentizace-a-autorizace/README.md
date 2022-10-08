# Atentizace a autorizace

Při realizaci spojení mezi Backendem a Frontendem je u většiny aplikací nutné udržovat informaci o uživateli. Ne všechny operace na Backendu mohou provádět všichni uživatelé. Webová aplikace proto musí poskytovat mechanismus, na základě kterého umožní:

* zjistit, kdo je daný uživatela - tzv. **autentizace**. Tento proces ověřuje identitu uživatele. Uživatel zadá svá **pověření** (například email+heslo, jméno+otisk prstu, přístupovou kartu apod.) a aplikace na základě toho ověří, zda uživatel je identitou, za kterou se vydává.
* zjistit, zda daná identita má dostatečná práva k požadované operaci - **autorizace**. Tento proces již **neověřuje** uživatele, jen na základě dřívějšího ověření zjišťuje, zda má uživatel práva nebo správnou roli dovolující provést určitou operaci. Pro autorizaci je identita typicky reprezentována nějakým otiskem (dlouhý nečitelný řetězec, ID apod.), kterou autorizační mechanismus umí rozložit a zjistit, co může daný uživatel provádět.

{% hint style="info" %}
Mnemotechnická pomůcka: Au**ten**tizace (ten) určuje "kdo" je daná identita. Au**to**rizace (to) určuje, co může daná identita provádět.
{% endhint %}

U běžných aplikací při autentizaci uživatel zadá uživatelské jméno a heslo a jako odpověd obdrží unikátní klíč, který ho identifikuje. Při dalších operacích (například změna obrázku uživatele) uživatel zároveň s požadavkem posílá tento klíč a mechanismus na jeho základě zjistí, zda má uživatel práva k provedení této operace.

Mechanismů, jak realizovat tyto operace, je několik. Záleží zejména, zda běží Frontend a Backend na stejném serveru, nebo zda je aplikace škálovaná na více Backendů (jak si budou předávat otisk uživatele?), zda je pro autentizaci vyhrazena specifická backendová služba atd.
