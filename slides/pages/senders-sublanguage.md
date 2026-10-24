---
layout: cover
background: /img/bg/bg1.jpg
---

<div class="pt-6 text-sm opacity-80">
Rachunek Lambda • Styl Przekazywania Kontynuacji (CPS) • Monady • Efekty Algebraiczne • 30 Adaptorów
</div>

---
layout: default
---

# Agenda

1. **Fundamenty teoretyczne:**
   * Rachunek lambda i styl przekazywania kontynuacji (CPS – Steele, Sussman, Plotkin)
   * Teoria kategorii: Monady, funktory i strzałki Kleisliego (Moggi, Milewski)
   * Ograniczone kontynuacje, efekty algebraiczne i typy sumaryczne (Danvy, Filinski, Griffin)
   * Logika liniowa, teoria procesów i dyscyplina regionów (Girard, Milner, Tofte)
2. **Katalog 30 algorytmów i adaptorów (omówienie + kod):**
   * Podnoszenie wartości (*Value Lifting*) & Konsumpcja (`just`, `just_error`, `just_stopped`, `sync_wait`)
   * Transformacja (*Functor fmap* – `then`, `upon_error`, `upon_stopped`)
   * Kompozycja monadyczna (*Kleisli Bind* – `let_value`, `let_error`, `let_stopped`)
   * Konteksty wykonania i planiści (`schedule`, `starts_on`, `continues_on`, `on`, `affine`, `schedule_from`)
   * Środowisko (*Reader Monad* – `read_env`, `write_env`, `unstoppable`)
   * Ustrukturyzowana współbieżność i alokacja (`when_all`, `when_all_with_variant`, `split`)
   * Adaptacja sygnałów (*Coproduct Morphisms* – `into_variant`, `stopped_as_optional`, `stopped_as_error`)
   * Równoległość danych (`bulk`, `bulk_chunked`, `bulk_unchunked`)
   * Zakresy asynchroniczne (*Async Scopes* – `associate`, `spawn_future`)
3. **Most z korutynami:** `std::execution::task<T>` (P3552R3)

---
layout: two-cols
---

# Rachunek Lambda i CPS (Steele, Sussman, Plotkin)

### Continuation-Passing Style (CPS)
* **The Lambda Papers (1975–1980):** Steele i Sussman sformalizowali CPS – funkcje nie zwracają wartości bezpośrednio do wywołującego, lecz przekazują ją do jawnej kontynuacji $k$ (*„co dzieje się dalej”*).
* **Rząd ewaluacji (Plotkin, 1975):** W rachunku call-by-value CPS sprawia, że kolejność ewaluacji staje się jawna w samej strukturze termu.
* **Kompilacja i brak alokacji:**
  * Wzorzec ten stanowi reprezentację pośrednią w kompilatorach funkcyjnych (SML/NJ, Chicken Scheme, GHC STG).
  * W C++26 faza `connect(sndr, rcvr)` konkretyzuje kontynuację, tworząc unikalny typ stanu operacji (*operation state*).
  * Cały graf zależności budowany jest statycznie w czasie kompilacji bez żadnej dynamicznej alokacji pamięci.

::right::

```
Tradycyjny powrót (Direct Style):
  Caller ──call──> [ Function ] ──return──> Caller

Styl CPS / Senders & Receivers:
  [ Sender ] ──set_value(val)──> [ Receiver (Continuation) ]
```

```cpp
// Konkretyzacja kontynuacji (CPS Reification):
auto op = std::execution::connect(sndr, rcvr);

// Ewaluacja grafu w stylu CPS:
std::execution::start(op);
```

*Graf operacji wykonuje się bez narzutu wirtualnego i bez pośrednictwa sterty.*

---
layout: two-cols
---

# Teoria Kategorii: Monady i Kleisli (Moggi, Wadler)

