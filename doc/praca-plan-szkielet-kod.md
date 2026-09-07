# Plan, szkielet, kod

**Metodyka pracy z agentami AI — propozycja dla zespołu**
Wersja robocza 1.0 · 2026-09-05
Wersja HTML do udostępnienia: https://claude.ai/code/artifact/5a594573-53b9-4d61-ba69-576f1e505bac

| | |
|---|---|
| Zespół | 6 developerów |
| Stack | Java, wiele repozytoriów |
| Agenci AI | 6 / 6 |
| Decyzje architektoniczne | jedna osoba |
| Wejście | Jira / Rally przez MCP (RW) |

---

## 1. Diagnoza

Obecny cykl kończy się średnio **trzema iteracjami poprawek po implementacji**, przed commitem.
Wszystkie dotyczą architektury i podejścia. Konwencje kodu są pokryte skillami, testy są w porządku.

Przyczyna jest mechaniczna:

1. **Plan opisuje zamiar, nie kształt.** Zdanie „dodamy walidację w `OrderService`" jest zgodne
   z pięcioma różnymi architekturami. Architektura mieszka w sygnaturach, granicach pakietów
   i grafie wywołań — a te powstają dopiero w implementacji.
2. **Agent nie pokazuje alternatyw.** Wybiera jedną drogę i opisuje ją tak, jakby była jedyna.
   Nie ma momentu, w którym podejmujesz decyzję — dostajesz ją już podjętą.

Stąd trzy zmiany w przepływie: **bramka decyzji** przed planem, **bramka szkieletu** przed
implementacją, **samokontrola agenta** przed pokazaniem czegokolwiek człowiekowi.

---

## 2. Przepływ — 10 kroków, 3 nowe bramki

| # | Krok | Artefakt | Status |
|---|------|----------|--------|
| 01 | Ticket → kontekst | `proposal.md` | bez zmian |
| 02 | Analiza | `analysis.md` | bez zmian |
| 03 | **Bramka decyzji** | `design.md` | **NOWE** |
| 04 | Plan jako kontrakt | `tasks.md` | zmienione |
| 05 | **Bramka szkieletu** | gałąź, bez ciał metod | **NOWE** |
| 06 | Implementacja | `deviations.md` | zmienione |
| 07 | **Samokontrola agenta** | — | **NOWE** |
| 08 | Przegląd autora | — | zmienione |
| 09 | Commit, PR, review | — | bez zmian |
| 10 | Archiwizacja | `specs/`, `docs/adr/`, ticket | zmienione |

**01 — Ticket → kontekst.** MCP do Jira/Rally wciąga treść defektu lub historyjki automatycznie.
Zero ręcznego przeklejania. Agent dopisuje wskazane repozytoria, pakiety, tabele.

**02 — Analiza.** Bez zmian. Agent czyta kod, opisuje stan faktyczny i wpływ zmiany.
Iterujesz, aż analiza jest prawdziwa.

**03 — Bramka decyzji.** Agent wypisuje każdą decyzję architektoniczną wraz z **minimum dwiema
odrzuconymi alternatywami** i konsekwencjami. Klasyfikuje ją jako zamkniętą, lokalną albo globalną.
Przy globalnej zatrzymuje się i pyta. Nie idzie dalej.

**04 — Plan jako kontrakt.** Pełna lista plików z rodzajem zmiany, sygnatury, kształt DTO i API,
DDL migracji dosłownie, granice transakcji, wsteczna kompatybilność, lista testów po nazwie.
Zasada twarda: *co nie jest w planie, nie wchodzi do implementacji bez pytania*.

**05 — Bramka szkieletu.** Agent generuje wyłącznie kształt: drzewo plików, klasy, interfejsy,
sygnatury, adnotacje, kolejność wywołań. Ciała metod puste albo `TODO`. Kompiluje się, nie robi nic.
Przegląd zajmuje trzy minuty i pokazuje dokładnie to, co dziś widać dopiero po pełnej implementacji.

**06 — Implementacja.** Agent wypełnia ciała metod. Każdą decyzję, której nie było w planie,
dopisuje do dziennika odstępstw wraz z uzasadnieniem — w trakcie, nie na końcu.

**07 — Samokontrola agenta.** Zanim cokolwiek zobaczy człowiek: testy ArchUnit, zgodność z planem
punkt po punkcie, przegląd własnego diffu. Czerwony ArchUnit wraca do agenta, nie do Ciebie.

