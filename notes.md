1. Tufin ma obsłużyć maksymalnie dużo standardowych requestów o otwarcie ruchu.
2. Panorama/Palo Alto ma być w pełni „configuration as code”, żeby konfiguracja nie żyła wyłącznie w GUI/Tufinie.

Aktualny provider PaloAltoNetworks/panos jest już całkiem mocny — obecnie 2.0.13 — i pozwala zarządzać Panorama/NGFW, security policy, NAT, address objects/groups, tagami, service objects, device groups, template stackami, routingiem itd.

Scenariusze ruchowe, które powinniście rozpatrzyć

Ja zrobiłbym katalog dokładnie takich przypadków:

Azure subnet → Azure subnet
ten sam VNet
różne VNety
różne subskrypcje
różne landing zones
Azure IP → Azure IP
private IP → private IP
pojedynczy host /32
wiele IP / grupa IP
Azure subnet → Azure IP
Azure IP → Azure subnet
Azure VNet → Azure VNet
cały address space VNetu → cały address space drugiego VNetu
Azure subnet → on-prem subnet
On-prem subnet → Azure subnet
Azure IP → on-prem IP
On-prem IP → Azure IP
Azure subnet/VNet → on-prem grupa sieci
On-prem → Azure VNet / grupa VNetów
Azure → Internet po IP
destination public IP
zakres public IP
Azure → Internet po FQDN
pojedynczy FQDN
wildcard FQDN, jeśli sposób implementacji na PAN na to pozwala
grupa FQDN
Azure → Azure PaaS przez public endpoint
Storage
Key Vault
SQL
ACR
Service Bus
Event Hub itd.
Azure → Azure PaaS przez Private Endpoint
wtedy technicznie robi się z tego private IP/FQDN + routing/DNS
Azure → Azure Service Tag
np. Storage
AzureContainerRegistry
AzureActiveDirectory
AzureMonitor
regionalne warianty typu Storage.WestEurope
Azure → Azure FQDN Tag / logiczna grupa usług Microsoft
tu trzeba osobno ustalić mechanizm translacji, bo PAN-OS nie rozumie natywnie konceptu Azure Firewall FQDN Tag.
AKS node subnet → Azure/on-prem
AKS pod → Azure/on-prem
Azure CNI
Azure CNI Overlay
zależnie od modelu adresacji podów
Azure/on-prem → AKS
node IP
pod IP
internal LoadBalancer
ingress private IP
AKS → Internet po IP
AKS → Internet po FQDN
AKS → usługi Microsoft wymagane przez platformę
ACR
Entra ID
Azure Monitor
Storage
MCR
management endpoints itd.
AKS według Azure Service Tags
AKS według dynamicznej tożsamości workloadu
namespace
cluster
application
label/tag
tutaj zaczyna się już integracja dynamiczna, nie zwykły firewall request.
Azure resource tag → Azure resource tag
np. environment=prod → application=sap
jeśli chcecie kiedyś abstrahować politykę od IP.
Azure tag → subnet/IP/FQDN
Dynamic Address Group → Dynamic Address Group
Dynamic Address Group → statyczny subnet/IP
Statyczny subnet/IP → Dynamic Address Group
User / AD identity → Azure resource
jeśli chcecie User-ID i reguły oparte na użytkownikach/grupach AD.
Azure workload → destination application
Palo Alto App-ID zamiast tylko portu.
Azure workload → URL category
Inbound Internet → Azure workload
DNAT
public LB
public IP
application ingress
Azure workload → Internet z SNAT
Azure → Azure z NAT
np. overlapping networks / translacja adresacji.
Azure → on-prem z NAT
On-prem → Azure z NAT
Ruch pomiędzy środowiskami
DEV → DEV
DEV → PROD
PROD → PROD
PROD → DEV
jako osobne klasy polityki.
Shared services
Azure workload → DNS
NTP
AD/DC
PKI
proxy
monitoring
backup
repozytoria
centralne API.

To jest według mnie minimalny katalog use-case'ów do przejścia z Tufinem.

A teraz najważniejsze: co z tego umie PAN-OS Terraform provider

W samym Terraformie możecie zakodować praktycznie całą warstwę policy.

Bez problemu

IP / subnet / CIDR

10.10.10.15
10.10.10.0/24

