---
title: Управління трафіком
description: Описує різні функції Istio, орієнтовані на маршрутизацію та контроль трафіку.
weight: 20
keywords: [traffic-management, pilot, envoy-proxies, service-discovery, load-balancing]
aliases:
    - /uk/docs/concepts/traffic-management/pilot
    - /uk/docs/concepts/traffic-management/rules-configuration
    - /uk/docs/concepts/traffic-management/fault-injection
    - /uk/docs/concepts/traffic-management/handling-failures
    - /uk/docs/concepts/traffic-management/load-balancing
    - /uk/docs/concepts/traffic-management/request-routing
    - /uk/docs/concepts/traffic-management/pilot.html
    - /uk/docs/concepts/traffic-management/overview.html
owner: istio/wg-networking-maintainers
test: n/a
---

Правила маршрутизації трафіку Istio дозволяють легко контролювати потік трафіку та API-викликів між сервісами. Istio спрощує конфігурацію властивостей на рівні сервісів, таких як запобіжники, тайм-аути та повторні спроби, і полегшує налаштування важливих завдань, таких як A/B тестування, канарейкові розгортання та поетапні розгортання з розподілом трафіку за відсотками. Також забезпечуються вбудовані функції надійності, які допомагають зробити ваш застосунок більш стійким до збоїв залежних сервісів або мережі.

Модель керування трафіком Istio базується на {{< gloss "envoy">}}проксі Envoy{{</ gloss >}}, які розгортаються разом з вашими сервісами. Весь трафік, який ваші сервіси надсилають та отримують ({{< gloss "панель даних">}}трафік панелі даних{{</ gloss >}}), проходить через проксі Envoy, що полегшує спрямування та контроль трафіку в межах вашої mesh-мережі без внесення змін у ваші сервіси.

Якщо вас цікавлять деталі роботи функцій, описаних у цьому посібнику, ви можете дізнатися більше про реалізацію управління трафіком Istio в [огляді архітектури](/docs/ops/deployment/architecture/). Решта цього посібника знайомить з функціями керування трафіком Istio.

## Впровадження керування трафіком в Istio {#introducing-isito-traffic-management}

Для того, щоб спрямовувати трафік у вашій мережі, Istio має знати, де знаходяться всі ваші точки доступу та до яких сервісів вони належать. Щоб заповнити власний {{< gloss >}}реєстр сервісів{{</ gloss >}}, Istio підключається до системи виявлення сервісів. Наприклад, якщо ви встановили Istio у кластер Kubernetes, Istio автоматично виявляє сервіси та точки доступу в цьому кластері.

Використовуючи цей реєстр сервісів, проксі Envoy можуть спрямовувати трафік до відповідних сервісів. Більшість мікросервісних застосунків мають кілька екземплярів кожного робочого навантаження сервісу для обробки трафіку сервісу, що іноді називається пулом балансування навантаження. Стандартно проксі Envoy розподіляють трафік між кожним пулом балансування навантаження сервісу за допомогою моделі найменшого числа запитів, коли кожен запит спрямовується до хосту з меншою кількістю активних запитів з випадкового вибору двох хостів із пулу; таким чином, найбільш завантажений хост не отримуватиме запити, доки він не буде завантажений не більше, ніж будь-який інший хост.

Хоча базове виявлення сервісів і балансування навантаження в Istio дає вам працюючу сервісну мережу, це далеко не все, що може зробити Istio. У багатьох випадках вам може знадобитися більш точний контроль над тим, що відбувається з вашим трафіком у мережі. Ви можете захотіти спрямовувати певний відсоток трафіку на нову версію сервісу в рамках A/B тестування або застосувати іншу політику балансування навантаження до трафіку для певної підмножини екземплярів сервісу. Також може виникнути потреба застосувати спеціальні правила до трафіку, що надходить у вашу мережу або виходить із неї, або додати зовнішню залежність вашої мережі до реєстру сервісів. Ви можете зробити все це і більше, додавши власну конфігурацію трафіку в Istio, використовуючи API керування трафіком Istio.

Як і інша конфігурація Istio, API визначається за допомогою визначень власних ресурсів Kubernetes ({{< gloss >}}CRD{{</ gloss >}}), які ви можете налаштовувати за допомогою YAML, як показано в прикладах.