**08 — Przegląd autora.** Najpierw `deviations.md` — dziesięć linijek zamiast czterystu.
Dopiero potem diff, celowany w miejsca wskazane przez dziennik.

**09 — Commit, PR, review.** Kolega recenzuje kod. Przy zmianach powyżej jednego modułu recenzuje
też `design.md`, jeszcze przed krokiem 04 — najdroższa poprawka na najtańszym etapie.

**10 — Archiwizacja.** Delty do specyfikacji, każda nowa decyzja globalna do katalogu ADR,
link do artefaktu wraca w komentarzu ticketu przez MCP.

---

## 3. Trzy klasy decyzji

| Klasa | Zakres | Kto decyduje |
|-------|--------|--------------|
| **Zamknięta** | Reguła ArchUnit albo wpis w `docs/adr/` pokrywa przypadek | agent, bez pytania |
| **Otwarta lokalna** | W granicach jednego modułu, nic nie wycieka na zewnątrz | developer, notuje w `design.md` |
| **Otwarta globalna** | Granice modułów, nowa abstrakcja, nowa zależność, cross-cutting (transakcje, cache, security, mapowanie, kontrakt API między serwisami) | eskalacja do architekta |

Obowiązek przeszukania katalogu ADR **przed** eskalacją siedzi w skillu.

> **Ryzyko wdrożenia numer jeden.** Dziś recenzujesz własne zadania. Kiedy pięć osób zacznie
> produkować sekcje „do decyzji", kolejka do jednej osoby zabije proces w miesiąc — ludzie zaczną
> omijać bramkę, żeby nie czekać. Katalog decyzji nie jest dodatkiem, tylko warunkiem, żeby to
> w ogóle skalowało się na zespół.

---

## 4. Architektura jako zależność, nie dokument

Osobne repo `architecture-rules` publikuje reguły jako artefakt Maven/Gradle.
Każde repo dodaje zależność i wybiera profil.

```java
// architecture-rules/src/main/java/.../StandardowyProfil.java
public final class StandardowyProfil {

    @ArchTest
    static final ArchRule warstwy = layeredArchitecture()
        .consideringOnlyDependenciesInLayers()
        .layer("API").definedBy("..api..")
        .layer("Domena").definedBy("..domain..")
        .layer("Infra").definedBy("..infrastructure..")
        .whereLayer("API").mayNotBeAccessedByAnyLayer()
        .whereLayer("Domena").mayOnlyBeAccessedByLayers("API", "Infra");

    @ArchTest
    static final ArchRule kontrolery_nie_dotykaja_repozytoriow =
        noClasses().that().resideInAPackage("..api..")
            .should().dependOnClassesThat().resideInAPackage("..persistence..");

    @ArchTest
    static final ArchRule transakcja_tylko_w_serwisie_domenowym =
        methods().that().areAnnotatedWith(Transactional.class)
            .should().beDeclaredInClassesThat().resideInAPackage("..domain.service..");
}
```

```java
// zamowienia-service/src/test/java/.../ArchitekturaTest.java
@AnalyzeClasses(packages = "com.firma.zamowienia")
class ArchitekturaTest {
    @ArchTest
    static final ArchRules standard = ArchRules.in(StandardowyProfil.class);
}
```

Repozytoria, które nie pasują do standardu, dostają własny profil z **jawną listą wyjątków** —
nigdy z brakiem reguł. Wyjątek zapisany to wyjątek, o którym wiadomo; wyjątek niezapisany to dryf.
Podniesienie wersji reguł jest świadomą decyzją i przechodzi przez PR.

Do tego `ARCHITECTURE.md` w każdym repo: ta sama treść prozą, bo to czyta agent przed planowaniem.
**Reguły weryfikują, dokument informuje — potrzebne są oba.**

---

## 5. Szablony

### `design.md` — bramka decyzji (krok 03)

```markdown
## D1 — Gdzie żyje wyliczanie rabatu

Klasa:        otwarta globalna        # zamknięta | otwarta lokalna | otwarta globalna
Katalog ADR:  sprawdzono, brak wpisu  # obowiązkowe przed eskalacją

Wybór:        nowy DiscountPolicy w com.firma.zamowienia.domain.policy
Odrzucone:
  (a) rozszerzenie OrderService — klasa rośnie do ~900 linii, łamie SRP
  (b) widok w bazie — nietestowalne jednostkowo, logika ucieka z domeny
Konsekwencje: +1 interfejs, +1 punkt rozszerzenia, granice transakcji bez zmian
Ryzyko:       brak, DiscountPolicy jest bezstanowe

STATUS: CZEKA NA DECYZJĘ — nie przechodzę do planu
```

