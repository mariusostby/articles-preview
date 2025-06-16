# Unngå fallgruver med Azure Policy: Den skjulte kompleksiteten i managed services

*Den skjulte kompleksiteten bak Azure Policy: hvorfor managed services endrer alt og hvordan implementere styring som fungerer i praksis*

## Introduksjon

Azure Policy virker enkelt i grunnleggende scenarioer--bruk en regel, håndhev styring, oppretthold compliance. Men denne enkelheten er villedende. Når du introduserer managed services som Azure Kubernetes Service (AKS), Azure Synapse og Azure Databricks i miljøet, avslører Azure Policy lag av kompleksitet som kan overraske selv erfarne devops-ingeniører.

Utfordringen handler ikke bare om å unngå "uforsiktige" policy-deployments. Det handler om å forstå hvordan Azure Policy og Managed Services fungerer sammen. Managed Services som f.eks AKS provisjonerer og administrerer underliggende Azure-ressurser automatisk, på måter som kan komme i konflikt med policies, og skaper arkitektoniske brudd som er vanskelige å forutsi og feilsøke.

I denne artikkelen bruker vi AKS som et detaljert case for å demonstrere hvorfor enkle policy-tilnærminger kan være risikabelt i sammenheng med Managed Services. Vi starter med et scenario hvor en grunnleggende policy i stillhet ødelegger grunnleggende Kubernetes funksjonalitet, og går deretter gjennom stadig mer sofistikerte løsninger--fra den taktiske "Monitor Først"-tilnærmingen for eksisterende miljøer til tilpassede policy-definisjoner som kreves for greenfield-deployments.

Denne progresjonen fra portal-basert styring til policy-as-code-løsninger reflekterer et større bilde: mens Azure Policy kan administreres enkelt for grunnleggende scenarioer, krever større miljø med Managed Services typisk Infrastructure as Code-tilnærminger og rammeverk som Enterprise Policy as Code (EPAC).

Etter å ha lest denne artikkelen vil du forstå ikke bare hvordan du unngår policy-relaterte tjenesteavbrudd, men når og hvorfor Azure Policy-kompleksitet blir uunngåelig. Du vil ha praktiske tilnærminger for både brownfield- og greenfield-scenarioer, og du vil gjenkjenne tegnene som indikerer når du har vokst ut av portal-basert policy-administrasjon.

### Hvem trenger å forstå dette?

Denne artikkelen er essensiell hvis du:
- Planlegger å implementere Azure Policy i miljøer med Managed Services
- Har opplevd uforklarlige feil etter policy-deployments
- Administrerer AKS, Synapse, Databricks eller andre tjenester som auto-provisjonerer ressurser
- Trenger å balansere styringskrav med operasjonell stabilitet
- Ønsker å unngå kostbare nedetid og frustrasjon pga policy-relaterte tjenesteavbrudd

## Policy-implementasjon: Forstå miljøet ditt

Før vi går inn på tilnærminger til implementasjon, er det viktig å forstå forskjellen på to typiske scenarier:

**Brownfield-miljø**: Eksisterende Azure-miljøer med etablerte workloads og tjenester. Men selv i brownfield-miljøer blir nye ressurser konstant opprettet--enten av brukere som deployer nye workloads eller av Managed Services som provisjonerer infrastruktur automatisk.

**Greenfield-miljø**: Nye Azure-miljøer hvor du starter på nytt, typisk med ønske om å ha policies på plass fra dag én.

Hovedutfordringen er at begge scenarioer involverer opprettelse av nye ressurser, men det er vesentlige forskjeller knyttet til hvordan du kan forutse hvilken effekt en policy deployment kan ha.


**Brownfield: Den risikable tilnærmingen: Enforce først**
- Raskt å implementere
- Umiddelbar effekt
- MEN: Kan forårsake uventede tjenesteavbrudd
- Vanskelig å feilsøke når ting bryter sammen

**Brownfield: Den sikre tilnærmingen: Monitor først**
- Krever planlegging
- Forsinket effekt
- MEN: Unngår operasjonelle overraskelser for eksisterende mønstre
- Gir tydelig vurderingsgrunnlag
- MEN: Kan fortsatt støte på problemer med nye ressurser eller Managed Services

*Merk: "Monitor Først"-tilnærmingen hjelper med kjente ressursmønstre, men eliminerer ikke alle risikoer. Avanserte scenarioer krever tilpassede policy-definisjoner som kan håndtere automatisk opprettelse og management av ressurser*

