# Díl 2. - Setup aplikace a vytvoření modelů

V tomto díle začneme psát naší ukázkovou aplikaci. Povedeme Vás od úplného začátku až po finální deployment do produkce. V průbehu si ukážeme využití všech hlavních vlastností IMA.js a jak vytvořit často používané prvky webových aplikací.

Úkázková aplikace bude zobrazovat počasí v místě, které:
- uživatel vyhledá pomocí našeho vyhladávání
- bude zadáno v URL parametru (SEO optimalizace)
- uživatel nastaví a uloží ho do cookies.
- je nastaveno v aplikaci jako výchozí, pokud se nepoužije žádná z výše uvedených možností.

Pro jednoduchost pojmenujeme aplikaci **WeatherApp**.

## Setup

Slibili jsme vám, že to vezmeme od začátku, ale také to nebudeme prodlužovat. Na instalaci taky není níc zajímavého :smile:

K vývoji budete potřebovat:
- **Node.js** ve verzi **10** a výše
- **NPM** ve verzi **6** a výše.

```console
cd <-- Váš adresář kde máte projekty -->

git clone https://github.com/seznam/IMA.js-skeleton.git weather-app

cd weather-app

git remote set-url origin <-- Váš remote Git repozitář -->

git remote -v # jen pro kontrolu

npm install --global gulp

npm install
```