### `deviations.md` — dziennik odstępstw (krok 06)

```markdown
## O1 — Mapowanie null w DiscountRequest
Plan mówił:  "mapuj pola żądania na model domenowy"
Zrobiłem:    null w polu couponCode traktuję jak brak kuponu, nie jak błąd
Dlaczego:    istniejący OrderMapper robi tak samo (OrderMapper.java:64)
Wpływ:       kontrakt API — klient nie dostanie już 400 dla pustego kuponu

## O2 — Kolejność walidacji
Plan mówił:  nic
Zrobiłem:    walidacja limitu przed walidacją kuponu
Dlaczego:    limit jest tańszy, unikamy zapytania do CouponRepository
Wpływ:       zmienia treść błędu przy dwóch naruszeniach naraz
```

Dziennik odstępstw czytasz **przed** diffem. To bezpośrednia odpowiedź na „zmiany widoczne dopiero
po implementacji": zamiast szukać niespodzianek w czterystu liniach, dostajesz ich listę na wejściu.

### `.claude/commands/szkielet.md` — bramka szkieletu (krok 05)

Plik ląduje w repo z procesem i jest instalowany do każdego projektu. Wywołanie: `/szkielet`
albo `/szkielet ZAM-1423`.

````markdown
---
description: Generuje sam kształt zmiany — sygnatury bez ciał metod — do przeglądu przed implementacją
argument-hint: [klucz ticketu lub nazwa zmiany]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(mvn:*), Bash(git diff:*), Bash(git status:*)
---

# Bramka szkieletu

Zmiana: $1

## Wejście

Przeczytaj i traktuj jako wiążący kontrakt:

- `openspec/changes/$1/design.md` — zatwierdzone decyzje architektoniczne
- `openspec/changes/$1/tasks.md` — plan
- `ARCHITECTURE.md` w repozytorium
- `docs/adr/` — katalog decyzji

Jeżeli w `design.md` jest choć jedna pozycja ze statusem `CZEKA NA DECYZJĘ`,
**zatrzymaj się i zgłoś to**. Nie generuj szkieletu.

## Zadanie

Wygeneruj **wyłącznie kształt** zmiany. Kod ma się kompilować i nie robić nic.

Tworzysz:

- pliki, klasy, interfejsy, rekordy, enumy
- pełne sygnatury metod publicznych i pakietowych, z typami i wyjątkami
- adnotacje (Spring, JPA, walidacja) — one są częścią architektury
- pola i wstrzykiwane zależności
- pliki migracji z pełnym DDL
- klasy testowe z nazwami metod i adnotacją `@Disabled("szkielet")`, bez ciał

Ciało każdej metody produkcyjnej to dokładnie jedna linia:

```java
throw new UnsupportedOperationException("TODO 4.2 — wyliczenie rabatu progowego");
```

Numer odsyła do punktu w `tasks.md`. Każda metoda musi mieć taki numer.

**Czego nie robisz:** logiki, obsługi błędów, mapowań pole po polu, ciał testów,
konfiguracji, logowania, komentarzy wyjaśniających. To wszystko przyjdzie w kroku 06.

## Weryfikacja przed pokazaniem

Uruchom i napraw, zanim cokolwiek zaraportujesz:

1. `mvn -q -DskipTests compile` — musi przejść
2. `mvn -q -Dtest=ArchitekturaTest test` — reguły ArchUnit muszą być zielone
   na samym szkielecie; jeżeli nie są, architektura jest zła **zanim** powstała logika

## Wyjście

Zapisz `openspec/changes/$1/szkielet.md` i wypisz w odpowiedzi:

1. **Drzewo plików** — nowe / zmienione / usunięte, z liczbą metod w każdym
2. **Graf wywołań** — ścieżka od wejścia (endpoint, listener, scheduler) do granicy
   (repozytorium, klient HTTP), jedna linia na krok
3. **Granice transakcji** — gdzie zaczyna się i kończy `@Transactional`
4. **Rozjazd z planem** — co musiałeś dodać, czego nie było w `tasks.md`, i dlaczego
5. **Pytania** — czego nie dało się rozstrzygnąć z `design.md`

Na końcu zatrzymaj się. **Nie przechodź do implementacji bez wyraźnej zgody.**
````

Trzy rzeczy, które ten szablon załatwia poza samym przeglądem kształtu:

- **ArchUnit na szkielecie.** Reguły warstw sprawdzają zależności między pakietami, a te są już
  w pełni widoczne w sygnaturach. Naruszenie architektury wychodzi, zanim powstanie choć jedna
  linia logiki — to jest najtańszy możliwy moment.
- **Sekcja „rozjazd z planem"** to dziennik odstępstw przesunięty o krok wcześniej.
  Odstępstwo na poziomie kształtu kosztuje jedną wiadomość, nie przepisanie implementacji.
- **Numer punktu planu w każdym `UnsupportedOperationException`** daje darmową kontrolę pokrycia:
  jeżeli punkt 4.2 nie ma swojego wyjątku, znaczy że agent go pominął.

---

## 6. Druga klasa wad: funkcjonalnie poprawne, wydajnościowo złe

### Dlaczego trzy bramki tego nie łapią

Architektura mieszka w sygnaturach — bramka szkieletu ją pokazuje.
Wydajność mieszka w dwóch miejscach, których szkielet nie pokazuje:

1. **W ciałach metod** — N+1, filtrowanie w pamięci zamiast w `WHERE`, zapytanie w pętli,
   brak batcha, mapowanie całej encji tam, gdzie wystarczy projekcja.
2. **Poza kodem, w wolumenie danych.** I to jest przyczyna źródłowa.

Agent pisze **poprawny kod dla złej skali**, bo nikt mu nie powiedział, jaka jest skala.
`orders.findByCustomerId()` bez paginacji jest bezbłędne przy 200 wierszach i jest awarią przy 12 mln.
Z kodu tego nie widać. To nie jest brak uwagi agenta, tylko brak danych wejściowych.

### Krok zerowy: daj agentowi wolumeny

Jeden plik w repo, odświeżany zaplanowanym zapytaniem:

```markdown
<!-- docs/data-volumes.md -->
| Tabela        | Wierszy | Przyrost/mies. | Uwagi                                   |
|---------------|---------|----------------|-----------------------------------------|
| orders        |   12,4M |          210 k | hot path GET /api/orders                |
| order_items   |   71,8M |          1,3 M | mediana 6 na zamówienie, p99 3 400      |
| customers     |    340 k |          4 k   |                                         |
| audit_log     |  980,0M |           22 M | tylko zapis, nigdy nie czytamy w runtime |
```

To jest najtańsza zmiana w całym dokumencie i usuwa większość zgadywania.
Alternatywa lub uzupełnienie: read-only dostęp do repliki przez MCP, żeby agent sam robił
`EXPLAIN` i `count(*)` zamiast zakładać.

### Co jednak widać w szkielecie — rozszerzenie kroku 05

Sporo wad wydajnościowych **jest** widocznych w sygnaturze, zanim powstanie logika:

- `List<Order>` vs `Page<Order>` / `Slice<Order>` — brak paginacji na nieograniczonym zbiorze
- zwracanie encji zamiast projekcji tam, gdzie potrzeba trzech pól
- `fetch = FetchType.EAGER` w adnotacji, `@OneToMany` bez `@BatchSize`
- zakres `@Transactional` — czy obejmuje wywołanie HTTP albo pętlę po 10 tys. elementów
- `readOnly = true` albo jego brak na ścieżkach odczytowych
- kształt metody: `process(Order)` wołane w pętli vs `process(List<Order>)` — jedno wymusza
  N zapytań, drugie pozwala na batch

Dlatego do sekcji wyjściowej komendy `/szkielet` dochodzi punkt **6. Wydajność**:
liczba zapytań na wywołanie dla każdej nowej ścieżki, użyte indeksy, zakres transakcji.

### Budżet wydajnościowy w `design.md`

Blok obowiązkowy dla każdej decyzji dotykającej odczytu lub zapisu danych:

```markdown
## Budżet wydajnościowy — D1

Ścieżka:       GET /api/orders?customerId=   (hot path, ~40 req/s w szczycie)
Wolumen:       orders 12,4M; na klienta mediana 6, p99 3 400
Limit zapytań: ≤ 2 na wywołanie
Limit czasu:   p95 ≤ 120 ms
Indeksy:       ix_orders_customer_created — istnieje, EXPLAIN poniżej
Ryzyko:        klienci z p99; wymuszona paginacja, zakaz fetch join po kolekcji
```

Brak budżetu = brak zgody na przejście do kroku 04. Tak samo jak brak decyzji.

### Zrób z tego czerwony test — dokładnie jak z ArchUnit

Liczba zapytań jest **mierzalna**, więc nie ma powodu, żeby była uwagą w review.
Wersja bez dodatkowych zależności, na statystykach Hibernate:

```java
// application-test.yml: spring.jpa.properties.hibernate.generate_statistics: true

@Test
void lista_zamowien_nie_generuje_n_plus_1() {
    Statistics stats = entityManagerFactory.unwrap(SessionFactory.class).getStatistics();
    stats.clear();

    orderQueryService.findRecent(customerId, PageRequest.of(0, 50));

    assertThat(stats.getPrepareStatementCount())
        .as("budżet z design.md: max 2 zapytania")
        .isEqualTo(2);
}
```

Dane testowe muszą mieć realny kształt — 50 zamówień po 6 pozycji, nie jedno po jednej.
Przy jednym wierszu N+1 nie istnieje. Testcontainers, nie H2, jeśli zapytania są nietrywialne.

Jeśli wolicie gotową bibliotekę, QuickPerf daje to samo adnotacjami w rodzaju `@ExpectSelect(2)`
i `@ExpectMaxQueryExecutionTime(...)` — sprawdźcie aktualne API przed wpięciem.

Dwa ustawienia Hibernate, które łapią klasyczne błędy same z siebie:

```properties
hibernate.generate_statistics=true
hibernate.query.fail_on_pagination_over_collection_fetch=true
```

Drugie zamienia ciche ładowanie całej kolekcji do pamięci przy paginacji w twardy wyjątek.

### ArchUnit też część złapie

```java
@ArchTest
static final ArchRule brak_eager =
    noFields().should().beAnnotatedWith(OneToMany.class)
        .andShould(haveFetchType(FetchType.EAGER));

@ArchTest
static final ArchRule odczyt_tylko_do_odczytu =
    classes().that().resideInAPackage("..domain.query..")
        .should().beAnnotatedWith(describe("@Transactional(readOnly = true)", ...));

@ArchTest
static final ArchRule repozytoria_nie_zwracaja_nieograniczonych_list =
    noMethods().that().areDeclaredInClassesThat().areAssignableTo(Repository.class)
        .and().haveNameStartingWith("findAll")
        .should().haveRawReturnType(List.class);
```

### Czego nie da się zabramkować

Poza zasięgiem testu jednostkowego zostają: rywalizacja o blokady, wyczerpanie puli połączeń,
zachowanie cache pod obciążeniem, GC, degradacja przy współbieżności.
Te wychodzą tylko pod obciążeniem albo na produkcji.

Dlatego jawna lista wyzwalaczy — zmiana wymaga testu obciążeniowego przed mergem, jeżeli:

- dotyka ścieżki oznaczonej w `docs/data-volumes.md` jako *hot path*, **albo**
- wprowadza nowy zapis w transakcji obejmującej więcej niż jedną tabelę, **albo**
- zmienia indeks, klucz albo typ kolumny na tabeli powyżej 1 mln wierszy, **albo**
- dodaje wywołanie zewnętrznego serwisu wewnątrz `@Transactional`

Reszta idzie na obserwowalność po merge’u: alert na p95 endpointu i na liczbę zapytań na żądanie.
Wada wydajnościowa wykryta w produkcji wraca do `docs/adr/` jako decyzja zamknięta — żeby
agent nie popełnił jej drugi raz.

---

## 7. OpenSpec — co bierzemy, co dokładamy, co odkładamy

| Element | Decyzja | Uzasadnienie |
|---------|---------|--------------|
| `openspec/changes/` | bierzemy | Formalizuje to, co i tak już robisz prywatnie: jeden folder na zmianę |
| `design.md` | bierzemy | Najważniejszy artefakt w naszym przypadku — ale z własnym szablonem |
| Requirement / Scenario | bierzemy | Recenzja WHEN/THEN szybsza i gęstsza niż recenzja prozy |
| `archive` → `specs/` | bierzemy | Delty akumulują się w żywą specyfikację, nie w stertę plików |
| `/opsx:explore` | bierzemy | Brakujący dziś moment „oto trzy drogi, wybierz" |
| bramka szkieletu | **nasze** | OpenSpec tego nie ma. Najtańsza zmiana o największym efekcie |
| `deviations.md` | **nasze** | OpenSpec tego nie ma, a to odpowiedź na nasz główny ból |
| `docs/adr/` + ArchUnit | **nasze** | Warunek, żeby jedna osoba nie była wąskim gardłem |
| Stores (beta) | później | Adresuje zmiany w kilku repo, ale beta — nie budujemy na tym wdrożenia |
| profil rozszerzony | później | Start na profilu domyślnym, więcej komend gdy cykl się utrwali |