panos_address obsługuje ip_netmask, zakres IP i wildcard.

FQDN

api.example.com

jest natywnie obsługiwany przez panos_address.fqdn.

Static Address Groups

APP01
APP02
SUBNET01

→ jedna grupa.

Dynamic Address Groups

Provider obsługuje:

dynamic {
  filter = ...
}

czyli DAG oparty na tagach PAN-OS.

Administrative tags

Te też można zarządzać Terraformem.

Service objects / Service Groups

TCP/UDP/porty oraz ich grupowanie.

Security Rules

Można kontrolować między innymi:

source zone
source addresses
destination zone
destination addresses
application
service
users
tags
logging
security profiles
schedule
action

Provider ma również osobny panos_security_policy_rules, dzięki któremu można zarządzać blokiem reguł i pozycjonować go np. first, last, before, after.

NAT

Jest pełne zarządzanie NAT policy, m.in. source/destination NAT i kolejnością reguł.

App-ID

Security policy obsługuje applications, więc polityki:

source
destination
application = ssl/web-browsing/...

mogą być również IaC.

User-ID / użytkownicy

Security rules mają source_users, więc samą politykę opartą na użytkownikach/grupach da się zapisać w kodzie. Oczywiście osobnym zagadnieniem jest dostarczenie Palo Alto mapowania user ↔ IP.

Najbardziej interesujący wariant dla was: DAG

Tu moim zdaniem leży rozwiązanie dla dynamicznego Azure/AKS.

Terraform tworzy tylko logikę:

DAG: azure-prod-web
filter:
  'azure' and 'prod' and 'web'

a zawartość DAG:

10.10.1.5
10.10.1.8
10.10.3.20
...

może się zmieniać bez modyfikacji security rule.

Provider wspiera dynamiczne address groups właśnie przez filtr tagowy.

Czyli potencjalnie:

Azure tags / AKS metadata
          ↓
     jakiś automat
          ↓
PAN-OS registered IP tags
          ↓
 Dynamic Address Group
          ↓
     Security Rule

To może być bardzo ważne przy waszym wymaganiu:

konfiguracja polityki w kodzie, ale membership dynamiczny.

Natomiast Azure Service Tags to osobny przypadek

Tu trzeba bardzo wyraźnie rozdzielić:

Palo Alto tag ≠ Azure Resource Tag ≠ Azure Service Tag.

Terraform PAN-OS nie przyjmie po prostu:

destination = "Storage.WestEurope"

i nie zrozumie tego jak Azure Firewall.

Trzeba zrobić translację:

Azure Service Tag
       ↓
lista prefixów IP
       ↓
PAN Address Group / EDL / DAG
       ↓
security rule

I ten element musi zrobić:

skrypt,
Azure Function,
pipeline,
integration service,
ewentualnie inny mechanizm dynamicznej rejestracji.

To samo dotyczy Azure FQDN Tags — one są konstruktem Azure Firewall, a nie uniwersalnym obiektem sieciowym PAN-OS.

FQDN jest łatwiejszy

Tutaj sytuacja jest dużo lepsza.

PAN-OS provider potrafi stworzyć:

panos_address {
    name = "api-microsoft"
    fqdn = "something.microsoft.com"
}

więc:

subnet → FQDN
IP → FQDN
DAG → FQDN

można elegancko mieć w IaC.

Co bym więc dał do Tufina

Jeżeli priorytetem jest „Tufinem ile tylko się da”, to katalog podzieliłbym na 3 klasy.

🟢 Tufin — podstawowy workflow

Te przypadki powinny być waszym „golden path”:

IP       → IP
IP       → subnet
subnet   → IP
subnet   → subnet
VNet     → VNet
Azure    → on-prem
on-prem  → Azure

IP       → FQDN
subnet   → FQDN

IP       → Address Group
subnet   → Address Group

service TCP/UDP
service group

Czyli zwykłe requesty aplikacyjne.

🟡 Tufin + przygotowane obiekty z IaC

Tufin nie musi wiedzieć, jak powstała grupa.

Może dostać:

source = DAG-AKS-PROD
destination = DAG-SQL-PROD
service = tcp/1433

i tylko dodać/zmienić security rule.

A Terraform/integracja utrzymuje:

