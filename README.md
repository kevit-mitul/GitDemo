# sbg-mono

Mono repo voor het SBG project: organisatie en participatie van sportieve activiteiten

Indeling:

- backend: De backend die toegankelijk is voor zowel backoffice als de app
- backoffice-web: Website voor het organiseren en managen van evenementen
- frontend-client-web: Publieke website voor deelnemers. Met uizondering van de registratie pagina, wordt hier ook alle content voor iFrames gehost.
- hydra-client: Gegenereerde code voor Hydra 1.11.10. \_De SDK voor deze versie was nog niet gereleased. Wanneer dit wel het geval is, kan dit project weer verwijderd worden en de package aan de csproj van Identity worden toegevoegd.
- identity-web: Frontend voor de inlog flows bijbehorend bij onze Identity Provider: [Ory Kratos](https://www.ory.sh/kratos/docs/)
- mobile-app: Android/iOS app hoofzakelijk voor het inschrijven voor evenementen. Bevat ook andere zaken zoals inzage voor partners.
- openapi-generator: bevat node module en script om code te generen (zie identity.sh)
- ops: Overige zaken die met ops te maken hebben zoals helm deployments

## OpenApi specs

Alle HTTP requests die de frontend nodig heeft worden via OpenApi gegenereerd. De relevante specs kan je genereren met het scriptje `generate-spec.sh`. Daarna kan je de specs vinden in de `openapi-specs` folder. Tevens zorgt dit scriptje ervoor dat de frontend code wordt gegenereerd. We zijn niet van plan om deze in de CI te genereren dus deze dien je mee te committen.

## Setup

Voor je begint, kopieer de `.envrc.example` naar `.envrc`, pas de `MountPath` aan naar een directory naar keuze (hier worden geuploade bestanden opgeslagen) en gebruik `direnv allow`.

<<<>>>UPDATE<<<>>>

Note : `direnv allow` export all the env variable from the `.envrc` file to system environments. if `direnv allow` works successfully it prints out all the exported variables in the terminal. if it doesn't print anything in your terminal. you need to hook it into the shell you're using.  
i.e. if you're using `zsh` shell Add the following line at the end of the ~/.zshrc file: It will hook direnv to shell when the shell starts.

```
`eval "$(direnv hook zsh)"`
```

Once the hook is configured, restart your shell for direnv to be activated. for more info browse the following link: https://github.com/direnv/direnv/blob/master/docs/hook.md

Niet alle databases worden automatisch aangemaakt. De Ory services hebben een lege database nodig voor ze van start kunnen met de migraties. Start daarom eerst de database alleen op en run de opvolgende commando's om de databases aan te maken.

```
docker compose up -d db
docker compose exec db psql -U postgres -c "CREATE DATABASE kratos;"
docker compose exec db psql -U postgres -c "CREATE DATABASE hydra;"
```

Het kan zijn dat je alles al had opgestart voordat je dit las. Vergeet in dit geval niet om de services opnieuw te starten. Anders mis je migraties en blijf je errors zien!

Na aanmaken van de databases kan je de identity server opstarten met `docker compose up -d identity_ui`. Controleer of alles is opgestart met `docker ps`. Je zou 5 containers moeten zien draaien:

1. identity
2. identity_ui
3. oidc
4. smtp
5. db

<<<>>>UPDATE<<<>>>

Note : If you get the `nuget restore error NU1301` error while performing `docker compose up -d identity_ui`. Open docker desktop, and go to setting then docker engine tab and add the "dns": ["8.8.8.8"] in the docker settings.

Start vervolgens de backend op in de Backend folder `dotnet run`. _Backend werkt helaas momenteel niet via docker compose_. Start als laatste de backoffice op vanuit backoffice-web: `yarn start`.

Open je browser in http://127.0.0.1:3000. Zie je het login scherm? Gefeliciteerd, je kan aan de slag. Er zijn een aantal gebruikers aangemaakt. De uitnodigingen kan je in de mailbox vinden: http://localhost:4436.

De app (mobile-app) is gekoppeld aan de testomgeving http://sbg.milvum.dev en kan je daarom los opstarten.

### Development

Voor een opgezet project voldoet het om 3 commando's te gebruiken:

- `docker compose up -d identity_ui`
- `dotnet run` (in `Backend`)
- `yarn start` (in `backoffice-web`)
  Hiermee start je de volledige infrastructuur op.

<<<>>>UPDATE<<<>>>

Note: If the backoffice-web is still not functioning and you receive a "This page isn’t working" error when accessing http://127.0.0.1:3000 in your web browser, you may need to manually restart the identity_ui-1 service. To do this, follow these steps:

1. Stop the identity_ui-1 service from Docker Desktop.
2. In the terminal, navigate to the Identity folder within the project directory.
3. Execute the command dotnet run to manually restart the identity_ui-1 service.
   After completing these steps, attempt to re-run the backoffice-web service and verify if the issue has been resolved.

Soms zijn er veranderingen in de OAuth client, nadat je het project al hebt draaien. De seeder van Identity maakt clients automatisch aan, mits ze nog niet bestaan. Als er aanpassingen zijn geweest, verwijder de client dan handmatig:

```bash
docker compose exec oidc hydra clients delete --endpoint http://localhost:4445 <client id>
```

## Identity infrastructuur

Ons identiteits systeem is gebouwd op de services van Ory, specifiek [Kratos](https://www.ory.sh/kratos/docs/) en [Hydra](https://www.ory.sh/hydra/docs/).

- Kratos is een user management systeem. Het verzorgt dat gebruikers kunnen inloggen, registreren, dat hun wachtwoorden netjes opgeslagen worden enz. Het is een API-only oplossing wat betekent dat we zelf onze UI mogen bouwen (het Identity project).
- Hydra is een service wat het hele OIDC protocol verzorgt. Het stuurt de gebruiker heen en weer naar pagina's waar hij moet wezen om het inloggen voor elkaar te krijgen. Het voert niet zelf het inloggen uit, maar laat dit over aan het systeem waaraan het gekoppeld wordt.
  Beide services van Ory zijn losstaand van elkaar met als enige overeenkomst dat ze te maken hebben met het authenticeren van gebruikers. Dit houdt ook in dat we wat werk moeten doen om ze samen te laten werken.

De flows die nodig zijn staan goed beschreven op hun website. Je moet je alleen indenken dat als je kijkt naar de [Login flow van Hydra](https://www.ory.sh/hydra/docs/concepts/login) dat het stukje "Authenticates user with credentials" vervangen moet [de login flow van Kratos](https://www.ory.sh/kratos/docs/self-service/flows/user-login). Details zijn het beste te vinden door de documentatie te bestuderen. Onthoud voor ons dat het inlogsysteem uit 3 componenten bestaat Kratos (identity in docker compose), Hydra (oidc in docker compose) en Identity (identity_ui in docker compose). Hierbij moet Identity worden gezien als de lijm tussen Kratos en Hydra, want deze bepaald wanneer er naar wie doorverwezen moet worden.

Mocht je Identity lokaal willen draaien, pas dan de kratos.yml aan. Hierin staat de admin.base_url naar http://identity:4434/ (de docker compose service). Wanneer je Identity zonder docker draait zal deze niet gevonden kunnen worden. Verander het daarom naar http://127.0.0.1:4434/ (of verander je hosts file om identity -> 127.0.0.1 te laten wijzen).

## Data import

Er zijn 2 manieren om data te importeren.

1. De seeder laad vanuit de appsettings.json een x aantal start objecten in. Dit is vooral voor kleinschalige data die voor de applicatie beschikbaar moet zijn.
2. Voor grotere zaken is er een import CLI die csv's kan inladen. Dit is met name gebouwd voor postcodes, buurten en wijken aangezien deze te groot zijn om in een json bestand te schrijven en in git op te slaan.

De CLI is aan te roepen vanuit de `Backend` folder. Ipv `dotnet run`, geef je wat extra argumenten mee. In de repo staat wat test data met de wijken/buurten van Den Haag en Brielle en de postcodes van Brielle. Deze kun je als volgt importeren:

```
dotnet run -- import -t Districts -f ../imports/dev_municipalities.csv -s $'\t'
dotnet run -- import -t Districts -f ../imports/dev_districts.csv -s $'\t'
dotnet run -- import -t Districts -f ../imports/dev_neighbourhoods.csv -s $'\t'
dotnet run -- import -t PostalCodes -f ../imports/dev_postalcodes.csv -s ","
dotnet run -- import -t TagVisibility -f ../imports/tagvisibility.csv -s ";"
```

De originele data komt van [het cbs](https://www.cbs.nl/nl-nl/maatwerk/2021/36/buurt-wijk-en-gemeente-2021-voor-postcode-huisnummer). Om toekomstige imports mogelijk te maken, houden we ons aan de structuur van die CSV's. Voor de testomgeving (en productie) kunnen deze imports lokaal gerunt worden, onder de aanname dat je toegang hebt tot de bijbehorende database en je je .envrc correct instelt.

### Synchroniseren van postcodes

Medio september wordt de lijst van postcodes bijgewerkt bij het CBS. Deze dienen opnieuw geimporteerd te worden via de import functie (wederom eerst Districts, dan PostalCodes). Naderhand moet de data echter geupdate worden. Hiermee krijgen onbekende postcodes opnieuw gemeentes/wijken/buurten toegewezen.

```
dotnet run -- resync -t PostalCodes
```

In 2023 is zijn de wijk codes dusdanig aangepast dat we niet meer op dezelfde manier ID's konden afleiden. Als gevolg daarvan moe(s)ten alle bestaande tags verwijderd worden. Hiervoor is de resync PresPostalCodes bedacht. Deze stap verwijderd alle bestaande tags die met gebieden te maken hebben. Daarnaast geeft het je een lijst met queries om de data te recoveren die niet verkregen kan worden met de PostalCode resync. Deze queries moet je als laatste uitvoeren. **Belangrijk dat je de queries niet verliest!**. Samenvattend:

1. **DEZE STAP VERWIJDERD ALLE AREA TAGS:** `dotnet run -- resync -t PrePostalCodes`
2. Sla output ergens op
3. Importeer alle municipalities, districts, neighbourhoods en postalcodes. In die volgorde. Uiteraard niet de `dev_`-files. _Notitie: CBS gebruikt "s-Gravenhage", maar SBG wilt graag "Den Haag", dus pas dit aan. In 2023 was er een wijk en buurt "Buitenland", maar uiteraard geen gemeente. Dit kan daardoor de import laten falen, dus verwijder deze rijen uit de CSV._
4. Link alle postcode gerelateerde data met `dotnet run -- resync -t PostalCodes`
5. Run queries die je in stap 2. hebt opgeslagen.

## Synchroniseren met Identity

De Identity en Backend server delen wat data, namelijk emails en rollen. Backend zorgt er grotendeels voor dat wijzigingen naar Identity doorgestuurd worden via de Kratos API. In uitzonderlijke gevallen (zoals bij toevoeging van een data veld) moeten we over alle gebruikers itereren en een update sturen naar Identity. Gebruik hier de `update-identity` functie in het Backend project voor:

```
cd Backend
dotnet run -- resync -t Identity
```

## Dynamic Links

Er worden dynamic links gebruikt om de gebruiker naar de app te sturen, de playstore of een andere website, afhankelijk v/d situatie. Het overgrote deel van onze links leunen op firebase. Deze zijn te herkennen aan het gebruik van het domein `link.sbg-app.nl`. In sommige gevallen zijn deze handmatig aangemaakt en kan je uitlezen aan de parameters wat er gebeurd. In het geval van een shortlink is dit niet duidelijk, maar kan je altijd `&d=1` erachter zetten en in je browser plakken.

**Belangrijk:** Momenteel is er 1 firebase voor zowel de productie als testomgeving. Vandaar dat voor beide omgevingen er gebruik wordt gemaakt van `link.sbg-app.nl`. In het geval dat we overstappen naar een splitsing in SBG-1280, moeten enige geconfigureerde links aangepast worden (bijv Identity - values.test.yaml).

Links:

- Instellingen dashboard: https://console.firebase.google.com/
- Documentatie: https://firebase.google.com/docs/dynamic-links/create-manually

## Email relay

Mocht je e-mails in MailDev willen doorsturen naar echte emails om styling te checken in verschillende email clients, dan kan je gebruik maken van de relay.

Stappen

1. Maak een App password aan voor een Google account, kan jezelf zijn of `apps@milvum.com`
2. Kopieer `docker-compose.override.yml.example` en hernoem naar `docker-compose.override.yml`
   - Kan ook via command line met `cp docker-compose.override.yml.example docker-compose.override.yml`
3. Vervang `MAILDEV_OUTGOING_USER` en `MAILDEV_OUTGOING_PASS` met de gegevens uit stap 1.
4. Start smtp opnieuw met docker-compose.
