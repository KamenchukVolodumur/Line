# 🚀 C# LINQ: Kompletny przewodnik i ściągawka (Z wizualizacją)

> **Language-Integrated Query (LINQ)** to wbudowane w język C# narzędzie umożliwiające operowanie na dowolnych kolekcjach danych (`List`, `Array`, bazy danych itp.) w sposób deklaratywny i czytelny.

---

## 📑 Spis treści
1. [Model myślowy: Jak działa LINQ](#1-model-myślowy-jak-działa-linq)
2. [Leniwe (Deferred) vs Natychmiastowe (Immediate) wykonywanie](#2-leniwe-deferred-vs-natychmiastowe-immediate-wykonywanie)
3. [Filtrowanie: `Where`, `OfType`, `DistinctBy`](#3-filtrowanie)
4. [Projekcja i rzutowanie: `Select`, `SelectMany`](#4-projekcja-i-rzutowanie)
5. [Sortowanie: `OrderBy`, `ThenBy`, `Reverse`](#5-sortowanie)
6. [Grupowanie: `GroupBy`, `ToLookup`](#6-grupowanie)
7. [Łączenie kolekcji: `Join`, `GroupJoin`, `Zip`](#7-łączenie-kolekcji-joins)
8. [Sprawdzanie warunków (bool): `Any`, `All`, `Contains`](#8-sprawdzanie-warunków-kwantyfikatory)
9. [Wybieranie pojedynczych elementów: `First`, `FirstOrDefault`, `Single`](#9-wybieranie-pojedynczych-elementów)
10. [Paginacja i podział: `Take`, `Skip`, `Chunk`](#10-paginacja-i-podział)
11. [Agregacja i obliczenia: `Count`, `Sum`, `MinBy`, `MaxBy`, `Aggregate`](#11-agregacja-i-obliczenia)
12. [Operacje na zbiorach: `Union`, `Intersect`, `Except`](#12-operacje-na-zbiorach)
13. [Konwersja do kolekcji: `ToList`, `ToDictionary`, `ToHashSet`](#13-konwersja-do-kolekcji)
14. [⚡️ Wielka tabela podsumowująca](#14-wielka-tabela-podsumowująca)
15. [Formatowanie stringów (`string.Format`, `$""`, maski)](#15-formatowanie-stringów-stringformat--i-maski-formatowania)
16. [Kompletny przegląd metod LINQ na przykładach (Lab 05 + .NET 8–10)](#16-kompletny-przegląd-metod-linq-na-przykładach-lab-05--net-810)
17. [Delegaty, Wyrażenia Lambda i Domknięcia (Closures)](#17-delegaty-wyrażenia-lambda-i-domknięcia-closures)
18. [Własne iteratory (`yield`), Enumerator i Bloki rozszerzające (C# 14)](#18-własne-iteratory-yield-enumerator-i-bloki-rozszerzające-c-14)

---

## 1. Model myślowy: Jak działa LINQ

Wyobraź sobie LINQ jako **taśmociąg w fabryce**. Dane wchodzą na początku i przechodzą przez kolejne stacje obróbki:

```mermaid
flowchart LR
    A["Kolekcja wejściowa (100 elementów)"] --> B["Where: Filtr (zostaje 20)"]
    B --> C["OrderBy: Sortowanie"]
    C --> D["Select: Rzutowanie na nowe pola"]
    D --> E["Take (5): Pierwsze 5"]
    E --> F["Wynik końcowy (5 elementów)"]
```

---

## 2. Leniwe (Deferred) vs Natychmiastowe (Immediate) wykonywanie

To najważniejsza koncepcja w LINQ:

* 🦥 **Metody leniwe (Deferred Execution):** zwracają `IEnumerable<T>`. **NIE wykonują pracy od razu**, a jedynie budują „plan zapytania”. Obliczenia ruszają dopiero wtedy, gdy zaczniesz iterować pętlą `foreach` lub wywołasz metodę natychmiastową.
  > *Przykłady:* `Where`, `Select`, `SelectMany`, `OrderBy`, `GroupBy`, `Take`, `Skip`.

* ⚡️ **Metody natychmiastowe (Immediate Execution):** zwracają konkretną wartość lub tworzą strukturę w pamięci RAM. Wykonują się **natychmiast w miejscu wywołania**.
  > *Przykłady:* `ToList()`, `ToArray()`, `Count()`, `Sum()`, `First()`, `Any()`.

---

## 3. Filtrowanie

### `.Where()` — Filtruje elementy według warunku
Przepuszcza tylko te elementy, dla których wyrażenie lambda zwraca `true`.

* **Wejście:** `[1, 2, 3, 4, 5, 6]`
* **Kod:**
```csharp
var evens = numbers.Where(x => x % 2 == 0);
```
* **Wynik:** `[2, 4, 6]`

---

### `.OfType<T>()` — Filtruje według typu danych
Wybiera z kolekcji tylko obiekty określonego typu i automatycznie rzutuje je na ten typ.

* **Wejście:** `object[] mixed = [1, "hello", 2, "world", 3.14]`
* **Kod:**
```csharp
var stringsOnly = mixed.OfType<string>();
```
* **Wynik:** `["hello", "world"]`

---

### `.Distinct()` oraz `.DistinctBy()` — Usuwa duplikaty
* `.Distinct()` usuwa identyczne elementy.
* `.DistinctBy(x => x.Pole)` usuwa duplikaty na podstawie wybranego klucza (.NET 6+).

* **Wejście:** `[1, 2, 2, 3, 3, 3]`
* **Kod:**
```csharp
var unique = numbers.Distinct();
```
* **Wynik:** `[1, 2, 3]`

---

## 4. Projekcja i rzutowanie

### `.Select()` — Przekształca każdy element (1 do 1)
Bierze każdy element i przekształca go w coś innego (nową wartość lub instancję klasy anonimowej).

* **Wejście:** `["apple", "banana"]`
* **Kod:**
```csharp
var lengths = fruits.Select(f => f.Length);
```
* **Wynik:** `[5, 6]`

**Tworzenie klasy anonimowej:**
```csharp
var usersDto = users.Select(u => new 
{ 
    u.Id, 
    FullName = $"{u.FirstName} {u.LastName}" 
});
```

---

### `.SelectMany()` — Spłaszcza zagnieżdżone kolekcje (1 do Wielu)
Gdy każdy element posiada własną listę podrzędną, `SelectMany` scala je wszystkie w **jedną wspólną, płaską listę**.

* **Wejście:** `[["apple", "banana"], ["orange", "kiwi"]]`
* **Kod:**
```csharp
var flatList = baskets.SelectMany(b => b);
```
* **Wynik:** `["apple", "banana", "orange", "kiwi"]`

**Praktyczny przykład (rozbijanie gatunków filmów `"Horror|Comedy"`):**
```csharp
// Filmy mają właściwość Genres: "Action|Sci-Fi", "Comedy"
var allGenres = movies.SelectMany(m => m.Genres.Split('|'));
// Wynik: ["Action", "Sci-Fi", "Comedy"]
```

#### `SelectMany` z drugim callbackiem (zachowanie elementu rodzica!)
Domyślnie `SelectMany(x => x.Children)` gubi obiekt rodzica (`x`). Jeśli w wyniku potrzebujesz **zarówno rodzica, jak i każdego jego pod-elementu**, przekaż drugi parametr (`resultSelector`): `(rodzic, dziecko) => wynik`.

```csharp
// Każdy artysta ma listę gatunków (artist.Genres)
var artistGenres = artists.SelectMany(
    artist => artist.Genres,                   // 1. kolekcja podrzędna
    (artist, genre) => (artist.Name, genre)    // 2. callback łączący rodzica z dzieckiem
);

// W składni zapytań (Query Syntax) to dokładnie dwie klauzule from:
var artistGenresQuery =
    from artist in artists
    from genre in artist.Genres
    select (artist.Name, genre);
```

> [!TIP]
> **Warianty z indeksem w LINQ:** Do metod `Select` i `Where` również można przekazać drugi parametr z indeksem elementu:
> ```csharp
> var numbered = items.Select((item, index) => $"{index}: {item}");
> var evenPos  = items.Where((item, index) => index % 2 == 0);
> ```

---

## 5. Sortowanie

> [!IMPORTANT]
> Pierwsze kryterium sortowania zawsze ustalamy za pomocą `OrderBy`. Każde kolejne dodatkowe kryterium ustalamy za pomocą **`ThenBy`**! Użycie drugiego `OrderBy` zresetuje poprzednie sortowanie!

### `.OrderBy()` oraz `.OrderByDescending()` — Sortowanie główne
* **Wejście:** `[5, 1, 4, 2]`
* **Kod:**
```csharp
var asc = numbers.OrderBy(x => x);              // Rosnąco  -> [1, 2, 4, 5]
var desc = numbers.OrderByDescending(x => x);   // Malejąco -> [5, 4, 2, 1]
```

### `.ThenBy()` oraz `.ThenByDescending()` — Sortowanie pomocnicze
Używane, gdy wartości z pierwszego sortowania są identyczne.

* **Wejście:** `[("Jan", 25), ("Piotr", 30), ("Jan", 20)]`
* **Kod:**
```csharp
var sorted = people
    .OrderBy(p => p.Name)     // 1. Najpierw po imieniu
    .ThenBy(p => p.Age);      // 2. Jeśli imię to samo -> po wieku
```
* **Wynik:** `[("Jan", 20), ("Jan", 25), ("Piotr", 30)]`

---

## 6. Grupowanie

### `.GroupBy()` — Dzielenie na grupy („pudełka”)
Dzieli elementy na grupy na podstawie wybranego klucza. Każda grupa `g` posiada:
* **`g.Key`** — wartość, według której grupowano (etykieta pudełka).
* Samą grupę **`g`** jako podkolekcję elementów znajdujących się w tym pudełku.

* **Wejście:** `["cat", "dog", "car", "apple"]`
* **Kod:**
```csharp
var groups = words.GroupBy(w => w[0]); // grupujemy po pierwszej literze

foreach (var g in groups)
{
    Console.WriteLine($"Litera '{g.Key}': {string.Join(", ", g)} (liczba: {g.Count()})");
}
```
* **Wynik:**
  * Litera `'c'`: `cat, car` (liczba: 2)
  * Litera `'d'`: `dog` (liczba: 1)
  * Litera `'a'`: `apple` (liczba: 1)

---

## 7. Łączenie kolekcji (Joins)

### `.Join()` — Złączenie wewnętrzne (Inner Join)
Łączy dwa zbiory na podstawie wspólnego identyfikatora (klucza). Jeśli element z jednej kolekcji nie ma pary w drugiej — zostaje odrzucony.

#### 📌 Szczegółowy rozkład parametrów:
```csharp
collection1.Join(
    collection2,                                  // 1. Druga kolekcja, z którą łączymy
    person => person.Id,                          // 2. Klucz pierwszej kolekcji (Outer Key)
    product => product.PersonId,                  // 3. Klucz drugiej kolekcji (Inner Key)
    (person, product) => $"{person.Name} bought {product.Name}" // 4. Rzutowanie wynikowej pary
);
```

#### Przykład z danymi:
* **Wejście:**
  * `People`: `[(Id: 1, "Jan"), (Id: 2, "Anna")]`
  * `Products`: `[(PersonId: 1, "Phone"), (PersonId: 1, "TV"), (PersonId: 3, "Bike")]`
* **Kod:**
```csharp
var purchases = people.Join(
    products,
    person => person.Id,
    product => product.PersonId,
    (person, product) => $"{person.Name} bought {product.Name}"
);
```
* **Wynik (Płaska lista par):** 
  ```text
  [
      "Jan bought Phone",
      "Jan bought TV"
  ]
  ```
  *(Anna odpada, bo nic nie kupiła; Bike odpada, bo PersonId: 3 nie ma w tabeli People).*

#### Porównanie: Składnia metod vs Składnia zapytań (Query Syntax)
```csharp
// Składnia metod (Method Syntax):
collection1.Join(collection2, p => p.Id, prod => prod.PersonId, (p, prod) => $"{p.Name} bought {prod.Name}");

// Składnia zapytań SQL (Query Syntax):
from person in collection1
join product in collection2 on person.Id equals product.PersonId
select $"{person.Name} bought {product.Name}";
```

---

### `.GroupJoin()` — Złączenie lewostronne (Left Outer Join)
W przeciwieństwie do `Join`, **`GroupJoin` nie odrzuca nikogo z pierwszej tabeli**! Zamiast płaskich par przypisuje do każdego elementu **całą listę pasujących rekordów** (nawet pustą).

```csharp
var result = people.GroupJoin(
    products,
    person => person.Id,
    product => product.PersonId,
    (person, userProducts) => new 
    { 
        person.Name, 
        Count = userProducts.Count(),
        Items = userProducts 
    }
);
```

#### Wynik danych w pamięci:
```text
[
    {
        Name = "Jan",
        Count = 2,
        Items = [ "Phone", "TV" ]
    },
    {
        Name = "Anna",
        Count = 0,
        Items = [ ]                 <-- pusta lista, ale Anna NIE została usunięta!
    }
]
```

---

### `.Zip()` — Łączenie parami (suwak)
Łączy odpowiadające sobie elementy z dwóch list (1. z 1., 2. z 2.).

* **Wejście:** `[1, 2, 3]` oraz `["A", "B", "C"]`
* **Kod:**
```csharp
var pairs = numbers.Zip(letters, (n, l) => $"{n}-{l}");
```
* **Wynik:** `["1-A", "2-B", "3-C"]`

---

## 8. Sprawdzanie warunków (Kwantyfikatory)

Zwracają wartość typu **`bool`**:

| Metoda | Co sprawdza | Przykład na `[2, 4, 6]` | Wynik |
| :--- | :--- | :--- | :--- |
| **`.Any(predykat)`** | Czy **chociaż jeden** element spełnia warunek? | `nums.Any(x => x > 5)` | `true` (jest 6) |
| **`.Any()`** | Czy kolekcja **nie jest pusta**? | `nums.Any()` | `true` |
| **`.All(predykat)`** | Czy **wszystkie** elementy spełniają warunek? | `nums.All(x => x % 2 == 0)` | `true` |
| **`.Contains(wartość)`**| Czy kolekcja zawiera dany element? | `nums.Contains(4)` | `true` |

---

## 9. Wybieranie pojedynczych elementów

| Metoda | Działanie | Gdy nic nie znaleziono |
| :--- | :--- | :--- |
| **`First()`** | Zwraca pierwszy pasujący element | 💥 Rzuca `InvalidOperationException` |
| **`FirstOrDefault()`** | Zwraca pierwszy element lub domyślny (`null`/`0`) | ✅ Zwraca wartość domyślną (`null` / `0`) |
| **`Single()`** | Zwraca jedyny pasujący element (musi być dokładnie 1!) | 💥 Błąd, jeśli znaleziono 0 lub > 1 |
| **`SingleOrDefault()`**| Zwraca jedyny element lub `null` | 💥 Błąd, jeśli znaleziono > 1 |
| **`Last()` / `LastOrDefault()`** | Zwraca ostatni pasujący element | Analogicznie jak First |

* **Przykład bezpiecznego pobierania:**
```csharp
var user = database.Users.FirstOrDefault(u => u.Id == 999);
if (user != null) 
{ 
    // użytkownik istnieje
}
```

---

## 10. Paginacja i podział

### `.Skip()` oraz `.Take()` — Pomiń i Pobierz
Klasyczny mechanizm dzielenia wyników na strony (paginacja).

* **Wejście:** `[10, 20, 30, 40, 50]`
* **Kod:**
```csharp
var page = numbers.Skip(2).Take(2); // pomiń 2, weź kolejne 2
```
* **Wynik:** `[30, 40]`

**Wzór na stronę `pageNumber` o rozmiarze `pageSize`:**
```csharp
var itemsOnPage = allItems.Skip((pageNumber - 1) * pageSize).Take(pageSize);
```

---

### `.Chunk(rozmiar)` — Podział na równe paczki (.NET 6+)
* **Wejście:** `[1, 2, 3, 4, 5]`
* **Kod:**
```csharp
var chunks = numbers.Chunk(2);
```
* **Wynik:** `[[1, 2], [3, 4], [5]]`

---

## 11. Agregacja i obliczenia

* **`.Count()`** — liczba elementów: `numbers.Count()`
* **`.Sum()`** — suma wartości: `numbers.Sum()`
* **`.Average()`** — średnia arytmetyczna: `numbers.Average()`
* **`.Min()` / `.Max()`** — wartość minimalna i maksymalna: `numbers.Min()`
* **`.MinBy(x => x.Age)`** (.NET 6+) — zwraca **cały obiekt**, który ma najmniejszą wartość danego pola:
  ```csharp
  var youngestUser = users.MinBy(u => u.Age); // zwraca instancję User!
  ```
* **`.MaxBy(x => x.Salary)`** — zwraca cały obiekt o największej wartości pola.

---

### `.Aggregate()` — Akumulacja / Redukcja (Fold / Reduce)
Przetwarza sekwencję krok po kroku za pomocą akumulatora:

* **Wejście:** `[1, 2, 3, 4]`
* **Kod:**
```csharp
// (seed, (acc, item) => ...)
int factorial = numbers.Aggregate(1, (acc, n) => acc * n);
```
* **Wynik:** `1 * 1 * 2 * 3 * 4 = 24`

---

## 12. Operacje na zbiorach

| Metoda | Działanie | Wejście A: `[1, 2]`, Wejście B: `[2, 3]` | Wynik |
| :--- | :--- | :--- | :--- |
| **`Union`** | Suma zbiorów (bez duplikatów) | `A.Union(B)` | `[1, 2, 3]` |
| **`Intersect`** | Część wspólna (przecięcie) | `A.Intersect(B)` | `[2]` |
| **`Except`** | Różnica zbiorów (elementy w A, których nie ma w B) | `A.Except(B)` | `[1]` |

---

## 13. Konwersja do kolekcji (Immediate)

Metody wymuszające natychmiastowe wykonanie zapytania i utrwalenie wyników w strukturach pamięci:

* **`ToList()`** $\rightarrow$ `List<T>`
* **`ToArray()`** $\rightarrow$ tablica `T[]`
* **`ToHashSet()`** $\rightarrow$ zbiór unikalnych wartości `HashSet<T>`
* **`ToDictionary()`** $\rightarrow$ słownik `Dictionary<TKey, TValue>`:
  ```csharp
  // Tworzymy słownik mapujący [Id -> Imię]:
  Dictionary<int, string> userMap = users.ToDictionary(u => u.Id, u => u.Name);
  ```

---

## 14. Wielka tabela podsumowująca

| Kategoria | Metoda | Działanie | Typ wykonania |
| :--- | :--- | :--- | :--- |
| **Filtrowanie** | `Where` | Filtruje elementy według predykatu | 🦥 Leniwe (Deferred) |
| | `Distinct` / `DistinctBy` | Eliminuje zduplikowane wartości | 🦥 Leniwe (Deferred) |
| | `OfType<T>` | Wybiera elementy określonego typu | 🦥 Leniwe (Deferred) |
| **Projekcja** | `Select` | Przekształca każdy element (1 $\rightarrow$ 1) | 🦥 Leniwe (Deferred) |
| | `SelectMany` | Spłaszcza kolekcje zagnieżdżone (1 $\rightarrow$ Wiele) | 🦥 Leniwe (Deferred) |
| **Sortowanie** | `OrderBy` / `OrderByDescending` | Sortowanie główne po kluczu | 🦥 Leniwe (Deferred) |
| | `ThenBy` / `ThenByDescending` | Sortowanie dodatkowe (w przypadku remisów) | 🦥 Leniwe (Deferred) |
| | `Reverse` | Odwraca kolejność elementów | 🦥 Leniwe (Deferred) |
| **Grupowanie** | `GroupBy` | Dzieli elementy na grupy o wspólnym kluczu | 🦥 Leniwe (Deferred) |
| **Łączenie** | `Join` | Złączenie wewnętrzne dwóch tabel (Inner Join) | 🦥 Leniwe (Deferred) |
| | `GroupJoin` | Złączenie lewostronne / hierarchiczne (Left Join) | 🦥 Leniwe (Deferred) |
| | `Zip` | Łączy odpowiadające elementy z dwóch sekwencji | 🦥 Leniwe (Deferred) |
| **Paginacja** | `Take` / `Skip` | Pobiera / pomija pierwsze N elementów | 🦥 Leniwe (Deferred) |
| | `Chunk` | Dzieli na tablice o stałej wielkości | 🦥 Leniwe (Deferred) |
| **Elementy** | `First` / `FirstOrDefault` | Pierwszy element pasujący do warunku | ⚡️ Natychmiastowe |
| | `Single` / `SingleOrDefault`| Jedyny element (błąd, jeśli jest ich więcej) | ⚡️ Natychmiastowe |
| **Sprawdzanie** | `Any` / `All` / `Contains` | Sprawdza warunki logiczne, zwraca `bool` | ⚡️ Natychmiastowe |
| **Agregacja** | `Count` / `Sum` / `Average` | Podstawowe obliczenia matematyczne | ⚡️ Natychmiastowe |
| | `Min` / `Max` / `MinBy` / `MaxBy`| Znajdowanie wartości skrajnych | ⚡️ Natychmiastowe |
| | `Aggregate` | Własna operacja akumulacji (odpowiednik Reduce) | ⚡️ Natychmiastowe |
| **Zbiory** | `Union` / `Intersect` / `Except` | Klasyczne operacje na zbiorach | 🦥 Leniwe (Deferred) |
| **Kolekcje** | `ToList` / `ToArray` / `ToDictionary` | Zapisuje wynik w pamięci RAM | ⚡️ Natychmiastowe |


---

## 15. Formatowanie stringów (`string.Format`, `$""` i maski formatowania)

Kiedy trzeba ładnie sformatować liczby, daty albo wyrównać kolumny w konsoli (jak `"{0,6:F2}"` czy `$"{index,3}: {elem}"`), w C# działa jeden wspólny mechanizm: **Composite Formatting**.

### Składnia placeholdera: `{indeks,szerokość:format}`

Każdy element w klamrach składa się z trzech części (dwie ostatnie są opcjonalne):

```
{ 0 , 6 : F2 }
  │   │    └── maska formatu (np. F2 = 2 miejsca po przecinku)
  │   └─────── szerokość kolumny i wyrównanie (+ do prawej, - do lewej)
  └─────────── indeks argumentu w string.Format (lub zmienna w $"...")
```

### `string.Format` vs interpolacja `$""`

| Sposób | Przykład | Kiedy używać |
| :--- | :--- | :--- |
| **`string.Format`** | `string.Format("\|{0,6:F2}\|", 5)` $\rightarrow$ `"\|  5.00\|"` | Gdy format przychodzi jako zmienna `string format` (np. parametr metody) |
| **Interpolacja `$""`** | `$"{index,3}: {elem}"` $\rightarrow$ `"  0: 1"` | Gdy piszesz tekst bezpośrednio w kodzie |

> **Uwaga:** Nie mieszaj ich razem w stylu `string.Format($"{format}", x)` — jeśli `format` to już `"{0,6:F2}"`, pisz po prostu `string.Format(format, x)`.

### Wyrównanie (Alignment)

* **`{0,5}`** (liczba dodatnia) $\rightarrow$ minimalna szerokość **5 znaków**, wyrównanie **do prawej** (`"   42"`).
* **`{0,-5}`** (liczba ujemna) $\rightarrow$ minimalna szerokość **5 znaków**, wyrównanie **do lewej** (`"42   "`).
* Jeśli tekst jest dłuższy niż podana szerokość, nie zostanie ucięty — po prostu zajmie tyle miejsca, ile potrzebuje.

### Najpopularniejsze maski formatowania (`:format`)

| Maska | Znaczenie | Przykład (`x = 5` lub `1234.567`) | Wynik |
| :--- | :--- | :--- | :--- |
| **`F2`** | Liczba stałoprzecinkowa (2 miejsca po `.`) | `string.Format("{0:F2}", 5)` | `"5.00"` |
| **`D4`** | Liczba całkowita z zerami wiodącymi | `$"{42:D4}"` | `"0042"` |
| **`N2`** | Liczba z separatorami tysięcy i 2 miejscami | `$"{1234.567:N2}"` | `"1,234.57"` |
| **`P1`** | Procenty (mnoży przez 100 i dodaje `%`) | `$"{0.256:P1}"` | `"25.6%"` |
| **`X`** | System szesnastkowy (hex) | `$"{255:X}"` | `"FF"` |
| **`yyyy-MM-dd HH:mm`** | Data i czas (`DateTime`) | `$"{dt:yyyy-MM-dd HH:mm}"` | `"2025-01-02 07:37"` |
| **`m\:ss`** | Czas trwania (`TimeSpan` — uwaga na `\:`!) | `$"{ts:m\\:ss}"` | `"3:31"` |

### Łączenie kolekcji w jeden napis (`string.Join`)

`string.Join(separator, kolekcja)` skleja elementy w jeden `string`, wstawiając `separator` pomiędzy nie (bez zbędnego przecinka na końcu):

```csharp
var formatted = SeqFormat(Enumerable.Range(5, 5), "|{0,6:F2}|");
Console.WriteLine(string.Join("~", formatted));
// Wynik: |  5.00|~|  6.00|~|  7.00|~|  8.00|~|  9.00|
```


---

## 16. Kompletny przegląd metod LINQ na przykładach (Lab 05 + .NET 8–10)

Poniżej znajduje się ściąga ze **wszystkich** metod omawianych na laboratoriach (w tym nowych z .NET 9/10), z krótkim przykładem działania.

> [!WARNING]
> **Pułapka wielokrotnej enumeracji:** Jeśli wywołasz na tym samym zapytaniu `query.Any()`, a potem `query.First()`, zapytanie wykona się **dwa razy**! Jeśli chcesz użyć wyniku wielokrotnie, najpierw zmaterializuj go przez `.ToList()` lub `.ToArray()`.
> **Nieskończone sekwencje:** Nigdy nie używaj `OrderBy`, `Reverse`, `Count`, `Sum` ani `ToList` na nieskończonych sekwencjach (np. `InfiniteSequence`) bez wcześniejszego `.Take(n)`!

---

### A. Operacje na zbiorach (Set operations)

Załóżmy: `a = [1, 2, 2, 3]`, `b = [2, 3, 4]`.

| Metoda | Przykład | Wynik | Co robi |
| :--- | :--- | :--- | :--- |
| **`Concat`** | `a.Concat(b)` | `[1, 2, 2, 3, 2, 3, 4]` | Skleja dwie sekwencje (zachowuje duplikaty) |
| **`Union`** | `a.Union(b)` | `[1, 2, 3, 4]` | Suma zbiorów (usuwa duplikaty) |
| **`UnionBy`** | `users1.UnionBy(users2, u => u.Id)` | unikalne wg `Id` | Suma zbiorów porównywana po kluczu |
| **`Intersect`** | `a.Intersect(b)` | `[2, 3]` | Część wspólna (elementy w obu) |
| **`IntersectBy`** | `users.IntersectBy(ids, u => u.Id)` | userzy o podanych `Id` | Część wspólna po kluczu |
| **`Except`** | `a.Except(b)` | `[1]` | Różnica zbiorów (`a \ b`) |
| **`ExceptBy`** | `tracks.ExceptBy(playedIds, t => t.Id)` | nieodtworzone utwory | Różnica po kluczu (świetne do "czego nie ma") |
| **`Distinct`** | `a.Distinct()` | `[1, 2, 3]` | Usuwa duplikaty |
| **`DistinctBy`** | `users.DistinctBy(u => u.City)` | po 1 osobie z miasta | Usuwa duplikaty według klucza |

---

### B. Partycjonowanie (krojenie sekwencji)

Załóżmy: `nums = [1, 2, 5, 2, 1]`.

| Metoda | Przykład | Wynik | Co robi |
| :--- | :--- | :--- | :--- |
| **`Take(n)`** | `nums.Take(3)` | `[1, 2, 5]` | Bierze pierwsze `n` elementów |
| **`TakeLast(n)`** | `nums.TakeLast(2)` | `[2, 1]` | Bierze ostatnie `n` elementów |
| **`TakeWhile(p)`** | `nums.TakeWhile(x => x < 4)` | `[1, 2]` | Bierze dopóki warunek jest prawdziwy (przerywa na `5`!) |
| **`Skip(n)`** | `nums.Skip(2)` | `[5, 2, 1]` | Pomija pierwsze `n` elementów |
| **`SkipLast(n)`** | `nums.SkipLast(2)` | `[1, 2, 5]` | Pomija ostatnie `n` elementów |
| **`SkipWhile(p)`** | `nums.SkipWhile(x => x < 4)` | `[5, 2, 1]` | Pomija dopóki warunek jest prawdziwy (od `5` bierze już wszystko) |
| **`Chunk(size)`** | `nums.Chunk(2)` | `[[1, 2], [5, 2], [1]]` | Dzieli na paczki (tablice) o zadanym rozmiarze |

---

### C. Łączenie (Join, GroupJoin, LeftJoin, RightJoin, Zip)

```csharp
// 1. Join (INNER JOIN) — tylko dopasowane pary
var inner = plays.Join(tracks, p => p.TrackId, t => t.Id, (p, t) => (p, t));

// 2. GroupJoin (1-do-wielu) — każdy element z lewej + kolekcja pasujących z prawej (może być pusta)
var withCounts = tracks.GroupJoin(plays, t => t.Id, p => p.TrackId,
    (track, trackPlays) => (track.Title, Count: trackPlays.Count()));

// 3. LeftJoin (.NET 10) — każdy z lewej; jeśli brak dopasowania, prawy = null (default)
var left = artists.LeftJoin(plays, a => a.Id, p => p.ArtistId,
    (artist, play) => (artist.Name, Play: play)); // play == null, gdy brak odtworzeń

// 4. RightJoin (.NET 10) — odwrotnie: każdy z prawej, lewy może być null
var right = artists.RightJoin(plays, a => a.Id, p => p.ArtistId,
    (artist, play) => (ArtistName: artist?.Name, play));

// 5. Zip — łączy "suwakiem" element po elemencie: [1,2] + ["a","b"] -> [(1,"a"), (2,"b")]
var pairs = new[] { 1, 2 }.Zip(new[] { "a", "b" });
```

---

### D. Sortowanie i tasowanie

| Metoda | Przykład | Co robi |
| :--- | :--- | :--- |
| **`Order()` / `OrderDescending()`** | `nums.Order()` | Sortuje bezpośrednio po wartości elementu (bez `x => x`) |
| **`OrderBy` / `OrderByDescending`** | `users.OrderBy(u => u.Age)` | Sortuje po wybranym kluczu |
| **`ThenBy` / `ThenByDescending`** | `.OrderBy(u => u.Age).ThenBy(u => u.Name)` | Drugie (kolejne) kryterium przy remisach |
| **`Reverse()`** | `nums.Reverse()` | Odwraca kolejność elementów |
| **`Shuffle()`** *(.NET 10)* | `nums.Shuffle()` | Losowo tasuje elementy sekwencji |

---

### E. Wybór pojedynczych elementów i kwantyfikatory

| Metoda | Gdy jest wynik | Gdy pusto | Gdy > 1 element |
| :--- | :--- | :--- | :--- |
| **`First()` / `FirstOrDefault()`** | pierwszy | wyjątek / `default` | pierwszy |
| **`Last()` / `LastOrDefault()`** | ostatni | wyjątek / `default` | ostatni |
| **`Single()` / `SingleOrDefault()`** | jedyny | wyjątek / `default` | **wyjątek!** |
| **`ElementAt(i)` / `ElementAtOrDefault(i)`** | element na indeksie `i` (obsługuje `^1`!) | wyjątek / `default` | element na `i` |
| **`DefaultIfEmpty(val)`** | oryginalna sekwencja | `[val]` (lub `[default]`) | oryginalna sekwencja |
| **`All` / `Any` / `Contains`** | `bool`: czy wszystkie / czy którykolwiek / czy zawiera konkretny element | | |

---

### F. Agregacja, grupowanie i generowanie

```csharp
// Aggregate — własna redukcja (np. iloczyn elementów: 1 * 2 * 3 * 4 = 24)
int product = new[] { 1, 2, 3, 4 }.Aggregate(1, (acc, x) => acc * x);

// AggregateBy (.NET 9+) — grupowanie i agregacja w jednym kroku (zwraca KeyValuePair<TKey, TAccumulate>)
var sumByCategory = items.AggregateBy(
    keySelector: i => i.Category,
    seed: 0m,
    func: (total, item) => total + item.Price);

// MinBy / MaxBy — zwraca cały obiekt, a nie samą wartość liczbową
var oldest = users.MaxBy(u => u.Age);

// SequenceEqual — porównuje dwie sekwencje element po elemencie (kolejność ma znaczenie!)
bool same = list1.SequenceEqual(list2);

// ToLookup — natychmiastowy słownik "klucz -> wiele elementów" (bezpieczny odczyt brakującego klucza zwraca pustą kolekcję!)
ILookup<string, User> byCity = users.ToLookup(u => u.City);
var warsawUsers = byCity["Warszawa"]; // nie rzuci wyjątku, nawet jeśli brak klucza

// Generowanie sekwencji:
var empty = Enumerable.Empty<int>();                  // []
var range = Enumerable.Range(1, 5);                   // [1, 2, 3, 4, 5]
var rep   = Enumerable.Repeat("hi", 3);               // ["hi", "hi", "hi"]
var prep  = nums.Prepend(0);                          // dodaje 0 na początek (Append na koniec)
var inf   = Enumerable.InfiniteSequence(0, x => x + 2); // (.NET 10) 0, 2, 4, 6, ... (użyj z .Take(n)!)
```

---

### G. Składnia zapytań (Query Syntax): `let` i `into`

* **`let`** — zapisuje zmienną pomocniczą wewnątrz zapytania, żeby nie liczyć jej dwa razy:
  ```csharp
  var res = from x in source
            let y = x.Price * 1.23m
            where y >= 100
            select new { x.Name, Brutto = y };
  ```
* **`into`** — przekazuje wynik `group` lub `join` do dalszego przetwarzania (np. `GroupJoin`):
  ```csharp
  var res = from a in artists
            join p in plays on a.Id equals p.ArtistId into artistPlays
            select new { a.Name, Count = artistPlays.Count() };
  ```
* **Łączenie obu składni:** `(from n in numbers where n > 5 select n).Take(3).ToList()`

---

### H. PLINQ (`AsParallel`) i Zakresy (`Index`, `Range`)

* **PLINQ** (zrównoleglenie obliczeń CPU dla dużych kolekcji w pamięci):
  ```csharp
  var sum = Enumerable.Range(0, 1_000_000)
      .AsParallel()        // dzieli pracę na wątki (nie wywołuj wielokrotnie!)
      .AsOrdered()         // opcjonalnie: zachowuje pierwotną kolejność (wolniejsze)
      .Where(x => ExpensiveCheck(x))
      .AsSequential()      // powrót do trybu jednowątkowego dla dalszych kroków
      .Sum();
  ```
  *(Do szybkiej akcji równoległej na każdym elemencie: `.ForAll(x => ...)`)*

* **Zakresy (`..`, `^`) vs LINQ:**
  Dla tablic `int[]`, `string` oraz `List<T>` zwykłe zakresy są **znacznie szybsze** i nie alokują enumeratorów:
  * `arr[..5]` zamiast `arr.Take(5).ToArray()`
  * `s[1..4]` zamiast `string.Concat(s.Skip(1).Take(3))`
  * `arr[^1]` — ostatni element (`Index` od końca)


---

## 17. Delegaty, Wyrażenia Lambda i Domknięcia (Closures)

### Co to jest delegat?
Delegat to **typ reprezentujący referencję do metody** (bezpieczny odpowiednik wskaźnika na funkcję). Pozwala przekazywać funkcje jako argumenty, zapisywać je w polach/zmiennych i zwracać z innych metod.

### Wbudowane typy delegat (nie trzeba tworzyć własnych `delegate`)

| Typ delegata | Sygnatura | Przykład |
| :--- | :--- | :--- |
| **`Action<T1, ...>`** | zwraca `void` | `Action<string, int> print = (label, val) => Console.WriteLine($"{label,-30}{val,6}");` |
| **`Func<T1, ..., TResult>`** | ostatni typ to wynik | `Func<Play, bool> isLong = p => p.MsPlayed > 60_000;` |
| **`Predicate<T>`** | zawsze `T -> bool` | `Predicate<Play> isSkip = p => p.Duration < TimeSpan.FromSeconds(30);` |
| **`Comparison<T>`** | `(T, T) -> int` | `Comparison<int> cmp = (a, b) => a.CompareTo(b);` |

### Delegaty wielokrotne (`+=`, `-=`)
Do zmiennej delegata można dopinać wiele metod:
```csharp
Action log = () => Console.WriteLine("one");
log += () => Console.WriteLine("two");
log(); // wypisze "one", a potem "two"
```
> [!NOTE]
> Jeśli delegat zwraca wartość (np. `Func`), wywołanie wielokrotne zwraca wynik **tylko ostatniej** przypisanej metody.

---

### Sposoby tworzenia delegatów

1. **Grupa metod (Method Group)** — przekazanie nazwy istniejącej metody:
   ```csharp
   Func<Play, bool> f = Filters.IsSkip;
   ```
2. **Wyrażenie lambda** — funkcja anonimowa pisana w locie:
   ```csharp
   Func<Play, bool> f = p => p.MsPlayed < 1000;
   ```
3. **Lambda zapisana w polu klasy (`static readonly`)** — zachowuje się jak metoda, ale jest już gotowym obiektem delegata:
   ```csharp
   // Modyfikator `static` przed lambdą ZABRANIA przechwytywania zmiennych z zewnątrz
   // (chroni przed ukrytymi alokacjami na stercie; przy próbie użycia zmiennej lokalnej -> błąd CS8820/CS8821)
   public static readonly Func<Play, bool> AtNight =
       static play => play.Timestamp.Hour is >= 22 or < 6;
   ```

---

### Domknięcia (Closures) i Fabryki delegatów
Gdy lambda używa zmiennej lub parametru z otaczającej ją metody, nazywamy ją **domknięciem (closure)**. Metoda zwracająca taką lambdę działa jak **fabryka filtrów**:

```csharp
// Każde wywołanie InMonth(5) tworzy nową funkcję pamiętającą, że month == 5
public static Func<Play, bool> InMonth(int month) =>
    play => play.Timestamp.Month == month;

// Optymalizacja z domknięciem: HashSet budujemy RAZ przy tworzeniu filtra, a nie przy każdym elemencie!
public static Func<Play, bool> ByArtist(MusicLibrary library, string artistName)
{
    var trackIds = new HashSet<int>();
    // ... pętla zbierająca ID utworów danego wykonawcy do trackIds ...
    return play => trackIds.Contains(play.TrackId); // lambda przechwytuje gotowy trackIds
}
```

### Kombinatory filtrów (łączenie delegatów)
```csharp
public static Func<Play, bool> Not(Func<Play, bool> filter) =>
    play => !filter(play);

public static Func<Play, bool> All(params Func<Play, bool>[] filters) =>
    play =>
    {
        foreach (var f in filters)
            if (!f(play)) return false;
        return true;
    };

public static Func<Play, bool> Any(params Func<Play, bool>[] filters) =>
    play =>
    {
        foreach (var f in filters)
            if (f(play)) return true;
        return false;
    };
```

---

### Pułapka 1: `Func<T, bool>` vs `Predicate<T>` (`List<T>.RemoveAll`)
Mimo że `Func<Play, bool>` i `Predicate<Play>` mają identyczną sygnaturę, to **są to różne, niekompatybilne typy**!
* **Grupa metod lub lambda** konwertuje się automatycznie na dowolny z nich.
* **Gotowy obiekt typu `Func<Play, bool>`** NIE konwertuje się automatycznie na `Predicate<Play>` — trzeba go jawnie opakować:

```csharp
List<Play> streams = [.. plays];
streams.RemoveAll(Filters.IsSkip);                       // ✅ OK — grupa metod
streams.RemoveAll(p => p.Timestamp.Hour == 7);           // ✅ OK — lambda
// streams.RemoveAll(Filters.AtNight);                   // ❌ Błąd CS1503: Func<Play,bool> != Predicate<Play>
streams.RemoveAll(new Predicate<Play>(Filters.AtNight)); // ✅ OK — jawna konwersja
```

### Pułapka 2: Przechwycona zmienna pętli `for`
```csharp
var monthFilters = new List<Func<Play, bool>>();
for (int month = 1; month <= 12; month++)
{
    monthFilters.Add(p => p.Timestamp.Month == month); // ❌ BŁĄD! Wszystkie 12 lambd wskazuje na TĘ SAMĄ zmienną `month`!
}
// Po pętli `month` ma wartość 13 -> każdy filtr zwraca false (0 wyników)!
```
**Rozwiązanie:**
1. Skopiuj zmienną w ciele pętli: `int m = month; monthFilters.Add(p => p.Timestamp.Month == m);`
2. Albo użyj fabryki: `monthFilters.Add(Filters.InMonth(month));` (parametr funkcji to osobna zmienna dla każdego wywołania).
*(W pętli `foreach` od C# 5 problem nie występuje, bo kompilator sam tworzy nową zmienną w każdej iteracji).*

---

## 18. Własne iteratory (`yield`), Enumerator i Bloki rozszerzające (C# 14)

### Ręczne użycie `IEnumerator<T>` (np. `Pairwise`)
Pętla `foreach` to tylko lukier składniowy na `GetEnumerator()`, `MoveNext()` i `Current`. Ręczne sterowanie przydaje się, gdy pierwszy element trzeba obsłużyć inaczej:

```csharp
public static IEnumerable<(T Previous, T Current)> Pairwise<T>(this IEnumerable<T> source)
{
    using var enumerator = source.GetEnumerator(); // IEnumerator<T> jest IDisposable
    if (!enumerator.MoveNext())
        yield break; // pusta sekwencja — kończymy iterator

    T previous = enumerator.Current;
    while (enumerator.MoveNext())
    {
        yield return (previous, enumerator.Current);
        previous = enumerator.Current;
    }
}
```

### Natychmiastowe sprawdzanie argumentów w iteratorach
Metoda zawierająca `yield return` wykonuje się **dopiero przy starcie iteracji** (`foreach`), a nie w momencie wywołania! Aby wyjątek (np. `ArgumentOutOfRangeException`) rzucić **od razu** przy wywołaniu, rozdzielamy walidację od lokalnej funkcji z `yield`:

```csharp
public static IEnumerable<T[]> Window<T>(this IEnumerable<T> source, int size)
{
    ArgumentOutOfRangeException.ThrowIfLessThan(size, 1); // ✅ sprawdzane NATYCHMIAST
    return Iterator();

    IEnumerable<T[]> Iterator() // lokalna funkcja iterująca z yield
    {
        var window = new Queue<T>(size);
        foreach (var item in source)
        {
            window.Enqueue(item);
            if (window.Count > size) window.Dequeue();
            if (window.Count == size) yield return window.ToArray();
        }
    }
}
```

### Bloki rozszerzające (Extension blocks — nowość w C# 14)
Klasyczne metody rozszerzające (`this IEnumerable<T> source`) pozwalają dodawać tylko **metody**. Nowa składnia `extension(Receiver receiver)` w klasie statycznej pozwala dodawać także **właściwości rozszerzające**, operatory i składowe statyczne:

```csharp
public static class ListeningExtensions
{
    // Blok rozszerzający dla IEnumerable<Play>
    extension(IEnumerable<Play> plays)
    {
        // Właściwość rozszerzająca (wywołanie: plays.TotalListeningTime bez nawiasów!)
        public TimeSpan TotalListeningTime
        {
            get
            {
                var total = TimeSpan.Zero;
                foreach (var play in plays) total += play.Duration;
                return total;
            }
        }
    }

    // Blok rozszerzający dla TimeSpan
    extension(TimeSpan ts)
    {
        // Metoda rozszerzająca (wywołanie: ts.ToHoursAndMinutes())
        public string ToHoursAndMinutes() => $"{(int)ts.TotalHours} h {ts.Minutes:D2} min";
    }
}
```
> Obie składnie (`this ...` oraz `extension(...)`) kompilują się do zwykłych metod statycznych i można je mieszać w jednym projekcie.