DAG-AKS-PROD
DAG-SQL-PROD

To jest moim zdaniem bardzo sensowny model.

Tak samo:

AZURE-STORAGE-WESTEUROPE
AZURE-ACR
AZURE-MONITOR
AKS-PROD
AKS-DEV
SHARED-DNS
SHARED-AD

Tufin operuje nazwami logicznymi, a nie tysiącami IP.

🔴 Poza Tufinem / pipeline automatyczny

Tu dałbym najbardziej dynamiczne przypadki:

Azure Service Tags → IP prefixes
Azure resource tags → IP
AKS labels → pod IP
AKS namespaces → pod IP
dynamiczna rejestracja IP/tagów
zmienne Microsoft FQDN/IP feeds

Tufin nie powinien być systemem, który co pięć minut synchronizuje membership takich grup.

I tutaj pojawia się bardzo ważne zagadnienie „config w kodzie”

Macie potencjalny konflikt:

Terraform
    ↓
Panorama
    ↑
Tufin

Jeżeli Terraform i Tufin modyfikują te same security rules, prędzej czy później będzie drift.

Dlatego trzeba ustalić granicę własności.

Najlepszy moim zdaniem model:

                 PANORAMA
                    │
       ┌────────────┴────────────┐
       │                         │
Terraform-owned            Tufin-owned
configuration              rule section

Przykładowo:

PRE-RULEBASE

00-platform       ← Terraform
10-shared         ← Terraform
20-automation     ← dynamic/system
30-application    ← Tufin
90-deny           ← Terraform

Provider wręcz pomaga w takim podejściu, bo panos_security_policy_rules może zarządzać grupą reguł i jej pozycją względem innych reguł, zamiast przejmować całą policy.

To jest istotne.

Nie używałbym jednego:

panos_security_policy

do przejęcia całej rulebase, jeśli Tufin ma również do niej pisać.

Czyli finalnie wasze scenariusze architektoniczne do zbadania

Ja na warsztat/projekt przygotowałbym dokładnie te 8 tematów:

Static network policy
IP/subnet/VNet ↔ IP/subnet/VNet
Hybrid policy
Azure ↔ on-prem
FQDN policy
IP/subnet/DAG → FQDN
Azure PaaS
Private Endpoint / public endpoint / Service Tag / FQDN
AKS static
node subnet / pod subnet / ingress IP
AKS dynamic
cluster / namespace / label / workload → DAG
Azure dynamic objects
Azure Resource Tags / Service Tags → PAN DAG/address groups
Policy ownership
Terraform-owned vs Tufin-owned vs dynamically maintained objects

Najważniejsze z tego wszystkiego: PAN-OS Terraform provider nie jest już tutaj takim ograniczeniem, jak mogłoby się wydawać. Obecna wersja potrafi trzymać w kodzie security policy, NAT, address/FQDN objects, static i dynamic address groups, tagi, service objects oraz znaczną część konfiguracji Panoramy.

Problemem nie będzie więc samo „czy da się zapisać Palo Alto w Terraformie”. Problemem będzie przede wszystkim ownership między Tufinem i Terraformem oraz automatyczna translacja dynamicznych konstrukcji Azure/AKS na obiekty zrozumiałe przez PAN-OS. To są dwa punkty, na których skupiłbym PoC.


Tak, i tutaj właśnie warto odseparować „otwieranie ruchu” od „budowy i utrzymania platformy firewall”, bo inaczej Tufin, Terraform Azure i Terraform PAN-OS zaczną sobie wchodzić w drogę.

Ja bym cały temat rozpisał na 4 warstwy własności:

AZURE INFRASTRUCTURE
        ↓
PAN-OS / PANORAMA CONFIGURATION
        ↓
DYNAMIC INTEGRATIONS / IDENTITY
        ↓
TRAFFIC POLICY / CHANGE MANAGEMENT

i każdej warstwie dał jednego właściciela.

1. Azure Infrastructure — tylko Terraform Azure

Tu powinno żyć wszystko, co jest fizycznym/logicalnym zasobem Azure:

VM-Series VM
NIC
subnet
VNet
peering
route table / UDR
Load Balancer
Public IP
NAT Gateway
NSG
Managed Identity
Availability Set / Zones
Storage/bootstrap