![Two Approaches](/DangersOfPolicy/img/1-twoapproaches-EN.png)

## Kjerneproblemet: Hvorfor Managed Services kompliserer Azure Policy

Azure Managed Services provisjonerer og administrerer underliggende Azure-ressurser på dine vegne automatisk. Dette skaper en fundamental utfordring når policies med restriktive effekter håndheves uten å vurdere deres operasjonelle krav.

For å illustrere denne utfordringen, la oss undersøke Azure Kubernetes Service (AKS) som et representativt eksempel. Når du deployer workloads til AKS, provisjonerer plattformen automatisk Azure-ressurser som managed disks for persistent storage, public IP-er for load balancers og network security groups. AKS bruker Azure Resource Manager for å opprette disse ressursene etter behov, men Kubernetes har ingen innebygde mekanismer for å tilfredsstille mange Azure Policy-krav.

Dette arkitektoniske bruddet mellom operasjoner på tjenestenivå og Azure Policy-krav er ikke unikt for AKS. Mange Azure Managed Services står overfor samme utfordring:

- **Azure Synapse**: Oppretter storage accounts, data lake storage og analytics-ressurser
- **Azure Databricks**: Provisjonerer virtual networks, VM-er og storage  
- **Azure Machine Learning**: Administrerer compute instances og tilhørende infrastruktur
- **Logic Apps**: Oppretter connection-ressurser og integration service environments
- **App Service**: Provisjonerer storage, nettverkskomponenter og sertifikater

Enhver policy som kan blokkere eller modifisere ressursopprettelse kan påvirke disse tjenestenes evne til å fungere ordentlig. Den følgende demonstrasjonen med AKS illustrerer mønstre som gjelder bredt på tvers av Azures Managed Services-økosystem.

## Testmiljøet: AKS som eksempel

For å demonstrere dette problemet satte jeg opp en test fokusert på AKS. Dette eksemplet representerer et typisk brownfield-scenario hvor policies introduseres til et eksisterende, fungerende miljø:

1. **Initielt oppsett**: Deployet et fungerende AKS-cluster og validerte grunnleggende funksjonalitet
2. **Introdusere policy**: Deployet en policy som blokkerer ressursopprettelse hvis en spesifikk tag mangler (som et representativt eksempel)
3. **Effekten**: Observerte hvordan policyen påvirket grunnleggende Kubernetes-funksjonalitet

### Initielt oppsett

Jeg deployet et standard AKS-cluster og logget inn i det som cluster-administrator. For å verifisere normal funksjonalitet deployet jeg en enkel WordPress-applikasjon, som krever persistent storage og nettverksressurser.

Som forventet deployet applikasjonen vellykket i standardkonfigurasjonen.

### Introdusere policy

For denne demonstrasjonen simulerte jeg et vanlig styringskrav: håndheve kostnadssporing ved å kreve en "costcenter"-tag på alle Azure-ressurser. Jeg brukte den innebygde policyen "Require a tag on resources" med DENY-effekt, som forhindrer opprettelse av enhver ressurs uten den spesifiserte taggen.

![Assign Policy](/DangersOfPolicy/img/2%20-%20Assign%20policy.png)

![Assign Policy](/DangersOfPolicy/img/3%20-%20Assign%20policy%202.png)

Dette eksemplet representerer en bredere klasse av policies som kan begrense ressursopprettelse basert på ulike styringskrav.

### Effekten

Da jeg prøvde å redeploye WordPress etter å ha deployet policyen, avslørte resultatene et kritisk problem:

1. Ingen umiddelbare feilmeldinger dukket opp på Kubernetes-nivå

![Consequence](/DangersOfPolicy/img/4%20-%20consequence1.png)

2. Pods og PersistentVolumeClaims (PVC-er) ble værende i "Pending"-tilstand

![Consequence](/DangersOfPolicy/img/5%20-%20consequence2.png)

3. Applikasjonen startet ikke, uten klar indikasjon på rotårsaken fra Kubernetes-perspektivet

Ved å grave dypere i loggene avslørte det faktiske problemet: "RequestDisallowedByPolicy." Når AKS prøvde å opprette managed disks for WordPress PVC-ene, blokkerte Azure Resource Manager operasjonene på grunn av policy-brudd.

