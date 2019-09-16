# Díl 3. - Načítání a vykreslování dat

Když jsme si v předchozím díle zajistili data a jejich zpracování pomocí modelů, můžeme nyní začít s jejich zobrazováním.

## Router

Celý chod aplikace začíná zde. Pro příchozí HTTP požadavek se IMA.js snaží najít shodnou routu. Každá routa má přiřazený Controller a View, který se má v případě shody načíst a vyrenderovat.
Seznam rout se nachází v konfiguračním souboru `app/config/routes.js`. Jednotlivé parametry routy si zde nebudeme popisovat, najdete je v [dokumentaci](https://github.com/seznam/IMA.js-skeleton/wiki/Routing). Doporučujeme si však dokumentaci routeru pročíst.

Pro naší aplikaci potřebujeme upravit pouze výchozí routu s názvem `home`. Abychom mohli vytvářet krásné SEO URL ve tvaru *weather.xyz/ostrava* nebo *weather.xyz/ceske-budejovice* přidáme do **path** parametr `:?location`.
Pokud jste si četli dokumentaci, tak už víte, že `:` se v routě označuje parametr a `?` nepovinný parametr.

```javascript
// app/config/routes.js
 router
    .add('home', '/:?location', HomeController, HomeView)
    .add('filtered', '/:filter', HomeController, HomeView)
    .add(RouteNames.ERROR, '/error', ErrorController, ErrorView)
    .add(RouteNames.NOT_FOUND, '/not-found', NotFoundController, NotFoundView);
```

## Controller

Router předává slovo Controlleru, který byl přiřazený k shodné routě. Controller tímto začíná svůj [životní cyklus](https://github.com/seznam/IMA.js-skeleton/wiki/Controller-lifecycle). Pro nás je důležitá metoda `load()`, ve které načteme všechna potřebná data. Tedy ne všechna, některá necháme, aby se načetla až na straně klienta.

Než ale začneme s načítáním, potřebujeme Controlleru předat **ForecastService** a **GeoCoderService**, které jsme vytvořili v předchozím díle. Uděláme to pomocí DI.

```javascript
export default class HomeController extends AbstractController {

	static get $dependencies() {
		return [Router, CookieStorage, ForecastService, GeoCoderService, '$Settings.App.defaultLocation'];
	}

	constructor(router, cookieStorage, forecastService, geoCoderService, defaultLocation) {
		super();

		this._router = router;
		this._cookieStorage = cookieStorage;
		
		this._forecastService = forecastService;
		this._geoCoderService = geoCoderService;

		this._defaultLocation = defaultLocation;
	}
```

Jak vidíte, nepředali jsme si jen zmiňované Services, ale i něco navíc. Konkrétně:
 - **Router** - budeme ho potřebovat pro přesměrování při zadání špatného názvu města.
 - **CookieStorage** - vytáhnutí informací o městě, které si uživatel nastavil (více si o tom povíme v 5. díle).
 - **Nastavení `defaultLocation`** - jak jsme popisovali v úvodu 2. dílu, pokud nebudeme mít žádný uživatelský vstup, použijeme výchozí město. Řetězec `$Settings` představuje soubor `app/config/settings.js`. Zbytek řetězce je cesta k nastavení.

```javascript
load() {
    let geoCoderPromise = Promise.resolve(this._getDefaultLocation());  // zde začneme s výchozím městem

    const { location, lat, lon } = this.params; // parametry u URL

    if (location) {
        geoCoderPromise = this._geoCoderService.geoCodeMunicipality(this.params.location); // načtení informací o městu podle parametru z URL

    } else if (!location && (typeof lat !== 'undefined' || typeof lon !== 'undefined')) {
        this._router.redirect(this._router.link('home')); // na dotazy se souřadnicemi bez názvy města neodpovídame
    }

    geoCoderPromise.then(geoLocation => {
        if (
            location && (
                typeof lat === 'undefined' ||
                typeof lon === 'undefined' ||
                lat !== geoLocation.lat ||
                lon !== geoLocation.lon
            )
        ) {
            // doplnění nebo oprava souřadnic do URL 
            this._router.redirect(this._router.link('home', { location, lat: geoLocation.lat, lon: geoLocation.lon }));
        }

        return geoLocation;
    });
    
    // Když známe město, můžeme načíst předpověď.
    const forecastPromise = geoCoderPromise.then(location => this._forecastService.getForecast(location.lat, location.lon));

    return {
        location: geoCoderPromise,
        forecast: forecastPromise,

        forecastDetail: null,  // detail předpovědi si necháme k načtení na klienta (ušetříme prostředky a čas na serveru)
        forecastDetailLoading: true
    };
}
```

#### Donačítání dat

Abychom nemuseli stahovat velké množství dat k vygenerování odpovědi ze serveru, můžeme odložit načítání dat na stranu klienta. Na serveru si tak stáhneme jen to nejnutnější k vytvoření smysluplné odpovědi, kterou můžou přečíst i vyhledávače.

K tomuto účelu se používá metoda Controlleru zvaná `activate()`. Zavolá se při oživení aplikace nebo aktivaci příslušné routy na straně klienta. Můžeme v ní tak načítat data pro první zobrazení ale i při každém vstupu na stránku.

```javascript
activate() {
    const { location } = this.getState(); // získání dat ze state

    if (location) {
        this._forecastService.getDetailedForecast(location.lat, location.lon)
            .then(forecastDetail => this.setState({ forecastDetail, forecastDetailLoading: false }));

    } else {
        this.setState({ forecastDetailLoading: false });
    }
}
```

#### Meta informace

V Controlleru využijeme ještě jednu metodu - `setMetaParams()`. Ta nám dovoluje nastavit meta informace do záhlaví stránky. Nastavené informace se potom využívají v [**DocumentView**](https://github.com/seznam/IMA.js-tutorial/tree/master/example-app/app/component/document/DocumentView.jsx) (komponenta, která zajišťujě render hlavního HTML markupu).

```javascript
setMetaParams(loadedResources, metaManager, router, dictionary, settings) {
    const { location, forecast } = loadedResources;

    if (!location || !forecast) {
        return;
    }

    const todayForecast = forecast.daily[0];
    const locationShortTitle = location.title.split(',')[0];

    const title = `Počasí ${locationShortTitle || location.title} - IMA.js Example`;
    const description = `${todayForecast.localDate}: ${todayForecast.summary}`;

    const url = router.getUrl();

    metaManager.setTitle(title);
    metaManager.setMetaName('description', description);

    metaManager.setMetaProperty('og:title', title);
    metaManager.setMetaProperty('og:description', description);
    metaManager.setMetaProperty('og:type', 'website');
    metaManager.setMetaProperty('og:url', url);
}
```

## View

Načtená data jsou předávána jako **props** speciální React komponentě. Tato komponenta se nazývá View. Od klasické komponenty, kterou si budeme popisovat v následujícím bodě, se nijak zásadně neliší.
**HomeView** se bude starat o zobrazení názvu města a předpovědi počasí. Každý den předpovědi bude renderovaný samostatnou komponentou, kterou pojmenujeme **ForecastDay**. Musíme také vyřešit stav, kdy změníme město a data se začnou načítat znovu (na straně klienta). Po určitý čas nebudeme mít dostupná žádná data a musíme vykreslit načítání. Práci si ulehčíme použitím předpřipravených komponent z balíčku IMA-UI-atoms.

1. Nejprve si tedy nainstalujeme `npm install --save ima-ui-atoms`.
2. V **app/build.js** přidáme `'ima-ui-atoms'` do **let vendors.common** a `'./node_modules/ima-ui-atoms/dist/*.less',` do `let less`

```diff
let less = [
  './app/assets/less/app.less',
+ './node_modules/ima-ui-atoms/dist/*.less',
...
let vendors = {
  common: [
    'ima',
+   'ima-ui-atoms',
```

```javascript
import React, { Fragment } from 'react';
import AbstractComponent from 'ima/page/AbstractComponent';
import { Loader } from 'ima-ui-atoms';

import ForecastDay from 'app/component/forecastDay/ForecastDay';
import ForecastDetail from 'app/component/forecastDetail/ForecastDetail';

export default class HomeView extends AbstractComponent {

    render() {
        return (
            <div className="container">
                { this._renderPlaceAndForecast() }
            </div>
        );
    }

    _renderPlaceAndForecast() {
        const { forecast, location } = this.props;
        const { activeDay } = this.state;

        if (!forecast || !location) { 
            return <Loader/>; // zde ještě nemáme načtená data, zobrazíme načítání
        }

        return (
            <Fragment>
                <div className="location">
                    <h1 className="location-title">{ location.title }</h1>
                </div>
                <div className="forecast-days">
                    <ul>
                        { forecast.daily.map((day, index) => (
                            <ForecastDay
                                key = { index }
                                forecast = { day }
                                place = { forecast.place }
                                isActive = { index === activeDay }  // activeDay máme uložený ve state komponenty (view)
                                onClick = { event => this.onDayClick(event, index)}/> // viz. níže
                        ))}
                    </ul>
                </div>
                { this._renderDetailedForecast() }
            </Fragment>
        );
    }

    onDayClick(event, index) {
	event.preventDefault();

	const { forecast } = this.props;

	if (forecast.daily[index] !== undefined) {
		this.setState({ activeDay: index });
	}
    }
}
```

#### Donačtená data

Jak už víme, tak detailní předpověď počasí pro jeden den donačítáme na straně klienta. Metoda `_renderDetailedForecast()` v našem View bude zobrazovat tato data.
Bohužel API, které využíváme vrací detailní předpověď pro všechny dny předpovědi. My si však tato zpracujeme a vykreslíme jen ta, která se týkají zvoleného dne (`activeDay`).

```javascript
_renderDetailedForecast() {
    const { forecastDetail, forecastDetailLoading } = this.props;
    const { activeDay } = this.state; 

    if (forecastDetailLoading) {
        return <Loader/>; // podobně jako v _renderPlaceAndForecast() zobrazíme Loader dokud nemáme data
    }

    return (
        <div className={this.cssClasses('forecast-detail')}>
            { forecastDetail
                .filter(day => day.dayId === activeDay) // vyfiltrujeme si data pro zvolený den
                .map((day, index) => (
                    <ForecastDetail
                        key = { activeDay + index }
                        { ...day }/>ą
                ))
            }
        </div>
    )
}
```

## Komponenty

Stejně tak jako View i komponenty používají React s tím rozdílem, že nerozšiřují `React.Component` ale `ima/page/AbstractComponent`.
**AbstractComponent** poskytuje přístup k utilitám, které jsme si nastavili v `app/config/bind.js` - řádek `oc.constant('$Utils')`.

Tyto utility jsou dostupné pod `this.utils`. Některé, jako například **Router**, **EventBus** nebo **Dictionary** mají speciální metody, které umožňují snadnější použití.
Odkazy můžete vytvářet pomocí `this.link('route', { param: 'value' })`, vytvářet eventy přes `this.fire('eventName', { data })`, překládat texty metodou `this.localize('string', { param })` a definovat CSS třídy pomocí `this.cssClasses({ classes })`.

Utilita `this.cssClasses` slouží k ulehčení definování CSS tříd. Můžete pomocí ní snadněji definovat třídy v závislosti na podmínce nebo nějáké hodnotě. Např.:

```javascript
this.cssClasses({
    'forecast-day': true,
    'forecast-day--active': this.props.isActive
})
```

Argumentem může být i string (např. `this.cssClasses('forecast-day')`) a nemusíte tak definovat objekt tříd.

#### ForecastDay a ForecastDetail

Aby jsme zlepšili čitelnost **HomeView** oddělili jsme opakované části kódu do komponent [**ForecastDay**](https://github.com/seznam/IMA.js-tutorial/tree/master/example-app/app/component/forecastDay/ForecastDay.jsx) a [**ForecastDetail**](https://github.com/seznam/IMA.js-tutorial/tree/master/example-app/app/component/forecastDetail/ForecastDetail.jsx). Na těchto komponentách je zvláštní pouze to, že místo **AbstractComponent** rozšiřují `ima/page/AbstractPureComponent`. Jak již název napovídá, jedná se o React Pure Componenty.

# Závěr

V tomto díle jsme v naší example aplikaci zajistili načítání a zobrazování dat pomocí Controlleru, View a komponent.

V dalším díle se podíváme na jeden detailní příklad komponenty, na které si ukážeme práci s eventy, HOC a Controller Extensions.



