# 📂 C# Strumienie, System plików, Archiwa, Zdarzenia i `IDisposable` — Kompletny przewodnik

> Kompletne notatki pokrywające cały materiał z wykładów **Strumienie**, **System plików**, **Archiwa**, **Zdarzenia**, **Interfejs `IDisposable`** oraz praktyczne wzorce z **Lab 06** (w tym `EnumerationOptions`, `FileSystemWatcher`, `leaveOpen`, `Zip Slip` i łańcuchy dekoratorów).

---

## 📑 Spis treści
1. [Czym jest Strumień (`Stream`) — model myślowy i podstawy](#1-czym-jest-strumień-stream--model-myślowy-i-podstawy)
2. [Tworzenie i otwieranie plików: `FileStream`, `FileMode`, `FileAccess`, `FileShare`](#2-tworzenie-i-otwieranie-plików-filestream-filemode-fileaccess-fileshare)
3. [Fasada vs Dekorator vs Adapter — kluczowe różnice i wzorce](#3-fasada-vs-dekorator-vs-adapter--kluczowe-różnice-i-wzorce)
4. [Łączenie strumieni: przesyłanie, kompresja i `leaveOpen`](#4-łączenie-strumieni-przesyłanie-kompresja-i-leaveopen)
5. [Adaptery tekstowe i binarne (`StreamReader`/`Writer`, `BinaryReader`/`Writer`, `Encoding`)](#5-adaptery-tekstowe-i-binarne-streamreaderwriter-binaryreaderwriter-encoding)
6. [System plików: `Path`, `File`/`FileInfo`, `Directory`/`DirectoryInfo`, `DriveInfo`](#6-system-plików-path-filefileinfo-directorydirectoryinfo-driveinfo)
7. [Przechodzenie po katalogach: `Get*` vs `Enumerate*`, `SearchOption` vs `EnumerationOptions`](#7-przechodzenie-po-katalogach-get-vs-enumerate-searchoption-vs-enumerationoptions)
8. [Archiwa: ZIP (`ZipFile`, `ZipArchive`) oraz TAR / `.tar.gz` (`TarFile`, `TarWriter`, `TarReader`)](#8-archiwa-zip-zipfile-ziparchive-oraz-tar-targz-tarfile-tarwriter-tarreader)
9. [Zdarzenia (`event`, `EventHandler<T>`, `EventArgs`, `CancelEventArgs`)](#9-zdarzenia-event-eventhandlert-eventargs-canceleventargs)
10. [Nasłuchiwanie zmian w plikach: `FileSystemWatcher`](#10-nasłuchiwanie-zmian-w-plikach-filesystemwatcher)
11. [Interfejsy, które najczęściej implementujemy w klasach (i co zwracają API)](#11-interfejsy-które-najczęściej-implementujemy-w-klasach-i-co-zwracają-api)

---

## 1. Czym jest Strumień (`Stream`) — model myślowy i podstawy

**Strumień (`System.IO.Stream`)** to **klasa abstrakcyjna** (nie interfejs!), która stanowi jednolitą abstrakcję **sekwencji bajtów** dla operacji wejścia/wyjścia (I/O).

* Źródłem bajtów może być **plik** (`FileStream`), **pamięć RAM** (`MemoryStream`), **sieć** (`NetworkStream`), **łącze** (`PipeStream`) czy **wpis w archiwum ZIP** (`ZipArchiveEntry.Open()`).
* Kod, który przyjmuje parametr typu `Stream`, zadziała tak samo z plikiem na dysku, tablicą bajtów w pamięci i plikiem wewnątrz ZIP-a!

### Trzy podstawowe operacje i właściwości `Can*`
Każdy strumień wspiera (lub nie) 3 operacje — sprawdzamy to właściwościami `bool`:

| Operacja | Właściwość | Główne metody | Które strumienie wspierają? |
| :--- | :--- | :--- | :--- |
| **Odczyt** | `CanRead` | `Read(...)`, `ReadByte()`, `ReadExactly(...)`, `ReadAtLeast(...)` | Pliki (do odczytu), sieć, pamięć, `GZipStream(Decompress)` |
| **Zapis** | `CanWrite` | `Write(...)`, `WriteByte()`, `Flush()` | Pliki (do zapisu), sieć, pamięć, `GZipStream(Compress)` |
| **Pozycjonowanie** | `CanSeek` | `Seek(offset, origin)`, `Position`, `Length` | **Pliki** i **pamięć** (sieć, łącza i strumienie kompresujące **NIE** wspierają `Seek`!) |

```csharp
using Stream stream = File.OpenRead("lorem.txt");
Console.WriteLine($"CanRead:  {stream.CanRead}");  // True
Console.WriteLine($"CanWrite: {stream.CanWrite}"); // False (otwarty tylko do odczytu)
Console.WriteLine($"CanSeek:  {stream.CanSeek}");  // True  (plik wspiera Seek)

// Typowe użycie w metodzie przyjmującej dowolny Stream:
if (stream.CanSeek)
{
    stream.Position = 0; // bezpieczne przewinięcie (bez sprawdzenia na GZipStream/sieci poleci NotSupportedException!)
}
```

> **Obsługa błędów (różnica względem C/POSIX):** Zamiast kodów błędu i `errno`, C# rzuca **wyjątki**: najczęściej `IOException` (oraz pochodne: `FileNotFoundException`, `DirectoryNotFoundException`, `EndOfStreamException`), `UnauthorizedAccessException` (brak uprawnień) lub `NotSupportedException` (np. próba `Seek` na strumieniu sieciowym).

### Odczyt z surowego strumienia (pętla + nowości z .NET 7)
> [!IMPORTANT]
> `int Read(byte[] buffer, int offset, int count)` może zwrócić **mniej bajtów niż `count`**, nawet przed końcem strumienia! Koniec strumienia (EOF) sygnalizuje dopiero zwrot **`0`**. Zawsze przetwarzaj dokładnie `read` bajtów!

```csharp
using Stream stream = new FileStream("lorem.txt", FileMode.Open);
byte[] buffer = new byte[4096];
int read;
while ((read = stream.Read(buffer, 0, buffer.Length)) > 0)
{
    // Przetwarzaj wyłącznie buffer[0..read]!
    Console.WriteLine($"Przeczytano {read} bajtów");
}
```

**Metody pomocnicze od .NET 7** (gdy znasz dokładny rozmiar nagłówka/bloku):
```csharp
byte[] header = new byte[16];
stream.ReadExactly(header);                       // czyta DOKŁADNIE 16 bajtów albo rzuca EndOfStreamException
int n = stream.ReadAtLeast(buffer, minimumBytes: 64); // czyta min. 64, maks. buffer.Length bajtów
```

### Zapis, pozycjonowanie i `RandomAccess` (.NET 6)
```csharp
// Zapis
using Stream outStream = new FileStream("data.bin", FileMode.Create);
byte[] bytes = { 1, 2, 3, 4 };
outStream.Write(bytes, 0, bytes.Length);

// Pozycjonowanie (gdy CanSeek == true)
stream.Position = 0;                       // na początek
stream.Position = stream.Length;           // na koniec
stream.Seek(offset: 10, SeekOrigin.Begin); // +10 od początku
stream.Seek(offset: -10, SeekOrigin.End);  // -10 od końca
stream.Seek(offset: -5, SeekOrigin.Current);

// RandomAccess (.NET 6) — odpowiednik pread/pwrite z POSIX (bezpieczny wielowątkowo, nie zmienia Position!)
using var handle = File.OpenHandle("lorem.txt", FileMode.Open);
byte[] buf = new byte[16];
int r = RandomAccess.Read(handle, buf, fileOffset: 100);
long len = RandomAccess.GetLength(handle);
```

### `Flush`, `Dispose` / `Close` i instrukcja `using`
* Operacje I/O są buforowane. `stream.Flush()` wysyła bufor z obiektu .NET do systemu operacyjnego; `fileStream.Flush(flushToDisk: true)` wymusza fizyczny zapis na dysk (odpowiednik `fsync`).
* `Dispose()` i `Close()` robią **to samo** (automatycznie robią `Flush()` i zamykają uchwyt OS).
* **Zawsze używaj `using`**, żeby plik nie został zablokowany nawet przy wyjątku:
  ```csharp
  using Stream s = File.OpenRead("lorem.txt"); // zamyka się na końcu bieżącego bloku (C# 8+)
  // kompilator zamienia to na: try { ... } finally { s?.Dispose(); }
  ```

---

## 2. Tworzenie i otwieranie plików: `FileStream`, `FileMode`, `FileAccess`, `FileShare`

### Jakie argumenty przyjmuje konstruktor `FileStream`?
```csharp
new FileStream(
    string path,           // 1. Ścieżka do pliku
    FileMode mode,         // 2. CO zrobić przy otwieraniu (utworzyć / otworzyć / nadpisać / dopisać)
    FileAccess access,     // 3. CO my będziemy robić (Read / Write / ReadWrite)
    FileShare share,       // 4. CO pozwalamy robić INNYM procesom/strumieniom w tym samym czasie
    int bufferSize,        // 5. Rozmiar wewnętrznego bufora (domyślnie 4096 B = 4 KB)
    FileOptions options    // 6. Flagi OS (np. Asynchronous, SequentialScan, DeleteOnClose)
);
```

### Tabela `FileMode` (KLUCZOWA na egzaminy!)

| `FileMode` | Gdy plik **ISTNIEJE** | Gdy plik **NIE ISTNIEJE** | Typowe użycie |
| :--- | :--- | :--- | :--- |
| **`CreateNew`** | ❌ Rzuca `IOException` | ✅ Tworzy nowy | Nie chcemy przypadkiem nadpisać istniejącego pliku |
| **`Create`** | ⚠️ **Nadpisuje** (czyści do 0 B) | ✅ Tworzy nowy | Zapis od zera (najczęstsze przy tworzeniu) |
| **`Open`** | ✅ Otwiera | ❌ Rzuca `FileNotFoundException` | Odczyt lub edycja istniejącego pliku |
| **`OpenOrCreate`** | ✅ Otwiera (**bez czyszczenia!**) | ✅ Tworzy nowy | Otwarcie bazy/pliku, jeśli jest, albo start od pustego |
| **`Truncate`** | ⚠️ Czyści zawartość do 0 B | ❌ Rzuca `FileNotFoundException` | Wyzerowanie już istniejącego pliku |
| **`Append`** | ✅ Otwiera i ustawia kursor na końcu | ✅ Tworzy nowy | Dopisywanie logów (tylko z `FileAccess.Write`!) |

### `FileAccess` i `FileShare`
* **`FileAccess`**: `Read`, `Write`, `ReadWrite` — jakie operacje wolno wykonać na *naszym* `FileStream`.
* **`FileShare`**: `None` (wyłączność — nikt inny nie otworzy), `Read` (inni mogą czytać), `Write`, `ReadWrite`, `Delete` — na co pozwalamy *innym* otwartym uchwytom w tym samym czasie.

### Metody skrótowe z klasy `File` (zwracają gotowy `FileStream`)

| Wywołanie | Odpowiednik konstruktora `FileStream` | Uwaga / Pułapka! |
| :--- | :--- | :--- |
| `File.Create(path)` | `FileMode.Create, FileAccess.ReadWrite` | Czyści plik, jeśli już istniał |
| `File.OpenRead(path)` | `FileMode.Open, FileAccess.Read, FileShare.Read` | Tylko do odczytu |
| `File.OpenWrite(path)` | `FileMode.OpenOrCreate, FileAccess.Write` | ⚠️ **NIE obcina starego pliku!** Jeśli nowy tekst jest krótszy niż stary, zostanie „ogon” ze starego pliku! |
| `File.Open(path, mode, access)` | `new FileStream(path, mode, access)` | Pełna kontrola |

---

## 3. Fasada vs Dekorator vs Adapter — kluczowe różnice i wzorce

To są **3 wzorce projektowe**, na których opiera się całe `System.IO`. Najważniejsza różnica to **typ obiektu i rola**:

| Wzorzec | Przykłady w `System.IO` | Czy jest `Stream`? | Co robi i kiedy używać? |
| :--- | :--- | :--- | :--- |
| **Fasada** *(Facade)* | `File.ReadAllText`, `File.ReadLines`, `File.WriteAllBytes`, `ZipFile.CreateFromDirectory` | ❌ Nie (klasa statyczna) | **Prosty interfejs „1 linijka”** — ukrywa tworzenie strumieni, adapterów i `using`. Świetny do prostych operacji na małych plikach. |
| **Dekorator** *(Decorator)* | `BufferedStream`, `GZipStream`, `DeflateStream`, `ZLibStream`, `BrotliStream`, `CryptoStream` | ✅ **TAK** (dziedziczy po `Stream`) | **Opakowuje inny `Stream`**, dodając mu nową funkcję (buforowanie, kompresję, szyfrowanie) w locie. Można je **łączyć w łańcuchy**! |
| **Adapter** *(Adapter)* | `StreamReader`, `StreamWriter`, `BinaryReader`, `BinaryWriter` | ❌ **NIE** (osobne klasy) | **Tłumaczy surowe bajty ze `Stream` na wygodny typ** (linie tekstu `string` albo `int`, `double`, `float`). Zwykle stoi na samym końcu łańcucha. |

### Prymitywny przykład Fasady (`File.*`)
```csharp
// Małe pliki — ładują WSZYSTKO naraz do pamięci RAM:
string text = File.ReadAllText("lorem.txt");
File.WriteAllText("copy.txt", text);
string[] allLines = File.ReadAllLines("lorem.txt");
byte[] allBytes = File.ReadAllBytes("image.bin");
File.AppendAllText("log.txt", "Nowy wpis\n");

// ⚠️ DUŻE pliki — NIE używaj ReadAllLines! Użyj leniwego ReadLines (zwraca IEnumerable<string>):
IEnumerable<string> lazyLines = File.ReadLines("huge.log");
int errors = lazyLines.Count(line => line.Contains("Error")); // czyta linia po linii!
```

**Jak Fasada wygląda w środku (pod spodem)?** Po prostu ukrywa rutynę tworzenia strumienia, adaptera i `using`:
```csharp
public static class MyFileFacade
{
    public static string ReadAllText(string path)
    {
        using FileStream fs = new FileStream(path, FileMode.Open, FileAccess.Read);
        using StreamReader sr = new StreamReader(fs, Encoding.UTF8);
        return sr.ReadToEnd();
    }
}
```

### Prymitywny przykład własnego Adaptera (jak działa w środku)
Adapter **NIE dziedziczy po `Stream`** — trzyma `Stream` w prywatnym polu i **zmienia interfejs** (z `byte[]` na np. `string`, jak przejściówka do gniazdka):

```csharp
public class SimpleTextWriter : IDisposable
{
    private readonly Stream _stream; // dowolny Stream pod spodem
    public SimpleTextWriter(Stream stream) => _stream = stream;

    // Zupełnie inny interfejs (string zamiast byte[])!
    public void WriteLine(string text)
    {
        byte[] bytes = Encoding.UTF8.GetBytes(text + Environment.NewLine);
        _stream.Write(bytes, 0, bytes.Length); // tłumaczy tekst na bajty i zapisuje
    }

    public void Dispose() => _stream.Dispose();
}
```

### Prymitywny przykład własnego Dekoratora (jak działa wzorzec pod spodem)
Dekorator dziedziczy po `Stream`, przyjmuje inny `Stream` w konstruktorze i deleguje do niego wywołania, dokładając własną logikę (np. liczenie zapisanych bajtów):

```csharp
public class CountingStream : Stream
{
    private readonly Stream _inner;
    public long BytesWritten { get; private set; }

    public CountingStream(Stream inner) => _inner = inner;

    public override void Write(byte[] buffer, int offset, int count)
    {
        _inner.Write(buffer, offset, count); // delegacja do opakowanego strumienia
        BytesWritten += count;               // dodatkowa funkcjonalność dekoratora
    }

    public override int Read(byte[] buffer, int offset, int count) => _inner.Read(buffer, offset, count);
    public override void Flush() => _inner.Flush();
    public override long Seek(long offset, SeekOrigin origin) => _inner.Seek(offset, origin);
    public override void SetLength(long value) => _inner.SetLength(value);
    public override bool CanRead => _inner.CanRead;
    public override bool CanSeek => _inner.CanSeek;
    public override bool CanWrite => _inner.CanWrite;
    public override long Length => _inner.Length;
    public override long Position { get => _inner.Position; set => _inner.Position = value; }
}
```

### `BufferedStream` — po co i kiedy?
* Opakowuje inny `Stream` i trzyma w RAM bufor (np. 20 KB), zmniejszając liczbę kosztownych wywołań systemowych I/O.
* **Uwaga:** `FileStream` ma już własny bufor (domyślnie 4 KB), więc `BufferedStream` ma największy sens dla strumieni **bez własnego bufora** (np. `NetworkStream`) lub gdy chcesz większy bufor.

---

## 4. Łączenie strumieni: przesyłanie, kompresja i `leaveOpen`

### Przesyłanie danych z jednego strumienia do drugiego (np. kompresja pliku do `.gz`)
Dostępne dekoratory kompresji (`System.IO.Compression`):
* `DeflateStream` (surowy algorytm Deflate), `ZLibStream` (.NET 6, format zlib), `GZipStream` (format `.gz` z nagłówkiem i stopką), `BrotliStream` (algorytm Brotli).

**Konstruktory (identyczne dla wszystkich 4 klas kompresji!):**
```csharp
// 1. Tryb Compress albo Decompress (+ opcjonalnie leaveOpen):
new GZipStream(Stream stream, CompressionMode mode, bool leaveOpen = false)

// 2. Zawsze kompresja — z wyborem poziomu (Optimal, Fastest, SmallestSize, NoCompression):
new GZipStream(Stream stream, CompressionLevel compressionLevel, bool leaveOpen = false)
```

> **Który strumień owinąć w `GZipStream` (`fsIn` czy `fsOut`)?**
> * **Kompresja (`Compress`):** owijamy **wyjście (`fsOut`)** $\rightarrow$ `fsIn.CopyTo(gz)` (dane wchodzą do `gz`, kompresują się i lecą do `fsOut`).
> * **Dekompresja (`Decompress`):** owijamy **wejście (`fsIn`)** $\rightarrow$ `gz.CopyTo(fsOut)` (`gz` czyta z `fsIn`, rozpakowuje i zapisuje do `fsOut`).

**Sposób 1 — nowoczesny i najprostszy (`CopyTo`):**
```csharp
// Kompresja: file.txt -> (FileStream) -> CopyTo -> GZipStream(Compress) -> FileStream(file.txt.gz)
using FileStream fsIn = File.OpenRead("file.txt");
using FileStream fsOut = File.Create("file.txt.gz");
using GZipStream gz = new GZipStream(fsOut, CompressionMode.Compress);
fsIn.CopyTo(gz); // kopiuje cały strumień porcjami w wewnętrznej pętli
```

**Sposób 2 — ręczna pętla (np. dekompresja):**
```csharp
// Dekompresja: file.txt.gz -> FileStream -> GZipStream(Decompress) -> FileStream(file_out.txt)
using FileStream fsIn = File.OpenRead("file.txt.gz");
using GZipStream gz = new GZipStream(fsIn, CompressionMode.Decompress);
using FileStream fsOut = File.Create("file_out.txt");

byte[] buffer = new byte[4096];
int read;
while ((read = gz.Read(buffer, 0, buffer.Length)) > 0)
{
    fsOut.Write(buffer, 0, read);
}
```

### Łańcuch: Adapter $\rightarrow$ Dekorator $\rightarrow$ Strumień bazowy
```csharp
// Zapis tekstu bezpośrednio przez kompresję na dysk:
// StreamWriter (tekst->bajty) -> GZipStream (kompresja) -> FileStream (dysk)
using (FileStream fs = File.Create("notes.txt.gz"))
using (GZipStream gz = new GZipStream(fs, CompressionLevel.Optimal))
using (StreamWriter writer = new StreamWriter(gz))
{
    writer.WriteLine("Hello, compressed world!");
} // Zwalnianie idzie OD KOŃCA: writer (Flush tekstu) -> gz (zapisuje stopkę gzip!) -> fs (zamyka plik)

// Odczyt z tego samego łańcucha:
using FileStream fsIn = File.OpenRead("notes.txt.gz");
using GZipStream gzIn = new GZipStream(fsIn, CompressionMode.Decompress);
using StreamReader reader = new StreamReader(gzIn);
Console.WriteLine(reader.ReadToEnd());
```

### ⚠️ Złota zasada `leaveOpen: true` i zamykania strumieni!
1. **Zwolnienie zewnętrznego dekoratora/adaptera automatycznie zamyka strumień w środku!**
2. **Strumienie kompresujące (`GZipStream`) zapisują końcówkę i stopkę DOPIERO przy `Dispose()`!** Jeśli chcesz np. skompresować dane do `MemoryStream`, a potem odczytać jego długość lub zawartość, musisz użyć **`leaveOpen: true`**:
   ```csharp
   using var ms = new MemoryStream();
   using (var gz = new GZipStream(ms, CompressionMode.Compress, leaveOpen: true))
   {
       gz.Write(new byte[1000]);
   } // Tutaj gz kończy kompresję i zapisuje stopkę, ALE ms pozostaje otwarty!

   Console.WriteLine($"Rozmiar po kompresji: {ms.Length} B");
   ms.Position = 0; // jeśli chcesz teraz czytać z ms, cofnij kursor na 0!
   ```
3. To samo dotyczy metody, która **pożycza** `Stream` jako parametr:
   ```csharp
   void WriteHeader(Stream stream)
   {
       using var writer = new StreamWriter(stream, leaveOpen: true); // NIE zamykaj cudzego strumienia!
       writer.WriteLine("# Header");
   }
   ```

---

## 5. Adaptery tekstowe i binarne (`StreamReader`/`Writer`, `BinaryReader`/`Writer`, `Encoding`)

### Adaptery tekstowe (`StreamReader` / `StreamWriter`)
Zamieniają znaki (`char`, `string`) $\leftrightarrow$ bajty za pomocą `System.Text.Encoding`.

```csharp
// Czytanie linia po linii:
using var sr = new StreamReader(File.OpenRead("lorem.txt"));
while (sr.ReadLine() is { } line) // ReadLine() zwraca null na końcu pliku
{
    Console.WriteLine(line);
}

// Zapisywanie tekstu:
using var sw = new StreamWriter(File.Create("out.txt"));
sw.Write("Klucz: ");
sw.WriteLine(42);
```

* **Kodowanie i BOM (Byte Order Mark):**
  * Domyślnie `StreamWriter` oraz `File.WriteAllText(path, text)` zapisują **UTF-8 bez BOM**.
  * Jawne przekazanie `Encoding.UTF8` dodaje **3-bajtowy BOM (`EF BB BF`)**!
  * `Encoding.Unicode` to **UTF-16 LE z BOM (`FF FE`)**.
  * `StreamReader` automatycznie wykrywa kodowanie po BOM (jeśli jest); inaczej używa UTF-8.

### Adaptery binarne (`BinaryReader` / `BinaryWriter`)
Zapisują i czytają typy proste (`int`, `long`, `float`, `double`, `bool`, `string`) w postaci binarnej (**zawsze little-endian**):

```csharp
// Zapis
using (var bw = new BinaryWriter(File.Create("player.bin")))
{
    bw.Write("Bob");    // zapisuje długość (7-bit int) + bajty UTF-8
    bw.Write(100);      // 4 bajty (Int32)
    bw.Write(48.5f);    // 4 bajty (Single)
}

// Odczyt — MUSI być w dokładnie tej samej kolejności i typach!
using (var br = new BinaryReader(File.OpenRead("player.bin")))
{
    string name = br.ReadString();
    int hp = br.ReadInt32();
    float x = br.ReadSingle();
}
```
> **Big-endian** (np. protokoły sieciowe): użyj `BinaryPrimitives.WriteInt32BigEndian(span, value)` i `BinaryPrimitives.ReadInt32BigEndian(span)` z `System.Buffers.Binary`.

---

## 6. System plików: `Path`, `File`/`FileInfo`, `Directory`/`DirectoryInfo`, `DriveInfo`

### Klasa `Path` (statyczna — operuje wyłącznie na napisach, NIE dotyka dysku!)

```csharp
string p = Path.Combine("data", "2026", "report.csv"); // data/2026/report.csv (na Windows \)
Path.GetFileName(p);                 // "report.csv"
Path.GetFileNameWithoutExtension(p); // "report"
Path.GetExtension(p);                // ".csv"
Path.ChangeExtension(p, ".json");    // "data/2026/report.json"
Path.GetDirectoryName(p);            // "data/2026"
Path.GetFullPath(p);                 // pełna ścieżka względem bieżącego katalogu roboczego
Path.GetRelativePath("/a/b", "/a/b/c/d.txt"); // "c/d.txt"
Path.DirectorySeparatorChar;         // '/' (Linux/macOS) lub '\\' (Windows)
Path.GetTempPath();                  // katalog tymczasowy OS
Path.GetRandomFileName();            // losowa nazwa pliku (nie tworzy pliku)
Path.GetTempFileName();              // tworzy pusty plik tymczasowy na dysku i zwraca ścieżkę!
```

> [!WARNING]
> **`Path.Combine` vs `Path.Join` (pułapka bezpieczeństwa!):**
> * `Path.Combine("data", "/etc/passwd")` $\rightarrow$ **`"/etc/passwd"`** (gdy któryś dalszy argument jest ścieżką bezwzględną, `Combine` **porzuca** wszystkie wcześniejsze!).
> * `Path.Join("data", "/etc/passwd")` $\rightarrow$ **`"data/etc/passwd"`** (zawsze po prostu skleja separatorem).

### Porównanie klas statycznych i instancyjnych

| Statyczna (jednorazowe wywołania) | Instancyjna (wiele operacji na 1 obiekcie + właściwości) | Co robi / zwraca |
| :--- | :--- | :--- |
| **`File`** (`Exists`, `Copy`, `Move`, `Delete`, `Replace`, `GetAttributes`, `GetLastWriteTime`, `GetUnixFileMode`) | **`FileInfo`** (`new FileInfo(path)` $\rightarrow$ `.Exists`, `.Name`, `.FullName`, `.Length`, `.Extension`, `.DirectoryName`, `.IsReadOnly`, `.CopyTo(...)`) | Operacje i metadane pojedynczego **pliku** |
| **`Directory`** (`CreateDirectory`, `Exists`, `Delete(p, recursive: true)`, `Move`, `GetCurrentDirectory`, `CreateTempSubdirectory`) | **`DirectoryInfo`** (`new DirectoryInfo(path)` $\rightarrow$ `.Name`, `.FullName`, `.Parent`, `.Root`, `.CreateSubdirectory(...)`, `.GetFiles()`, `.EnumerateFiles()`) | Operacje i przeglądanie **katalogu** (`DirectoryInfo.GetFiles()` zwraca `FileInfo[]`, a `Directory.GetFiles()` zwraca `string[]`!) |
| — | **`DriveInfo`** (`DriveInfo.GetDrives()`, `new DriveInfo("/")` $\rightarrow$ `.TotalSize`, `.TotalFreeSpace`, `.AvailableFreeSpace`) | Informacje o dyskach |

* **Katalogi specjalne systemu:** `Environment.GetFolderPath(Environment.SpecialFolder.UserProfile)` / `.MyDocuments`.

---

## 7. Przechodzenie po katalogach: `Get*` vs `Enumerate*`, `SearchOption` vs `EnumerationOptions`

### `Get*` vs `Enumerate*` — czym się różnią?

| Metoda | Zwracany typ (`Directory` / `DirectoryInfo`) | Sposób działania | Kiedy używać? |
| :--- | :--- | :--- | :--- |
| **`GetFiles` / `GetDirectories` / `GetFileSystemEntries`** | `string[]` / `FileInfo[]` | ⚡ **Natychmiastowo** ładuje wszystkie ścieżki do tablicy | Gdy katalog jest mały LUB gdy **usuwasz/modyfikujesz pliki** podczas iteracji! |
| **`EnumerateFiles` / `EnumerateDirectories` / `EnumerateFileSystemEntries`** | `IEnumerable<string>` / `IEnumerable<FileInfo>` | 🦥 **Leniwie** (element po elemencie z dysku) | Duże drzewa katalogów, szukanie pierwszego pasującego pliku (oszczędność RAM i czasu) |

> [!CAUTION]
> **Nie usuwaj ani nie dodawaj plików w katalogu podczas leniwego `EnumerateFiles`!** Wynik jest nieprzewidywalny — do modyfikowania zawartości użyj `GetFiles` (najpierw zbiera tablicę).

### `SearchOption` vs `EnumerationOptions` (opcje rekursji, uprawnień i pomijania!)

**1. Starszy sposób — enum `SearchOption`:**
* `SearchOption.TopDirectoryOnly` (tylko bieżący folder)
* `SearchOption.AllDirectories` (rekurencyjnie cały podfolder)
* ⚠️ **Wady `SearchOption`:**
  1. Jeśli trafi na katalog bez uprawnień odczytu $\rightarrow$ rzuca `UnauthorizedAccessException` i **przerywa całe przeszukiwanie**!
  2. **NIE pomija** plików ukrytych i systemowych.

**2. Nowoczesny sposób — obiekt `EnumerationOptions` (ZALECANY!):**
Pozwala precyzyjnie sterować rekursją, błędami dostępu, plikami ukrytymi i wielkością liter:

```csharp
var options = new EnumerationOptions
{
    RecurseSubdirectories = true,            // flaga rekursji (jak AllDirectories)
    IgnoreInaccessible = true,               // pomija pliki/foldery bez uprawnień (domyślnie true!)
    AttributesToSkip = FileAttributes.Hidden | FileAttributes.System, // co pomijać (to jest domyślne!)
    MatchCasing = MatchCasing.CaseInsensitive, // dopasowanie wielkości liter wzorca
    MaxRecursionDepth = 5                    // maksymalna głębokość rekursji (domyślnie int.MaxValue)
};

// Zwraca leniwie IEnumerable<FileInfo> (mamy od razu .Length, .LastWriteTime, .FullName!)
foreach (FileInfo file in new DirectoryInfo(".").EnumerateFiles("*.cs", options))
{
    Console.WriteLine($"{file.FullName} ({file.Length} B)");
}
```
> **Wskazówka:** Jeśli chcesz jednak uwzględnić też pliki ukryte (np. `.gitignore` na Linux/macOS), ustaw `AttributesToSkip = FileAttributes.None` (lub `0`).
> **Dopasowanie wzorców (`*`, `?`) ręcznie:** `FileSystemName.MatchesSimpleExpression("*.bak", fileName)` z przestrzeni nazw `System.IO.Enumeration`.

---

## 8. Archiwa: ZIP (`ZipFile`, `ZipArchive`) oraz TAR / `.tar.gz` (`TarFile`, `TarWriter`, `TarReader`)

### Porównanie formatów i klas

| Format | Przestrzeń nazw | Kompresja wbudowana? | Prosta fasada | Praca wpis po wpisie |
| :--- | :--- | :--- | :--- | :--- |
| **ZIP (`.zip`)** | `System.IO.Compression` | ✅ Tak (domyślnie) | `ZipFile.CreateFromDirectory` / `ExtractToDirectory` | `ZipArchive` (`ZipFile.Open`) + `ZipArchiveEntry` |
| **TAR (`.tar`)** | `System.Formats.Tar` (.NET 7) | ❌ Nie (tylko kontener, zachowuje uprawnienia Unix) | `TarFile.CreateFromDirectory` / `ExtractToDirectory` | `TarWriter` / `TarReader` + `TarEntry` (`PaxTarEntry`) |
| **`.tar.gz`** | `Tar` + `GZipStream` | ✅ Przez owinięcie `GZipStream` | `TarFile.*` ze strumieniem `GZipStream` | `TarWriter`/`TarReader` na `GZipStream` |

### ZIP: Fasada (`ZipFile`) vs Wpis po wpisie (`ZipArchive`)
```csharp
using System.IO.Compression;

// 1. Prosta fasada (cały katalog <-> plik .zip):
ZipFile.CreateFromDirectory("my_folder", "archive.zip");
ZipFile.ExtractToDirectory("archive.zip", "extracted_folder", overwriteFiles: true);

// 2. Tworzenie ZIP wpis po wpisie (ZipArchiveMode: Read, Create, Update):
using (ZipArchive zip = ZipFile.Open("custom.zip", ZipArchiveMode.Create))
{
    // Dodanie pliku z dysku (metoda rozszerzająca):
    zip.CreateEntryFromFile("README.md", "docs/README.md", CompressionLevel.Optimal);

    // Dodanie wpisu bezpośrednio przez strumień:
    ZipArchiveEntry entry = zip.CreateEntry("manifest.txt", CompressionLevel.Optimal);
    using (Stream entryStream = entry.Open()) // ⚠️ W trybie Create naraz może być otwarty TYLKO 1 wpis!
    using (var sw = new StreamWriter(entryStream))
    {
        sw.WriteLine("Plik wygenerowany w locie!");
    }
} // Dopiero Dispose() na ZipArchive zapisuje spis treści i kończy archiwum!

// 3. Odczyt ZIP wpis po wpisie:
using (ZipArchive zip = ZipFile.OpenRead("custom.zip")) // skrót dla ZipArchiveMode.Read
{
    foreach (ZipArchiveEntry entry in zip.Entries)
    {
        Console.WriteLine($"{entry.FullName} ({entry.Length} B)");
        using Stream s = entry.Open();
        // ... lub entry.ExtractToFile("dest.txt", overwrite: true);
    }
}
```
> **Asynchroniczne API (.NET 10):** `ZipFile.OpenReadAsync`, `ZipFile.CreateFromDirectoryAsync`, `entry.OpenAsync()` + `await using` (`ZipArchive` implementuje `IAsyncDisposable`).
> **Bezpieczeństwo — Zip Slip:** Złośliwy ZIP może zawierać wpis `../../.bashrc`. Przed `ExtractToFile` sprawdź, czy `Path.GetFullPath(destPath)` leży wewnątrz katalogu docelowego!

### TAR i skompresowane `.tar.gz` (łączenie ze strumieniem `GZipStream`)
> **Różnica wobec `ZipFile`:** `TarFile.ExtractToDirectory` wymaga, aby **folder docelowy już istniał** (`Directory.CreateDirectory(dest)`).

```csharp
using System.Formats.Tar;
using System.IO.Compression;

// Tworzenie .tar.gz z całego katalogu (TarFile -> GZipStream -> FileStream):
using (FileStream fs = File.Create("backup.tar.gz"))
using (GZipStream gz = new GZipStream(fs, CompressionMode.Compress))
{
    TarFile.CreateFromDirectory("my_folder", gz, includeBaseDirectory: false);
}

// Rozpakowanie .tar.gz:
Directory.CreateDirectory("out_dir");
using (FileStream fs = File.OpenRead("backup.tar.gz"))
using (GZipStream gz = new GZipStream(fs, CompressionMode.Decompress))
{
    TarFile.ExtractToDirectory(gz, "out_dir", overwriteFiles: true);
}

// Tworzenie i odczyt wpis po wpisie (TarWriter / TarReader na GZipStream):
using (FileStream fs = File.Create("logs.tar.gz"))
using (GZipStream gz = new GZipStream(fs, CompressionMode.Compress))
using (TarWriter writer = new TarWriter(gz))
{
    writer.WriteEntry("log.txt", entryName: "logs/log.txt");
    var memEntry = new PaxTarEntry(TarEntryType.RegularFile, "logs/info.txt")
    {
        DataStream = new MemoryStream("Hello TAR"u8.ToArray())
    };
    writer.WriteEntry(memEntry);
}

using (FileStream fs = File.OpenRead("logs.tar.gz"))
using (GZipStream gz = new GZipStream(fs, CompressionMode.Decompress))
using (TarReader reader = new TarReader(gz))
{
    while (reader.GetNextEntry() is TarEntry entry)
    {
        Console.WriteLine($"{entry.Name} ({entry.Length} B, mode: {entry.Mode})");
    }
}
```

---

## 9. Zdarzenia (`event`, `EventHandler<T>`, `EventArgs`, `CancelEventArgs`)

**Zdarzenie (`event`)** to implementacja wzorca **Obserwator** w C#: **Wydawca (Publisher)** powiadamia dowolną liczbę **Subskrybentów (Subscribers)**, nic o nich nie wiedząc.

### Czym różni się `event` od zwykłego publicznego pola delegata?
* Zwykły `public EventHandler? Ev;` pozwoliłby kodowi z zewnątrz **wywołać** zdarzenie (`obj.Ev(null, null)`) albo **wykasować wszystkich subskrybentów** (`obj.Ev = null`).
* Słowo kluczowe **`event`** nakłada **enkapsulację**: z zewnątrz dozwolone są **TYLKO `+=` (subskrypcja)** oraz **`-=` (wypisanie)**. Wywołać zdarzenie (`?.Invoke`) może **wyłącznie klasa-właściciel**.

### Prymitywny, kompletny przykład (3 elementy wzorca .NET)

```csharp
// 1. Klasa danych zdarzenia (dziedziczy po EventArgs)
public class PriceChangedEventArgs(decimal oldPrice, decimal newPrice) : EventArgs
{
    public decimal OldPrice { get; } = oldPrice;
    public decimal NewPrice { get; } = newPrice;
}

// 2. Wydawca (Publisher)
public class Product(string name, decimal price)
{
    public string Name { get; } = name;
    private decimal _price = price;

    // Standardowy delegat: void EventHandler<TEventArgs>(object? sender, TEventArgs e)
    public event EventHandler<PriceChangedEventArgs>? PriceChanged;

    public decimal Price
    {
        get => _price;
        set
        {
            if (value == _price) return;
            decimal old = _price;
            _price = value;
            OnPriceChanged(new PriceChangedEventArgs(old, value));
        }
    }

    // Konwencja: protected virtual On... (pozwala klasom pochodnym reagować) + ?.Invoke (bo może być null!)
    protected virtual void OnPriceChanged(PriceChangedEventArgs e)
        => PriceChanged?.Invoke(this, e);
}

// 3. Subskrybent (Subscriber)
var laptop = new Product("Laptop", 3000m);

// Subskrypcja (metodą lub lambdą):
void OnPriceChanged(object? sender, PriceChangedEventArgs e)
{
    if (sender is Product p)
        Console.WriteLine($"{p.Name}: {e.OldPrice} -> {e.NewPrice}");
}

laptop.PriceChanged += OnPriceChanged;
laptop.Price = 2500m; // Wypisze: Laptop: 3000 -> 2500
laptop.PriceChanged -= OnPriceChanged; // Wypisanie (ważne przeciwko wyciekom pamięci!)
```

### Ważne detale na egzamin:
1. **Wyciek pamięci (Memory Leak):** Wydawca trzyma referencje do wszystkich subskrybentów. Jeśli długo żyjący wydawca ma krótko żyjącego subskrybenta, który **nie zrobi `-=`**, GC nie usunie subskrybenta! Często subskrybent implementuje `IDisposable` i robi `-=` w `Dispose()`.
2. **Odsubskrybowanie lambdy:** `x.Ev += (s, e) => ...; x.Ev -= (s, e) => ...;` **NIE DZIAŁA** (to dwa różne obiekty delegatów!). Żeby odsubskrybować lambdę, zapisz ją w zmiennej `EventHandler<...> handler = ...;`.
3. **Zdarzenia z możliwością anulowania (`CancelEventArgs` z `System.ComponentModel`):**
   * Klasa argumentów dziedziczy po `CancelEventArgs` (ma właściwość `bool Cancel`).
   * Wydawca wywołuje `OnFileFound(args)`, a potem sprawdza `if (!args.Cancel) { ... }`.
   * Subskrybent ustawia tylko `e.Cancel = true;` (nigdy nie resetuje na `false`, żeby nie nadpisać decyzji poprzedniego subskrybenta!).
4. **Własne akcesory `add` / `remove`:** Kompilator tłumaczy `public event ...` na prywatne pole + bezpieczne wątkowo `add`/`remove`. Można je napisać ręcznie (np. do logowania subskrypcji), ale wtedy wywołujemy prywatne pole `_ev?.Invoke(...)`, a ręczne akcesory domyślnie nie są thread-safe. Od C# 14 istnieją też `partial event`.
5. **Kiedy `event`, a kiedy parametr `Action<T>`?** `event` — gdy obiekt ma wielu niezależnych subskrybentów przez czas swojego życia; `Action<T>` (callback) — gdy metoda ma jednego wywołującego podczas jednego wywołania.

---

## 10. Nasłuchiwanie zmian w plikach: `FileSystemWatcher`

Klasa `FileSystemWatcher` (`System.IO`) to wbudowany w .NET „nasłuchiwacz zdarzeń systemu plików” (pod spodem `inotify` na Linux / `FSEvents` na macOS / WinAPI na Windows). Implementuje **`IDisposable`**!

### Podstawowy przykład użycia
```csharp
using var watcher = new FileSystemWatcher(path: ".", filter: "*.txt");

// Podpinamy obsługę zdarzeń:
watcher.Created += (s, e) => Console.WriteLine($"Utworzono: {e.FullPath}");
watcher.Changed += (s, e) => Console.WriteLine($"Zmieniono: {e.FullPath} ({e.ChangeType})");
watcher.Deleted += (s, e) => Console.WriteLine($"Usunięto:  {e.FullPath}");
watcher.Renamed += (s, e) => Console.WriteLine($"Zmiana nazwy: {e.OldFullPath} -> {e.FullPath}");
watcher.Error   += (s, e) => Console.WriteLine($"Błąd watchera: {e.GetException()}");

// Konfiguracja:
watcher.IncludeSubdirectories = true; // czy obserwować też podkatalogi
watcher.NotifyFilter = NotifyFilters.FileName | NotifyFilters.DirectoryName
                     | NotifyFilters.LastWrite | NotifyFilters.Size;

watcher.EnableRaisingEvents = true;   // ⚠️ BEZ TEGO NIE BĘDZIE ŻADNYCH ZDARZEŃ!

Console.WriteLine("Nasłuchuję... Naciśnij Enter, aby zakończyć.");
Console.ReadLine();
```

### 4 kluczowe fakty o `FileSystemWatcher`
1. **`EnableRaisingEvents = true`** — domyślnie jest `false`; dopiero ustawienie na `true` uruchamia nasłuchiwanie.
2. **Typy argumentów zdarzeń:**
   * `Created`, `Changed`, `Deleted` $\rightarrow$ **`FileSystemEventArgs`** (`ChangeType`, `Name`, `FullPath`).
   * `Renamed` $\rightarrow$ **`RenamedEventArgs`** (dziedziczy po `FileSystemEventArgs`, dodaje `OldName` i `OldFullPath`). Dzięki **kontrawariancji** tę samą metodę `void OnAny(object s, FileSystemEventArgs e)` można podpiąć także pod `Renamed`!
   * `Error` $\rightarrow$ **`ErrorEventArgs`** (`GetException()`). Zgłasza wewnętrzne błędy samego watchera (np. **przepełnienie bufora** przy lawinie zmian — informuje, że część zdarzeń mogła zaginąć!).
3. **Wątek w tle i `lock`:** Zdarzenia `FileSystemWatcher` są wywoływane **asynchronicznie w osobnym wątku**! Wyjątek rzucony z handlera nie trafi do `try/catch` w `Main` (zabije proces) — trzeba go łapać w samym handlerze, a wspólną kolekcję (np. `_files`) chronić przez **`lock`**:
   ```csharp
   private readonly Lock _lock = new(); // w .NET 9+ (w starszych: private readonly object _lock = new();)
   private readonly HashSet<string> _files = new();

   private void OnFileCreated(object sender, FileSystemEventArgs e)
   {
       if (e.Name is null) return;
       lock (_lock) // w danej chwili tylko 1 wątek może modyfikować _files
       {
           _files.Add(e.Name);
       }
       DirectoryChanged?.Invoke(this, EventArgs.Empty); // zdarzenie wywołujemy POZA lock!
   }

   public string[] Files
   {
       get { lock (_lock) return _files.ToArray(); } // odczyt też pod lock!
   }
   ```
   > **Dlaczego `lock (_lock)`, a nie `lock (_files)`?** `lock` nie blokuje samego obiektu, tylko sekcję kodu w `{ ... }`. Zła praktyka to blokować samą kolekcję (`_files`), bo ktoś z zewnątrz mógłby ją widzieć — dlatego tworzymy osobne prywatne pole `_lock` („klucz do pokoju”).
4. **Debouncing:** Jedno „Zapisz” w edytorze często generuje kilka zdarzeń `Changed`/`Created` naraz. Dlatego w praktyce używa się timera (`System.Threading.Timer`), który resetuje odliczanie po każdej zmianie i uruchamia akcję dopiero po chwili ciszy (np. 2 s).

---

## 11. Interfejsy, które najczęściej implementujemy w klasach (i co zwracają API)

### A. Kiedy w .NET zwracamy **interfejs**, a kiedy **klasę abstrakcyjną**?

| Co zwraca metoda | Typ zwracany | Interfejs czy klasa? | Dlaczego? |
| :--- | :--- | :--- | :--- |
| `File.OpenRead(...)`, `entry.Open()` | `FileStream` / `Stream` | **Klasa abstrakcyjna** (`Stream`) | Wspólna implementacja bazowa (`CopyTo`, `ReadExactly`, właściwości) |
| `File.ReadLines(...)`, `Directory.EnumerateFiles(...)` | `IEnumerable<string>` | **Interfejs** | Leniwa sekwencja — nie narzuca konkretnej kolekcji w pamięci |
| `File.ReadAllLines(...)`, `Directory.GetFiles(...)` | `string[]` | **Konkretna tablica** | Natychmiastowy odczyt wszystkiego do RAM |
| Własne API zwracające kolekcję (np. `Entries`) | `IReadOnlyList<T>` / `IReadOnlyCollection<T>` | **Interfejs** | Enkapsulacja — wywołujący może czytać, ale nie może zrobić `.Add()`/`.Clear()` |

### B. Najważniejsze interfejsy do implementacji w klasach — ściąga metod

| Interfejs | Kiedy implementować? | Jakie metody/właściwości MUSISZ zaimplementować? |
| :--- | :--- | :--- |
| **`IDisposable`** | Klasa posiada otwarte zasoby (`Stream`, `StreamWriter`, `FileSystemWatcher`, `Timer`) lub subskrypcje zdarzeń | `void Dispose()` *(po zwolnieniu pozostałe metody rzucają `ObjectDisposedException`; wielokrotne `Dispose()` nie rzuca błędu!)* |
| **`IAsyncDisposable`** | Klasa posiada zasoby zwalniane asynchronicznie (`await using`, np. `ZipArchive` w .NET 10) | `ValueTask DisposeAsync()` |
| **`IEnumerable<T>`** | Chcesz, aby po obiekcie Twojej klasy dało się iterować w `foreach` i używać LINQ | `IEnumerator<T> GetEnumerator()` + niegeneryczne `IEnumerator IEnumerable.GetEnumerator()` |
| **`IEnumerator<T>`** | Piszesz ręczny iterator (bez `yield return`) | `T Current`, `object IEnumerator.Current`, `bool MoveNext()`, `void Reset()`, `void Dispose()` |
| **`IEquatable<T>`** | Własne porównywanie równości obiektów (ważne dla `Dictionary`, `HashSet`, `Distinct`) | `bool Equals(T? other)` (+ nadpisz `object.Equals` i `GetHashCode()`) |
| **`IComparable<T>`** | Domyślna kolejność sortowania obiektów (`Order()`, `List.Sort()`) | `int CompareTo(T? other)` (`< 0`, `0`, `> 0`) |

### Wzorcowa implementacja `IDisposable` (standardowa + z zasobem niezarządzanym)

**Wersja 1 — najczęstsza (klasa posiada obiekty zarządzane implementujące `IDisposable`, np. `StreamWriter`):**
```csharp
public class FileLogger : IDisposable
{
    private readonly StreamWriter _sw;
    private bool _disposed;

    public FileLogger(string path) => _sw = new StreamWriter(path, append: true);

    public void Log(string message)
    {
        ObjectDisposedException.ThrowIf(_disposed, this); // rzuca ObjectDisposedException po Dispose!
        _sw.WriteLine(message);
    }

    public void Dispose()
    {
        if (_disposed) return; // wielokrotne Dispose() musi być bezpieczne!
        _sw.Dispose();         // zwalniamy wszystkie posiadane zasoby IDisposable
        _disposed = true;
    }
}
```

**Wersja 2 — pełny Dispose Pattern (TYLKO gdy klasa trzyma bezpośrednio niezarządzany uchwyt, np. `IntPtr`):**
* Dodajemy `protected virtual void Dispose(bool disposing)` oraz finalizator `~MyClass() => Dispose(false);`.
* W `Dispose()` wywołujemy `Dispose(true); GC.SuppressFinalize(this);`.
* W `if (disposing)` zwalniamy obiekty **zarządzane** (`_sw?.Dispose()`).
* Poza `if (disposing)` (czyli także z finalizatora) zwalniamy **WYŁĄCZNIE niezarządzane** zasoby (np. `Marshal.FreeHGlobal(ptr)` — w finalizatorze **nie wolno** dotykać innych obiektów zarządzanych!).
* W nowoczesnym .NET zamiast ręcznego finalizatora i `IntPtr` lepiej używać `SafeHandle`.


### Kiedy `using var ...;` vs `using (...) { ... }` (i co wymaga `using`)?

* **Wymaga `using` (implementuje `IDisposable`):** wszystkie strumienie (`FileStream`, `MemoryStream`, `GZipStream`, `BufferedStream`, `entry.Open()`), adaptery (`StreamReader`/`Writer`, `BinaryReader`/`Writer`), archiwa (`ZipArchive`, `TarWriter`/`Reader`), `FileSystemWatcher`, `Timer`, `IncrementalHash`, `File.OpenHandle(...)`.
* **NIE wymaga `using`:** `FileInfo`, `DirectoryInfo`, `DriveInfo`, `Path`, fasady `File.ReadAll*` (nie są `IDisposable` albo same zamykają plik).

| Wariant | Kiedy używać? | Przykład |
| :--- | :--- | :--- |
| **`using var x = ...;`** *(bez klamr)* | **Domyślnie (90% przypadków)** — zwalnia na końcu bieżącej metody / bloku `if` / iteracji `foreach`. Czytelniejszy kod. | `using var fs = File.OpenRead("a.txt");` |
| **`using (var x = ...) { ... }`** *(z klamrami)* | Gdy musisz **zamknąć zasób WCZEŚNIEJ**, np. zaraz potem czytasz ten sam plik, sprawdzasz `ms.Length` po `GZipStream(..., leaveOpen: true)` albo tworzysz kilka wpisów w `ZipArchive(Create)` po kolei. | `using (var sw = new StreamWriter("a.txt")) { sw.WriteLine("Hi"); }` $\rightarrow$ potem `File.ReadAllText("a.txt")` |
| **`await using var x = ...;`** | Dla obiektów implementujących **`IAsyncDisposable`** (np. `await ZipFile.OpenReadAsync(...)` w .NET 10). | `await using var zip = await ZipFile.OpenReadAsync("a.zip");` |