![K8slogs](/DangersOfPolicy/img/6%20-%20Kubernetes%20logs.png)

Ved å se på Activity Logs i MC_*-ressursgruppen bekreftet det mange mislykkede operasjoner initiert av AKS-clusterets Managed Identity, alle blokkert av policyen.

![Activity logs](/DangersOfPolicy/img/7%20-%20Azure%20activity%20logs.png)

![Activity logs](/DangersOfPolicy/img/8%20-%20Activity%20log%20entry.png)

MC_*-ressursgruppen opprettes automatisk av Azure når du deployer et AKS-cluster--det er hvor AKS lagrer alle de underliggende infrastrukturkomponentene den administrerer på dine vegne, inkludert virtuelle maskiner, managed disks, network security groups, load balancers og public IP-adresser. 

Denne separasjonen lar AKS administrere infrastruktur uten å kreve at brukere har direkte tilgang til disse underliggende ressursene, men det betyr også at policy-brudd i dette skjulte laget kan forårsake mystiske feil på applikasjonsnivå.

## Den grunnleggende utfordringen

Kjerneproblemet strekker seg utover bare AKS og gjelder mange Azure Managed Services:

1. **Arkitektonisk brudd**: Tjenestespesifikke API-er har ingen bevissthet om Azure Policy-krav
2. **Transparent ressursopprettelse**: Managed Services oppretter Azure-ressurser uten å eksponere alle Azure-spesifikke egenskaper
3. **Begrenset kontroll**: Tjenestegrensesnitt gir ikke nødvendigvis noen mulighet til å tilfredstille mange Azure-spesifikke policy-krav
4. **Stille feil**: Policy-brudd manifesterer seg ofte som generiske feil med minimal diagnostikk

![Architectural disconnect](/DangersOfPolicy/img/AKS%20Policy%20Disconnect.png)

Når du deployer policies med restriktive effekter som DENY, MODIFY, eller til og med DEPLOYIFNOTEXISTS uten riktig forberedelse, kan du skape situasjoner hvor Managed Services ikke kan oppfylle sine operasjonelle behov.

## Quickfix: Policy-exemption

For å løse det umiddelbare problemet med AKS opprettet jeg en policy-exemption for MC_*-ressursgruppen:

![Exemption](/DangersOfPolicy/img/9%20-%20Exemption%20Assignment.png)

Etter å ha satt exemption og redeployet WordPress, kunne applikasjonen deployes uten problemer. Policyen fortsatte å gjelde for alle andre ressurser, men AKS kunne nå provisjonere den nødvendige infrastrukturen automatisk.

![Post-exemption Deploy](/DangersOfPolicy/img/10%20-%20DeployWPPostExemption.png)

Lignende exemptions ville vært nødvendig for ressursgrupper administrert av andre tjenester som Synapse, Databricks og andre.

Selv om exemptions gir en taktisk løsning, representerer de en reaktiv tilnærming. En mer strategisk tilnærming er nødvendig.

## Løsningsstrategier: Fra reaktiv til proaktiv

Basert på denne demonstrasjonen er her en strukturert tilnærming til Azure Policy-implementasjon som balanserer styringskrav med operasjonell pålitelighet:

### For brownfield-miljøer: Monitor Først-prinsippet

Når du jobber med eksisterende miljøer, reduserer denne tilnærmingen risikoen for å forstyrre etablerte workloads når du evaluerer policy-deployments:

1. **Initial deployment**: Deploy alle policies med audit-effekt først
2. **Påvirkningsvurdering**: Evaluer resultater over en definert periode (typisk 1-2 uker)
3. **Identifiser unntak**: Dokumenter alle ressursmønstre som ikke kan møte policy-krav
4. **Implementer unntak**: Opprett policy-exemptions for identifiserte scenarioer
5. **Gradvis håndhevelse**: Skift til enforce-effekter etter å ha adressert alle identifiserte problemer

![Monitor First Approach](/DangersOfPolicy/img/MonitorFirstApproach.png)

### For greenfield-miljøer: Proaktiv planlegging

For nye miljøer hvor du vil ha styrings-policies aktive fra dag én:

1. **Identifiser Managed Services**: Katalogiser alle planlagte Azure Managed Services i miljøet ditt
2. **Dokumenter ressursmønstre**: Forstå hvilke Azure-ressurser hver Managed Service provisjonerer
3. **Design policy-unntak**: Opprett exemptions proaktivt for alle administrerte ressursgrupper
4. **Implementer med unntak**: Deploy policies med både enforce-effekter og nødvendige exemptions
5. **Kontinuerlig overvåking**: Etabler overvåking for å fange opp nye ressursmønstre

