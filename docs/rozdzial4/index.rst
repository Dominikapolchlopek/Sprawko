Analiza bazy danych oraz optymalizacja zapytań
==============================
:Autor: - Dominika Półchłopek

Migracja danych między bazami SQLite i PostgreSQL
----------------


Migracja danych pomiędzy bazami PostgreSQL i SQLite wymaga dostosowania typów danych, ograniczeń oraz sposobu przechowywania niektórych informacji.

Migracja z SQLite do PostgreSQL
########

**Dostosowanie typów danych**
  * Zamiana typów ogólnych SQLite (`TEXT`, `REAL`, `INTEGER`) na bardziej restrykcyjne typy PostgreSQL (`VARCHAR`, `DECIMAL`, `SERIAL`, `JSONB`).
  * Przykład: `INTEGER PRIMARY KEY AUTOINCREMENT` w SQLite odpowiada `SERIAL PRIMARY KEY` w PostgreSQL.

**Eksport i import danych**
  * Eksport danych z SQLite do plików CSV lub dump SQL.
  * Import danych do PostgreSQL przy użyciu polecenia `COPY`, narzędzi GUI lub narzędzi migracyjnych, np. `pgloader`.

**Dostosowanie schematu**
  * Zamiana `produkty TEXT` na `produkty JSONB` w tabeli `Koszyki`.
  * Uzupełnienie ograniczeń i kluczy obcych zgodnie z wymaganiami PostgreSQL.

**Testowanie**
 * Weryfikacja poprawności danych, relacji i działania ograniczeń.

Migracja z PostgreSQL do SQLite
############

**Dostosowanie typów danych**
  * Zamiana typów specyficznych dla PostgreSQL (`SERIAL`, `VARCHAR`, `DECIMAL`, `JSONB`) na typy obsługiwane przez SQLite (`INTEGER PRIMARY KEY AUTOINCREMENT`, `TEXT`, `REAL`).
  * Przykład: `DECIMAL(10, 2)` w PostgreSQL zamieniamy na `REAL` w SQLite.

**Eksport i import danych**
  * Eksport danych z PostgreSQL do CSV lub dump SQL.
  * Import do SQLite przy użyciu polecenia `.import` lub narzędzi GUI.

**Dostosowanie schematu**
  * Zamiana `produkty JSONB` na `produkty TEXT` (przechowywanie danych w formacie JSON jako tekst).
  * Uproszczenie ograniczeń i kluczy obcych, ponieważ SQLite nie egzekwuje ich tak rygorystycznie jak PostgreSQL.

**Testowanie**
  * Weryfikacja poprawności migracji, szczególnie relacji i unikalności danych.

Wnioski
#####

- **Typy danych:** Należy zamieniać typy specyficzne dla jednej bazy na odpowiedniki w drugiej, np. `SERIAL` ↔ `INTEGER PRIMARY KEY AUTOINCREMENT`, `DECIMAL` ↔ `REAL`, `JSONB` ↔ `TEXT`.
- **Ograniczenia:** SQLite jest mniej restrykcyjny, dlatego część ograniczeń z PostgreSQL może wymagać uproszczenia lub ręcznego sprawdzenia po migracji.
- **Dane złożone:** W PostgreSQL można używać typu `JSONB`, w SQLite należy przechowywać dane w formacie tekstowym (JSON jako `TEXT`).
- **Proces migracji:** Najwygodniej używać narzędzi takich jak `pgloader` (SQLite → PostgreSQL) lub eksportować/importować dane przez CSV.
- **Testowanie:** Po migracji konieczne jest sprawdzenie poprawności danych, relacji i działania aplikacji.


Analiza i pomiar wydajności zapytań
###########

Najprostszym sposobem oceny wydajności kodu SQL jest **zmierzenie czasu wykonania poszczególnych zapytań**. W PostgreSQL można to zrobić na kilka sposobów:

1. **Pomiar czasu wykonania wybranych zapytań**
   
   - Można użyć narzędzi takich jak `EXPLAIN ANALYZE`, które pokazuje rzeczywisty czas wykonania zapytania oraz szczegóły planu wykonania.
   - Alternatywnie, można mierzyć czas „ręcznie” (np. za pomocą polecenia `\timing` w psql lub funkcji time w aplikacji).
   - Przykład porównania:
     ::
     
        EXPLAIN ANALYZE SELECT * FROM Produkty WHERE cena > 100;
        EXPLAIN ANALYZE SELECT * FROM Produkty WHERE nazwa LIKE 'A%';

   - Porównując czasy wykonania tych zapytań, można wskazać, które są bardziej wydajne i zidentyfikować potencjalne miejsca do optymalizacji.