У решті цього посібника розглядаються кожен із ресурсів API керування трафіком і те, що ви можете з ними робити. До цих ресурсів належать:

- [Віртуальні сервіси](#virtual-services)
- [Правила призначення](#destination-rules)
- [Шлюзи](#gateways)
- [Входи сервісів](#service-entries)
- [Sidecars](#sidecars)

Цей посібник також надає огляд деяких [функцій мережевої стійкості та тестування](#network-resilience-and-testing), які вбудовані в ресурси API.

## Віртуальні сервіси {#virtual-services}

[Віртуальні сервіси](/docs/reference/config/networking/virtual-service/#VirtualService), разом із [правилами призначення](#destination-rules), є ключовими будівельними блоками функціональності маршрутизації трафіку в Istio. Віртуальний сервіс дозволяє вам налаштувати маршрутизацію запитів до сервісу в сервісній мережі Istio, спираючись на базові можливості підключення та виявлення, що надаються Istio та вашою платформою. Кожен віртуальний сервіс складається з набору правил маршрутизації, які оцінюються послідовно, дозволяючи Istio зіставляти кожен запит до віртуального сервісу з конкретним реальним призначенням у межах мережі. Ваша мережа може вимагати кілька віртуальних сервісів або не вимагати жодного, залежно від вашого випадку використання.

### Навіщо користуватися віртуальними сервісами? {#why-use-virtual-services}

Віртуальні сервіси відіграють ключову роль у забезпеченні гнучкості та потужності керування трафіком в Istio. Вони досягають цього, чітко розділяючи місце, куди клієнти надсилають свої запити, від робочих навантажень, які фактично їх обробляють. Віртуальні сервіси також надають широкий набір способів визначення різних правил маршрутизації трафіку для надсилання трафіку до цих навантажень.

Чому це так корисно? Без віртуальних сервісів Envoy розподіляє трафік за допомогою балансування навантаження з найменшою кількістю запитів між усіма екземплярами сервісів, як описано у вступі. Ви можете покращити цю поведінку, використовуючи наявну інформацію про робочі навантаження. Наприклад, деякі з них можуть представляти іншу версію. Це може бути корисним при A/B тестуванні, коли ви хочете налаштувати маршрути трафіку на основі відсотків між різними версіями сервісів або спрямувати трафік від внутрішніх користувачів до певного набору екземплярів.

З віртуальним сервісом ви можете задати поведінку трафіку для одного або кількох імен хостів. Ви використовуєте правила маршрутизації у віртуальному сервісі, які вказують Envoy, як надсилати трафік віртуального сервісу до відповідних пунктів призначення. Пункти призначення маршруту можуть бути різними версіями одного і того ж сервісу або зовсім іншими сервісами.

Типовий випадок використання — це спрямування трафіку до різних версій сервісу, які визначаються як підмножини сервісу. Клієнти надсилають запити до хосту віртуального сервісу так, ніби це єдиний субʼєкт, а Envoy потім маршрутизує трафік до різних версій залежно від правил віртуального сервісу: наприклад, "20% викликів ідуть до нової версії" або "виклики від цих користувачів ідуть до версії 2". Це дозволяє, наприклад, створити канарейковий реліз, у якому ви поступово збільшуєте відсоток трафіку, який надсилається до нової версії сервісу. Маршрутизація трафіку повністю відокремлена від розгортання екземплярів, що означає, що кількість екземплярів, які реалізують нову версію сервісу, може збільшуватися або зменшуватися залежно від навантаження трафіку без будь-якого посилання на маршрутизацію трафіку. Для порівняння, платформи оркестрування контейнерів, такі як Kubernetes, підтримують розподіл трафіку лише на основі масштабування екземплярів, що швидко ускладнюється. Ви можете дізнатися більше про те, як віртуальні сервіси допомагають із канарейковими релізами, у статті [Канарейкові релізи з використанням Istio](/blog/2017/0.1-canary/).

Віртуальні сервіси також дозволяють:

- Звертатися до кількох сервісів застосунків через один віртуальний сервіс. Наприклад, якщо ваша мережа використовує Kubernetes, ви можете налаштувати віртуальний сервіс для обробки всіх сервісів у певному просторі імен. Призначення одного віртуального сервісу для кількох "реальних" сервісів є особливо корисним при перетворенні монолітного застосунку на зіставний сервіс, побудований з окремих мікросервісів, без необхідності адаптувати споживачів сервісу до цього переходу. Ваші правила маршрутизації можуть вказувати "виклики до цих URI `monolith.com` йдуть до `microservice A`" тощо. Ви можете побачити, як це працює, в [одному з наших прикладів нижче](#more-about-routing-rules).
- Налаштовувати правила трафіку у поєднанні зі [шлюзами](/docs/concepts/traffic-management/#gateways) для контролю вхідного та вихідного трафіку.

У деяких випадках вам також потрібно налаштувати правила призначення для використання цих функцій, оскільки саме там ви визначаєте підмножини сервісу. Визначення підмножин сервісу та інших політик, специфічних для призначення, в окремому обʼєкті дозволяє чисто повторно використовувати їх між віртуальними сервісами. Ви можете дізнатися більше про правила призначення в наступному розділі.

### Приклад віртуального сервісу {#virtual-service-example}

Наведений нижче віртуальний сервіс маршрутизує запити до різних версій сервісу залежно від того, чи надходить запит від певного користувача.

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
{{< /text >}}

#### Поле hosts {#the-hosts-field}

Поле `hosts` містить список хостів віртуального сервісу — іншими словами, адресовані користувачеві призначення або призначення, до яких застосовуються ці правила маршрутизації. Це адреса або адреси, які клієнт використовує під час надсилання запитів до сервісу.

{{< text yaml >}}
hosts:
- reviews
{{< /text >}}

Імʼя хосту віртуального сервісу може бути IP-адресою, DNS-імʼям або, залежно від платформи, коротким іменем (наприклад, коротким іменем сервісу Kubernetes), яке розвʼязується неявно або явно до повного доменного імені (FQDN). Ви також можете використовувати префікси із символом підставлення ("*"), що дозволяє створити єдиний набір правил маршрутизації для всіх відповідних сервісів. Хости віртуального сервісу не обовʼязково мають бути частиною реєстру сервісів Istio, вони є просто віртуальними призначеннями. Це дозволяє моделювати трафік для віртуальних хостів, які не мають записів маршрутів у межах мережі.

#### Правила маршрутизації {#routing-rules}

Розділ `http` містить правила маршрутизації віртуального сервісу, які описують умови відповідності та дії для маршрутизації трафіку HTTP/1.1, HTTP2 і gRPC, надісланого до призначення(нь), зазначених у полі hosts (ви також можете використовувати розділи `tcp` і `tls` для налаштування правил маршрутизації для трафіку [TCP](/docs/reference/config/networking/virtual-service/#TCPRoute) та нетермінованого [TLS](/docs/reference/config/networking/virtual-service/#TLSRoute)). Правило маршрутизації складається з призначення, куди ви хочете направити трафік, і нуля або більше умов відповідності, залежно від вашого випадку використання.

##### Умова відповідності {#match-condition}

Перше правило маршрутизації в прикладі має умову, тому починається з поля `match`. У цьому випадку ви хочете, щоб це правило застосовувалося до всіх запитів від користувача "jason", тому використовуєте поля `headers`, `end-user` і `exact`, щоб вибрати відповідні запити.

{{< text yaml >}}
- match:
   - headers:
       end-user:
         exact: jason
{{< /text >}}

##### Призначення {#destination}

Поле `destination` у розділі маршруту вказує фактичне призначення для трафіку, який відповідає цій умові. На відміну від хостів віртуального сервісу, хост призначення має бути реальним призначенням, яке існує в реєстрі сервісів Istio, інакше Envoy не знатиме, куди направити трафік. Це може бути сервіс у межах мережі з проксі-серверами або сервіс поза мережею, доданий за допомогою запису сервісу. У цьому випадку ми працюємо в Kubernetes, і імʼя хосту є іменем сервісу в Kubernetes:

{{< text yaml >}}
route:
- destination:
    host: reviews
    subset: v2
{{< /text >}}

Зверніть увагу, що в цьому та інших прикладах на цій сторінці ми використовуємо коротке імʼя Kubernetes для хостів призначення для спрощення. Коли це правило оцінюється, Istio додає суфікс домену на основі простору імен віртуального сервісу, що містить правило маршрутизації, щоб отримати повне доменне імʼя для хосту. Використання коротких імен у наших прикладах також означає, що ви можете скопіювати та спробувати їх у будь-якому просторі імен, який вам подобається.

{{< warning >}}
Використання коротких імен таким чином працює лише в тому випадку, якщо хости призначення та віртуальний сервіс фактично знаходяться в одному просторі імен Kubernetes. Оскільки використання короткого імені Kubernetes може призвести до помилкових конфігурацій, ми рекомендуємо вказувати повністю кваліфіковані імена хостів в операційних середовищах.
{{< /warning >}}

У розділі призначення також вказується, до якої підмножини цього сервісу Kubernetes ви хочете, щоб запити, які відповідають умовам цього правила, надходили; у цьому випадку підмножина з назвою v2. Ви дізнаєтесь, як визначити підмножину сервісу в розділі про [правила призначення](#destination-rules) нижче.

#### Пріоритет правил маршрутизації {#routing-rule-precedence}

Правила маршрутизації **оцінюються послідовно зверху вниз**, причому перше правило у визначенні віртуального сервісу має найвищий пріоритет. У цьому випадку ви хочете, щоб усе, що не відповідає першому правилу маршрутизації, спрямовувалося до стандартного місця призначення, яке визначено в другому правилі. Через це друге правило не має умов відповідності й просто направляє трафік до підмножини v3.

{{< text yaml >}}
- route:
  - destination:
      host: reviews
      subset: v3
{{< /text >}}

Ми рекомендуємо вказувати стандартне правило "без умов" або на основі ваги (дивись нижче) як останнє правило в кожному віртуальному сервісі, щоб забезпечити наявність хоча б одного маршруту, що відповідає трафіку до віртуального сервісу.

### Більше про правила маршрутизації {#more-about-routing-rules}

Як ви бачили вище, правила маршрутизації — це потужний інструмент для маршрутизації певних підмножин трафіку до конкретних призначень. Ви можете встановлювати умови відповідності для портів трафіку, заголовків, URI тощо. Наприклад, цей віртуальний сервіс дозволяє користувачам надсилати трафік до двох окремих сервісів, `ratings` та `reviews`, так, ніби вони є частиною більшого віртуального сервісу за адресою `http://bookinfo.com/`. Правила віртуального сервісу зіставляють трафік на основі URI запиту та спрямовують запити до відповідного сервісу.

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
{{< /text >}}

Для деяких умов збігу ви також можете вибрати їх за точним значенням, префіксом або регулярним виразом.

Ви можете додати кілька умов відповідності до одного блоку `match`, щоб поєднати їх за допомогою оператора AND, або додати кілька блоків відповідності до одного правила, щоб поєднати їх за допомогою оператора OR. Ви також можете мати кілька правил маршрутизації для будь-якого віртуального сервісу. Це дозволяє зробити умови маршрутизації настільки складними або простими, як ви хочете, у межах одного віртуального сервісу. Повний список полів умов відповідності та їх можливих значень можна знайти в довідці [`HTTPMatchRequest`](/docs/reference/config/networking/virtual-service/#HTTPMatchRequest).

Крім використання умов відповідності, ви можете розподіляти трафік за відсотковою "вагою". Це корисно для A/B тестування та канаркових розгортань:

{{< text yaml >}}
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
{{< /text >}}

Ви також можете використовувати правила маршрутизації для виконання певних дій з трафіком, наприклад:

- Додавання або видалення заголовків.
- Переписування URL.
- Встановлення [політики повторних спроб](#retries) для викликів до цього призначення.

Щоб дізнатися більше про доступні дії, дивіться довідку [`HTTPRoute`](/docs/reference/config/networking/virtual-service/#HTTPRoute).

## Правила призначення {#destination-rules}

Разом із [віртуальними сервісами](#virtual-services) правила призначення (англ. [destination rules](/docs/reference/config/networking/destination-rule/#DestinationRule)) є ключовою частиною функціональності маршрутизації трафіку в Istio. Ви можете розглядати віртуальні сервіси як спосіб маршрутизації вашого трафіку **до** певного призначення, а правила призначення використовувати для налаштування того, що відбувається з трафіком **для** цього призначення. Правила призначення застосовуються після того, як будуть оцінені правила маршрутизації віртуальних сервісів, тому вони стосуються "реального" призначення трафіку.

Зокрема, ви використовуєте правила призначення для вказівки іменованих підмножин сервісів, таких як групування всіх екземплярів даного сервісу за версіями. Потім ви можете використовувати ці підмножини сервісів у правилах маршрутизації віртуальних сервісів для контролю трафіку до різних екземплярів ваших сервісів.

Правила призначення також дозволяють налаштовувати політики трафіку Envoy під час виклику всього призначеного сервісу або певної підмножини сервісів, таких як ваш улюблений метод балансування навантаження, режим безпеки TLS або налаштування автоматичного вимикача. Повний список опцій правил призначення можна знайти в довідці [Destination Rule reference](/docs/reference/config/networking/destination-rule/).

### Опції балансування навантаження {#load-balancing-options}

Стандартно Istio використовує політику балансування навантаження за принципом найменшої кількості запитів, коли запити розподіляються серед екземплярів з найменшою кількістю запитів. Istio також підтримує такі моделі, які можна вказати в правилах призначення для запитів до конкретного сервісу або підмножини сервісів:

- Випадковий (Random): Запити пересилаються випадковим чином до екземплярів у пулі.
- Зважений (Weighted): Запити пересилаються до екземплярів у пулі відповідно до певного відсотка.
- Кільцевий (Round robin): Запити пересилаються кожному екземпляру послідовно.

Докладніше про кожен варіант можна дізнатися в [документації з балансування навантаження Envoy](https://www.envoyproxy.io/docs/envoy/v1.5.0/intro/arch_overview/load_balancing).

### Приклад правила призначення {#destination-rule-example}

Наступний приклад правила призначення налаштовує три різні підмножини для сервісу `my-svc`, використовуючи різні політики балансування навантаження:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
{{< /text >}}

Кожна підмножина визначається на основі одного або декількох `labels` (міток), які в Kubernetes є парами "ключ/значення", що привʼязуються до обʼєктів, таких як Podʼи. Ці мітки застосовуються під час розгортання сервісу в Kubernetes як `metadata`, щоб ідентифікувати різні версії.

Крім визначення підмножин, це правило призначення має як загальну політику трафіку для всіх підмножин цього сервісу, так і специфічну політику для окремої підмножини, яка перевизначає загальну. Загальна політика, визначена над полем `subsets`, встановлює простий випадковий балансувальник навантаження для підмножин `v1` і `v3`. У політиці для `v2` зазначено балансувальник навантаження з циклічним алгоритмом, що вказано у відповідному полі підмножини.

## Шлюзи {#gateways}

Ви використовуєте [gateway](/docs/reference/config/networking/gateway/#Gateway) для управління вхідним та вихідним трафіком у вашій mesh-системі, що дозволяє вам вказати, який трафік ви хочете допустити або випустити з mesh. Конфігурації шлюзів застосовуються до автономних проксі Envoy, які працюють на межі mesh, а не до sidecar проксі Envoy, що працюють поруч із вашими сервісними навантаженнями.

На відміну від інших механізмів контролю трафіку, який входить у вашу систему, таких як API Kubernetes Ingress, шлюзи Istio дозволяють використовувати повну потужність та гнучкість маршрутизації трафіку Istio. Це можливо, тому що ресурс шлюзу Istio дозволяє налаштувати властивості балансування навантаження на рівнях 4-6, такі як порти для експозиції, налаштування TLS і так далі. Замість того щоб додавати маршрутизацію трафіку на рівні застосунків (L7) до того ж ресурсу API, ви звʼязуєте звичайний Istio [віртуальний сервіс](#virtual-services) зі шлюзом. Це дозволяє вам керувати трафіком через шлюз так само, як і будь-яким іншим трафіком у mesh-системі Istio.

Шлюзи в основному використовуються для управління вхідним трафіком, але ви також можете налаштувати шлюзи для вихідного трафіку. Шлюз для вихідного трафіку дозволяє налаштувати спеціальний вузол виходу для трафіку, що залишає mesh, дозволяючи вам обмежити, які сервіси можуть або повинні мати доступ до зовнішніх мереж, або забезпечити [безпечний контроль вихідного трафіку](/blog/2019/egress-traffic-control-in-istio-part-1/), наприклад, для додавання безпеки до вашої mesh-системи. Ви також можете використовувати шлюз для налаштування виключно внутрішнього проксі.

Istio надає кілька попередньо налаштованих розгортань шлюзових проксі (`istio-ingressgateway` і `istio-egressgateway`), які ви можете використовувати — обидва вони розгортаються, якщо ви використовуєте наш [демонстраційний інсталяційний профіль](/docs/setup/getting-started/), тоді як лише шлюз для вхідного трафіку розгортається з нашим [типовим профілем](/docs/setup/additional-setup/config-profiles/). Ви можете застосувати свої власні конфігурації шлюзу до цих розгортань або розгорнути та налаштувати свої власні шлюзові проксі.

### Приклад конфігурації шлюзу {#gateway-example}

Наведений нижче приклад показує можливу конфігурацію шлюзу для зовнішнього трафіку HTTPS:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      credentialName: ext-host-cert
{{< /text >}}

Ця конфігурація шлюзу дозволяє трафіку HTTPS з `ext-host.example.com` увійти до mesh-системи через порт 443, але не вказує жодних правил маршрутизації для цього трафіку.

Щоб вказати маршрутизацію і щоб шлюз працював належним чином, потрібно також зв’язати шлюз із віртуальним сервісом. Це робиться за допомогою поля `gateways` у віртуальному сервісі, як показано в наступному прикладі:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:
  - ext-host-gwy
{{< /text >}}

Після цього ви можете налаштувати віртуальний сервіс з правилами маршрутизації для зовнішнього трафіку.

## Записи сервісів {#service-entries}

Ви використовуєте [запис сервісу (service entry)](https://istio.io/docs/reference/config/networking/service-entry/#ServiceEntry), щоб додати запис до реєстру сервісів, який Istio підтримує внутрішньо. Після додавання запису сервісу, проксі Envoy можуть надсилати трафік до сервісу, наче він є частиною вашого mesh. Конфігурація записів сервісів дозволяє керувати трафіком для сервісів, що працюють поза межами mesh, включаючи такі завдання:

- Перенаправлення та пересилання трафіку на зовнішні призначення, такі як API, споживані з вебу, або трафік до сервісів у застарілій інфраструктурі.
- Визначення політик [повторних спроб](#retries), [таймаутів](#timeouts) і [інʼєкції збоїв](#fault-injection) для зовнішніх призначень.
- Запуск mesh-сервісу у віртуальній машині (VM) шляхом [додавання VM до вашого mesh](https://istio.io/docs/examples/virtual-machines/).

Вам не потрібно додавати запис сервісу для кожного зовнішнього сервісу, який ви хочете використовувати у своїх mesh-сервісах. Стандартно Istio налаштовує проксі Envoy для пропуску запитів до невідомих сервісів. Однак ви не можете використовувати функції Istio для керування трафіком до призначень, які не зареєстровані в mesh.

### Приклад запису сервісу {#service-entry-example}

Наступний приклад запису сервісу для зовнішнього ресурсу додає зовнішню залежність `ext-svc.example.com` до реєстру сервісів Istio:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
{{< /text >}}

Ви вказуєте зовнішній ресурс за допомогою поля `hosts`. Ви можете вказати його повністю або використовувати доменне імʼя з префіксом wildcard.

Ви можете налаштувати віртуальні сервіси та правила призначень для контролю трафіку до запису сервісу більш детально, так само як ви налаштовуєте трафік для будь-якого іншого сервісу в mesh. Наприклад, наступне правило призначення регулює таймаут TCP-зʼєднання для запитів до зовнішнього сервісу `ext-svc.example.com`, який ми налаштували за допомогою запису сервісу:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: ext-res-dr
spec:
  host: ext-svc.example.com
  trafficPolicy:
    connectionPool:
      tcp:
        connectTimeout: 1s
{{< /text >}}

Дивіться [посилання на документацію Service Entry](https://istio.io/docs/reference/config/networking/service-entry) для додаткових можливостей конфігурації.

### Sidecars {#sidecars}

Стандартно Istio налаштовує кожен проксі Envoy так, щоб він приймав трафік на всіх портах повʼязаного робочого навантаження і міг досягти кожного робочого навантаження в mesh при пересиланні трафіку. Ви можете використовувати конфігурацію [sidecar](https://istio.io/docs/reference/config/networking/sidecar/) для наступного:

- Тонкої настройки набору портів і протоколів, які проксі Envoy приймає.
- Обмеження набору сервісів, до яких проксі Envoy може дістатися.

Вам може знадобитися обмежити досяжність sidecars у більших застосунках, де налаштування кожного проксі для досягнення кожного іншого сервісу в mesh може потенційно вплинути на продуктивність mesh через значне використання памʼяті.

Ви можете вказати, що ви хочете, щоб конфігурація sidecar застосовувалася до всіх робочих навантажень у певному просторі імен або вибрати конкретні робочі навантаження за допомогою `workloadSelector`. Наприклад, наступна конфігурація sidecar налаштовує всі сервіси в просторі імен `bookinfo`, щоб вони могли досягати лише сервісів, які працюють у тому ж просторі імен, і панелі управління Istio (необхідно для функцій egress та телеметрії Istio):

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: bookinfo
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
{{< /text >}}

Дивіться [посилання на документацію Sidecar](https://istio.io/docs/reference/config/networking/sidecar/) для більш детальної інформації.

## Стійкість мережі та тестування {#network-resilience-and-testing}

Окрім допомоги в напрямку трафіку по вашій мережі, Istio надає функції відновлення після збоїв та інʼєкції помилок, які ви можете налаштувати динамічно в режимі реального часу. Використання цих функцій допомагає вашим застосункам працювати надійно, забезпечуючи, щоб сервісна мережа могла витримувати несправності вузлів і запобігати локальним збоям, які можуть поширюватися на інші вузли.

### Тайм-аути {#timeouts}

Тайм-аут — це проміжок часу, протягом якого проксі Envoy має чекати відповіді від певного сервісу, щоб гарантувати, що сервіси не залишаються в очікуванні відповідей нескінченно і що виклики успішно закінчуються або зазнають невдачи в передбачувані терміни. Тайм-аут для HTTP-запитів стандартно вимкнено в Istio.

Для деяких застосунків та сервісів стандартний тайм-аут в Istio може бути не підходящим. Наприклад, занадто довгий тайм-аут може призвести до надмірної затримки через очікування відповідей від збоїв сервісів, тоді як занадто короткий тайм-аут може призвести до невиправданих помилок при очікуванні завершення операції, яка залучає кілька сервісів. Щоб знайти та використовувати оптимальні налаштування тайм-аутів, Istio дозволяє легко налаштовувати тайм-аути динамічно для кожного сервісу за допомогою [віртуальних сервісів](#virtual-services), без необхідності редагувати код вашого сервісу. Ось приклад віртуального сервісу, який задає тайм-аут у 10 секунд для викликів до підмножини v1 сервісу оцінок:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
{{< /text >}}

### Повторні спроби {#retries}

Налаштування повторних спроб визначає максимальну кількість разів, коли проксі Envoy намагається підключитися до сервісу, якщо початковий виклик не вдався. Повторні спроби можуть підвищити доступність сервісу та продуктивність застосунку, забезпечуючи, щоб виклики не зазнавали постійних невдач через тимчасові проблеми, такі як тимчасове перевантаження сервісу або проблеми з мережею. Інтервал між повторними спробами (25 мс і більше) є змінним і визначається автоматично Istio, щоб запобігти перевантаженню сервісу запитами. Стандартно, поведінка повторних спроб для HTTP-запитів полягає в повторенні спроб двічі перед поверненням помилки.

Як і у випадку з тайм-аутами, стандартна поведінка повторних спроб в Istio може не відповідати вашим потребам у термінах затримки (надто багато повторних спроб до невдалого сервісу може уповільнити процес) або доступності. Так само, як і тайм-аути, ви можете налаштувати повторні спроби для кожного сервісу у [віртуальних сервісах](#virtual-services), не торкаючись коду вашого сервісу. Ви також можете додатково уточнити поведінку повторних спроб, додавши тайм-аути на кожну спробу, зазначаючи час, який ви хочете почекати для кожної спроби успішно підключитися до сервісу. Ось приклад, який налаштовує максимум 3 повторні спроби для підключення до цієї підмножини сервісу після невдачі початкового виклику, кожна з тайм-аутом у 2 секунди.

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 3
      perTryTimeout: 2s
{{< /text >}}

### Переривачі ланцюгів (запобіжники) {#circuit-breakers}

Переривачі ланцюгів є ще одним корисним механізмом, який Istio надає для створення стійких застосунків на основі мікросервісів. У переривачі ланцюгів ви встановлюєте обмеження для викликів до окремих хостів у межах сервісу, такі як кількість одночасних підключень або кількість невдалих спроб викликів до цього хоста. Як тільки це обмеження досягається, запобіжник "спрацьовує" і зупиняє подальші підключення до цього хоста. Використання паттерну переривача ланцюга дозволяє швидко реагувати на збої, замість того щоб клієнти намагалися підключитися до перевантаженого або несправного хосту.

Оскільки переривання ланцюгів застосовується до "реальних" призначень у мережі в пулі навантаження, ви налаштовуєте пороги переривання ланцюгів у [правилах призначень](#destination-rules), і ці налаштування застосовуються до кожного окремого хоста в сервісі. Ось приклад, який обмежує кількість одночасних підключень для сервісів `reviews` у підмножині v1 до 100:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 100
{{< /text >}}

Детальніше про створення запобіжників можна дізнатися в розділі [переривання ланцюгів](/docs/tasks/traffic-management/circuit-breaking/).

### Інʼєкція збоїв {#fault-injection}

Після налаштування мережі, включаючи політики відновлення після збоїв, ви можете використовувати механізми інʼєкції збоїв Istio для перевірки здатності вашого застосунку до відновлення після збоїв. Інʼєкція збоїв — це метод тестування, який вводить збої в систему, щоб перевірити, чи вона може витримати і відновитися після таких станів. Використання інʼєкції збоїв може бути особливо корисним для перевірки сумісності ваших політик відновлення після збоїв та їхньої ефективності, запобігаючи критичним неполадкам у важливих сервісах.

{{< warning >}}
Наразі конфігурація інʼєкції збоїв не може бути поєднана з конфігурацією повторних спроб або тайм-аутів в одному віртуальному сервісі, див. [Проблеми управління трафіком](https://istio.io/latest/docs/ops/common-problems/network-issues/#virtual-service-with-fault-injection-and-retrytimeout-policies-not-working-as-expected).
{{< /warning >}}

На відміну від інших механізмів введення збоїв, таких як затримка пакетів або завершення роботи Podʼів на рівні мережі, Istio дозволяє вводити збої на рівні застосунків. Це дозволяє ввести більш релевантні збої, такі як HTTP-коди помилок, щоб отримати більш точні результати.

Ви можете вводити два типи збоїв, обидва сконфігуровані за допомогою [віртуального сервісу](#virtual-services):

- Затримки: Затримки є помилками часу. Вони імітують збільшену мережеву затримку або перевантажений висхідного сервісу.
- Переривання: Переривання є помилками збоїв. Вони імітують збої у висхідних сервісах. Переривання зазвичай проявляються у вигляді HTTP-кодів помилок або помилок TCP-зʼєднань.

Наприклад, цей віртуальний сервіс вводить затримку в 5 секунд для 1 з кожних 1000 запитів до сервісу `ratings`.

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: ratings
        subset: v1
{{< /text >}}

Для детальних інструкцій про налаштування затримок і переривань див. [Інʼєкція збоїв](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/).

### Робота з вашими застосунками {#working-with-your-applications}

Функції відновлення після збоїв в Istio повністю прозорі для застосунку. Застосунки не знають, чи обробляє збій проксі Envoy для викликаного сервісу перед поверненням відповіді. Це означає, що якщо ви також налаштовуєте політики відновлення після збоїв у коді вашого застосунку, потрібно враховувати, що обидва механізми працюють незалежно один від одного і, отже, можуть конфліктувати. Наприклад, якщо ви можете мати два тайм-аути, один налаштований у віртуальному сервісі, а інший у застосунку. Застосунок встановлює тайм-аут у 2 секунди для API-виклику до сервісу. Проте ви налаштували тайм-аут у 3 секунди з 1 повторною спробою у вашому віртуальному сервісі. У цьому випадку тайм-аут застосунку спрацює першим, тому тайм-аут і повторна спроба Envoy не матимуть жодного ефекту.

Хоча функції відновлення після збоїв Istio покращують надійність та доступність сервісів у мesh, застосунки повинні обробляти збої або помилки та вживати відповідні резервні заходи. Наприклад, коли всі екземпляри в пулі балансування навантаження зазнали збою, Envoy повертає код `HTTP 503`. Застосунок повинен реалізувати будь-яку резервну логіку, необхідну для обробки коду помилки `HTTP 503`.