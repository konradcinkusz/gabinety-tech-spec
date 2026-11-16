# Gabinety Tech: materiał do przeglądu prawnego (T-10.02)

Wersja robocza z 2026-11-16. Autor: sesja dyrektora (model językowy), nie prawnik. Produkt jest w fazie specyfikacji, kod nie istnieje, nie ma kont u żadnych dostawców. Przepisy są wskazane jako obszary do weryfikacji; to prawnik ocenia, co ma zastosowanie.

## 0. Czego oczekujemy od prawnika

Teksty do dostarczenia (wersjonowane, z datą):
1. Regulamin dla klientów (konsumentów i przedsiębiorców).
2. Regulamin dla praktyków (programistów).
3. Polityka prywatności i klauzule informacyjne (RODO art. 13/14).
4. Informacja o plikach cookie (tylko sesyjne, wymagane do działania).
5. Opis zasad rankingu (`/zasady-rankingu`, wersja `2026-11-v1`).
6. Opis zasad opinii i ich weryfikacji.
7. Procedura zgłaszania i rozstrzygania sporów oraz zgłaszania treści (punkt kontaktowy, notice and action).
8. Umowy powierzenia lub podpowierzenia przetwarzania danych (dostawcy z §8.3).

Plus odpowiedzi na pytania z §12 i na założenia A-06, A-07, A-13 (SPEC §12.1).

## 1. Co to za usługa