2. **Wykorzystanie EXPLAIN do analizy wydajności zapytań**

   - Polecenie **EXPLAIN** pokazuje, jak PostgreSQL planuje wykonać zapytanie – wyświetla tzw. plan zapytania, czyli drzewo operacji (np. skanowanie sekwencyjne, skanowanie indeksu, złączenia, filtrowanie)[1][2][8].
   - **EXPLAIN ANALYZE** uruchamia zapytanie i pokazuje rzeczywisty czas wykonania oraz liczbę przetworzonych wierszy na każdym etapie planu.
   - Dzięki EXPLAIN można:
     - Rozpoznać, czy wykorzystywane są indeksy (Index Scan) czy pełne skanowanie tabeli (Seq Scan)
     - Sprawdzić typy złączeń (Nested Loop, Hash Join itp.)
     - Porównać szacowaną i rzeczywistą liczbę wierszy (co pozwala wykryć błędne statystyki)
     - Zidentyfikować kosztowne operacje, które warto zoptymalizować (np. sortowania, duże złączenia).

   - Przykład użycia:
     ::
     
        EXPLAIN SELECT * FROM Zamowienia WHERE status = 'zrealizowane';
        EXPLAIN ANALYZE SELECT * FROM PozycjeZamowienia WHERE ilosc > 10;

   - EXPLAIN pozwala także porównać różne warianty zapytań i wybrać najbardziej efektywny.

3. **Systematyczny opis wydajności i punkty kontroli**

   Aby efektywnie monitorować i opisywać wydajność zapytań w bazie danych, warto:

   - **Dokonywać regularnych pomiarów czasu wykonania kluczowych zapytań** (np. SELECT, INSERT, UPDATE, DELETE na dużych tabelach).
   - **Analizować plany zapytań EXPLAIN/EXPLAIN ANALYZE** dla najważniejszych operacji – identyfikować sekwencyjne skany, nieużywane indeksy, kosztowne złączenia.
   - **Monitorować kluczowe metryki systemowe**:
     - Zużycie CPU i pamięci przez PostgreSQL
     - Liczbę aktywnych połączeń i sesji
     - Liczbę zapytań na sekundę, czas odpowiedzi, throughput
     - Użycie i fragmentację indeksów oraz tabel
     - Wskaźnik trafień w cache (buffer cache hit ratio)
   - **Weryfikować statystyki i regularnie wykonywać ANALYZE oraz VACUUM** – nieaktualne statystyki mogą prowadzić do wyboru nieoptymalnych planów zapytań.
   - **Tworzyć i utrzymywać indeksy** na kolumnach często używanych w WHERE, JOIN, ORDER BY.
   - **Analizować logi PostgreSQL** pod kątem długotrwałych zapytań, błędów i deadlocków.

Wnioski
####

- Pomiar czasu wykonania zapytań oraz analiza planu wykonania (EXPLAIN/EXPLAIN ANALYZE) to podstawowe narzędzia oceny wydajności SQL w PostgreSQL.
- EXPLAIN pozwala zrozumieć, jak baza realizuje zapytanie i wskazuje miejsca do optymalizacji (np. brak indeksu, nieefektywne złączenie, kosztowny sort).
- Systematyczne monitorowanie wydajności powinno obejmować zarówno analizę pojedynczych zapytań, jak i obserwację kluczowych metryk systemowych i bazodanowych.
- Regularne testy, analiza planów zapytań, aktualizacja statystyk oraz optymalizacja indeksów pozwalają na wczesne wykrycie i usunięcie wąskich gardeł w bazie danych, co przekłada się na stabilność i wysoką wydajność całego systemu.

Dokumentacja kodu dla SQLuser_price
------------

.. code:: python

  def SQLuser_price(self) -> None:
      """
      Wyświetla użytkowników wraz z sumą ich zakupów.

      Zapytanie łączy dane z tabel:
      - Uzytkownicy (dane osobowe)
      - Zamowienia (historia zamówień)
      - PozycjeZamowienia (szczegóły zamówień)
      - Produkty (ceny)

      Zwraca:
          None: Wyniki są wyświetlane bezpośrednio w konsoli

      Uwagi:
          - Wyniki są posortowane malejąco według sumy zakupów
          - Zwracane są pełne imiona i nazwiska użytkowników
          - Wartości są formatowane z dwoma miejscami po przecinku
      """
        try:
            conn = self._get_connection()
            c = conn.cursor()
        
            sql = """
                SELECT 
                u.imie || ' ' || u.nazwisko AS nazwa_uzytkownika,
                SUM(p.cena * pz.ilosc) AS suma_zakupow
                FROM Uzytkownicy u
                JOIN Zamowienia z ON u.id_uzytkownika = z.id_uzytkownika
                JOIN PozycjeZamowienia pz ON z.id_zamowienia = pz.id_zamowienia
                JOIN Produkty p ON pz.id_produktu = p.id_produktu
                GROUP BY u.id_uzytkownika
                ORDER BY suma_zakupow DESC;
            """
        
            print("\nLista użytkowników wraz z sumą ich zakupów:")
            print("-" * 50)
            for row in c.execute(sql):
              print(f"{row[0]}: {row[1]:.2f} zł")
            
        except Exception as e:
            print(f"Błąd podczas wykonywania zapytania: {e}")
        finally:
            if 'conn' in locals():
              conn.close()
