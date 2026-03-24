# WeeChat
WeeChat to aplikacja czatu z naciskiem na bezpieczeństwo dzieci w sieci. Wiadomości wysyłane do dzieci obsługiwane są poprzez moduł moderacji – wszystkie wiadomości przychodzące pokazują się w panelu moderacji powiązanego rodzica. W zależności od wybranej polityki wiadomości mogą wymagać ręcznego zatwierdzenia lub odrzucenia. Dodatkowo przy moderacji asystuje (lub dokonuje decyzji) wydany przez NASK przetrenowany model HerBERT-PL-Guard, który klasyfikuje wiadomości na bezpieczne lub niebezpieczne (14 group niebezpiecznych).  


## Architektura i stack technologiczny 

### Frontend: 
TypeScript/React (toolset Vite) z obsługą logowania rodzica i dziecka, czatu w czasie rzeczywistym i paneli moderacji. 

### Backend: 
Java/Spring Boot (REST + WebSocket STOMP). 

### Bazy danych: 

PostgreSQL: dane kont, relacji rodzic-dziecko, znajomości. 

MongoDB: 
wątki i wiadomości czatu. 

### Asystent moderacji AI: 
Python/ FastAPI - HerBERT-PL-Guard - używany asynchronicznie do oceny wiadomości względem bezpieczeństwa. 

### Reverse proxy: 
nginx 

### CDN: 
również symulowany przez osobny nginx 

## Zabezpieczenia 

### Uwierzytelnianie 

Uwierzytelnianie zrealizowane zostało w o Spring Security w połączeniu z JSON Web Tokenami tworzonymi i weryfikowanymi z pomocą JJWT (io.jsonwebtoken).  

Z tego powodu wyłączony jest springowy CSRF, w zamian na zapytaniach aplikowane są filtry JWT. 

Tworzone są dwa tokeny: 

Access token o TTL 5 minut, służący do uwierzytelniania przy korzystanie z endpointów. REST API odczytuje go z ciasteczek, w przypadku Web Socketów jest on przekazywany w postaci nagłówka Authorization: Bearer. 

Refresh token o TTL 30 dni, służący do odświeżania sesji (generowania nowego access tokenu). 

Tokeny podpisywane są HS256 i przechowywane są w ciasteczkach z ustawieniami HttpOnly, SameSite=Lax, Secure (w środowisku developerskim Secure jest ustawione na false), dzięki czemu skrypty JS nie mają do nich dostępu, nie są przekazywane przy zapytaniach międzydomenowych, przekazywane są przez HTTPS. 

Większość endpointów, oprócz tych odpowiedzialnych za logowanie, wymaga przekazania access tokenu by uzyskać odpowiedź, w przeciwnym wypadku zwrócą 401 Unauthorized. 

### CORS 

Ograniczono również możliwość wysyłania żądań z innych domen z pomocą security.cors.allowed-origins, czyli listy dozwolonych domen. 

### Autoryzaja 

Autoryzacja na REST API egzekwowana jest filtrami JWT. W przypadku WebSocketów JWT jest sprawdzany w momencie nawiązywania połączenia (STOMP interceptor), następnie: 

Dostęp do wątków kontrolowany przez członkostwo w wątku. 

Moderacja wiadomości wymaga bycia rodzicem powiązanym z dzieckiem z wątku. 

## Architektura baz danych 

PostgreSQL 

Baza relacyjna PostgreSQL została użyta do przechowywania kont użytkowników oraz relacji między nimi. 

Poniżej znajduje się schemat bazy danych tak jak ją zaprojektowano, natomiast w końcowej wersji projektu nieużywana pozostaje tabela account_block reprezentująca zablokowane kontakty. Dodatkowo nieużywana jest kolumny app_user.two_factor_secret i app_user.two_factor_enabled – funkcjonalność uwierzytelniania wieloskładnikowego nie została zaimplementowana w obecnej wersji. 

<img width="614" height="684" alt="image" src="https://github.com/user-attachments/assets/7b5eec62-9d67-47b1-8e89-d65e00feb694" />

Zaimplementowano migrację bazy z wykorzystaniem flyway. Skrypt został wyedytowany 
ręcznie w taki sposób, by wypełniał bazę przykładowymi danymi.  

## MongoDB 
Do przechowywania wątków, wiadomości oraz decyzji moderacyjnych na wiadomościach 
użyto MongoDB – bazy no-SQL przechowującej dokumenty JSON. Schemat poniżej 
przedstawia w konwencji UML strukturę dokumentów – bloki oznaczone <<collection>> 
reprezentują kolekcję (a więc dokument), a bloki opisane <<embedded>> są 
zagnieżdżone w dokumentach (należą do nich/są ich częścią). 
Podobnie jak w przypadku PostgreSQL, tak i struktura dokumentów MongoDB zawiera 
elementy nieużywane: nie zaimplementowano funkcjonalności dodawania załączników 
oraz potwierdzeń dostarczenia i odczytania wiadomości.