### Monada $(T, \eta, \mu)$
* **Eugenio Moggi (1991):** Monady modelują obliczenia z efektami ubocznymi.
* **Jednostka monady ($\eta : \text{Id} \to T$):** Podnosi czystą wartość do kontekstu obliczeniowego. W C++26 odpowiada jej `just(x)`.
* **Mnożenie monady ($\mu : T(T(A)) \to T(A)$):** Spłaszcza zagnieżdżone konteksty obliczeń.
* **Strzałki Kleisliego ($A \to T(B)$):** Funkcje zwracające nowe obliczenia asynchroniczne.
* **Monadyczny Bind ($\gg=$):**
  $$(m \gg= f) = \mu(T(f)(m))$$
  W C++26 realizowany bezpośrednio przez `let_value(f)`.

::right::

### Funktor Endomorficzny
* Każda monada jest funktorem wyposażonym w transformację strzałek:
  $$fmap : (A \to B) \to (T(A) \to T(B))$$
* W standardzie C++26 funktor $fmap$ jest reprezentowany przez adaptor **`then(f)`**.
* Nie modyfikuje on struktury samego potoku – transformuje czystą wartość przed przekazaniem do odbiornika.

### Monada Czytelnika (*Reader Monad*)
* Philip Wadler (1992): Obliczenia zależne od niezmiennego środowiska $E \to A$.
* W C++26: `read_env` to funkcja `ask`, a `write_env` to funkcja `local`.

---
layout: default
---

# Ograniczone Kontynuacje i Efekty Algebraiczne

### Trójkanałowy Koprojekt jako Efekt Algebraiczny (Danvy & Filinski, 1990)
* Zamiast pojedynczego wyjścia model C++26 definiuje **koprojekt trzech kanałów**:
  $$\text{Completion} \cong \text{Value} + \text{Error} + \text{Stopped}$$
* Operatory `shift`/`reset` pozwalają na obsługę i maskowanie poszczególnych efektów obliczenia.
* **Korespondencja Curry'ego-Howarda dla sterowania (Timothy Griffin, 1990):** Operatory sterowania odpowiadają regułom typowania w logice klasycznej. Sygnatury ukończenia (*Completion Signatures*) deklarują w systemie typów C++ zbiór wszystkich dopuszczalnych kanałów i typów argumentów.

### Logika Liniowa i Regiony
* **Logika Liniowa (Girard, 1987):** Sendery są domyślnie liniowe – ulegają konsumpcji przy wywołaniu `connect`. Adaptor `split` realizuje regułę kontrakcji ($\Delta : A \to A \otimes A$) kosztem współdzielonego stanu.
* **Dyscyplina Regionów (Tofte & Talpin, 1994):** Zakresy asynchroniczne (*Async Scopes*) narzucają leksykalne i dynamiczne granice czasu życia zadań.

---
layout: default
---

# Tabela odwzorowań: Matematyka a C++26

| Pojęcie w C++26 | Odpowiednik matematyczny / teoretyczny | Literatura źródłowa |
| :--- | :--- | :--- |
| `just(x)` | Monadyczna jednostka / `return` ($\eta$) | Moggi (1991) |
| `then(f)` | Akcja funktora na morfizmach ($fmap$) | Moggi (1991), Milewski (2019) |
| `let_value(f)` | Monadyczny bind ($\gg=$) / Strzałka Kleisliego | Moggi (1991), Wadler (1992) |
| `set_value / set_error / set_stopped` | Kanały efektów algebraicznych / Koprojekt sumaryczny | Danvy & Filinski (1990) |
| `connect(sndr, rcvr)` | Reifikacja kontynuacji w CPS | Steele & Sussman (1975–1980) |
| `start(op)` | Wymuszenie ewaluacji termu CPS | Plotkin (1975) |
| *Completion signatures* | Zbiory sygnatur na poziomie typów | Griffin (1990) |
| `read_env / write_env` | Monada Czytelnika (`ask` / `local`) | Wadler (1992) |
| `when_all` | N-arny produkt tensorowy w symetrycznej kategorii monoidalnej | Milner (1989, CCS) |
| `split` | Reguła kontrakcji w logice liniowej ($\Delta: A \to A \otimes A$) | Girard (1987) |
| *Async scopes* (`associate`) | Ograniczenie czasu życia oparte na regionach | Tofte & Talpin (1994) |
| Rodzina `bulk` | Data-parallel map | Hillis & Steele (1986) |

