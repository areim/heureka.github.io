---
layout: post
title: "Ikony bez kompromisů"
permalink: /ikony-bez-kompromisu/
date: 2016-11-15 13:00:00 +0200
author: Adam Havel
tags: front-end web development
category: blog
---


I přes svou malou velikost představují ikony na webu zajímavý problém. Jeden přístup střídá další — každý o trochu lepší než ten předchozí, každý nějakým způsobem nedokonalý. Poslední systém, kterému se delší dobu daří držet na vrcholu, je takzvaný SVG sprite, který vznikne tak, že jednotlivé ikony vložíme do elementů `symbol` v rámci jednoho SVG souboru. Důvodem pro použití symbolů je to, že se nevykreslí v místě definice, ale až tehdy, kdy je skutečně použijeme. Pokud takový sprite nechceme vytvářet ručně, lze použít některé z mnoha dostupných řešení, jako je [grunt-svgstore][grunt-svgstore] nebo [gulp-svg-sprite][gulp-svg-sprite].

{% highlight html %}
<svg>
    <symbol viewBox="0 0 32 32" id="close">
        <path d="..." />
    </symbol>
    <symbol viewBox="0 0 32 32" id="search">
        <path d="..." />
    </symbol>
    ...
</svg>
{% endhighlight %}

Pro vykreslení konkrétní ikonky stačí přímo do šablony napsat jeden řádek SVG kódu obsahující element `use`. Pomocí jeho atributu `xlink:href` pak odkážeme na `id` dané ikony. To je ta jednodušší část.

{% highlight html %}
<svg><use xlink:href="sprite.svg#search"></use></svg>
{% endhighlight %}

## Nahrání spritu

Ve skutečnosti je to však o trochu složitější, protože žádná verze Internet Exploreru (na rozdíl od Edge) nedokáže nahrát externí soubor přes `xlink:href`. Řešení jsou dvě: buď sprite vložíme přímo do všech šablon, nebo použijeme AJAX. První případ lze sice řešit automaticky na straně serveru, ale to nás nezbaví problému cachování dvou nezávislých souborů jako jednoho. Druhá možnost spočívá v použití malého skriptu, který dáme do hlavičky v šabloně; krom ikon ho lze využít i na ostatní nekritické soubory, třeba fonty.

{% highlight javascript %}
(function(w) {
    'use strict';

    function getType(src) {
        return src.match(/\.([^\.\?]+)(?:\?.*)?$/)[1];
    }

    function injectContent(content, type) {
        if (type === 'css') {
            var style = document.createElement('style');
            document.head.appendChild(style);
            style.textContent = content;
        } else {
            document.head.insertAdjacentHTML('beforeend', content);
        }
    }

    w.loadFile = function(src) {
        if (!w.localStorage || !w.addEventListener) {
            return false;
        }

        var content = localStorage[src];

        if (content) {
            injectContent(content, getType(src));
        } else {
            var xhr = new XMLHttpRequest();

            xhr.addEventListener('load', function() {
                try {
                    localStorage[src] = xhr.responseText;
                } catch (err) {
                    // localStorage not allowed or quota exceeded
                } finally {
                    injectContent(xhr.responseText, getType(src));
                }
            });

            xhr.open('GET', src);
            xhr.send();
        }
    };

})(window);
{% endhighlight %}

Funkce po zavolání asynchronně načte zadaný soubor. Jeho obsah pak zároveň uloží do `localStorage`, a vloží do šablony — styly v podobě `style` elementu, SVG soubory přímo do hlavičky. V druhém případě by jistě bylo lepší cílit na tělo dokumentu, ale to v momentu zavolání funkce ještě neexistuje. SVG sice není standardní součástí `head`, ale podstatné je, že předchozí metoda funguje.

Výhoda skriptu spočívá v tom, že jím načtené soubory neblokují vykreslení dokumentu. Má to ovšem jeden háček: ikony se nenačtou, pokud je JavaScript vypnutý, a bohužel nevím o žádném řešení (jako je `noscript` v případě stylů), které nevyžaduje, aby byl sprite dopředu součástí dokumentu. Závěrem je, že na ikony není spoleh a vždy by u nich měl být textový popisek.

Pokud se nic nepokazí, ikony se vykreslí v podstatě okamžitě — samozřejmě s ohledem na rychlost sítě. Při druhém načtení dokumentu by prodleva měla být ještě kratší, jelikož se ikony nahrávají přímo z `localStorage`. To sice pro čtení a zápis vyžaduje synchronní — tedy blokující — operaci, ale přesto by mělo být dostatečně rychlé a zároveň spolehlivější než klasická cache v prohlížeči.

## SVG a DOM

Stinné stránky máme za sebou a konečně můžeme přejít k výhodám spritu. V momentě, kdy SVG vložíme přímo do dokumentu, se stane součastí DOMu — včetně vnitřní struktury. To znamená, že můžeme upravovat všechny jeho součásti, například měnit barvu podle kontextu pomocí CSS proměnné `currentColor`. Ale to umí i ikony zabalené do podoby fontu. Podstatné je, že v případě SVG lze měnit jakoukoliv vlastnost, nejen barvu.

