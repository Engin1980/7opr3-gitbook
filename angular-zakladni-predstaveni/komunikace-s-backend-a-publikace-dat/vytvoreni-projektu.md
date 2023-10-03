# Vytvoření projektu

Prvním krokem je vytvoření nového Angular projektu. Projekt byl vytvářen Node.JS verzí 18.18.0,  `npm` verzí 10.1.0 a Angular (`ng`) verzí 16.2.4.

{% hint style="info" %}
Před vytvářením projektu doporučujeme provést aktualizaci na nejnovější verzi. Odlišné/nekompatibilní verzie npm/ng mohou při následném generování a spouštění projektu způsobit problémy.
{% endhint %}

Z příkazové řádky/shellu vytvoříme nový projekt `event-reminder-front`, do projektu přidáme routování a CSS:

```powershell
ng new event-reminder-front
```

Do projektu následně vstoupíme a přidáme knihovnu `rxjs`:

```powershell
cd event-reminder-front
npm install rxjs
```

### Příprava souborové struktury a Bootstrapu

V projektu - složce `src/app` vytvoříme následující adresářovou strukturu:

* components - pro vytvářené komponenty
* model - pro DTO objekty
* pipes - pro formátovací třídy pipe
* services - pro vytvářené služby

Dále do souboru `index.html` zavedeme Boostrap 5 přes CDN (řádky 9-16):

{% code title="index.html" lineNumbers="true" %}
```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>EventReminderFront</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
  <link rel="stylesheet" 
        href="https://cdn.jsdelivr.net/npm/bootstrap@5.2.3/dist/css/bootstrap.min.css"
        integrity="sha384-rbsA2VBKQhggwzxH7pPCaAqO46MgnOM80zW1RWuH61DGLwZJEdK2Kadq2F9CUG65" 
        crossorigin="anonymous">
  <link rel="stylesheet" 
        href="https://cdn.jsdelivr.net/npm/bootstrap@5.2.3/dist/css/bootstrap.min.css"
        integrity="sha384-rbsA2VBKQhggwzxH7pPCaAqO46MgnOM80zW1RWuH61DGLwZJEdK2Kadq2F9CUG65" 
        crossorigin="anonymous">
</head>
<body>
<div class="container">
  <app-root></app-root>
</div>
</body>
</html>

```
{% endcode %}