---
layout: two-cols
---

# 1. Podnoszenie wartości: `just`, `just_error`

### 3.1 `just`
Podnosi czyste wartości do potoku na kanale sukcesu (`set_value`). Odpowiednik monadic unit $\eta$.
```cpp
auto sndr = just(42, std::string("test"));

// Czysty C++:
// int a = 42; string b = "test";
```

* Wykonuje się natychmiastowo, synchronicznie i bez alokacji.
* Punkt startowy każdego czystego potoku obliczeń.

::right::

### 3.2 `just_error`
Wstrzykuje błąd do potoku na kanale `set_error`. Wartość podróżuje do najbliższego handlera.
```cpp
auto sndr = just_error(
  std::make_error_code(
    std::errc::invalid_argument));

// Czysty C++:
// throw std::system_error(
//   make_error_code(invalid_argument));
```

* Podnosi błąd jako pierwszorzędny typ bez rzucania wyjątku CPU.
* Obsługiwany niżej przez `upon_error` lub `let_error`.

---
layout: two-cols
---

# 1. Podnoszenie wartości: `just_stopped`, `sync_wait`

### 3.3 `just_stopped`
Inicjuje sygnał anulowania na dedykowanym kanale zatrzymania (`set_stopped`).
```cpp
auto sndr = just_stopped();

// Czysty C++:
// Brak bezpośredniego odpowiednika w tradycyjnym C++
// (odpowiednik przerwania transakcji / return)
```

* Trzeci, niezależny kanał koprojektu ukończeń.
* Reprezentuje rezygnację z wykonania bez błędu.

::right::

### 3.4 `sync_wait`
Most łączący świat senderów z synchronicznym C++. Odpowiednik monadic counit / runST.
```cpp
auto opt = sync_wait(just(42));
if (opt) {
  auto [v] = *opt; // v == 42
}

// Czysty C++:
// int result = 42;
```

* Blokuje bieżący wątek do momentu zakończenia grafu.
* Jedyny konsument sendera w standardzie; wymusza ewaluację.

---
layout: two-cols
---

# 2. Transformacja: `then`, `upon_error`

### 4.1 `then`
Aplikuje czystą funkcję $f: A \to B$ do wartości z kanału `set_value`. Funktorowe $fmap$.
```cpp
auto sndr = just(3, 4)
  | then([](int x, int y) {
      return x * x + y * y;
    });
// set_value(25)

// Czysty C++:
// int result = 3 * 3 + 4 * 4;
```
* Błędy i zatrzymanie omijają `f` bez zmian.
* Wyjątek z `f` trafia automatycznie na kanał `set_error`.

::right::

### 4.2 `upon_error`
Przekształca błąd z kanału `set_error` w wartość sukcesu na kanale `set_value`.
```cpp
auto sndr = just_error(
  make_error_code(errc::connection_reset))
  | upon_error([](error_code ec) {
      return -1; // Ratunek: powrót do set_value(-1)
    });

// Czysty C++:
// try { throw ... } catch (...) { return -1; }
```
* Zamienia ścieżkę awarii w wartość domyślną.
* Wartości sukcesu przechodzą nienaruszone.

---
layout: default
---

# 2. Transformacja: `upon_stopped`

### 4.3 `upon_stopped`
Przechwytuje sygnał zatrzymania z kanału `set_stopped` i zamienia go w normalną wartość na kanale sukcesu:

```cpp
auto sndr = just_stopped()
  | upon_stopped([] {
      return 0; // Przekształcenie anulowania w wartość set_value(0)
    });

// Czysty C++:
// Klasyczny C++ nie posiada wydzielonego kanału anulowania
```

* **Semantyka:** Pozwala potokowi na zdefiniowanie wartości awaryjnej (fallback) w sytuacji, gdy poprzedzające podzadanie zostało anulowane przez token zatrzymania.
* **Transformacja koprojektu:** Redukuje kanał `set_stopped`, kierując wynik ponownie na ścieżkę `set_value`.