### Policy-design for Managed Services

For mer sofistikerte implementasjoner kan du opprette tilpassede policy-definisjoner som er bevisst på Managed Services:

```json
{
    "policyRule": {
        "if": {
            "allOf": [
                {
                    "field": "type",
                    "equals": "Microsoft.Compute/disks"
                },
                {
                    "not": {
                        "field": "Microsoft.Compute/disks/managedBy",
                        "exists": "false"
                    }
                },
                {
                    "field": "tags['costcenter']",
                    "exists": "false"
                }
            ]
        },
        "then": {
            "effect": "deny"
        }
    }
}
```

## Utover AKS: Generelle prinsipper for Azure Policy og Managed Services

Selv om denne artikkelen har fokusert på AKS som en case-studie, bør den samme metodikken anvendes når du implementerer policies som kan påvirke:

- **Azure Synapse**: Sjekk ressursgrupper som inneholder Synapse workspaces
- **Azure Databricks**: Se gjennom administrerte ressursgrupper med navnemønstre som "databricks-rg-*"
- **App Service**: Se etter App Service-planer og deres tilstætende ressurser
- **Azure Machine Learning**: Undersøk ML workspace-ressursgrupper
- **Ethvert PaaS-tilbud**: Vurder hvorvidt tjenesten provisjonerer og administrerer ressurser

For hver tjeneste, identifiser ressursgruppene og ressursene som automatisk administreres, og evaluer policy-påvirkning før håndhevelse.

## Avanserte strategier: Forstå policy-regler og greenfield-scenarioer

"Monitor Først"-tilnærmingen fungerer godt for eksisterende miljøer, men hva om du implementerer styring i et greenfield-scenario hvor du vil ha DENY-policies håndhevet fra starten av? For å forstå hvorfor dette krever en annen tilnærming, la oss undersøke hvordan Azure Policy-regler fungerer.

### Greenfield-utfordringen

I greenfield-miljøer står du overfor en umulig situasjon: du vil ha restriktive policies aktive fra dag én, men du kan ikke bruke "Monitor Først"-tilnærmingen til å identifisere unntak fordi det ikke finnes eksisterende ressurser å evaluere.

For vårt AKS-eksempel skaper dette et problem:
- Du vil at tagging-policyen skal DENY ressurser uten riktige tags
- Men AKS vil opprette MC_*-ressursgruppen under deployment
- Policyen vil blokkere denne opprettelsen, noe som forårsaker deployment-feil
- Du kan ikke unntaksbehandle ressursgruppen på forhånd fordi den ikke eksisterer enda

### Portal-begrensninger og behovet for custom policies

Azure-portalen har betydelige begrensninger for å håndtere disse scenarioene:
- **Ingen wildcard-støtte i notScopes**: Du kan ikke ekskludere MC_*-mønstre i policy assignments gjennom portalen
- **Kun statiske exemptions**: Exemptions må rettes mot spesifikke, eksisterende ressurser
- **Begrenset conditional logic**: Komplekse ekskluderingsmønstre krever custom policy definitions

### Greenfield-realiteten: Pipeline-feil

For å illustrere denne utfordringen, her er hva som skjer når du prøver å deploye AKS i et greenfield-miljø med restriktive policies allerede aktive:

![Pipeline failure](/DangersOfPolicy/img/15%20-%20greenfieldpipefail1.png)
![Pipeline failure2](/DangersOfPolicy/img/16%20-%20greenfieldpipefail2.png)

Deploymentet feiler når AKS forsøker å opprette MC_*-ressursgruppen og de nødvendige ressursene innenfor. Det finnes ingen god måte å forhindre dette gjennom portal-basert policy-administrasjon.

Pipeline-feilen viser tydelig policy-bruddet, men timingen gjør det umulig å opprette exemptions reaktivt, ettersom scopet hvor du vil opprette et exemption for ikke eksisterer enda.

Dette illustrerer når forståelse av hvordan policy-regler fungerer blir avgjørende.

## Sammenligning av built-in vs custom policy-regler

For å forstå hvorfor vi trenger custom policies for greenfield-scenarioer, la oss sammenligne hvordan de built-in og custom policy-reglene håndterer vår AKS-utfordring.