Czyli:

azurerm provider
        ↓
wasze utils
        ↓
wasze classes
        ↓
product palo_alto_transit

Tufin nie powinien tego tworzyć.

Panorama również nie powinna tworzyć infrastruktury Azure.

Przykład:

Terraform Azure

FW01
 ├─ nic-mgmt 10.10.1.4
 ├─ nic-untrust 10.10.2.4
 └─ nic-trust 10.10.3.4

FW02
 ├─ nic-mgmt 10.10.1.5
 ├─ nic-untrust 10.10.2.5
 └─ nic-trust 10.10.3.5

To jest Source of Truth = repo Azure.

2. PAN-OS networking — Terraform PAN-OS

Potem masz drugą warstwę.

Azure stworzył NIC:

Azure NIC
10.10.3.4

ale Palo musi wiedzieć, że:

ethernet1/2
IP 10.10.3.4
zone TRUST
virtual-router VR1

I to już nie jest Azure configuration.

To powinno trafiać przez provider panos.

Obecny provider PAN-OS 2.0.13 potrafi zarządzać m.in. Ethernet interfaces, virtual routers, zones i routingiem.

Czyli:

PAN-OS Terraform

Template
Template Stack
    ↓
ethernet1/1 = untrust
ethernet1/2 = trust
ethernet1/3 = mgmt / jeśli dotyczy dataplane
    ↓
Zones
    ↓
Virtual Router
    ↓
routing

To jest moim zdaniem drugi osobny state Terraformowy.

Nie:

jeden gigantyczny Terraform state
Azure + PANOS + policies

tylko:

01-azure-infra
02-panos-platform
03-panos-policy-baseline
3. Statyczne obiekty PAN-OS

Tu dochodzą:

Address Objects
Address Groups
Service Objects
Service Groups
Tags
FQDN objects

I tutaj musisz podzielić obiekty na dwa rodzaje.

Statyczne

Np.:

NET-AZURE-HUB      = 10.10.0.0/16
NET-AZURE-PROD     = 10.20.0.0/16
NET-ONPREM-DC1     = 10.100.0.0/16

HOST-DNS01         = 10.100.1.10
HOST-DNS02         = 10.100.1.11

FQDN-EXAMPLE       = api.example.com

SRV-HTTPS          = tcp/443
SRV-MSSQL          = tcp/1433

To świetnie nadaje się do Terraform.

Czyli:

Terraform → Panorama → Address Objects

Możecie nawet generować część automatycznie z waszego inventory Azure.

Na przykład:

class subnet:
  name = aks-prod
  cidr = 10.50.10.0/24

może dawać output:

10.50.10.0/24

który pipeline PAN-OS używa do stworzenia:

AZ-SUBNET-AKS-PROD

I to byłoby bardzo eleganckie.

4. Dynamiczne obiekty — tutaj jest największy potencjał

I właśnie tego nie traktowałbym jako klasycznego IaC.

Przykładowo macie VM:

vm01
IP = 10.20.1.15
tags:
  application = SAP
  environment = PROD
  role = WEB

Nie chcę, żeby ktoś za każdym razem robił:

terraform apply

bo IP VM się zmieniło.

Lepiej:

Azure Resource Graph / Azure API
             ↓
Panorama Azure Plugin / automation
             ↓
IP → TAG mapping
             ↓
Dynamic Address Group

Panorama ma Azure monitoring plugin, który może użyć Service Principal do pobierania informacji o workloadach i przekazywania firewallom mapowania IP ↔ tag. Następnie Dynamic Address Groups mogą na tych tagach filtrować workloady.

To jest bardzo ciekawe dla was.

Przykład

Azure:

VM:
10.20.1.15

tags:
application = sap
environment = prod

Panorama dostaje mapping.

DAG:

DAG-SAP-PROD

filter:
'application.sap' AND 'environment.prod'

Reguła:

DAG-SAP-PROD
    →
DAG-SQL-PROD
tcp/1433

I VM mogą:

powstawać,
znikać,
zmieniać IP,

a security rule się nie zmienia.

5. Czy DAG powinien być w Terraformie?

Tak.

Ale membership DAG już niekoniecznie.

Terraform:

tworzy:
DAG-SAP-PROD

filter:
tag = application.sap
AND environment.prod