---
layout: two-cols
---

# 3. Kompozycja monadyczna: `let_value`, `let_error`

### 5.1 `let_value`
Monadyczny bind ($\gg=$) / strzałka Kleisliego. Przyjmuje funkcję zwracającą **nowy sender**.
```cpp
auto sndr = just(std::string("hello"))
  | let_value([](std::string s) {
      return just(s + " world")
        | then([](std::string r) {
            return r.size();
          });
    });
// Kończy z set_value(11)
```
* Umożliwia dynamiczne rozgałęzianie i sekwencjonowanie całych kolejnych podgrafów asynchronicznych.

::right::

### 5.2 `let_error`
Obsługa błędu poprzez uruchomienie nowego, asynchronicznego potoku naprawczego.
```cpp
auto sndr = just_error(
  make_error_code(errc::timed_out))
  | let_error([](error_code ec) {
      // Zwraca cały nowy sender naprawczy:
      return just(ec.message());
    });
// Kończy z set_value("timed out")
```
* Pozwala na asynchroniczne próby odzyskania sprawności (np. ponowne pobranie z bazy danych lub zapasowego serwera).

---
layout: default
---

# 3. Kompozycja monadyczna: `let_stopped`

### 5.3 `let_stopped`
Monadyczna obsługa anulowania. Zastępuje sygnał zatrzymania uruchomieniem zupełnie nowego sendera:

```cpp
auto sndr = just_stopped()
  | let_stopped([] {
      // Uruchomienie asynchronicznej ścieżki zapasowej po anulowaniu
      return just(std::string("operacja_zastępcza"));
    });
// Kończy z set_value("operacja_zastępcza")
```

* **Zastosowanie:** Gdy anulowanie jednego procesu wymaga podjęcia asynchronicznego działania kompensacyjnego (np. zapisu stanu do dziennika transakcyjnego lub asynchronicznego zwolnienia zasobów zewnętrznych).
* **Teoria:** Rozszerzenie Kleisliego zaaplikowane bezpośrednio do składowej koprojektu odpowiadającej zatrzymaniu.

---
layout: two-cols
---

# 4. Konteksty wykonania: `schedule`, `starts_on`

W modelu C++26 planista (*Scheduler*) reprezentuje zasób obliczeniowy (wątek, pulę wątków, kolejkę I/O, GPU).

### 6.1 `schedule`
Tworzy sender kończący się na wskazanym planiście (punkt wejścia w kontekst).
```cpp
auto sndr = schedule(pool.get_scheduler())
  | then([] {
      return handle_request();
    });

// Czysty C++:
// pool.post([&] { handle_request(); });
```

::right::

### 6.2 `starts_on`
Wymusza rozpoczęcie całego zadanego potoku na wybranym planiście.
```cpp
auto sndr = starts_on(
  pool.get_scheduler(),
  just(req) | then([](auto r) {
    return handle(r);
  })
);

// Czysty C++:
// pool.post([&] { auto res = handle(req); });
```

---
layout: two-cols
---

# 4. Konteksty wykonania: `continues_on`, `on`

### 6.3 `continues_on`
Przenosi dalszą realizację potoku na inny kontekst wykonania po ukończeniu poprzednika.
```cpp
auto sndr =
  starts_on(io_ctx.get_scheduler(),
            async_read(socket, buf))
  | continues_on(pool.get_scheduler())
  | then([](auto data) {
      return parse(data); // na puli wątków!
    });
```
* Klasyczny wzorzec: odczyt na pętli I/O, ciężkie przetwarzanie na puli roboczej.

::right::

### 6.4 `on`
Wykonuje podzadanie na zadanym planiście i automatycznie przywraca pierwotny kontekst.
```cpp
auto sndr = on(
  pool.get_scheduler(),
  just(req) | then([](auto r) {
    return compute_heavy(r);
  })
);
// Wynik wraca na scheduler wywołującego
```
* Semantyka "round-trip": delegacja obliczeń na zewnątrz z gwarancją powrotu.