### Built-in policy-regelen

Built-in "Require a tag on resources"-policy har en enkel regelstruktur:

```json
"policyRule": {
    "if": {
        "field": "[concat('tags[', parameters('tagName'), ']')]",
        "exists": "false"
    },
    "then": {
        "effect": "deny"
    }
}
```

Denne regelen stiller et enkelt spørsmål: "Har denne ressursen den nødvendige taggen?" Hvis ikke, blokker. Det er enkelt, men lite fleksibelt - vi eksluderer ikke ressurgrupper styrt av Managed Services.

### Custom Policy Regel: Gjør regelen smartere

Denne custom policy regelen tar en mer sofistikert tilnærming med flere betingelser samlet i en ```allOf``` gruppering:

```json
"policyRule": {
    "if": {
        "allOf": [
            // Condition 1: Resource type filtering
            {
                "anyOf": [
                    {
                        "field": "type",
                        "in": "[parameters('resourceTypeList')]"
                    },
                    {
                        "allOf": [
                            {
                                "value": "[length(parameters('resourceTypeList'))]",
                                "equals": 0
                            },
                            {
                                "count": {
                                    "value": "[parameters('excludedResourceTypes')]",
                                    "name": "excludedResourceTypes",
                                    "where": {
                                        "field": "type",
                                        "like": "[current('excludedResourceTypes')]"
                                    }
                                },
                                "equals": 0
                            }
                        ]
                    }
                ]
            },
            // Condition 2: Tag requirement (same as built-in)
            {
                "field": "[concat('tags[', parameters('tagName'), ']')]",
                "exists": "false"
            },
            // Condition 3: Resource group exclusions
            {
                "count": {
                    "value": "[parameters('excludedRG')]",
                    "name": "excludedRG",
                    "where": {
                        "value": "[resourceGroup().name]",
                        "like": "[current('excludedRG')]"
                    }
                },
                "equals": 0
            }
        ]
    },
    "then": {
        "effect": "[parameters('effect')]"
    }
}
```

**NB**: Denne regelen er hentet fra Enterprise Policy as Code (EPAC) rammeverket.

### Hovedpoenget: Ekslusjon på ressursgruppe-nivå

Custom policy regelen håndterer AKS scenariet med sitt ```excludeRG``` parameter:

```json
"excludedRG": {
    "defaultValue": [
        "MC_*",
        "synapseworkspace-managedrg-*",
        "managed-rg-*", 
        "databricks-*",
        "DefaultResourceGroup*",
        "NetworkWatcherRG",
        "LogAnalyticsDefault*",
        "cloud-shell-storage*"
    ]
}
```

**Når AKS prøver å opprette ressurser:**

- MC_myakscluster_eastus_12345-ressursgruppen matcher MC_*-mønsteret og blir ekskludert.
- Alle ressurser opprettet innenfor den ressursgruppen blir automatisk ekskludert - managed disks, network security groups, public IPs, load balancers og VMs.
- AKS kan administrere hele all nødvendig infrastruktur uten å bli blokkert av policy.

Denne ekskluderingen på ressursgruppe-nivå er langt mer effektiv enn å prøve å ekskludere individuelle ressurstyper. Ett wildcard-mønster håndterer det hele behovet AKS måtte ha for underliggende infrastruktur.

### Policy-evaluering: Runtime vs assignment time
Denne sammenligningen illustrerer et grunnleggende prinsipp: Azure Policy-regler evalueres når ressurser opprettes, ikke når policyen deployes. Custom policyen kan gjøre dynamisk evaluering basert på ressursegenskaper som ikke eksisterer når policyen blir deployet.

Dette illustrerer:

- Built-in policies fungerer godt for enkle, universelle krav
- Custom policies er nødvendige for komplekse scenarioer med managed services
- Portal-baserte unntak kan ikke håndtere "fremtidige" ressurser som ikke eksisterer ennå

### Enterprise Policy as Code (EPAC) Framework

Custom policyen som vises her kommer fra Microsofts Enterprise Policy as Code (EPAC) framework, en open-source, Microsoft-kuratert løsning som er mye brukt i store enterprise-organisasjoner verden rundt. EPAC tilbyr et omfattende bibliotek av custom policy-definisjoner som håndterer reelle scenarioer som managed service-ekskluderinger.

