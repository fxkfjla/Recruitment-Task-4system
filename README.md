# Opis aplikacji

### Aplikacja jest rozbita na dwa współgrające ze sobą moduły:

## Aplikacja Spring Boot

Głównym celem tej aplikacji jest dostarczenie endpointów takich jak:

1. Importowanie danych w formacie XML do bazy danych.
2. Zwracanie wszystkich użytkowników z bazy danych.
3. Wyszukiwanie użytkowników po imieniu, nazwisku lub loginie.

Dodatkowo endpointy korzystają ze stronnicowania aby zapobiec zwracania wszystkich użytkowników na raz oraz sortowania po id, imieniu, nazwisku i loginie. 

Struktura aplikacji jest oparta na architekturze `Spring model-view-controller (MVC)` i separuje implementację między kontrollerem, serwisem i repozytorium. Przy uruchomieniu automatycznie tworzy strukturę bazy danych, a przy zatrzymaniu ją usuwa.

#### Aplikacja implementuje następujące funkcjonalności:

##### 1. Repozytorium użytkownika
Korzysta ze `Spring Data JPA`, które generuje podstawowe metody `CRUD` podczas działania aplikacji oraz między innymi implementuje sortowanie oraz stronnicowanie. Implementuje również własne metody takie jak czyszczenie tabeli lub wyszukiwanie użytkownika po imieniu, nazwisku lub loginie, które również są generowane podczas działania aplikacji.

##### 2. Serwis użytkownika
Separuje logikę od kontrolera dzięki czemu pozwala na przejrzyste wywoływanie metod w kontrolerze. Korzysta z `UserRepository` i konwertera typów aby kod był przejrzysty. Implementuje metody takie jak:

1. `uploadXMLFile(MultipartFile)`: metoda służąca do importu pliku XML; sprawdza czy format pliku zgadza się z oczekiwanym, jeżeli tak to konwertuje plik do listy za pomocą `XMLDataHandler`, którą zapisuje do tableli w bazie danych z wcześniejszym usunięciem znajdujących się tam użytkowników. Jeżeli plik nie jest formatu XML to wyrzuca wyjątek, który jest obsługiwany przez `GlobalExceptionHandler`. W przypadku niewłaściwej zawartości pliku, obsługa wyjątków jest przeprowadzana w `XMLDataHandler`.

2. `findAll(PageRequestDTO)`: zwraca wszystkich użytkowników z bazy danych; konwertuje własny typ danych `PageRequestDTO` na `PageRequest` przy użyciu konwertera i przy użyciu `UserRepository` zdobywa listę użytkowników z wcześniej określonym stronnicowaniem i sortowaniem. Funkcja zwraca listę aby w prostszy sposób obsłużyć ją w aplikacji implementującej interfejs użytkownika, więc dodatkowo zawiera w nagłówku ilość wszystkich elementów strony.

3. `findByNameOrSurnameOrLogin(String, PageRequestDTO)`: wykonuje to samo co funkcja `findAll(PageRequestDTO)`, ale dodatkowo pozwala na wyszukanie konkretnych użytkowników po podanym ciągu znaków.

4. `getXTotalCountHeader(long)`: tworzy, zwraca i dodaje nagłówek `X-Total-Count` do `HttpHeaders` z ilością wszystkich elementów strony.

##### 3. Kontroler użytkownika
Jest to typowo RESTowy kontroler, nie zajmuje się zwracaniem widoków, a jedynie implementuje i udostępnia endpointy. Domyślna ścieżka tego kontrolera to `/api/users`.

1. `uploadXMLFile(MultipartFile)`: metoda oznaczona jest `PostMapping` aby dodatkowo zasugerować, że wysyła dane i powoduje zmianę na serwerze. Argument posiada adnotację `@RequestPart`, ponieważ oczekuje danych typu `multipart/form-data`.

2. `findAll(PageRequestDTO)`: posiada adnotację `GetMapping`, ponieważ ta metoda tylko zwraca dane z serwera i nie powoduje żadnych zmian. Oznaczenie `Valid` uruchamia walidację, która sprawdza czy obiekt spełnia wymagania takie jak nieujemna ilość stron itd.