Dynamiczna integracja:

10.20.1.15 → application.sap
10.20.1.15 → environment.prod

Czyli:

Terraform
    ↓
definicja DAG / logika

Azure Plugin/API
    ↓
aktualna zawartość DAG

To jest bardzo czyste rozdzielenie.

6. Subnety

Subnet również ma dwa „życia”.

Azure

Terraform Azure tworzy:

VNet
Subnet
Route Table
NSG
Panorama

Panorama może mieć reprezentację tego subnetu jako:

address object

np.:

AZ-PROD-AKS-SUBNET = 10.50.20.0/24

Nie robiłbym tego ręcznie.

Jeśli wasze monorepo jest Source of Truth dla subnetów, pipeline może automatycznie wygenerować inventory dla Palo Alto.

Czyli:

Azure monorepo
       ↓
outputs / inventory
       ↓
PANOS Terraform
       ↓
Address Objects

To byłoby według mnie bardzo mocne rozwiązanie.

7. VM

VM jako Azure Resource:

Terraform Azure

VM jako firewall policy object:

nie tworzyłbym ręcznie w Terraform PANOS, jeśli mówimy o setkach/tysiącach VM.

Do tego właśnie:

Azure tags
    ↓
Azure Plugin / automation
    ↓
DAG

Statyczny host /32 zostawiłbym tylko dla:

infrastruktury,
DNS,
Domain Controllers,
management,
appliance'ów,
legacy,
wyjątków.
8. Interfejsy firewalli

To jest jeszcze inny rodzaj interfejsów.

Azure:

NIC0
NIC1
NIC2

Terraform Azure.

PAN-OS:

ethernet1/1
ethernet1/2
zones
IP
MTU
management profile
virtual router membership

Terraform PANOS.

Provider to obsługuje.

Czyli ważna zasada:

Azure NIC != PANOS interface configuration

ale muszą mieć wspólne dane wejściowe.

Najlepiej:

platform definition:

trust:
 subnet = 10.10.3.0/24
 fw01 = 10.10.3.4
 fw02 = 10.10.3.5

untrust:
 subnet = 10.10.2.0/24
 fw01 = 10.10.2.4
 fw02 = 10.10.2.5

Z tego:

Azure TF → NIC
PANOS TF → ethernet interfaces
9. Routing

Tu znowu masz dwie strony.

Azure routing
UDR
VNet peering
gateway propagation
LB frontend
next hop virtual appliance

Terraform Azure.

Palo routing
Virtual Router
Static Routes
BGP
ECMP
route redistribution

Terraform PANOS.

Provider ma natywne zasoby Virtual Router i static route.

Czyli np.:

Azure UDR

Spoke:
0.0.0.0/0
    ↓
Internal LB
    ↓
Palo Alto

a Palo:

VR
0.0.0.0/0
    ↓
Azure gateway / upstream

To muszą być dwa skoordynowane kawałki IaC.

10. BGP

BGP również traktowałbym jako platform configuration, a nie traffic change.

Czyli absolutnie nie Tufin.

Terraform PANOS
    ↓
BGP configuration

W kodzie trzymacie:

ASN
router-id
peers
peer groups
authentication
import policy
export policy
redistribution
BFD
timers

A Azure-side:

VPN Gateway
ExpressRoute
Route Server
vWAN

jest tworzony przez Azure Terraform.

Czyli przykład:

Azure Route Server
ASN 65515
10.10.255.4
10.10.255.5
       ↕ BGP
Palo Alto
ASN 65001

Azure side:

azurerm

PAN side:

panos
11. VPN

Podobnie.

VPN ma zwykle kilka warstw.

Azure

Jeśli tunnel endpoint jest Azure VPN Gateway:

VPN Gateway
Local Network Gateway
Connection
Public IP

→ Azure Terraform.

Jeżeli Palo Alto sam terminates IPSec:

Palo:
IKE Gateway
IPSec Crypto Profile
IKE Crypto Profile
IPSec Tunnel
Tunnel Interface
Virtual Router
Route/BGP
Zone

→ PANOS Terraform.

I dopiero:

Security Policy

→ Tufin albo Terraform baseline.

Czyli VPN jest przede wszystkim platform/network configuration, nie requestem o otwarcie portu.