EPAC-frameworket tilbyr betydelige fordeler når man takler dette nivået av policy-kompleksitet:

- Ferdiglagde policies som håndterer vanlige managed service-mønstre
- Aktiv community-støtte fra erfarne policy administratorer
- Microsoft-backing og regelmessige oppdateringer
- Utprøvde tilnærminger fra store miljø

Jeg kan personlig vitne om verdien av EPAC-frameworket siden jeg har brukt det til å implementere og vedlikeholde governance i store miljøer med over 10000 ressurser, inkludert stort sett alle managed services som eksisterer, i nesten 3 år.

### Ulike tilnærminger til policy as code

**ARM/Bicep Templates:** Deploy custom policies sammen med infrastrukturen din ved å bruke Infrastructure as Code.

**Terraform/Pulumi:** Inkluder policy-definisjoner som del av din Infrastructure as Code.

**EPAC Framework:** Bruk en komplette enterpris-grade policy management-løsning, og dra fordel av et aktivt open-source community.

### Når man skal anvende ulike prinsipper

**Monitor først (Brownfield):**

- Eksisterende miljøer med etablerte workloads
- Gradvis implementering av overordnet styring
- Lærings- og oppdagelsesfaser
- Portal-basert administrasjon er tilstrekkelig

**Custom Policies (Greenfield):**

- Nye miljøer der man ønsker å ha overordnet styring på plass fra dag 1
- Komplekse ekskluderingsmønstre er nødvendig for Managed Services
- Standardisert governance på tvers av flere miljøer
- Enterprise-skala policy-administrasjon

### Den tekniske realiteten

Denne progresjonen fra portal-basert til kode-basert policy-administrasjon reflekterer en viktig realitet: Azure Policy ser enkelt ut, men blir raskt komplekst i praksis. Mens grunnleggende policies kan administreres gjennom portalen, krever enterprise governance typisk Infrastructure as Code-tilnærminger for å håndtere den fulle kompleksiteten i moderne Azure-miljøer.

For organisasjoner som implementerer sofistikert governance i stor skala, tilbyr rammeverk som EPAC både det tekniske fundamentet og community-ekspertisen som trengs for å navigere Azure Policys kompleksitet uten problemer.

## Konklusjon
Azure Policy kan fremstå enkelt, men man støter på betydelig kompleksitet når det kombineres med managed services som AKS, Synapse og Databricks. Disse tjenestene provisjonerer og styrer Azure-ressurser på måter som kan komme i konflikt med governance-policies, og skaper operasjonelle utfordringer som er vanskelige å forutse og feilsøke.

Denne artikkelen har demonstrert to essensielle tilnærminger for ulike scenarioer:

For brownfield-miljøer med eksisterende arbeidsbelastninger gir "Monitor First"-metodikken en trygg vei til policy-håndhevelse. Ved å starte med audit-modus, evaluere påvirkning og planlegge unntak før håndhevelse, kan du implementere governance samtidig som du unngår uventede forstyrrelser.

For greenfield-miljøer eller komplekse ekskluderingskrav blir custom policy-definisjoner med innebygd ekskluderingslogikk nødvendige. Portalens begrensninger -- ingen wildcard-støtte, kun statiske unntak -- betyr at enterprise governance typisk krever Infrastructure as Code-tilnærminger og rammeverk som EPAC.

Progresjonen fra enkle portal-baserte policies til sofistikerte policy-as-code-løsninger er ikke en svakhet ved de enklere tilnærmingene -- det er den naturlige utviklingen når man håndterer den virkelige kompleksiteten i moderne Azure-miljøer. Å forstå når du har vokst fra portal-basert administrasjon er avgjørende for å opprettholde både sterk governance og operasjonell pålitelighet.

Dette har vært en relativt tung artikkel. Hensikten er ikke å fraråde bruk av azure policy, men å understreke at et så kraftig verktøy må brukes riktig. Og kompleksiteten kombinert med skadepotensialet ved feil bruk betyr at dette er noe man må sette av betydelig tid og ressurser til før man går i gang. Det vil lønne seg når man først lykkes, da man kan få en uovertruffen grad av overordnet kontroll.

Husk: God governance forbedrer operasjoner -- den skal ikke forstyrre dem. Start med monitoring, gå videre til custom løsninger når det trengs, og ikke nøl med å ta i bruk enterprise-rammeverk når kompleksiteten krever det.