3. `findByNameOrSurnameOrLogin(String, PageRequestDTO)`: działa na tej samej zasadzie co metoda `findAll(PageRequestDTO)` z dodatkową funkcją wyszukiwania po podanym ciągu znaków.

##### 4. Modele

1. Klasa `User` reprezentuje strukturę danych użytkownika i jest ona wykorzystywana do mapowania obiektowo-relacyjnego przy użyciu `Java Persistance API`. Posiada pola takie jak imię, nazwisko i login oraz gettery i settery do tych pól. Nadpisuje również funkcje `equals` i `hashCode` aby obiekty tej klasy mogły być wykorzystywane w strukturach danych bazujących na hashowaniu.

2. Klasa `UserList` to klasa stworzona na potrzebny poprawnego konwertowania listy użytkowników na plik XML i na odwrót. Przechowuje referencję do listy użytkowników oraz posiada gettery, settery i adnotacje potrzebne do poprawnego skonwertowania obiektu. 

3. Klasa `PageRequestDTO` to typowa klasa `Data Transfer Object (DTO)`, służąca do przesyłania danych między warstwami aplikacji. Jest to własna implementacja klasy `PageRequest` korzystająca z mechanizmu walidacji; stworzona po to by jednoznacznie określić wymogi, które muszą spełniać parametry wysyłane przez żądania `HTTP`. 

##### 5. Klasy pomocnicze

Klasa `XMLDataHandler` została stworzona na potrzebę odizolowania logiki konwersji i generacji pliku XML od reszty programu. Deklaruje i definiuje metody statyczne aby pominąć potrzebę alokacji obiektów tej klasy.

1. `convertXMLToList(MultipartFile)`: jako argument przyjmuje plik z rozszerzeniem XML. Jeżeli dane w pliku są poprawnie sformatowane, wywołuje metodę `unmarshalFile(MultipartFile, Unmarshaller)` i konwertuje plik XML na obiekt `UserList`, z którego wyciągamy listę użytkowników. Jeżeli dane w pliku nie są poprawnie sformatowane, to metoda wyrzuci wyjątek `InvalidXMLDataException`, który zostanie obsłużony przez `GlobalExceptionHandler`. 

2. `generateUsersToXML(int, String, boolean)`: jako argument przyjmuje ilość użytkowników do generacji, nazwa pliku do jakiego użytkownicy mają się wygenerować oraz flagę, która określa czy zawartość wygenerowanego pliku będzie sformatowana, czy też będzie napisana ciągiem jako jeden łańcuch znaków. Metoda korzysta z właściwej metody `generateUsersToList(int)`, która generuje listę użytkówników. Po wygenerowaniu listy użytkowników, funkcja używa obiektu typu `Marshaller` do skonwertowania obiektu typu `UserList` na plik z rozszerzeniem XML.

3. `generateUsersToList(int)`: generuje podaną ilość użytkowników do list.

4. `unmarshalFile(MultipartFile, Unmarshaller)`: konwertuje podany plik XML na obiekt typu `UserList`.

Klasa `PageRequestDTOToPageRequestConverter` została stworzona po to aby zwiększyć przejrzystość kodu. Konwersja z `PageRequestDTO` na `PageRequest` odbywa się kilka razy w programie i ta klasa pozwala na zachowanie czystości kodu. Implementuje interfejs `Converter<S, T>` i nadpisuje metodę `convert(S)`.

##### 6. Obsługa wyjątków

Do obsługi wyjątków została stworzona klasa `GlobalExceptionHandler` i wyłapuje takie wyjątki jak `InvalidXMLDataException`, `JAXBInitializationRuntimeException` oraz `HttpRequestMethodNotSupportedException`. Wyjątki związane z formatem XML i inicjalizacją `JAXBContext` są własną implementacją wyjątków i powstały po to aby wyróżnić te konkretne wyjątki od innych. Klasa `GlobalExceptionHandler` używa adnotacji `ControllerAdvice`, która jest specjalizacją adnotacji `Component` i służy do automatycznego wykrywania wyjątków i przypisywania ich do odpowiednich oznaczeń `ExceptionHandler`.