12. AD / Entra ID

Tu trzeba rozdzielić management authentication od identity-based firewall policy.

A. Logowanie administratorów

Np.:

Admin
  ↓
Entra ID / SAML
  ↓
Panorama

To jest konfiguracja platformowa.

IaC, jeśli provider obsługuje wymagane elementy, albo bootstrap/API/Ansible, jeśli któregoś elementu w providerze brakuje.

Nie Tufin.

B. User-ID

Czyli:

Jan Kowalski
       ↓
IP 10.20.1.55
       ↓
AD group Finance
       ↓
Palo User-ID

i:

source-user = DOMAIN\Finance
destination = SAP

To jest drugi rodzaj integracji.

Tutaj potrzebujesz mechanizmu:

AD / Entra
     ↓
identity mapping
     ↓
Palo User-ID

A sama Security Rule może potem odnosić się do użytkowników/grup.

13. Entra workload identity ≠ User-ID

To też trzeba wyraźnie oddzielić.

Jeżeli AKS używa:

Managed Identity
Workload Identity
Service Principal

to Palo Alto normalnie nie zobaczy:

"to jest Managed Identity X"

w pakiecie IP.

Firewall zobaczy:

src IP
dst IP
port
application

Więc jeśli chcecie robić:

AKS workload identity X
    →
SQL

potrzebujecie dodatkowej warstwy mapowania:

workload metadata
       ↓
IP / tags
       ↓
DAG

a nie klasycznego User-ID.

14. AKS

AKS zdecydowanie wyciągnąłbym jako osobny temat.

Macie kilka poziomów granularności:

Cluster
Node pool
Node
Pod subnet
Namespace
Workload
Pod
Service
Ingress

I wybór zależy od CNI.

Najprostsze
AKS subnet
    →
destination

Tufin świetnie.

Lepsze
AKS-PROD
    →
destination

DAG.

Bardzo granularne
namespace=payments
app=backend
env=prod
    →
SQL-PROD

Tu potrzebujecie:

AKS API / Azure
     ↓
automation
     ↓
PAN tags
     ↓
DAG

I to jest już osobny PoC.

15. Service Tags

Tak samo.

Nie trzymałbym w Terraformie:

AzureCloud = 9 000 prefixów

bo wtedy każdy Microsoft update daje ogromny Terraform diff.

Lepszy model:

Terraform
    ↓
tworzy logiczny obiekt
AZ-SERVICETAG-STORAGE-WEU

automation
    ↓
synchronizuje membership

Tu rozważyłbym:

DAG,
EDL,
address group,
automatyczny feed.

Który mechanizm najlepszy — to warto osobno przetestować.

16. FQDN

Statyczny FQDN:

api.company.com

spokojnie Terraform.

Dynamiczne listy Microsoft typu:

*.azurecr.io
*.blob.core.windows.net
...

już raczej:

feed / EDL / automation

niż setki ręcznie tworzonych Terraform resources.

17. Gdzie w tym wszystkim Tufin?

I tu robi się najważniejsze rozdzielenie.

Tufin powinien być systemem realizacji change requestów dotyczących security policy, a nie provisioning engine'em całej platformy.

Czyli:

Tufin TAK
source
destination
service/application
action
security rule

add
modify
remove

workflow
approval
risk analysis
recertification
Tufin RACZEJ NIE
tworzenie VM-Series
NIC
VNet
Subnet
BGP
IPSec
Virtual Router
Ethernet interfaces
Template Stack
Panorama onboarding
Azure Service Principal
Azure Plugin
DAG synchronization

Te rzeczy należą do platform engineering / IaC.

18. Moim zdaniem najczystszy model dla was wygląda tak
                     GIT
                      │
        ┌─────────────┴──────────────┐
        │                            │
     AZURE IaC                   PANOS IaC
        │                            │
        │                            │
    AzureRM                     PANOS Provider
        │                            │
        ▼                            ▼
 Azure Resources               Panorama
 ─────────────                 ──────────────
 VM-Series                     Templates
 NIC                           Template Stack
 VNet                          Interfaces
 Subnet                        Zones
 UDR                           Virtual Router
 LB                            BGP
 PIP                           IPSec
 NAT GW                        Static objects
 Identity                      DAG definitions
                               baseline rules

        │                            │
        └─────────────┬──────────────┘
                      │
                      ▼
               DYNAMIC LAYER
               ─────────────
               Azure Plugin
               Azure API
               Resource Graph
               AKS API
               scripts/functions
                      │
                      ▼
                 IP ↔ TAG
                      │
                      ▼
                     DAG

                      ▲
                      │
                    TUFIn
              ─────────────────
              Application rules
              Change workflow
              Approval
              Risk analysis