---
layout: two-cols
---

# 4. Konteksty wykonania: `affine`, `schedule_from`

### 6.5 `affine` (P4151R1 / P3941R4)
Wymusza ukończenie na planiście początkowym odbiornika (`get_start_scheduler`), optymalizując zbędne skoki.
```cpp
auto sndr = some_operation()
  | affine;
```
* **Kluczowa optymalizacja:** Jeśli sender już kończy się na docelowym wątku, kosztowne przełączenie kontekstu jest całkowicie pomijane.
* Fundament mechanizmu korutyn w C++26.

::right::

### 6.6 `schedule_from`
Niskopoziomowy adaptor gwarantujący, że **wszystkie trzy sygnały** (wartość, błąd, stop) dotrą na dany scheduler.
```cpp
auto sndr = schedule_from(
  pool.get_scheduler(),
  async_read(socket, buf)
);
```
* Głębszy niż `continues_on`, który transferuje wyłącznie kanał wartości sukcesu.
* Narzędzie implementatorów bibliotek i sterowników.

---
layout: two-cols
---

# 5. Środowisko: `read_env`, `write_env`

Środowisko (*Environment*) to niejawny kontekst przekazywany w górę przez odbiornik (np. stop token, alokator, scheduler).

### 7.1 `read_env`
Odpowiednik `ask` w monadzie czytelnika. Odpytuje odbiornik o właściwość środowiska.
```cpp
auto sndr = read_env(get_scheduler)
  | let_value([](auto sch) {
      return schedule(sch)
        | then([] { return load_config(); });
    });
```
* Umożliwia pisanie kodu polimorficznego względem kontekstu wykonania.

::right::

### 7.2 `write_env`
Odpowiednik `local` w monadzie czytelnika. Nakłada nowe właściwości na środowisko odbiornika.
```cpp
auto sndr = write_env(
  make_env(get_allocator, pool_alloc),
  process_file(path)
);
```
* Przekazuje np. niestandardowy alokator PMR do wszystkich operacji potomnych w potoku.

---
layout: default
---

# 5. Środowisko: `unstoppable`

### 7.3 `unstoppable`
Maskowanie efektu zatrzymania w rachunku efektów algebraicznych. Chroni sender przed sygnałem anulowania:

```cpp
auto sndr = unstoppable(
  commit_transaction(db, txn)
);

// Czysty C++:
// commit_transaction(db, txn);
```

* **Semantyka:** Nawet jeśli nadrzędny potok otrzyma żądanie zatrzymania (`stop_token`), opakowana operacja wykona się do końca i nie zostanie przerwana.
* **Typowanie:** Wyprowadza typ sygnatur ukończenia, z którego usuwany jest znacznik `set_stopped_t()`.
* **Zastosowanie:** Zamykanie połączeń, zwalnianie zasobów sprzętowych, zatwierdzanie transakcji w bazie danych.

---
layout: two-cols
---

# 6. Ustrukturyzowana współbieżność: `when_all`, `when_all_with_variant`

### 8.1 `when_all`
Uruchamia sendery równolegle. Łączy ich wartości w jeden wspólny ciąg argumentów.
```cpp
auto sndr = when_all(
  fetch_user(id),
  fetch_orders(id),
  fetch_prefs(id)
) | then([](auto user, auto ords, auto prfs) {
    return build_profile(user, ords, prfs);
  });
```
* **Gwarancja strukturalna:** Żadne zadanie potomne nie przeżyje punktu złączenia.
* Jeśli którekolwiek rzuci błąd lub stop, pozostałe są natychmiast anulowane.

::right::

### 8.2 `when_all_with_variant`
Współbieżne złączenie dla operacji zwracających niejednorodne sygnatury ukończenia.
```cpp
auto sndr = when_all_with_variant(
  fetch_json(url_a),
  fetch_binary(url_b)
) | then([](auto json_var, auto bin_var) {
    return combine(json_var, bin_var);
  });
```
* Każdy wynik jest bezpiecznie opakowany w <code>std::variant&lt;std::tuple&lt;...&gt;&gt;</code>.
* Odpowiednik tensora z koprojektami w każdym czynniku.

