# Wstęp do TDD

**Test-Driven Development (TDD)** to technika tworzenia oprogramowania sterowana przez testy:

* Utrzymujemy kompletny zestaw testów programisty (Programmer Tests)
* Kod nie powinien trafić do produkcji, jeśli nie ma powiązanych testów
* Najpierw piszemy testy
* Testy określają, jaki kod powinniśmy napisać

## Testy programisty

Testy programisty służą do sprawdzenia, czy klasa wykazuje prawidłowe zachowanie. Są tworzone przez programistę, który pisze kod do przetestowania.

Podobne do testów jednostkowych, ale tworzone z innego powodu:

* Testy jednostkowe tworzone są w celu sprawdzenia, czy napisany już kod działa
* Testy programisty definiują, co to znaczy, że kod działa

Testy programisty nazywane są w ten sposób również w celu odróżnienia od testów tworzonych przez klienta, których zadaniem jest sprawdzenie, czy system działa prawidłowo z punktu widzenia użytkownika.

Używanie TDD oznacza teoretycznie, że dysponujemy kompletnym zestawem testów. Dzieje się tak, ponieważ nie może istnieć kod, jeśli nie istnieje test, który ten kod powinien przejść. Piszemy test, a potem (i nie wcześniej) piszemy kod, który jest testowany przez ten test. W systemie nie powinien istnieć kod, który nie został napisany w odpowiedzi na test.

Jeśli mamy do zaimplementowania jakiś fragment funkcjonalności, to najpierw tworzymy kod, który sprawdzi, czy ta funkcjonalność działa zgodnie z wymaganiami, a dopiero potem implementujemy samą funkcjonalność. Tworzymy test, a następnie tylko tyle kodu, żeby test mógł przejść.

Testy określają, jaki kod powinniśmy napisać. Pisząc tylko kod wymagany do przejścia testu ograniczamy ilość kodu do napisania. Do weryfikacji testu tworzymy najprostszy, działający kod.

## Trzy prawa TDD

TDD zakłada pisanie testów jednostkowych na początku, przed napisanie kodu produkcyjnego. 

Możemy zdefiniować trzy podstawowe prawa TDD:

1. Nie można zacząć pisać kodu produkcyjnego przed zakończeniem pisania testu jednostkowego, który nie jest spełniony.
2. Kod testu jednostkowego powinien być tylko tak długi, aby wystarczył do niespełnienia testu, a błędna kompilacja jest jednocześnie nieudanym testem.
3. Nie można pisać większej ilości kodu produkcyjnego, niż jest wymagana do przejścia testu jednostkowego.

Te trzy prawa zamykają się w cyklu, który trwa prawdopodobnie kilkadziesiąt sekund. Testy i kod produkcyjny są pisane razem, przy czym testy są pisane kilka sekund wcześniej niż kod produkcyjny.

```{important}
Kod testów jest tak samo ważny, jak kod produkcyjny.
```