To jest według mnie właściwy docelowy obraz.

19. I bardzo ważne: Panorama Template vs Device Group

To wam pomoże uporządkować repo.

Ja zastosowałbym naturalny podział Palo Alto:

Template / Template Stack

network/device/platform configuration

Interfaces
Zones
Virtual Router
BGP
IPSec
DNS
NTP
logging settings
User-ID config
Panorama integration
Device Group

objects + policies

Address Objects
Address Groups
DAG
Services
Service Groups

Security Policy
NAT Policy
profiles

To bardzo dobrze mapuje się na Git:

panos/
├── templates/
│   ├── interfaces/
│   ├── zones/
│   ├── routing/
│   ├── bgp/
│   ├── ipsec/
│   └── identity/
│
├── device-groups/
│   ├── objects/
│   ├── dynamic-groups/
│   ├── services/
│   └── baseline-policy/
│
└── integrations/
    ├── azure/
    ├── entra/
    └── aks/
20. Source of Truth — warto sobie zrobić taką tabelę
Element	Source of Truth	Narzędzie
VM-Series VM	Git	Azure Terraform
NIC	Git	Azure Terraform
VNet	Git	Azure Terraform
Subnet	Git	Azure Terraform
UDR	Git	Azure Terraform
LB	Git	Azure Terraform
Panorama Template	Git	PANOS Terraform
Ethernet Interface	Git	PANOS Terraform
Zone	Git	PANOS Terraform
Virtual Router	Git	PANOS Terraform
BGP	Git	PANOS Terraform
IPSec	Git	PANOS Terraform
Static address objects	Git	PANOS Terraform
Static groups	Git	PANOS Terraform
DAG definition	Git	PANOS Terraform
DAG membership	Azure/runtime	Azure Plugin/API
Azure Resource Tags	Azure	Azure
AKS metadata	Kubernetes	AKS/API integration
baseline security rules	Git	PANOS Terraform
application changes	Tufin	Tufin
emergency rules	Tufin / controlled process	Tufin
rule approval	Tufin	Tufin

I wtedy praktycznie znika pytanie:

„kto może to zmieniać?”

Bo każdy typ obiektu ma właściciela.

Jedna rzecz jeszcze bym rozważył: commit/deploy

Nowy provider PAN-OS ma już również akcję commit, w tym możliwość commitowania konkretnych Device Groups na Panoramie.

Czyli pipeline może być:

PR
 ↓
terraform plan
 ↓
review
 ↓
merge
 ↓
terraform apply
 ↓
Panorama commit
 ↓
push/commit-all do FW
 ↓
test

I wtedy rzeczywiście możecie mówić, że platformowa konfiguracja Palo Alto jest zarządzana jako kod, a nie że Terraform tylko wrzuca candidate config i ktoś potem ręcznie klika commit.

Podsumowując, ja bym wam ten cały projekt rozbił na 5 dużych strumieni, które potem można osobno PoC-ować:

Azure infrastructure IaC – VM, NIC, subnety, UDR, LB itd.
PAN-OS platform IaC – interfejsy, zone, VR, BGP, VPN, Panorama templates.
PAN-OS objects IaC – statyczne obiekty, grupy, DAG definitions, services.
Dynamic integrations – Azure tags, Service Tags, AKS, Entra/AD, IP-tag registration.
Security change management – Tufin jako główny mechanizm otwierania ruchu.

I moim zdaniem właśnie strumień 4 jest najbardziej interesujący do zbadania, bo 1, 2 i 3 są technicznie dość klasyczne, a 5 to Tufin. Największa wartość będzie w odpowiedzi na pytanie: jak automatycznie przekształcać rzeczy typu Azure tag / AKS workload / Service Tag / identity w stabilne obiekty Palo Alto, które potem Tufin może wykorzystywać w regułach.