Existuje několik způsobů, jak ikony nastylovat. První z nich předpokládá použití atributů — jako třeba `fill="currentColor"` — přímo na samotné ikoně (nebo jejích částech). Jelikož se však většina ikon bude chovat podobně, vyplatí se jejich vzhled nastavit globálně. Možností je CSS vložit přímo do spritu a tím zacílit všechny symboly. Ale s ohledem na to, jak se sprite vytváří, může jít o zbytečnou komplikaci. Schůdnější je vydat se cestou globálních stylů a vlastnosti jako `fill` nebo `stroke` nastavit přímo v nich. Nesmíme ale zapomenout na dva důležité faktory: kaskádu a specificitu.

Předpokládáme-li ve všech případech stejný selektor, pak CSS nastavené přes atribut `style` přímo na původních elementech `symbol` přebíjí vše ostatní. Následují styly vložené navrch samotného spritu a hned za nimi atributy typu `fill` nebo `stroke` (znovu použité na úrovni samotných ikon). Jako poslední se bere ohled na globální styly ve vnějším dokumentu.

Zmíněná kaskáda pravidel ukazuje, že můžeme zvolit řadu způsobů, jak ovlivnit vzhled ikon. Podle mě je nejlepší držet se globálních stylů a při tvorbě spritu z ikon odstranit veškeré atributy. Pokud zjistíme, že potřebujeme ikonku, která by neměla měnit barvu v závislosti na kontextu (např. loga sociálních sítí), můžeme využít atribut `style` a ten neodstraňovat. Další trik spoléhá na proměnnou `currentColor`: část ikony obarvíme pomocí vlastnosti `fill` s konkrétní hodnotou, jinou pak s hodnotou nastavenou právě na `currentColor`. Tím získáme způsob, jak dynamicky ovlivnit dvě různé části ikony změnou barvy buď ve `fill` nebo v `color`.

## Atributy

Pro správné vykreslení je třeba, aby každý symbol obsahoval atribut `viewbox`. Krom toho ovšem není od věci nastavit i základní rozměry na úrovni SVG elementu v šabloně. Pokud SVG žádné nemá — ať už ze strany atributů `width` a `height` nebo v rámci CSS — zobrazí se o velikosti 300 na 150 pixelů. To se může stát relativně snadno: když se styly nepodaří nahrát, anebo předtím, než se projeví (v případě, kdy se načítají asynchronním způsobem).

{% highlight html %}
<svg width="32" height="32" class="e-icon">
    <use xlink:href="#search"></use>
</svg>
{% endhighlight %}

Dalším zajímavým atributem je `role`. Pokud chcete, aby čtečky obsahu považovaly ikonku za obrázek, nastavte jej na `image`. Ale vzhledem k tomu, že by se ikony vždy měly vyskytovat v páru s textovým popiskem, je lepší skrýt je před zraky čteček úplně. Toho snadno dosáhneme pomocí atributu `aria-hidden` s hodnotou `true`. Pokud chcete za každou cenu použít ikonku bez popisku, popište její význam alespoň pomocí atributu `aria-label`.

{% highlight html %}
<button>
    <svg aria-hidden="true">
        <use xlink:href="#search"></use>
    </svg>
    <span>Search</span>
</button>
{% endhighlight %}

{% highlight html %}
<button>
    <svg role="image" aria-label="Search">
        <use xlink:href="#search"></use>
    </svg>
</button>
{% endhighlight %}

## Co dělat bez SVG?

V případě, že vyžadujete, aby se ikonky zobrazovaly i v prohlížečích, které nepodporují SVG, je nutné provést několik dalších kroků. Zaprvé je třeba vytvořit PNG verze všech ikonek. Druhým krokem je ověřit podporu SVG — toho lze docílit buď pomocí knihovny [Modernizr][modernizr] nebo dotazem na objekt `document.implementation`. Další postup spočívá v jednoduchém nahrazení SVG elementů klasickým `img` s odkazem na PNG verzi dané ikonky.

Pro začátek je třeba získat `id` ikonky, abychom mohli vytvořit URL rastrové verze. Jelikož však prohlížeč nepodporuje SVG, nerozumí ani jeho struktuře, což znamená, že neumí přímo přečíst atribut `xlink:href`, kde se `id` nachází. Řešením je použít regulární výraz, který spustíme nad rodičem ikonky, respektive jeho `innerHTML`. Další problém se týká Internet Exploreru 8, který náhradní ikoně z nějakého důvodu přisuzuje nulové rozměry. Pomůžeme mu tím, že vedle ikony vložíme další prvek (třeba `div`) o stejné velikosti.


{% highlight javascript %}
if (!document.implementation.hasFeature('http://www.w3.org/TR/SVG11/feature#Image', '1.1')) {
    [].forEach.call(document.querySelectorAll('svg.e-icon'), function(icon) {
        var fallbackIcon = document.createElement('img');
        var placeholder = document.createElement('div');
        var parent = icon.parentNode;
        var type = parent.innerHTML.match(/xlink:href=["']?#([^"'>\s]+)["']?/i)[1];

        fallbackIcon.classList.add('icon');
        placeholder.classList.add('j-icon-placeholder');
        fallbackIcon.src = 'img/' + type + '.png';

        parent.insertBefore(placeholder, icon.nextSibling);
        parent.replaceChild(fallbackIcon, icon);
    });
}
{% endhighlight %}