Instalacja: `npm install -g @fission-ai/openspec@latest`, potem `openspec init` w repo
(wymaga Node ≥ 20.19). `openspec update` regeneruje instrukcje dla agenta — trzeba pilnować,
żeby nie nadpisywało naszych własnych reguł.

**Werdykt:** OpenSpec jest dobrym pojemnikiem, ale nie jest lekarstwem na iteracje architektoniczne.
Iteracje wycinają bramki (sekcje 3–5), OpenSpec sprawia, że reszta zespołu robi to samo,
w tym samym miejscu i formacie.

---

## 8. Wdrożenie — trzy etapy z warunkami przejścia

### Etap 1 — Reguły i bramka szkieletu · ~1 tydzień · tylko architekt

- Repo `architecture-rules`, standardowy profil ArchUnit, wpięcie w dwa najważniejsze repozytoria
- `ARCHITECTURE.md` w tych repozytoriach
- Slash-command na szkielet, używany na własnych zadaniach

**Warunek przejścia:** liczba iteracji po implementacji spada poniżej dwóch.

### Etap 2 — OpenSpec na jednym repo · ~2 tygodnie · architekt + 1 developer

- `openspec init`, profil domyślny, własny szablon `design.md`
- `deviations.md` i klasyfikacja decyzji zapisane w skillu
- Pierwsze wpisy w `docs/adr/` — każda decyzja podjęta raz
- Domknięcie pętli z Jira/Rally przez MCP: ticket na wejściu, link na wyjściu

**Warunek przejścia:** drugi developer przechodzi cały cykl bez pytania architekta o proces.

### Etap 3 — Zespół i dystrybucja procesu · ~1 miesiąc · cały zespół

- Wspólne repo z `.claude/`: skille, slash-commandy, fragment CLAUDE.md, instalowane w każdym projekcie
- Zmiana procesu = PR do tego repo, nie ogłoszenie na standupie
- Reguły ArchUnit w pozostałych repozytoriach, z jawnymi wyjątkami
- Recenzja `design.md` przez kolegę przy zmianach powyżej jednego modułu

**Warunek utrzymania:** eskalacje do architekta spadają z tygodnia na tydzień.

---

## 9. Metryki kontrolne

| Metryka | Kierunek | Co mówi |
|---------|----------|---------|
| Iteracje po implementacji | dziś ~3 → cel ≤ 1 | Główna metryka. Brak spadku po etapie 1 = przyczyna leży poza procesem |
| Eskalacje na sprint | ma spadać | Płaski wykres po dwóch miesiącach = katalog ADR nie działa |
| Odstępstwa na zmianę | ma spadać | Czy plan naprawdę zyskuje rozdzielczość |
| Uwagi architektoniczne w PR | ma spadać | Powinny przenieść się na krok 03 |
| Regresje wydajnościowe po merge | ma spadać | Każda oznacza brakujący budżet w `design.md` albo brakujący test na liczbę zapytań |

---

## 10. Do rozstrzygnięcia przed startem

**A. Nazewnictwo warstw w regułach ArchUnit.**
`..api..` / `..domain..` / `..infrastructure..` to placeholdery. Trzeba je podmienić na faktyczną
strukturę pakietów. Uwaga na klasyczną pułapkę: ArchUnit nie zgłasza błędu, gdy pakiet w regule
nie istnieje — reguła jest zielona przez przypadek. Zabezpieczenie:

```java
@AnalyzeClasses(packages = "com.firma.zamowienia",
                importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitekturaTest {
    @ArchTest
    static final ArchRule r = ...
        .allowEmptyShould(false);   // albo globalnie: archunit.properties
}
```

W `archunit.properties`: `archRule.failOnEmptyShould=true`.

**B. Kryterium „powyżej jednego modułu"** przy recenzji `design.md` przez kolegę.
Musi być zdaniem rozstrzygalnym bez dyskusji, inaczej każdy zinterpretuje po swojemu.
Propozycja: *zmiana dotyka plików w więcej niż jednym module Maven/Gradle, albo dodaje/zmienia
publiczny kontrakt (endpoint REST, zdarzenie, schemat tabeli)*.

**C. Gdzie żyją artefakty przy zmianie obejmującej kilka repo.**
Do czasu wyjścia Stores z bety: artefakt w repo, w którym leży większość zmiany,
plus link krzyżowy w pozostałych. Do ustalenia: kto pilnuje spójności.