Polski marketplace. Klient z problemem programistycznym rezerwuje jedną płatną konsultację wideo, 30 lub 60 min, w stałej cenie, u zweryfikowanego programisty („gabinet"). Programista ma bezpłatny profil, kalendarz i obsługę płatności. Opinię może zostawić tylko klient, którego konsultacja się odbyła.

- Platforma nie hostuje wideo: spotkanie jest w narzędziu praktyka, a link klient widzi dopiero po płatności.
- Poza zakresem MVP: wycena projektów, umowy, escrow projektowy, fakturowanie automatyczne, abonament, czat, aplikacje mobilne, wiele walut i języków. Tylko PLN, tylko polski interfejs.
- Pierwsza nisza: .NET (migracja legacy, integracje AI).
- Model jest wzorowany na mechanice ZnanyLekarz. Nie kopiujemy marki, treści, profili ani opinii z żadnego serwisu (SPEC §1.2). Robocza nazwa „znany-programista" została zmieniona z powodu bliskości do znaku ZnanyLekarz.

## 2. Strony i role

| Strona | Kim jest | Co robi |
|---|---|---|
| Platforma (operator) | Do ustalenia: forma prawna, NIP, adres (§12 pyt. 1). Administratorem i moderatorem jest na starcie właściciel | Prowadzi serwis, pobiera opłatę, rozstrzyga spory |
| Klient | Zalogowany użytkownik: konsument lub przedsiębiorca (system tego nie rozróżnia) | Rezerwuje, płaci, ocenia |
| Praktyk | Programista z NIP (format i suma kontrolna, bez sprawdzenia w rejestrze) | Sprzedaje konsultację, ustala cenę |
| Dostawca płatności | Nie wybrany (Stripe Connect, Przelewy24, Tpay; żaden nie zweryfikowany) | Przyjmuje płatność, dzieli środki, zwraca |

## 3. Przepływ pieniędzy (założenie A-06)

Decyzja architektoniczna (ADR-0005): platforma nigdy nie przechowuje salda ani cudzych środków. Płatność idzie przez dostawcę z podziałem płatności lub kontami połączonymi. Opłata platformy trafia do platformy, reszta do praktyka. Odrzucone alternatywy: własny rachunek escrow (ekspozycja regulacyjna) oraz opłata do praktyka z fakturą później (odtwarza wyciek).

Czas: rezerwacja blokuje termin na 10 minut. Wypłata dla praktyka następuje po `koniec + 48 h`, o ile nie otwarto sporu.

Do oceny: czy ta struktura jest dopuszczalna w prawie polskim i unijnym (usługi płatnicze, agent rozliczeniowy, wyłączenie dla platform handlowych), przy jakim dostawcy i na jakich warunkach umowy z nim.

## 4. Opłata platformy (D-23, ADR-0008)

- 15 % ceny pierwszej zakończonej konsultacji danej pary klient–praktyk, zaokrąglone w górę od połowy do grosza. Kolejne konsultacje tej pary: 0 %. Koszty dostawcy płatności ponosi platforma.
- Stawka to placeholder właściciela, nie wynik analizy. Zmiana wymaga decyzji właściciela i nowej wersji tekstów.
- Wzór: `opłata = (cena_gr × 1500 + 5000) / 10000` (dzielenie całkowite). Wektory: 250 zł → 37,50 zł; 333,33 zł → 50,00 zł; 50 zł → 7,50 zł; 0,01 zł → 0.
- Przy częściowym zwrocie opłata liczona od kwoty zatrzymanej.
- Cel (ADR-0008): ograniczenie wycieku (klient i praktyk umawiają się poza platformą po pierwszym spotkaniu).

Do oceny: czy „opłata tylko za pierwszą konsultację pary" i identyfikacja pary w księdze są zgodne z prawem; czy zakaz obchodzenia platformy może być w regulaminie i w jakiej formie.

## 5. Reklamacje, zwroty i spory (D-25, CONTRACTS §8.1, §9.2)

| Zdarzenie | Zwrot dla klienta | Praktyk otrzymuje | Opłata |
|---|---|---|---|
| Klient odwołuje ≥ 24 h przed startem | 100 % | nic | brak |
| Klient odwołuje < 24 h przed startem | 0 % | cena − opłata | tak |
| Praktyk odwołuje (przed końcem) | 100 % | nic | brak |
| Płatność przyszła po wygaśnięciu blokady | 100 % | nic | brak |
| Konsultacja odbyta, brak sporu | 0 % | cena − opłata po `koniec + 48 h` | tak |
| Nieobecność klienta (zgłasza praktyk w 72 h) | 0 % | cena − opłata | tak |
| Nieobecność praktyka (klient zgłasza w 72 h) | 100 % + kara w rankingu | nic | brak |
| Spór rozstrzygnięty „release" | 0 % | cena − opłata | tak |
| Spór „refund_full" | 100 % | nic | brak |
| Spór „refund_partial" | kwota ustalona przez administratora | reszta − opłata od reszty | tak |

Klient może otworzyć spór w ciągu 48 h od końca konsultacji. Spór rozstrzyga administrator (na starcie właściciel).

Do oceny:
- Zgodność z prawami konsumenta: prawo odstąpienia od umowy zawartej na odległość, wyjątki dla usług świadczonych w określonym terminie, informacja przed zawarciem umowy i zgoda na rozpoczęcie świadczenia.
- Brak zwrotu przy odwołaniu < 24 h i traktowanie jej jak odbytej (klauzule abuzywne).
- Czy właściciel jako administrator może rozstrzygać spory i na jakiej podstawie (regulamin, polubowne rozstrzyganie, ODR).
- Kto jest sprzedawcą konsultacji wobec klienta: praktyk czy platforma.

## 6. Opinie (D-29, SPEC §1.1, CONTRACTS §5)

- Opinię może napisać tylko klient rezerwacji zakończonej jako odbyta (nie odwołanej późno), raz na rezerwację, w ciągu 14 dni; ocena 1–5, tekst 20–2000 znaków.
- Praktyk może raz odpowiedzieć, nie może usunąć opinii.
- Reguły automatyczne (linki, dane kontaktowe, zakazane słowa, powtórzenia) kierują opinię do publikacji albo wstrzymania. Opcjonalnie triaż modelu AI, tylko doradczy: brak odpowiedzi modelu nie blokuje publikacji, a o usunięciu decyduje człowiek (administrator).
- Autor jest pokazany jako „Zweryfikowany klient", bez imienia.
- Średnia ocena jest pokazywana od 3 opinii; wcześniej strona mówi „Nowy".
- Administrator może wstrzymać lub odrzucić opinię (także opublikowaną). Praktyk nie może.
- Praktyk nie może zarezerwować własnej oferty. Lista administratora wyróżnia pary z więcej niż 2 rezerwacjami i samymi pięciogwiazdkowymi opiniami.

Do oceny: wymogi dotyczące opinii konsumenckich (informacja o sposobie weryfikacji, zakaz fałszywych opinii, nieuczciwe praktyki rynkowe); moderacja i usuwanie opinii a DSA (uzasadnienie, odwołanie); dane autora opinii (nazwa niepokazywana, ale powiązana w bazie z identyfikatorem użytkownika).

## 7. Ranking (D-30, CONTRACTS §9.3)

Wersja `2026-11-v1`. Wynik = `100 × (0,45·B + 0,25·C + 0,15·A + 0,15·P)`:
- B: wygładzona średnia ocen (waga prior 5, średnia 3,8),
- C: terminowość i rzetelność: (zrealizowane + 3) / (zrealizowane + odwołane przez praktyka + podtrzymane nieobecności + 4),
- A: dostępność w 14 dni: min(1, wolne terminy / 10),
- P: kompletność profilu.

Żaden parametr nie jest płatny (ani plan, ani promocja, ani opłata, ani wynik modelu). Zasady mają być publiczne na `/zasady-rankingu` z wersją i datą; każda zmiana to nowa wersja z wpisem w historii.

Do oceny: wymogi przejrzystości rankingu wobec użytkowników biznesowych (rozporządzenie P2B 2019/1150) i wobec konsumentów; czy wzór trzeba ujawniać w całości, czy wystarczy opis parametrów.

## 8. Dane osobowe (A-13, SPEC §8)

### 8.1 Inwentarz

| Dane | Gdzie | Uwagi |
|---|---|---|
| Dane logowania, zgody, e-mail | usługa tożsamości authservice (jeszcze nie zbadana, T-01.01) | Prawa RODO (zgody, eksport, usunięcie) realizowane przez nią |
| Profil praktyka: nazwa, nagłówek, opis (≤ 2000 zn.), login GitHub, link do portfolio, NIP, domyślny link do spotkania | `directorydb` | Profil publiczny po zatwierdzeniu; czy NIP jest pokazywany publicznie: do potwierdzenia w kodzie |
| Rezerwacja: opis problemu klienta (≤ 2000 zn.), termin, cena, link do spotkania | `bookingdb` | Opis może zawierać dane wrażliwe lub tajemnice firmowe, czego nie kontrolujemy |
| Płatności, księga rozliczeń, wypłaty | `bookingdb` | Księga tylko-dopisywana, służy księgowości |
| Opinie i odpowiedzi | `reputationdb` | |
| Powiadomienia e-mail | `bookingdb` + dostawca e-mail (nie wybrany) | |
| Zużycie asystenta AI | `assistantdb` | Bez treści zapytań i odpowiedzi; identyfikator użytkownika tylko jako solony skrót |

Zasada projektowa: teksty użytkowników (opis problemu, opinie, bio, oferty) nigdy nie trafiają do logów, śladów ani metryk; loguje się długość i skrót. E-mail, imię, NIP, treść i token nigdy w logach.

### 8.2 Prawo do usunięcia a księgowość

Po usunięciu konta w authservice właściciel uruchamia usuwanie (runbook) w trzech usługach:
- Directory: profil zanonimizowany („Usunięty użytkownik", tekst wyczyszczony, status zawieszony).
- Reputation: tekst opinii zastąpiony „Opinia usunięta na wniosek autora", ocena zostaje.
- Booking: wiersze księgi zostają (A-13: okres przechowywania do potwierdzenia).

Do oceny: okres przechowywania dokumentów księgowych, podstawa zatrzymania wiersza księgi z identyfikatorem użytkownika, dopuszczalność pozostawienia oceny po usunięciu autora, ujęcie tego w polityce prywatności, los opisu problemu w rezerwacji.

### 8.3 Dostawcy i przekazywanie danych (do wypełnienia po wyborze)

Każdy to potencjalny procesor lub współadministrator. Kraje i umowy powierzenia nieznane do czasu wyboru:
- hosting: Fly.io (region Frankfurt, baza PostgreSQL w tym samym regionie) [do potwierdzenia],
- płatności: nie wybrany,
- e-mail transakcyjny: nie wybrany,
- model językowy (asystent): konto BYOK (Anthropic, OpenAI lub Azure), nie wybrane,
- GitHub (logowanie i weryfikacja konta),
- magazyn kopii zapasowych (dzienne zrzuty baz).

Do oceny: role (administrator/procesor), umowy powierzenia, transfery poza EOG (zwłaszcza dostawca modelu), rejestr czynności przetwarzania, DPIA, potrzeba IOD.

## 9. Asystent AI (D-31, SPEC §5 reguła 6, COPILOT-SDK)

Dwie funkcje: (1) asystent opisu problemu dla klienta (propozycja kategorii, technologii, streszczenia i do 3 pytań), (2) doradczy triaż opinii dla moderatora.

- Model nigdy nie decyduje o pieniądzach, rankingu, ocenach, weryfikacji ani usunięciu treści. Jego brak nie blokuje człowieka (zwykły formularz działa).
- Treść klienta jest wysyłana do zewnętrznego dostawcy modelu (dla ruchu publicznego tylko modele BYOK; subskrypcja Copilot właściciela nie obsługuje użytkowników, ADR-0004). Dzienny budżet 5 USD; po wyczerpaniu funkcja się wyłącza.
- Wynik modelu jest walidowany schematem i traktowany jako dane (nigdy nie renderowany jako HTML).

Do oceny: informacja dla klienta o użyciu AI i przekazaniu tekstu dostawcy, podstawa prawna, art. 22 RODO (AI jest doradcze, ale triaż wpływa na wstrzymanie opinii przez regułę), obowiązki z rozporządzenia o AI, warunki licencji dostawcy modelu wobec danych użytkowników.

## 10. Weryfikacja praktyka i odznaki (D-28)

Cztery niezależne odznaki: e-mail zweryfikowany, konto GitHub powiązane, NIP o poprawnym formacie i sumie kontrolnej (bez sprawdzenia w rejestrze), zatwierdzenie ręczne przez administratora. Warunki publikacji profilu: portfolio lub link do repozytorium, opis ≥ 200 znaków, co najmniej jedna aktywna oferta.

Ryzyko do oceny: słowo „zweryfikowany" przy NIP bez sprawdzenia w rejestrze może wprowadzać w błąd; jak oznaczać odznaki, żeby nie obiecywać więcej, niż system sprawdza. Platforma nie weryfikuje kompetencji, tylko wymienione fakty.

## 11. Obowiązki platformy internetowej i handlowe

- DSA (rozporządzenie 2022/2065): zakres obowiązków platformy online, identyfikowalność przedsiębiorców (praktycy mają NIP), punkt kontaktowy, zgłaszanie treści i odwołania, raportowanie.
- Faktury i VAT (A-07): kto wystawia fakturę klientowi, kto za opłatę 15 %, VAT od opłaty, KSeF. W MVP oprogramowanie pokazuje zestawienie opłaty, a fakturowanie jest ręczne.
- Cookies: tylko sesyjne (wymagane do działania). Brak analityki i skryptów podmiotów trzecich na stronach publicznych; dodanie ich wymaga decyzji. Czy tekst informacyjny wystarczy bez baneru.
- Znak towarowy: nazwa „Gabinety Tech" i domena jeszcze nie sprawdzone. Właściciel zrobi wyszukiwanie znaków przed zakupem domeny.
- Własność intelektualna: treści użytkowników (opisy, opinie), licencja dla platformy, odpowiedzialność za treści.
- Charakter porady: konsultacja nie jest gwarancją wyniku; odpowiedzialność praktyka i platformy.

## 12. Pytania do prawnika

1. Jaka forma prowadzenia działalności jest właściwa dla operatora (JDG właściciela, spółka) w kontekście odpowiedzialności za rozliczenia i przepisów dla platform?
2. Czy struktura płatności z §3 wymaga zezwolenia lub wpisu do rejestru, przy założeniu, że platforma nie trzyma środków?
3. Jakie wymogi stawia wybór dostawcy płatności (umowa, KYC praktyków, konta połączone)?
4. Kto jest sprzedawcą wobec klienta, kto odpowiada za reklamację, kto wystawia fakturę i jak liczyć VAT od opłaty 15 % (§11)?
5. Czy opłata 15 % tylko od pierwszej konsultacji pary i klauzula zakazu obchodzenia platformy są dopuszczalne i jak je zapisać?
6. Czy brak zwrotu przy odwołaniu < 24 h i traktowanie jej jako zrealizowanej jest zgodne z prawem konsumentów? Jak ująć prawo odstąpienia i zgodę na rozpoczęcie świadczenia?
7. Czy administrator (właściciel) może rozstrzygać spory? Jakie procedury reklamacyjne, polubowne i ODR są wymagane?
8. Jak opisać sposób weryfikacji opinii i oznaczenie „Zweryfikowany klient" zgodnie z przepisami o opiniach konsumenckich?
9. Czy publiczny opis rankingu z §7 spełnia wymogi przejrzystości (P2B i prawo konsumenckie)? Czy trzeba ujawniać wzór?
10. Okres przechowywania księgi rozliczeń i podstawa zatrzymania danych po usunięciu konta (§8.2)?
11. Role RODO wobec dostawców z §8.3; konieczność DPIA i IOD; transfery poza EOG.
12. Informacja dla klienta o użyciu AI, przekazaniu opisu problemu dostawcy modelu, oraz obowiązki z rozporządzenia o AI.
13. Czy odznaki „zweryfikowany" są bezpieczne przy braku sprawdzenia NIP w rejestrze?
14. Które obowiązki DSA mają zastosowanie i jak je spełnić (punkt kontaktowy, notice and action, identyfikowalność)?
15. Czy cookies sesyjne wymagają baneru i jak sformułować informację?
16. Czy nazwa i domena nie naruszają cudzych znaków (wyszukanie wykona właściciel)?
17. Które z powyższych wymagają zmiany produktu, a nie tylko tekstu (np. dodatkowa zgoda w rejestracji, pole „jestem konsumentem", zakres danych)?

## 13. Czego ten dokument nie pokazuje

- Kod nie istnieje; opisy zachowań to specyfikacja, nie zmierzone działanie.
- authservice (zgody, eksport, usunięcie) nie został jeszcze zbadany (A-01).
- Dostawcy (płatności, e-mail, model, hosting) nie są wybrani; warunki ich umów są nieznane.
- Stawka 15 %, ceny 50–1500 zł, okna 24/48/72 h i 14 dni to wartości domyślne, nie wyniki analizy.
- Nie sprawdzono, czy zasady odwołań i opłaty są zgodne z prawem.

## 14. Źródła w repozytorium

SPEC.md, CONTRACTS.md §8–§10, ADR-0004/5/7/8, DECISIONS.md D-23 do D-31, SPEC §12.1 (A-04 do A-07, A-13).
