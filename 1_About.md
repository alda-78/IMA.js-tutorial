---------------------------------------------------------
TODO:

- prelozit HelloWorld tutorial a upravit ho pripadne pro aktualni verzi
---------------------------------------------------------

# Co je IMA.js?

"Isomorphic Application in JavaScript", zkráceně tedy [IMA.js](https://imajs.io/), je framework pro vývoj plně izomorfních aplikací. Je vyvíjen pod hlavičkou [Seznam.cz, a.s.](https://www.seznam.cz/) jako open-source projekt pod licencí MIT. Repozitář projektu můžete nalézt na [GitHubu](https://github.com/seznam/IMA.js-core). Pod skupinou [seznam](https://github.com/seznam) najdete další open-source projekty, především nás budou v dalších dílech zajímat projekty související s IMA.js, třeba UI atomy a další užitečné pluginy.

V Seznamu.cz jej používáme pro naše vlastní služby, některé z nich možná navštěvujete každý den, mimojiné [Proženy](https://www.prozeny.cz/), [Garáž.cz](https://www.garaz.cz/), [HRY.cz](https://www.hry.cz/) nebo třeba [Seznamzpravy.cz](https://www.seznamzpravy.cz/) a další budou přibývat. I z toho důvodu má IMA.js integrovány spousty unit testů a do jejího vývoje neustále investujeme.

# Izomorfní aplikace?

Izomorfní aplikace kombinuje výhody single-page (SPA) a multi-page aplikací (MPA). To konkrétně znamená, že jsme schopni spouštět stejný kód na serveru i na klientu. Aby byl takový přístup možný, musíme poskytnout na obou stranách abstraktní API, které na dané části chybí. Například na serveru nebude dostupný Window objekt prohlížeče.

Takový přístup má spoustu výhod. Například je možné se podle zátěže serveru rozhodnout, zda budeme obsah do DOMu vykreslovat už na serveru nebo půjdeme cestou SPA (bez server-side renderingu). Pro roboty vyhledávačů nebo starší prohlížeče můžeme servírovat MPA verzi apod.

Sdílené jsou i další komponenty, třeba [Router](https://imajs.io/doc/router/router-abstract-router.html) nebo [SEO](https://imajs.io/doc/meta/meta-meta-manager-impl.html) manažer. IMA.js obsahuje také injektor závislostí, centralizovanou konfiguraci, REST API [localhost proxy](https://github.com/seznam/IMA.js-skeleton/blob/master/server/server.js#L116) a cache. Konkrétní komponenty včetně příkladů použití si popíšeme v dalších dílech.

# Technologie a platformy

Ve výčtu použitých technologií nepřekvapí Node.js, express.js a React. K testování používáme enzyme a Jest, CSS styly píšeme v Less, k balení zase Babel a k dodržování code-style ESLint. Práci nám zjednodušuje Gulp a spousta procesů je dostupná jako [plugin](https://github.com/seznam/IMA.js-gulp-tasks). Používáme balíčkovací systém NPM, kde publikujeme všechny naše balíčky, které budeme v průběhu seriálu potřebovat. Veškeré závislosti můžete nalézt v souboru 'package.json' v repozitáři projektu.

Izomorfismus frameworku se odráží i v podporovaných platformách. Podporujeme všechny hlavní prohlížeče, u starších a nepodporovaných prohlížečů vydáváme MPA verzi webu a podporujeme je tedy také, byť funkčnost může být omezená. Apache Cordova a Adobe PhoneGap podporujeme v SPA režimu.

# HelloWorld

Budeme potřebovat poslední LTS verzi Node.js a NPM.

Vyklonujeme si skeleton projektu, abychom nemuseli vše zakládat na zelené louce.
```bash
git clone https://github.com/seznam/IMA.js-skeleton.git
```
Přepeneme se do nově vytvořeného repozitáře.
```bash
cd IMA.js-skeleton
```
Doporučujeme nainstalovat si nástroj Gulp jako globální balíček.
```bash
npm install --global gulp
```
Nainstalujeme si lokální závisalosti.
```bash
npm install
```
Nainstalujeme HelloWorld demo, které nám vytvoří i základní adresářovou strukturu pro další práci.
```bash
npm run app:hello
```
Poslední příkaz spustí vývojový server, který přeloží naši aplikaci.
```bash
npm run dev
```
Teď už se můžeme podívat na náš HelloWorld v prohlížeči na portu 3001 - [http://localhost:3001/](http://localhost:3001/). Vývojový server běží na pozadí a hlídá změny v souborech. Pokud zaznamená modifikaci nějakého souboru, přeloží znovu aplikaci.