{% highlight scss %}
.e-icon,
.j-icon-placeholder {
    width: 1em;
    height: 1em;
    display: inline-block;
    vertical-align: middle;
    position: relative;
    top: -.075em;
}

.e-icon {
    fill: currentColor;
}
{% endhighlight %}

## Sprite a CSS?

Poslední příklad se týká situace, kdy chceme ikonku použít v seznamu položek místo klasické odrážky. Zároveň předpokládáme, že je odrážka graficky natolik složitá, že se nedá vytvořit jen pomocí CSS nebo Unicode symbolů. Možným řešením je vložit ikonku (pomocí elementu `use`) do každé položky zvlášť. Pokud se však chceme vyvarovat zbytečného opakování, nezbyde nám, než se obrátit na CSS. Ale i kdybychom SVG vložili přímo do globálních stylů (přes `data-uri`), ikonky svou barvu — v závislosti na tom, kde se seznam vyskytuje — nezmění. Pro tyto případy by se daly použít ikony zabalené do podoby fontu, stačila by pak jednoduchá změna barvy textu. To by ovšem vyžadovalo dva nezávislé systémy.

Ve snaze vyřešit tento problém jsem vyzkoušel přístup, který využívá (nebo spíše zneužívá) CSS filtry, konkrétně `drop-shadow`. Narozdíl od vlastnosti `box-shadow`, která vytváří obdélníkové stíny (včetně zakulacených rohů a případných transformací), bere `drop-shadow` v potaz přesný tvar prvku. Pokud skryjeme odrážku, která stín vytváří, stačí správně nastavit barvu samotného stínu. To nám umožňuje na základě jedné ikony vytvořit různě barevné verze.

Nechat odrážku zmizet ovšem není tak jednoduché, jak se zdá. Pokud ji skryjeme pomocí průhlednosti nebo vlastnosti `background-position`, zmizí spolu s ní i stín. Zbývá nám ještě jeden způsob: posunout ikonu pomocí `position` a nastavit rodičovskému prvku `overflow` na `hidden`. Relevantní posun v `drop-shadow` filtru nastavíme na zápornou hodnotu posunu samotné odrážky a výsledkem je, že zůstane vidět právě jen stín. Heuréka!

O elegantní řešení však nejde ani zdaleka. Zaprvé, podpora CSS filtrů chybí ve všech verzích Internet Exploreru. Zadruhé, používat filtry v podobném rozsahu může (ale nemusí) zpomalit vykreslení stránky. A to nejhorší na konec: ač funkční v předchozích verzích, v posledním vydání Chrome nefunguje ani metoda s posunem odrážky mimo rodičovský element (stín se jednoduše nezobrazí).

{% highlight scss %}
.c-bullet-list {

    > li {
        $size: 1.5em;

        position: relative;
        padding-left: $size + .5em;
        overflow: hidden;

        &:before {
            width: $size;
            height: $size;
            content: '';
            position: absolute;
            top: 0;
            left: -$size;
            background: url('img/icon/arrow.svg') center no-repeat;
            background-size: auto 100%;
            color: cyan;
            filter: drop-shadow($size 0 0 currentColor);
        }

        .no-cssfilters & {
            overflow: visible;

            &:before {
                left: 0;
            }
        }
    }
}
{% endhighlight %}

Takže nám zbývá jen jedno rozumné řešení, a to sice vkládání ikon přímo do stylů v podobě data URI. Tato technika nám ušetří síťový požadavek a umožní upravovat vlastnosti ikon přímo v CSS. V případě, že použijeme preprocesor jako LESS nebo Sass, a máme v plánu ikonku použít na více než jednom místě, můžeme navíc vytvořit funkci (nebo mixin), která poskytne způsob jak ji obarvit pomocí parametrů. To nám sice nepomůže s repeticí ve výsledném CSS, ale zachová to alespoň jediný zdroj pro případnou úpravu ikonky. Z principu techniky dále vyplývá, že by ikonka neměla být příliš komplexní. Na druhou stranu, čím více ji pak v CSS použijeme, tím menší podíl bude mít na velikosti výsledného komprimovaného souboru.

{% highlight scss %}
@function bullet($color) {
    @return url('data:image/svg+xml;charset=utf-8,%3Csvg%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20viewBox%3D%220%200%2010.7%2016%22%3E%3Cpath%20fill%3D%22#{$color}%22%20d%3D%22M.7%202.7L6.4%208%20.6%2013.3%202.4%2015%2010%208%202.4%201%20.7%202.7z%22%2F%3E%3C%2Fsvg%3E');
}
{% endhighlight %}

[grunt-svgstore]: https://github.com/FWeinb/grunt-svgstore
[gulp-svg-sprite]: https://github.com/jkphl/gulp-svg-sprite
[modernizr]: https://modernizr.com
