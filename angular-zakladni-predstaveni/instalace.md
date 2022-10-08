---
description: Tato stránka popisuje postup instalace prostředí Angular.
---

# Instalace

## Instalace NPM

NPM - Node Package Manager - je nástroj pro správu, instalaci balíčků. Byl vytvořen jako součást Node.JS jako jeho balíčkovací systém pro správu a instalaci balíčků souvisejících s vytvářeným projektem. Součástí NPM je CLI - command line interface, tj. možnost jednoduchého vykonávání příkazů přes příkazovou řádku/konzoli. NPM později využijeme k instalaci prostředí Angular.

{% embed url="https://nodejs.org/en/download/" %}

NPM je ke stažeí na adrese [https://nodejs.org/en/download/](https://nodejs.org/en/download/). Na stránce vybereme požadovaný operační systém a architekturu. Vybíráme také ze dvou verzí:

* LTS - Long Term Support - je stabilní verze s garantovanou podporou
* Current - je verze s nejnovějšími vlastnostmi, ale negaratnovanou podporou v konkrétní verzi.

Stáhneme požadovaný soubor a nainstalujeme jej běžným způsobem pro daný operační systém.

## Instalace Angular

Pro instalaci Angular využijeme příkazový řádek, na kterém spustíme NPM.

```
npm install -g @angular/clint
```

{% hint style="info" %}
Na počítačích Windows s využitím PowerShell mohou být omezena práva na spouštění externích skriptů. Proto v prostředí PowerShell můžeme/musíme posílit práva spouštění skriptů příkazem:\
\`Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned\`.

Tento příkaz ale nastavuje práva na spouštění v rámci celého uživatelského účtu. Je proto vhodné pro instalaci Angular práva zpět povýšit.
{% endhint %}
