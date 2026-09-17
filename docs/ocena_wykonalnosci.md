# Ocena wykonalności — raport / dashboard struktury i ustawień klienta

## Cel

Biznes chce **jednego, kompleksowego raportu** prezentującego pełną strukturę
klienta w formie **hierarchicznego drzewka („kto pod kim")** — podobnie do
drzewka widocznego dziś w Webshopie — wraz z kluczowymi danymi
konfiguracyjnymi. Dane są dziś rozproszone po wielu narzędziach, a do części
biznes nie ma dostępu.

Niniejszy dokument to **ocena wykonalności**: dla każdej wnioskowanej kolumny
sprawdzamy, czy da się ją zbudować z bazy **HALO (Oracle: POLAND / SHARED /
WORK_POLAND)**, czy jej tam nie ma.

**Podstawa oceny:** zrzut struktury bazy `RAG_XML_HALO.xml` (366 tabel + próbki
danych) oraz notatka architektoniczna `DB_KNOWLEDGE (2).txt` z repozytorium
`BAZA_WIEDZY_HALO_RAG`. Nazwy tabel i kolumn zweryfikowano w zrzucie struktury.

Legenda statusów: ✅ **MAMY** · 🟡 **CZĘŚCIOWO** · ❌ **BRAK w HALO**

---

## Wniosek ogólny

- **W pełni wykonalne z bazy HALO** (~60% wnioskowanych kolumn): hierarchiczne
  drzewko + dane podstawowe klienta + data ostatniego zamówienia + eksperci
  PPE/Higiena/NBS + promocje + godziny + reguły aktywności (2 lata / HADES /
  blokady).
- **Luki wymagające osobnego źródła** (Webshop / inne narzędzie, do którego
  biznes „w ogóle nie ma dostępu"): **BPO**, **budżety i okresy**,
  **minikatalog**, **maks. liczba zamówień**, **CROSS-y na produktach**, pełne
  **MPK per odbiorca**, **opłata administracyjna** jako osobna pozycja oraz
  jawna **lista dopuszczeń**.

To pokrywa się z uwagą biznesu, że dane są rozproszone i do części nie ma
dostępu — brakujące pola to w większości ustawienia po stronie Webshopu, które
nie są replikowane do bazy Oracle HALO.

---

## A. Szkielet drzewka „kto pod kim" — ✅ MAMY

Hierarchia klienta jest w bazie kompletna i wystarcza do odwzorowania drzewka
z Webshopu:

- `POLAND.T_ACCOUNT` — master wszystkich kont: `NATIONAL_LEADER_NUMBER`,
  `INTERNATIONAL_LEADER_NUMBER`, `HIERARCHY_LEVEL`, flagi ról
  (`SOLDTO_FLAG` / `SHIPTO_FLAG` / `BILLTO_FLAG` / `PAYER_FLAG`), `NAME`, adres.
- `POLAND.T_SOLDTO` → `POLAND.T_SHIPTO` (join po `SOLDTO_NUMBER`).
- Szczyt hierarchii: `TOP_LEVEL = COALESCE(NATIONAL_LEADER_NUMBER, SOLDTO_NUMBER)`.
- Poziomy drzewka: **LEADER (grupa zakupowa) → SOLDTO (zamawiający) → SHIPTO
  (odbiorca dostaw)**.

---

## B. Dane podstawowe klienta — ✅ MAMY

| Pole wnioskowane | Status | Źródło (tabela.kolumna) |
|---|---|---|
| Numer klienta | ✅ | `T_ACCOUNT.ACCOUNT_NUMBER` / `T_SOLDTO.SOLDTO_NUMBER` / `T_SHIPTO.SHIPTO_NUMBER` |
| Dane kontaktowe | ✅ | `T_ECOM_ACCOUNT` (`USER_EMAIL`, `USER_PHONE_NUMBER`, `USER_NAME`), `T_ACCOUNT` |
| Adres | ✅ | `T_ACCOUNT` (`STREET`, `BUILDING`, `ZIP_CODE`, `CITY`, `GPS_LATITUDE/LONGITUDE`) |
| NIP | ✅ | `T_PAYER.TAX_NUMBER1` (normalizacja `REGEXP_REPLACE(...,'[^0-9]','')`) |
| Cennik | ✅ | `T_SOLDTO.PRICE_LIST_CODE` (+ `PRICE_BAND_CODE`) |
| Warunki płatności | ✅ | `T_PAYER.PAYMENT_TERMS_CODE` → `SHARED.T_PAYMENT_TERMS` |

---

## C. Pozostałe wnioskowane kolumny

| Pole wnioskowane | Status | Źródło / uwaga |
|---|---|---|
| Data ostatniego zamówienia | ✅ MAMY | `T_SOLDTO.LAST_ORDER_DATE` (+ `FIRST_ORDER_DATE`, `LAST_SALES_ACTIVITY_DATE`) |
| Eksperci PPE / Higiena / NBS | ✅ MAMY | `POLAND.T_SALES_REP_VS_EXPERT`: `EXPERT_PPE`, `EXPERT_HYG` (Higiena), `EXPERT_NBS`, `EXPERT_IS` — join po `AREA_CODE` |
| Promocje | ❌ **BRAK** | Brak danych VISTEX|
| Godziny dostaw / otwarcia | ✅ MAMY (semantyka do potwierdzenia) | `T_SITI_CUSTOMER_OPENING_HOURS` (`MONDAY_OPENING_HOURS`…`FRIDAY_OPENING_HOURS`), `T_SF_PARTNER` (`START_OPENING_HOUR`, `END_OPENING_HOUR`), `T_SF_PARTNER_BUSINESS.DELIVERY_TIME_CODE` |
| Listy dopuszczeń i wykluczeń | 🟡 CZĘŚCIOWO | słownik `SHARED.T_EXCLUSION_LIST`; przypisanie **wykluczeń** na poziomie produktu: `T_ACCOUNT_PRODUCT.EXCLUSION_LIST_CODE`. Brak jawnej **listy dopuszczeń** (najbliżej: `EXCLUSIVE_PROPOSAL`) |
| Numer zamówienia klienta | 🟡 CZĘŚCIOWO | `T_INVOICE_LINE` / `T_ORDER_LINE`: `PURCHASE_ORDER_NUMBER`, `BLANKET_PO_NUMBER` — to dane **transakcyjne** (per linia dokumentu), nie stałe ustawienie klienta |
| MPK (numery, na których założone) | 🟡 CZĘŚCIOWO | `T_SF_PARTNER.COST_CENTER_FLAG` = tylko **flaga** „czy klient używa MPK". Kolumna `MPK` istnieje wyłącznie w tabeli szkoleń `T_LD_MANDAYS_TRAININGS_EMPLOYEE` (niezwiązana z klientem). **Brak numerów MPK per odbiorca** |
| Opłata administracyjna | 🟡  | są opłaty transportowe: `DELIVERY_CHARGE_AMOUNT`, `FREIGHT_CHARGES_AMOUNT`, `DEPOSIT_FEE` |
| **BPO** (numery + rodzaje BPO) | ❌ BRAK | brak jakiejkolwiek kolumny/tabeli BPO w HALO (jedyne „BPO" w danych to fragment nazwy firmy). Funkcja Webshopu — nie replikowana do Oracle |
| **Budżety** (wartości + okresy obowiązywania) | ❌ BRAK | tabele `*_BUDGET` (`T_SALES_BL_BUDGET`, `T_SALES_SD_BUDGET`, `T_SALES_BL_BUDGET_PER_SECTION`…) dotyczą **wewnętrznych targetów sprzedaży** per Business Line/Section, a nie budżetów zakupowych klienta z Webshopu |
| **Podpięty minikatalog** | ❌ BRAK | brak powiązania klient → minikatalog (jest tylko flaga `NBS_KATALOG` oraz `CATALOG_PAGE` na poziomie produktu) |
| **Maks. liczba zamówień w okresie budżetowym** | ❌ BRAK | brak kolumny limitu zamówień |
| **CROSS-y na produktach** | ❌ BRAK | brak tabel/kolumn typu cross-sell / substytutów / powiązań produktowych |

---

## D. Reguły aktywności odbiorców — ✅ WYKONALNE

Wszystkie trzy proponowane reguły da się zrealizować z HALO:

- **Numery bez zakupu > 2 lata → nie uwzględniać w raporcie:** filtr po
  `T_SOLDTO.LAST_ORDER_DATE` (i/lub flagach `SOLDTO_PURCHASING_ACTIVITY_FLAG_12M`
  / `_6M` / `_3M`); dla poziomu SHIPTO — po historii `T_INVOICE_LINE.INVOICE_DATE`.
- **Nieaktywni max 2 lata w raporcie:** ta sama data graniczna (2 lata wstecz
  od daty raportu).
- **Wykluczyć odbiorców z HADES:** anti-join do `WORK_POLAND.T_HADES_CA` po
  numerze klienta.
- **Wykluczyć odbiorców z BLOKADĄ:** `T_ORDER_BLOCK` / `T_DELIVERY_BLOCK`,
  `FINANCIAL_BLOCKING_FLAG`, `ECOM_ACCESS_BLOCKED_FLAG`, `QUARANTINE_FLAG`.

---

## E. Rekomendacje dla luk

Pola oznaczone ❌ (BPO, budżety i okresy, minikatalog, maks. liczba zamówień,
CROSS-y) oraz pełne MPK per odbiorca to konfiguracja **po stronie Webshopu /
innych narzędzi**. Aby znalazły się w raporcie, potrzebny jest osobny, cykliczny
**eksport z Webshopu** (lub innego systemu źródłowego), który następnie można:

1. wgrać jako tabelę roboczą do `WORK_POLAND` (procedura
   `WORK_POLAND.WORK_SCHEMA_CREATE_TABLE`, wg konwencji z `DB_KNOWLEDGE`), a potem
2. dołączyć do raportu po `SOLDTO_NUMBER` / `SHIPTO_NUMBER` / `NIP`.

Dopóki takiego eksportu nie ma, raport można zbudować w wersji **MVP** na
kolumnach ✅/🟡 z sekcji A–D, a pola ❌ zostawić jako miejsca do uzupełnienia po
udostępnieniu źródła.
