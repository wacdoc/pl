# Zbuduj swój własny serwer wysyłania poczty SMTP

## preambuła

SMTP może bezpośrednio kupować usługi od dostawców usług w chmurze, takich jak:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Push e-mail w chmurze Ali](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Możesz także zbudować własny serwer pocztowy - nieograniczone wysyłanie, niski całkowity koszt.

Poniżej pokazujemy krok po kroku jak zbudować własny serwer pocztowy.

## Wybór serwera

Samoobsługowy serwer SMTP wymaga publicznego adresu IP z otwartymi portami 25, 456 i 587.

Powszechnie stosowane chmury publiczne domyślnie zablokowały te porty i być może uda się je otworzyć wydając zlecenie, ale jest to jednak bardzo uciążliwe.

Polecam kupowanie od hosta, który ma otwarte te porty i obsługuje konfigurowanie odwrotnych nazw domen.

Tutaj polecam [Contabo](https://contabo.com) .

Contabo to dostawca hostingu z siedzibą w Monachium w Niemczech, założony w 2003 roku z bardzo konkurencyjnymi cenami.

Jeśli wybierzesz euro jako walutę zakupu, cena będzie tańsza (serwer z 8 GB pamięci i 4 procesorami kosztuje około 529 juanów rocznie, a wstępna opłata instalacyjna jest bezpłatna przez rok).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Składając zamówienie, pamiętaj, `prefer AMD` , a serwer z procesorem AMD będzie miał lepszą wydajność.

Poniżej wezmę VPS Contabo jako przykład, aby zademonstrować, jak zbudować własny serwer pocztowy.

## Konfiguracja systemu Ubuntu

System operacyjny to Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Jeśli serwer na ssh wyświetla `Welcome to TinyCore 13!` (jak pokazano na poniższym rysunku), oznacza to, że system nie został jeszcze zainstalowany. Odłącz ssh i poczekaj kilka minut, aby zalogować się ponownie.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Gdy pojawi się `Welcome to Ubuntu 22.04.1 LTS` , inicjalizacja została zakończona i można kontynuować wykonywanie poniższych czynności.

### [Opcjonalnie] Zainicjuj środowisko programistyczne

Ten krok jest opcjonalny.

Dla wygody umieściłem instalację i konfigurację systemu oprogramowania ubuntu w [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Uruchom następujące polecenie, aby zainstalować jednym kliknięciem.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Użytkownicy z Chin powinni zamiast tego użyć następującego polecenia, a język, strefa czasowa itp. zostaną ustawione automatycznie.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo umożliwia IPV6

Włącz IPV6, aby SMTP mógł również wysyłać e-maile z adresami IPV6.

edytuj `/etc/sysctl.conf`

Zmodyfikuj lub dodaj następujące wiersze

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Kontynuuj, korzystając z [samouczka contabo: Dodawanie łączności IPv6 do serwera](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Edytuj `/etc/netplan/01-netcfg.yaml` , dodaj kilka linii, jak pokazano na poniższym rysunku (domyślny plik konfiguracyjny Contabo VPS ma już te linie, po prostu odkomentuj je).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Następnie `netplan apply` , aby zmodyfikowana konfiguracja zaczęła obowiązywać.

Po pomyślnej konfiguracji możesz użyć `curl 6.ipw.cn` , aby wyświetlić adres IPv6 swojej sieci zewnętrznej.

## Sklonuj operacje repozytorium konfiguracji

```
git clone https://github.com/wactax/ops.soft.git
```

## Wygeneruj bezpłatny certyfikat SSL dla swojej nazwy domeny

Wysyłanie poczty wymaga certyfikatu SSL do szyfrowania i podpisywania.

Używamy [acme.sh](https://github.com/acmesh-official/acme.sh) do generowania certyfikatów.

acme.sh to zautomatyzowane narzędzie do podpisywania certyfikatów typu open source,

Wejdź do hurtowni konfiguracji ops.soft, uruchom `./ssl.sh` , a w **górnym katalogu** zostanie utworzony folder `conf` .

Znajdź swojego dostawcę DNS z [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , edytuj `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Następnie uruchom `./ssl.sh 123.com` , aby wygenerować certyfikaty `123.com` i `*.123.com` dla swojej nazwy domeny.

Pierwsze uruchomienie automatycznie zainstaluje [acme.sh](https://github.com/acmesh-official/acme.sh) i doda zaplanowane zadanie do automatycznego odnowienia. Możesz zobaczyć `crontab -l` , jest taka linia jak poniżej.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Ścieżka do wygenerowanego certyfikatu jest podobna do `/mnt/www/.acme.sh/123.com_ecc。`

Odnowienie certyfikatu wywoła skrypt `conf/reload/123.com.sh` , edytuj ten skrypt, możesz dodać polecenia, takie jak `nginx -s reload` aby odświeżyć pamięć podręczną certyfikatów powiązanych aplikacji.

## Zbuduj serwer SMTP za pomocą chasquid

[chasquid](https://github.com/albertito/chasquid) to serwer SMTP typu open source napisany w języku Go.

Jako substytut starożytnych programów serwerów pocztowych, takich jak Postfix i Sendmail, chasquid jest prostszy i łatwiejszy w użyciu, a także łatwiejszy do wtórnego programowania.

Uruchom `./chasquid/init.sh 123.com` zostanie zainstalowany automatycznie jednym kliknięciem (zastąp 123.com nazwą domeny wysyłającej).

## Skonfiguruj podpis e-mail DKIM

DKIM służy do wysyłania podpisów e-maili, aby zapobiec traktowaniu listów jako spamu.

Po pomyślnym uruchomieniu polecenia zostaniesz poproszony o ustawienie rekordu DKIM (jak pokazano poniżej).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Po prostu dodaj rekord TXT do swojego DNS (jak pokazano poniżej).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Wyświetl stan usługi i dzienniki

 `systemctl status chasquid` Wyświetl stan usługi.

Stan normalnej pracy przedstawiono na poniższym rysunku

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` lub `journalctl -xeu chasquid` może przeglądać dziennik błędów.

## Odwróć konfigurację nazwy domeny

Odwrotna nazwa domeny ma na celu umożliwienie przetłumaczenia adresu IP na odpowiednią nazwę domeny.

Ustawienie odwrotnej nazwy domeny może zapobiec identyfikowaniu e-maili jako spamu.

Po odebraniu poczty serwer odbierający przeprowadzi odwrotną analizę nazwy domeny na adresie IP serwera wysyłającego, aby potwierdzić, czy serwer wysyłający ma prawidłową odwrotną nazwę domeny.

Jeśli serwer wysyłający nie ma odwrotnej nazwy domeny lub jeśli odwrotna nazwa domeny nie odpowiada adresowi IP serwera wysyłającego, serwer odbierający może uznać wiadomość e-mail za spam lub ją odrzucić.

Odwiedź [https://my.contabo.com/rdns](https://my.contabo.com/rdns) i skonfiguruj, jak pokazano poniżej

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Po ustawieniu odwrotnej nazwy domeny należy pamiętać o skonfigurowaniu rozdzielczości przekazywania nazwy domeny ipv4 i ipv6 na serwer.

## Edytuj nazwę hosta chasquid.conf

Zmodyfikuj `conf/chasquid/chasquid.conf` na wartość odwrotnej nazwy domeny.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Następnie uruchom `systemctl restart chasquid` aby ponownie uruchomić usługę.

## Wykonaj kopię zapasową konfiguracji w repozytorium git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Na przykład tworzę kopię zapasową folderu conf w moim własnym procesie github w następujący sposób

Najpierw utwórz prywatny magazyn

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Wejdź do katalogu conf i prześlij do magazynu

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Dodaj nadawcę

uruchomić

```
chasquid-util user-add i@wac.tax
```

Może dodać nadawcę

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Sprawdź, czy hasło jest ustawione poprawnie

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Po dodaniu użytkownika nastąpi aktualizacja `chasquid/domains/wac.tax/users` , pamiętaj o przesłaniu go do hurtowni.

## DNS dodaje rekord SPF

SPF (Sender Policy Framework) to technologia weryfikacji wiadomości e-mail używana w celu zapobiegania oszustwom e-mailowym.

Weryfikuje tożsamość nadawcy poczty, sprawdzając, czy adres IP nadawcy jest zgodny z rekordami DNS nazwy domeny, za którą się podaje, zapobiegając wysyłaniu fałszywych wiadomości e-mail przez oszustów.

Dodanie rekordów SPF może w jak największym stopniu zapobiec identyfikowaniu wiadomości e-mail jako spam.

Jeśli Twój serwer nazw domen nie obsługuje typu SPF, po prostu dodaj rekord typu TXT.

Na przykład SPF `wac.tax` jest następujący

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF dla `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Zauważ, że mam tutaj `include:_spf.google.com` , ponieważ później skonfiguruję `i@wac.tax` jako adres wysyłania w skrzynce pocztowej Google.

## Konfiguracja DNS DMARC

DMARC to skrót od (Domain-based Message Authentication, Reporting & Conformance).

Służy do przechwytywania odrzuceń SPF (być może spowodowanych błędami konfiguracji lub kimś innym, kto podszywa się pod Ciebie, aby wysyłać spam).

Dodaj rekord TXT `_dmarc` ,

Treść jest następująca

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Znaczenie każdego parametru jest następujące

### p (Zasady)

Wskazuje, jak postępować z wiadomościami e-mail, które nie przeszły weryfikacji SPF (Sender Policy Framework) lub DKIM (DomainKeys Identified Mail). Parametr p można ustawić na jedną z trzech wartości:

* brak: żadne działanie nie jest podejmowane, tylko wynik weryfikacji jest przekazywany z powrotem do nadawcy za pośrednictwem mechanizmu raportowania e-mail.
* Kwarantanna: Umieść wiadomość, która nie przeszła weryfikacji, w folderze ze spamem, ale nie odrzuci jej bezpośrednio.
* odrzuć: Bezpośrednio odrzucaj e-maile, które nie przeszły weryfikacji.

### fo (opcje awarii)

Określa ilość informacji zwracanych przez mechanizm raportowania. Można go ustawić na jedną z następujących wartości:

* 0: Zgłoś wyniki sprawdzania poprawności dla wszystkich wiadomości
* 1: Zgłaszaj tylko wiadomości, które nie przeszły weryfikacji
* d: Zgłaszaj tylko niepowodzenia weryfikacji nazwy domeny
* s: zgłaszaj tylko błędy weryfikacji SPF
* l: Zgłaszaj tylko błędy weryfikacji DKIM

### rua & ruf

* rua (identyfikator URI raportowania dla raportów zbiorczych): adres e-mail do otrzymywania raportów zbiorczych
* ruf (identyfikator URI raportowania dla raportów kryminalistycznych): adres e-mail, na który należy otrzymywać szczegółowe raporty

## Dodaj rekordy MX, aby przekazywać e-maile do Google Maila

Ponieważ nie mogłem znaleźć darmowej firmowej skrzynki pocztowej, która obsługuje uniwersalne adresy (Catch-All, może odbierać dowolne e-maile wysyłane do tej nazwy domeny, bez ograniczeń dotyczących prefiksów), użyłem chasquid do przekierowania wszystkich e-maili na moją skrzynkę pocztową Gmail.

**Jeśli masz własną płatną firmową skrzynkę pocztową, nie modyfikuj MX i pomiń ten krok.**

Edytuj `conf/chasquid/domains/wac.tax/aliases` , ustaw przekierowanie skrzynki pocztowej

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` oznacza wszystkie wiadomości e-mail, `i` to prefiks adresu e-mail użytkownika wysyłającego utworzonego powyżej. Aby przesłać pocztę dalej, każdy użytkownik musi dodać wiersz.

Następnie dodaj rekord MX (wskazuję tutaj bezpośrednio na adres odwrotnej nazwy domeny, jak pokazano w pierwszym wierszu na poniższym rysunku).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Po zakończeniu konfiguracji możesz użyć innych adresów e-mail do wysyłania wiadomości e-mail na `i@wac.tax` i `any123@wac.tax` , aby sprawdzić, czy możesz odbierać wiadomości e-mail w Gmailu.

Jeśli nie, sprawdź dziennik chasquid ( `grep chasquid /var/log/syslog` ).

## Wyślij wiadomość e-mail na adres i@wac.tax za pomocą Google Mail

Po otrzymaniu wiadomości przez Google Mail miałem oczywiście nadzieję, że odpowiem za pomocą `i@wac.tax` zamiast i.wac.tax@gmail.com.

Odwiedź [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) i kliknij „Dodaj kolejny adres e-mail”.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Następnie wprowadź kod weryfikacyjny otrzymany w wiadomości e-mail, na którą został przekazany.

Wreszcie można go ustawić jako domyślny adres nadawcy (wraz z opcją odpowiedzi z tym samym adresem).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

W ten sposób zakończyliśmy tworzenie serwera pocztowego SMTP i jednocześnie korzystamy z Google Mail do wysyłania i odbierania wiadomości e-mail.

## Wyślij testową wiadomość e-mail, aby sprawdzić, czy konfiguracja przebiegła pomyślnie

Wpisz `ops/chasquid`

Uruchom `direnv allow` aby zainstalować zależności (direnv został zainstalowany w poprzednim procesie inicjalizacji jednym klawiszem, a do powłoki dodano hak)

następnie uruchomić

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Znaczenie parametrów jest następujące

* użytkownik: nazwa użytkownika SMTP
* hasło: hasło SMTP
* do: odbiorcy

Możesz wysłać testową wiadomość e-mail.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Zaleca się używanie Gmaila do otrzymywania e-maili testowych w celu sprawdzenia, czy konfiguracja przebiegła pomyślnie.

### Standardowe szyfrowanie TLS

Jak pokazano na poniższym rysunku, jest tam ta mała kłódka, co oznacza, że ​​certyfikat SSL został pomyślnie włączony.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Następnie kliknij „Pokaż oryginalny e-mail”

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Jak pokazano na poniższym rysunku, strona oryginalnej poczty Gmaila wyświetla DKIM, co oznacza, że ​​konfiguracja DKIM zakończyła się pomyślnie.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Sprawdź Otrzymano w nagłówku oryginalnej wiadomości e-mail, a zobaczysz, że adres nadawcy to IPV6, co oznacza, że ​​IPV6 również został pomyślnie skonfigurowany.
