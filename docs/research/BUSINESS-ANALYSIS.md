# ZnanyLekarz: analiza i przełożenie na gabinety-tech (materiał wejściowy właściciela)

> **Status dokumentu.** To materiał wejściowy od właściciela produktu, zapisany w języku polskim jako źródło dla
> [`SPEC.md`](../spec/SPEC.md). Nie jest specyfikacją: nic w nim nie zobowiązuje implementacji. Liczby o ZnanyLekarz i
> Docplannerze pochodzą z deklaracji i agregatorów, nie były weryfikowane przez autora specyfikacji i są oznaczone
> jako orientacyjne. Cennik ZnanyLekarz **nie jest potwierdzony** (patrz §3).
>
> **Nazwa.** Robocza nazwa `znany-programista.pl` została zmieniona na **gabinety-tech** z powodu bliskości do znaku
> ZnanyLekarz (patrz §7). Decyzje, które z tej analizy wynikają, są w [`DECISIONS.md`](../spec/DECISIONS.md) (D-21 i dalej).
>
> **Zasada pracy.** Kopiujemy mechanikę modelu, nie markę i nie treści. Żaden profil, opinia ani tekst z serwisu
> ZnanyLekarz nie jest pobierany, kopiowany ani używany jako dane testowe (SPEC §1.2, DECISIONS D-03).

## 1. Co to jest

ZnanyLekarz to polska platforma do szukania lekarzy i umawiania wizyt. Ma ponad 240 tys. lekarzy i specjalistów. Dla
pacjentów jest darmowa. Według deklaracji DSA z czerwca 2026 ma ok. 15,4 mln użytkowników miesięcznie. Powstał jako serwis
z opiniami o lekarzach. Założyciel Docplannera kupił go w 2010 r. i zbudował na nim globalną grupę. Docplanner działa
w 13 krajach, ma ponad 2,8 mln profesjonalistów i obsługuje ponad 20 mln rezerwacji miesięcznie. Po drodze przejął m.in.
Doctoralię, tuOtempO i jamedę.

## 2. Jak to działa

**Strona pacjenta (darmowa):**
- Wyszukiwanie po specjalizacji, zabiegu lub nazwisku oraz po mieście lub dzielnicy.
- Rezerwacja terminu, a potem przypomnienia SMS i e-mail.
- Opinie, pytania do lekarza z odpowiedziami publicznymi oraz strony zabiegów i chorób (to ruch z SEO).
- Wizyta w gabinecie albo online.

**Strona profesjonalisty:**
- Darmowy profil, który tworzy ZnanyLekarz albo sam lekarz.
- Płatny pakiet ZnanyLekarz Pro z kalendarzem online, rezerwacjami i komunikacją z pacjentami. Do tego asystent AI Noa Notes do
  notatek z wizyt.
- Każdy profesjonalista ma jeden profil, a uprawnienia są weryfikowane w ciągu 7 dni od podpisania umowy. Poświadczana jest też
  każda specjalizacja.

**Mechanizmy zaufania (to jest sedno produktu):**
- Platforma sama decyduje o weryfikacji, publikacji i usuwaniu opinii. Lekarz może odpowiadać, ale nie może z tego powodu
  zerwać umowy.
- Opinie od osób, które umówiły wizytę przez platformę, są oznaczane jako najbardziej wiarygodne. Publikuje się ponad 1500
  opinii dziennie, każda jest moderowana.
- Pozycja w rankingu zależy od jawnych "Zasad Rankingu", które platforma może zmieniać jednostronnie. Żadna pozycja nie jest
  gwarantowana.

## 3. Model biznesowy

To dwustronny marketplace z SaaS-em i prowizją za pierwszą wizytę:

| Strumień | Jak działa |
|---|---|
| Pacjenci | Darmowi, to strona podażowa ruchu |
| Opłata za rezerwację | Klient płaci opłatę za gotowość oraz opłatę za każdą pierwszą rezerwację pacjenta przez platformę. Stawki są w ofercie i mogą się zmienić z 15-dniowym wyprzedzeniem. |
| No-show | Za nieobecność pacjenta nie ma opłaty, ale tylko jeśli lekarz zgłosi ją w ciągu 7 dni. To wymusza lojalne raportowanie. |
| Rozliczenia | Miesięcznie, płatność w 14 dni, przy zaległościach blokada dostępu po 14 dniach. |
| Terapia (osobny produkt) | Konsultacje 149–449 zł, platforma pobiera prowizję od realnej ceny. Zakaz kierowania pacjentów poza platformę. |
| SaaS | Pro, kalendarz, AI do notatek i oprogramowanie dla klinik (tuOtempO, MyDr) |

**Cennik: nie potwierdzony.** Oficjalna strona /cennik nie otworzyła się. Agregator Subger podaje 199–499 zł/mies. dla lekarza,
ale to słabe źródło. Cennik trzeba zdjąć ręcznie, wchodząc na stronę jako "lekarz" (zadanie właściciela, nie agenta).