---
layout: default
---

# 6. Logika Liniowa: `split`

### 8.3 `split`
Przekształca sender jednokrotny (*single-shot*) w sender wielokrotnego użytku (*multi-shot*):

```cpp
// Domyślnie sendery są liniowe (niszczone przy connect):
auto shared = split(fetch_data(url));

auto sndr = when_all(
  shared | then([](auto data) { return analyze(data); }),
  shared | then([](auto data) { return archive(data); })
);

// Czysty C++:
// auto data = fetch_data(url);
// auto a = analyze(data); auto b = archive(data);
```

* **Podstawa w logice liniowej (Girard, 1987):** Reguła kontrakcji $\Delta : A \to A \otimes A$. Ponieważ duplikacja stanu liniowego nie jest darmowa, `split` alokuje wewnętrzny stan ze zliczaniem referencji, który buforuje wynik i rozsyła go do wszystkich odbiorników.

---
layout: two-cols
---

# 7. Adaptacja sygnałów: `into_variant`, `stopped_as_optional`

### 9.1 `into_variant`
Unifikuje wielorakie sygnatury wartości do pojedynczego `std::variant`.
```cpp
auto sndr = query_database(stmt)
  | into_variant;
// set_value(variant<tuple<Row>, tuple<int>>)
```
* Sprowadza dowolny sender do postaci o dokładnie jednej sygnaturze `set_value`, co umożliwia podpięcie go np. pod standardowe `when_all`.

::right::

### 9.2 `stopped_as_optional`
Mapuje kanał zatrzymania na pusty `std::optional` na kanale sukcesu.
```cpp
auto sndr = dequeue(work_queue)
  | stopped_as_optional;
// value(x)  -> set_value(optional(x))
// stopped() -> set_value(nullopt)
```
* Pozwala konsumentowi obsłużyć zatrzymanie zwykłym blokiem `if (opt)`.

---
layout: default
---

# 7. Adaptacja sygnałów: `stopped_as_error`

### 9.3 `stopped_as_error`
Przekształca anulowanie operacji (`set_stopped`) w formalny błąd na kanale `set_error`:

```cpp
auto sndr = timed_operation(deadline)
  | stopped_as_error(
      make_error_code(std::errc::timed_out)
    );

// Czysty C++:
// auto res = timed_operation(deadline);
// if (!res) throw system_error(make_error_code(errc::timed_out));
```

* **Semantyka:** Eliminuje kanał zatrzymania z sygnatur ukończenia i scala go ze ścieżką błędów.
* **Zastosowanie:** Przydatne, gdy w dalszej części potoku infrastruktura obsługuje błędy i kody odpowiedzi, ale nie implementuje semantyki anulowania.

---
layout: two-cols
---

# 8. Równoległość danych: `bulk`, `bulk_chunked`

Zrównoleglenie operacji oparte na modelu Hillisa i Steele'a (1986).

### 10.1 `bulk`
Równoległa pętla po przestrzeni indeksów `[0, shape)`.
```cpp
auto sndr = just(records)
  | bulk(std::par, records.size(),
      [](size_t i, auto& recs) {
        recs[i].score = compute(recs[i]);
      });
```
* Odpowiednik `std::for_each(par, ...)`.
* Zachowuje referencję do przekazanej wartości bez alokacji.

::right::

### 10.2 `bulk_chunked`
Dzieli przestrzeń indeksów na paczki dopasowane do pamięci podręcznej i wektoryzacji.
```cpp
auto sndr = just(data)
  | bulk_chunked(std::par, data.size(),
      [](auto chunk, auto& d) {
        for (auto i : chunk)
          d[i] = transform(d[i]);
      });
```
* Pozwala na wektoryzację SIMD wewnątrz każdej paczki.

---
layout: default
---

# 8. Równoległość danych: `bulk_unchunked`

### 10.3 `bulk_unchunked`
Wywołuje zadaną funkcję raz dla każdego indeksu bez żadnych gwarancji grupowania:

```cpp
auto sndr = just(pixels)
  | bulk_unchunked(std::par, pixels.size(),
      [](size_t i, auto& px) {
        px[i] = apply_filter(px[i]);
      });

// Czysty C++:
// std::for_each(std::execution::par, pixels.begin(), pixels.end(),
//   [](auto& p) { p = apply_filter(p); });
```

* **Relacja w rodzinie:** `bulk` domyślnie deleguje wykonanie do `bulk_chunked`, który planista może wyspecjalizować pod kątem wątków sprzętowych. `bulk_unchunked` stanowi najprostszy model poindeksowego zrównoleglenia.

---
layout: two-cols
---

# 9. Zakresy asynchroniczne: `associate`, `spawn_future`

Dyscyplina regionów (Tofte & Talpin, 1994) przeniesiona w realia asynchroniczne.

### 11.1 `associate`
Wiąże czas życia swobodnie działającego sendera z nadrzędnym zakresem (`counting_scope`).
```cpp
counting_scope scope;
auto token = scope.get_token();

for (auto& conn : connections) {
  auto sndr = associate(token,
    starts_on(pool.get_scheduler(),
              handle(conn)));
}
// scope.join() zawiesi shutdown do końca zadań
```
* Gwarantuje brak "wiszących" wątków w tle przy zamykaniu serwera.

::right::

### 11.2 `spawn_future`
Uruchamia zadanie w tle wewnątrz zakresu, zwracając uchwyt do odbioru wyniku.
```cpp
auto sndr = spawn_future(token,
  starts_on(pool.get_scheduler(),
            fetch_data(key)))
  | then([](auto data) {
      return process(data);
    });

// Czysty C++:
// auto fut = std::async(launch::async, ...);
```
* Asynchroniczny odpowiednik `std::async` ze strukturalnym ograniczeniem czasu życia.

---
layout: two-cols
---

# Korutyny C++26: `execution::task<T>` (P3552R3)

Most łączący składnię `co_await` z grafem senderów:

* **Leniwa ewaluacja:** Korutyna nie startuje do momentu wywołania `connect` i `start`.
* **Dwukierunkowa integracja:**
  * Wewnątrz `task<T>` można wykonać `co_await sndr`.
  * Sam `task<T>` jest pełnoprawnym senderem (może być łączony operatorem `|`).
* **Niejawny `affine`:** Każdy punkt zawieszenia `co_await` automatycznie wznawia wykonanie na macierzystym planiście korutyny.
* **Typowane błędy:** `co_yield with_error(e)` wysyła błąd kanałem `set_error` bez rzucania wyjątków!

::right::

```cpp
task<sensor_data> read_sensors(
    sensor_array const& sensors) {
  // co_await na ustrukturyzowanym potoku
  auto [lidar, radar, cam] = co_await
    when_all(
      starts_on(sensors.lidar_ctx(),
                read_lidar()),
      starts_on(sensors.radar_ctx(),
                read_radar()),
      starts_on(sensors.cam_ctx(),
                read_cam())
    );

  co_return fuse(lidar, radar, cam);
}
```

---
layout: default
---

# Podsumowanie: Kompletny ekosystem C++26

```
                    ┌────────────────────────┐
                    │      Wartości          │
                    │ just, just_error, ...  │
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │      Transformacja     │
                    │  then, upon_error, ... │
                    └───────────┬────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│  Współbieżność│       │  Konteksty    │       │   Środowisko  │
│   when_all    │       │   starts_on   │       │   read_env    │
│     bulk      │       │  continues_on │       │   write_env   │
└───────┬───────┘       └───────┬───────┘       └───────┬───────┘
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ execution::task<T>     │
                    │ co_await / Pipelines   │
                    └────────────────────────┘
```

* **Pełna spójność:** 30 komplementarnych algorytmów opartych na teorii kategorii i logice liniowej.
* **Zero niejawnych alokacji:** Wszystkie przejścia typowane statycznie podczas konkretyzacji grafu CPS.

---
layout: end
---

# Dziękuję za uwagę!
### Pytania i dyskusja