##### 7. Pliki konfiguracyjne

Z racji korzystania z endpointów przez źródła zewnętrzne, została stworzona konfiguracja `CorsConfig` dla `Cross-Origin Resource Sharing (CORS)`. Obecna konfiguracja zezwala na wszystkie żądania, metody i nagłówki oraz udostępnia do użytku nagłówek `X-Total-Count`. W fazie produkcyjnej najpewniej należałoby ograniczyć ten plik konfiguracyjny tylko do zezwalania na żądania z zaufanych źródeł.

Konfiguracja `WebConfig` jest wymagana po to aby powiadomić aplikację o istnieniu konwertera `PageRequestDTOToPageRequestConverter`.

##### 8. Testy

W plikach źródłowych zawarte są również pliki testowe, w których zaimplementowana jest większość ważnych testów dla tego typu aplikacji.

##### Link do repozytorium na githubie: https://github.com/fxkfjla/Recruitment-Task-Back

## Aplikacja Angular 

Głównym celem tej aplikacji jest dostarczenie użytkownikowi przejrzystego interfejsu graficznego i wykorzystanie funkcjonalności, które dostarcza aplikacja serwerowa. Do implementacji interfejsu użytkownika aplikacja dodatkowo korzysta z takich narzędzi jak `Bootstrap` i `SASS`.

#### Aplikacja implementuje następujące funkcjonalności:

##### 1. Serwis użytkownika

Klasa `UserService` została stworzona aby odizolować logikę zapytań do serwera od komponentów, które na ich podstawie tworzą interfejs użytkownika. Korzysta z `HttpClient` do wysyłania zapytań.

1. `uploadXMLFile(File)`: przyjmuje plik typu `multipart/form-data` i wysyła z nim żadanie do serwera na odpowiedni adres.

2. `findAll(number, number, string, string)`: przyjmuje argumenty takie jak strona, ilość rekordów na stronie, kierunek sortowania oraz pole po którym chcemy posortować dane i wysyła zapytanie do serwera z odpowiednimi parametrami.

3. `findByNameOrSurnameOrLogin(string, number, number, string, string)`: działa na tej samej zasadzie co metoda `findAll(number, number, string, string)` z dodatkową funkcją wyszukiwania po podanym ciągu znaków.

##### 2. Modele, enumeracje i zmienne środowiskowe

Interfejs `User` jest odzwierciedleniem klasy `User` po stronie serwera i jest używany do mapowania danych z serwera na konkretne obiekty. Posiada pola takie jak id, imię, nazwisko i login.

W zmiennych środowiskowych przetrzymywana jest zmienna `apiUrl`, która powstała po to aby w łatwy sposób podmienić adres serwera na etapie produkcyjnym.

Enumeracja `ApiPaths` przechowuje wszystkie ścieżki kontrolerów z aplikacji serwerowej.

##### 3. Komponenty

1. Ekran startowy `start-screen`: przez to, że zdecydowałem się na stworzenie paska nawigacyjnego aby trochę polepszyć doświadczenie użytkownika końcowego, ten ekran nie posiada przycisków przenoszących do innych ekranów, ale posiada pasek nawigacyjny, który ma te przyciski zaimplementowane.

2. Pasek nawigacyjny `header`: posiada prosty `CSS` i `HTML`, który implementuje dwa przyciski przenoszące na dwa pozostałe ekrany. Korzysta z `navbar` zaimplementowany przez `Bootstrap` oraz rozszerza `navbar-brand` o dodatkową stylistykę.

3. Ekran z formularzem `form-screen`: posiada prosty ostylowany formularz pozwalający na przesłanie plików z systemu. Korzysta z `UserService` do importu wybranego pliku na serwer oraz `ngx-toastr` do wyświetlania odpowiednich powiadomień dotyczących importu danych. Posiada również flagę informującą o skończeniu importu danych, z której korzysta przycisk, który pojawia się po imporcie danych i pozwala na przejście do ekranu z podglądem użytkowników. Import plików zajmuje trochę czasu, więc proszę o cierpliwość i przejście na stronę wyświetlającą listę użytkowników dopiero po pojawieniu się przycisku i komunikatu o pomyślnym zaimportowaniu danych.