**Skala:** przychód spółki ZnanyLekarz w 2024 r. to ok. 158 mln zł, wobec ok. 125 mln zł w 2023 i ok. 94 mln zł w 2022. To dane
z agregatora cytującego Biznes Gazele, więc traktować je orientacyjnie. Łącznie Docplanner zebrał około 130 mln euro w rundach
do Series E.

## 4. Co jest tam prawdziwą przewagą (a nie kodem)

1. **SEO.** Strony "specjalizacja × miasto" i "zabieg × miasto" generują darmowy ruch. To jest główny kanał pozyskiwania
   pacjentów.
2. **Opinie powiązane z rezerwacją.** Daje to zaufanie, a rezerwacja przez platformę generuje dane do rozliczeń.
3. **Zablokowanie przez kalendarz.** Gdy lekarz prowadzi grafik w platformie, trudno mu odejść.
4. **Płatność za wynik** (pierwsza wizyta nowego pacjenta), nie za samo ogłaszanie się.

## 5. Przełożenie na programistów: gdzie analogia działa, a gdzie nie

| ZnanyLekarz | gabinety-tech |
|---|---|
| Specjalizacja × miasto | Technologia/problem × typ zlecenia (np. "migracja .NET", "automatyzacja z AI") |
| Wizyta (slot w kalendarzu) | Konsultacja/audyt 30–60 min w stałej cenie, potem projekt |
| Opinie po wizycie | Opinie po zleceniu, oznaczone jako zweryfikowane tylko przez platformę |
| Weryfikacja uprawnień | Weryfikacja: GitHub, portfolio, referencje, firma (NIP) |
| Opłata za pierwszą rezerwację | Opłata za pierwszą konsultację lub lead |
| Pro (kalendarz, CRM) | Kalendarz, wycena, umowa, faktura, escrow |

**Uczciwa ocena słabych punktów, bo od tego zależy czy to się opłaca:**

- **Wyciek (leakage).** To największy problem. U lekarza pacjent wraca do gabinetu, ale pierwsza wizyta już jest opłacona
  i terapia jest regulowana. Klient programisty po pierwszej rozmowie pójdzie bezpośrednio, i to w 100% przypadków. Pobieranie
  opłaty tylko za pierwsze spotkanie ratuje model, ale lojalność trzeba zbudować przez wartość: escrow, umowa wzorcowa,
  fakturowanie, ubezpieczenie, rozliczanie w jednym miejscu.
- **Brak lokalnej rzadkości.** Lekarza wybierasz po dzielnicy, programistę możesz wziąć z całego świata. Konkurencja to Useme,
  Upwork, Toptal, Contra i polskie bazy freelancerów i software house'ów. Przewaga "miasto" odpada, więc trzeba innej osi
  (stały zakres, cena, stack).
- **Częstotliwość popytu.** Wizyta lekarska jest potrzebą powszechną i powtarzalną. Zlecenie programistyczne to rzadka decyzja,
  więc ruch z SEO będzie dużo mniejszy.
- **Ocena jakości.** Pacjent nie umie ocenić lekarza i dlatego opinie są tak ważne. Zleceniodawca może przynajmniej obejrzeć
  kod, ale zwykle też nie umie, więc zaufanie nadal jest wartością.
- **Hipoteza "gabinetów AI".** Zakłada, że rośnie liczba solo-programistów sprzedających usługi dzięki AI. To plus, bo po
  stronie podaży jest dużo chętnych, ale zbudowanie popytu to osobny, trudniejszy problem.

## 6. Proponowany MVP

1. Strony wizytówek programistów (darmowy profil) i strony kategorii pod SEO.
2. "Rezerwacja" jednej płatnej konsultacji przez kalendarz.
3. Opinie dopuszczone tylko po zrealizowanej konsultacji.
4. Opłata: prowizja od pierwszej konsultacji. Dopiero później abonament Pro (CRM, umowy, faktury).
5. Zacząć od jednej niszy (np. .NET, migracje legacy, integracje AI), nie od "wszystkich programistów".

## 7. Dwie praktyczne uwagi

- **Nazwa.** "znany-programista.pl" jest bardzo blisko znaku ZnanyLekarz. Ryzyko pomyłki i sporu o znak jest realne, więc
  wybrano własną nazwę (gabinety-tech). Kopiujemy model, nie markę.
- **Dane.** Kopiujemy mechanikę, a nie treści: nie scrapujemy profili ani opinii lekarzy z ich serwisu.

## Źródła podane przez właściciela

- ZnanyLekarz, strona główna (znanylekarz.pl)
- Warunki umowy dla klientów biznesowych (znanylekarz.pl/crm/doc/premium_terms/20230303060001)
- Umowa ZnanyLekarz Terapia (terapia.znanylekarz.pl/documents/terms/umowa-wspolpracy-terapeuci-pl.pdf)
- Wywiad z CEO ZnanyLekarz, Menedżer Zdrowia
- Docplanner, Series E (TechCrunch)
- Forbes.pl, przejęcie MyDr
- Tear sheet z danymi o przychodach (siliconvalleyinvestclub.com)
- Subger, orientacyjne ceny (niepotwierdzone)