Tímto máme vytvořený skeleton aplikace a nainstalované závislosti. Skeleton doplníme o demo soubory, které nám pomohou odstartovat vývoj aplikace.
IMA.js nabízí 3 demo *"aplikace"*, ze kterých si můžete vybrat. Jejich ukázky najdete na [imajs.io/examples](https://imajs.io/examples).

Pro naši aplikaci postačí základní [Hello World](https://imajs.io/examples/hello) - `npm run app:hello`. Můžete však použít i [TODO list](https://imajs.io/examples/todos) - `npm run app:todos` nebo [Twitter-like feed](https://imajs.io/examples/feed) - `npm run app:feed`.

## Adresářová struktura

V adresáři naší aplikace vzniklo několik pod-adresářů: `app`, `build`, `node_modules` a `server`. My se budeme zabývat pouze adresářem `app` (ostatní jsou vysvětleny svým názvem).

- `assets` - obsahuje soubory, které jsou zpracovány pre-processory a zkopírovány do `build` adresáře.
  - `less` - LESS soubory definující obecná pravidla, makra, mixins a základní strukturu UI.
  - `static` - jakékoliv soubory, které nepotřebují pre-processing (JS soubory 3. stran, obrázky, ...)

- `component` - naše React komponenty, které budeme používat ve views. Více o komponentách si řekneme v 3. díle tohoto seriálu.
- `config` - konfigurační soubory aplikace. Nyní se konfigurací nebudeme zabývat a ukážeme si jednotlivé možnosti až je budeme potřebovat v průběhu vytváření aplikace.
- `page` - controllery, views a LESS soubory k views.
  - `error` - stránka, která se zobrazí pokud se vyskytne chyba v průběhu aplikace.
  - `home` - hlavní stránka aplikace
  - `notFound` - stránka, která se zobrazí pokud uživatel zadá URL na neexistující routu.

Adresáře `assets` a `config` jsou povinné a měly by se v aplikaci vždy nacházet. Ostatní jsou volitelné a můžete si je přejmenovat. Je však potřeba upravit některé konfigurace a přihlížet k tomu v následném vývoji aplikace.

## Modely

Modely nám v IMA.js aplikacích umožǔjí pracovat s daty. Pomocí modelů se data načítají, odesílají a tranformují. V našem případě budeme využívat základní sestavu modelů (`Service`, `Entity`, `Resource` a `Factory`).
- `Entity` představuje objekt držící data nějákého celku (uživatele, článku, ...).
- `Factory` vytváří ze surových dat instance `Entity`.
- `Resource` stahuje data ze serveru pomocí HTTP požadavků.
- `Service` se stará o načítání dat z `Resource` a následné vytvoření entit pomocí `Factory`.

Může se vám zdát, že `Factory` a `Resource` jsou zbytečné. Data můžeme přece stahovat už v `Service` a tam také vytváře instance `Entity`. Ano to jistě můžeme, ale jen v jednodušších případech. Jak se bude aplikace rozrůstat a začnou se objevovat zanořené závislosti, věřte, že samostatné `Factory` a `Resource` příjdou vhod. 

Pokud by aplikace využívala REST API, můžeme `Factory` a `Resource` uplně nahradit použitím [**ima-plugin-rest-client**](https://www.npmjs.com/package/ima-plugin-rest-client). Více o tomto pluginu se dočtete v jeho README.

Na začátku jsme si stanovili co naše aplikace bude umět. Podle toho si teď rozvrhneme datovou strukturu a vytvoříme modely.

#### 1. Proxy

Data o počasí nám poskytne služba [Počasí.cz](https://pocasi.seznam.cz/) přes jejich API rozhraní `https://wapi.pocasi.seznam.cz`. My ale na toto API rozhraní budeme přistupovat přes naši proxy. Vyhneme se tak problémům s [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

Proxy vytvoříme na stejném serveru, který se stará o výdej webové aplikace. Ve výchozím stavu je server připravený pouze pro jednu proxy. My jich však budeme potřebovat více.
V souboru `weather-app/server/server.js` - funkce `runNodeApp` - najdeme řádek:

```javascript
.use(environment.$Proxy.path + '/', proxy(environment.$Proxy.server, environment.$Proxy.options)
```

Vidíme, že server používá [**Express** framework](https://expressjs.com/) a pro proxy jeden z jeho doplňků [**express-http-proxy**](https://www.npmjs.com/package/express-http-proxy). Nastavení proxy se nachází v souboru `weather-app/app/environment.js`. Soubor obsahuje jednoduchý Node.js modul, který vrací objekt. Hlavními klíči objektu jsou `prod`, `test` a `dev`. Tyto klíče představují nastavení pro jednotlivé vývojové prostředí s tím, že všechny vycházejí z `prod` prostředí. 

> **Poznámka:** Tento styl konfigurace (prod - test - dev) si zapamatujte, neobjevuje se naposledy.

Naší proxy potřebujeme nastavit pouze v produkčním nastavení.  

```javascript
// app/environment.js

module.exports = (() => {
    return {
        prod: {
            // ...
            $Proxy: {
                path: '/api',   // na této URL bude proxy poslouchat
                server: 'https://wapi.pocasi.seznam.cz', // zde bude proxy přeposílat požadavky
                options: {
                    proxyReqPathResolver: function (req) {
                        const queryString = req.url.split('?')[1];

                        return '/v2/forecast' + (queryString ? '?' + queryString : '');
                    }
                }
            }
        }
        // ...
    }
```

#### 2. Načítání dat o předpovědi počasí

V předchozím bodě jsme si nastavili proxy, která poslouchá na URL `/api`. Aby jsme URL neopakovali v aplikaci několikrát přidáme ji do nastavení v souboru `weather-app/app/config/settings.js`. Zde se používá stejný styl konfigurace jako v `environment.js`.

```javascript
// app/config/settings.js

export default (ns, oc, config) => {
  // ...
  return {
    prod: {
        // ...
        App: {
            api: config.$Protocol + '//' + config.$Host + '/api'
        }
    }
  }
}
```

V adresáři `weather-app/app/model` vytvoříme podadresář pro předpověď počasí - `forecast`. V tomto adresáři vytvoříme soubory `ForecastService.js`, `ForecastResource.js`, `ForecastFactory.js` a `ForecastEntity.js`.

Začneme vytvořením **ForecastResource**. V aplikaci IMA.js funguje **DI (Dependency Injection)** zkrz **OC (Object Container)**. Více o **OC** a **DI** se dočtete v [dokumentaci](https://github.com/seznam/IMA.js-skeleton/wiki/Object-Container#1-dependency-injection)

```javascript
// app/model/forecast/ForecastResource.js
import HttpAgent from 'ima/http/HttpAgent';

export default class ForecastResource {
	static get $dependencies() {
		return [HttpAgent, '$Settings.App.api']; // $Settings umožňuje vkládat nastavení pomocí DI
	}

	constructor(http, apiUrl) {
		this._http = http;
		this._api = apiUrl;
    }
    
    async getForecast(lat, lon) {
		const response = await this._http.get(
            this._api,
            { lat, lon, include: ['place', 'daily'] } // query parametry
        );
		
		return response.body;
	}
}
```

Následuje **ForecastFactory**. Prozatím bude obsahovat jen jednu metodu, která z předaných dat vytvoří instanci **ForecastEntity**.

```javascript
// app/model/forecast/ForecastFactory.js
import ForecastEntity from './ForecastEntity';

export default class ForecastFactory {

    static get $dependencies() {
        return [];
    }

    createEntity(data) {
        return new ForecastEntity(data);
    }
}
```

**ForecastEntity** pouze vezme data z constructoru a vytvoří si z nich vlastní properties.

```javascript
// app/model/forecast/ForecastEntity.js
export default class ForecastEntity {
    
    constructor(data) {
        this.place = data.place;

        this.daily = data.daily;

        Object.freeze(this);
    }
}  
```

**ForecastService** bude obsahovat prozatím pouze jednu metodu `getForecast`. Ta pomocí **ForecastResource** načte data ze serveru a poté vytvoří instance **ForecastEntity** pomocí **ForecastFactory**.

```javascript
// app/model/forecast/ForecastService.js
import ForecastResource from './ForecastResource';
import ForecastFactory from './ForecastFactory';

export default class ForecastService {
    static get $dependencies() {
        return [ForecastResource, ForecastFactory];
    }

    constructor(resource, factory) {
        this._resource = resource;
        this._factory = factory;
    }

    async getForecast(lat, lon) {
        const result = await this._resource.getForecast(lat, lon);

        return this._factory.createEntity(result);
    }
```

Vytvořili jsme také model pro **geocoder** - vyhledávání místa podle názvu z URL. Nebudeme zde ale rozepisovat kompletní postup, protože se v mnohém shoduje s modelem pro předpověď počasí. Kompletní podobu **geocoder** modelu najdete v kódu ukázkové aplikace.


 