<img width="638" height="339" alt="image" src="https://github.com/user-attachments/assets/4b51a8f0-5b45-4af6-a3e6-0dc0b0335958" />


## Funkcjonalność oraz widoki

Użytkownik może stworzyć pełnoprawne konto przed zalogowaniem:

<img width="617" height="599" alt="image" src="https://github.com/user-attachments/assets/598a84e4-df76-49be-827c-1ebb1c36ce6e" />

Użytkownicy pełnoprawnych kont (app_user) uwierzytelniają się loginem i hasłem:

<img width="613" height="474" alt="image" src="https://github.com/user-attachments/assets/efe0023a-3f12-4102-975d-6d69353b59b8" />

Aplikacja umożliwia przesyłanie wiadomości między dwoma użytkownikami:

<img width="541" height="340" alt="image" src="https://github.com/user-attachments/assets/2387116d-eeda-4d0d-986b-f4810e4b134a" />

Użytkownicy mogą nawiązywać znajomości:

<img width="594" height="356" alt="image" src="https://github.com/user-attachments/assets/2b11475b-331b-4ac8-96b5-5fd3f329b607" />

Zaproszenia do znajomych skierowane do dzieci są potwierdzane przez powiązane konto opiekuna: 

<img width="612" height="303" alt="image" src="https://github.com/user-attachments/assets/1331839b-4870-4c3f-9423-027dd2e8e7b9" />

Wiadomości skierowane do dzieci przechodzą przez kolejkę moderatorską. Model LLM podpowiada, czy wiadomości są bezpieczne:

<img width="604" height="416" alt="image" src="https://github.com/user-attachments/assets/236410bb-a855-485b-9656-494939d43b3e" />

Niezatwierdzone oraz odrzucone wiadomości nie są widoczne z poziomu konta dziecka:

<img width="608" height="361" alt="image" src="https://github.com/user-attachments/assets/1595b493-b4d6-4183-93d5-1d8cde06661d" />

Rodzic może tworzyć nowe konta dzieci: 

<img width="610" height="500" alt="image" src="https://github.com/user-attachments/assets/abd0bd35-6b40-44d9-803e-c6cfcf319d28" />

Logowanie do konta dziecka wymaga wygenerowania jednorazowego kodu w panelu rodzica:

<img width="504" height="412" alt="image" src="https://github.com/user-attachments/assets/84ce25d8-48f8-464e-9bd3-d8cd3bbd0363" />

<img width="525" height="441" alt="image" src="https://github.com/user-attachments/assets/155148e5-55e3-4b38-bdf0-535227b639bc" />

Ten sam kod może zostać wykorzystany do połączenia istniejącego konta dziecka z innym rodzicem:

<img width="510" height="605" alt="image" src="https://github.com/user-attachments/assets/a753634f-02d6-4bc7-95e2-58f51ebc733b" />

## Konteneryzacja 

Na diagramie poniżej przedstawiono stack kontenerów, wraz z zależnościami między nimi, 
przynależnościami do sieci oraz ekspozycją portów.

<img width="629" height="353" alt="image" src="https://github.com/user-attachments/assets/af0079a7-6a8a-46fd-a5e2-4277bdd64d74" />


Kontener nginx-edge jest jedynym z portami wystawionymi publicznie – jest to reverse 
proxy i serwer staticów. 
Kontener pl-guard-weechat, choć umieszczony w sieci public (oprócz private) nie 
wystawia żadnych portów. Taki zabieg wynika z faktu, że jest to kontener usługi HerBER
PL-Guard i przy pierwszym uruchomieniu wymagane jest pobranie modelu z repozytorium 
HugginFace (https://huggingface.co/NASK-PIB/HerBERT-PL-Guard), a więc wymaga 
dostępu do internetu (sieć private jest siecią wewnętrzną i takiego dostępu nie ma). 
Umieszczenie modelu w obrazie kontenera byłoby niezgodne z licencją modelu. 
Backend, CDN oraz bazy danych umieszczono wyłącznie w sieci private. API backendu 
oraz CDN wystawione są na widok publiczny przez reverse proxy nginx-edge, a bazy 
danych komunikują się wyłącznie z backendem. 
Dodatkowy kontener nginx-init korzysta z obrazu Alpine 
(https://hub.docker.com/_/alpine) by uruchomić komendę mkdir i utworzyć katalogi 
wymagane przez kontenery nginx. 
Umieszczenie backendu oraz symulowanego CDN za reverse proxy (nginx-edge) w jednej 
sieci wynikało umieszczenia wszystkich serwisów na pojedynczej maszynie i chęci 
serwowania na standardowych portach 80 oraz 443. W środowisku produkcyjnym, 
ponieważ CDN i serwer frontendu zupełnie nie są zależne od backendu, można się 
pokusić o uruchomienie ich kontenerów na zupełnie osobnych maszynach. Poniżej 
uproszczony schemat pomijający reverse proxy i load balancing.

<img width="600" height="484" alt="image" src="https://github.com/user-attachments/assets/171217f8-91e4-458c-884d-655214a5e903" />