4. Ekran z listą użytkowników `list-screen`: posiada prostą ostylowaną tabelę wyświetlającą użytkowników oraz pola do sortowania i wyszukiwania. Przechowywuje zmienne odnoszące się do aktualnie wybranej opcji sortowania, wyszukiwania i ilości stron oraz implementuje następujące funkcje:

    * `findUsers()`: wywołuje wyszukiwanie wszystkich użytkowników lub użytkowników po wpisanym ciągu znaków w zależności czy zmienna przechowująca ten ciąg znaków jest pusta.
    
    * `findAllUsers()`: korzysta z `UserService` do zdobycia listy wszystkich użytkowników i funkcji `countTotalPages(HttpHeaders)`, `updateVisiblePages()` oraz `addMd5Hash()` do odebrania liczby wszystkich elementów, zaktualizowania interfejsu użytkownika oraz dodania hashy z imienia do nazwiska.
        
    * `findByNameOrSurnameOrLogin()`: wykonuje to samo co funkcja `findAllUsers()`, ale dodatkowo pozwala na wyszukanie konkretnych użytkowników po podanym ciągu znaków.
    
    * `countTotalPages(HttpHeaders)`: odzystkuje wartość `X-Total-Count` z nagłówka i na jej podstawie wylicza ilość stron.
    
    * `updateVisiblePages()`: aktualizuje tablicę z indeksami stron, które powinny wyświetlać się pod tabelą. Od strony na której aktualnie jesteśmy odejmuje połowę wartości ze zmiennej odpowiadającej za określenie ile widocznych stron powinno się wyświetlać i przypisuje to do zmiennej przechowywującej indeks pierwszej widocznej strony. Ostatnia widoczna strona to indeks pierwszej widocznej strony plus wszystkie widoczne strony - 1, ponieważ liczymy też stronę, która wyświetla się jako pierwsza. Jeżeli ostatnia widoczna strona wybiega poza ilość wszystkich stron, to przyjmuje wartość maksymalną, a strona początkowa liczona jest od końca.
    
    * `addMd5Hash()`: wykorzystuje funkcję `MD5` z `CryptoJS`, liczy hash z imienia każdego użytkownika w liście po kolei i dodaje go do nazwiska.

    * `onSortOptionChange()`: wyłuskuje kierunek sortowania i pole, po którym chcemy sortować, ze zmiennych, których wartość zmienia się w zależności od zmiany komponentów w `HTML` i `Two-way binding`.
    
    Reszta funkcji to gettery i settery lub funkcje wywołujące inne funkcje ze zmianą wartości jednej zmiennej.


Wszystkie style komponentów importują plik `variable.sass`, w którym znajdują się zmienne określające paletę kolorów aplikacji i nadpisane zmienne z `Bootstrap` oraz plik `style.sass`, który implementuje kilka uniwersalnych klas.

##### 4. Testy

W plikach źródłowych zawarte są również pliki testowe, w kórych zaimplementowane są testy komponentów, serwisu i interfejsu graficznego.

##### Link do repozytorium na githubie: https://github.com/fxkfjla/Recruitment-Task-Front

# Instrukcja uruchomienia

1. Uruchom serwer mysql i połącz się do niego jako root: `mysql -u root -p`

2. Utwórz bazę danych i użytkownika:

        CREATE DATABASE db_example;
        CREATE USER 'test_user'@'localhost' IDENTIFIED BY 'password';
        GRANT ALL PRIVILEGES ON db_example.* TO 'test_user'@'localhost';

3. Przejdź do folderu Spring-Boot i uruchom aplikację: `./mvnw spring-boot:run`

4. Przejdź do folderu Angular i uruchom aplikację:

        npm install
        ng serve

5. Wejdź na adres `localhost:4200` w przeglądarce.