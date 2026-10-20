# Week 7 RAG Evidence

## 1. Cilj zadatka

Cilj je bio da se na početnom RAG sistemu pronađe konkretan problem, primeni tačno jedna kontrolisana izmena i izmeri razlika između početnog i završnog stanja na istom eval skupu.

Eval skup `eval/eval_set.json` nije menjan.

## 2. Okruženje i početna provera

Korišćeno okruženje:

- Windows
- Python 3.14.0
- pytest 8.3.3
- virtuelno okruženje `.venv`

Komanda:

```powershell
python -m pytest
```

Rezultat:

```text
18 passed
```

## 3. Početno merenje

Komanda:

```powershell
python -m src.eval_runner --out results/before.json
```

Početna konfiguracija je koristila:

```text
CHUNK_STRATEGY = "chars"
CHUNK_SIZE = 140
CHUNK_OVERLAP = 30
TOP_K = 2
RETRIEVAL_MIN_SCORE = 0.01
STOPWORDS_ENABLED = false
CURRENT_ONLY = false
REFUSAL_THRESHOLD = 0.02
CITATION_MIN_SCORE = null
```

Početni rezultat:

| Metrika                 | Vrednost |
| ----------------------- | -------: |
| Passed                  |     6/12 |
| Behaviour accuracy      |   0.8333 |
| Retrieval hit rate      |   0.7778 |
| Mean citation precision |   0.4545 |

SHA-256 eval skupa:

```text
afe46db1e75a0b85bb5c9a93b6763e3b96e5cd7b289e12da6d2671e4d8650064
```

Početni rezultati su sačuvani u `results/before.json`.

## 4. Uočeni problemi

### E04 — problem na granici chunk-a

Pitanje:

```text
How many business days does a laptop replacement take?
```

Očekivano je bilo da odgovor sadrži činjenicu `5 business days` iz dokumenta `hb-01`.

Sistem je pronašao očekivani dokument, ali činjenica nije bila u jednom citiranom chunku. Ručna provera je pokazala da je tekst presečen:

```text
hb-01-chunk-0: ... handover can take up to
hb-01-chunk-2: iness days. Report lost or stolen ...
```

Rezultat:

```text
Hit: yes
Fact: no
CitPrec: 1.00
Pass: FAIL
```

Tip problema: `chunk-boundary`.

### E09 — problem na granici chunk-a

Pitanje:

```text
Where do I request a replacement laptop and how long does it take?
```

Očekivano je bilo da citati podrže i mesto zahteva i trajanje zamene laptopa.

Očekivani dokument `hb-01` jeste pronađen, ali činjenica `5 business days` ponovo nije bila sačuvana u odgovarajućem celom chunku.

Rezultat:

```text
Hit: yes
Fact: no
CitPrec: 1.00
Pass: FAIL
```

Tip problema: `chunk-boundary`.

### Ostali uočeni problemi

- `E05` — parafraza „notebook computer“ nije pronašla dokument o laptopu; sistem je vratio onboarding i annual leave chunkove. Tip: `retrieval/paraphrase`.
- `E06` — početno je vraćena zastarela politika rada na daljinu (`hb-02`, verzija 2024), umesto aktuelne politike (`hb-03`, verzija 2026). Tip: `versioned/retrieval`.
- `E10` — sistem je odgovorio na pitanje o trenutnoj ceni akcija, iako priručnik to ne pokriva. Tip: `pogrešno odbijanje`.
- `E11` — sistem je odgovorio na pitanje o roditeljskom odsustvu koristeći nepovezane informacije o godišnjem odmoru. Tip: `pogrešno odbijanje`.

## 5. Specifikacija promene

Izabrana je promena `M1 — Sentence-aware chunking`.

### Problem

Chunkovanje po karakterima može da preseče jednu rečenicu i time odvoji ključnu činjenicu od ostatka konteksta. To se konkretno desilo kod pitanja `E04` i `E09`.

### Hipoteza

Ako se dokumenti dele po celim rečenicama, ključna činjenica `5 business days` ostaće u istom chunku i biće pravilno pronađena i citirana.

### Planirana promena

- Implementirati `split_text_by_sentences` u `src/chunking.py`.
- Uključiti `CHUNK_STRATEGY = "sentences"` u `src/config.py`.
- Rečenice pakovati do `CHUNK_SIZE` karaktera.
- Rečenicu dužu od `CHUNK_SIZE` ostaviti kao samostalan chunk.
- Ne koristiti overlap kod sentence-aware strategije.

### Očekivani efekat

Očekivano je poboljšanje pitanja `E04` i `E09`, uz mogućnost da se skorovi i rezultati drugih pitanja promene zbog drugačije strukture chunkova.

### Rizik regresije

Promena veličine i granica chunkova može promeniti retrieval skorove, redosled rezultata i citate kod pitanja koja su ranije prolazila.

### Provera

Promena je proverena pomoću:

```powershell
python -m pytest
python -m src.eval_runner --out results/after.json
python -m src.eval_compare results/before.json results/after.json
```

## 6. Implementirana promena

U `src/chunking.py` implementirano je deljenje teksta na rečenice pomoću granica posle tačke, uzvičnika ili upitnika.

Rečenice se zatim pakuju u chunkove dok ukupna dužina ne pređe `CHUNK_SIZE`. Rečenice duže od ograničenja ostaju samostalne.

U `src/config.py` promenjeno je samo:

```diff
-CHUNK_STRATEGY = "chars"
+CHUNK_STRATEGY = "sentences"
```

Nisu menjani:

- `eval/eval_set.json`,
- kod za računanje metrika,
- retrieval pragovi,
- `TOP_K`,
- refusal prag,
- stopword podešavanja,
- filtriranje verzija dokumenata.

Promena je jedna po nameri: promena strategije chunkovanja.

## 7. Provera testova posle izmene

Komanda:

```powershell
python -m pytest
```

Rezultat:

```text
18 passed
```

## 8. Merenje posle izmene

Komanda:

```powershell
python -m src.eval_runner --out results/after.json
```

Rezultat po pitanjima:

| ID  | Očekivanje | Rezultat | Hit | Fact | CitPrec | Pass |
| --- | ---------- | -------- | --- | ---- | ------: | ---- |
| E01 | answer     | answer   | yes | yes  |    0.50 | PASS |
| E02 | answer     | answer   | yes | yes  |    0.50 | PASS |
| E03 | answer     | answer   | yes | yes  |    1.00 | PASS |
| E04 | answer     | answer   | yes | yes  |    1.00 | PASS |
| E05 | answer     | answer   | no  | no   |    0.00 | FAIL |
| E06 | answer     | answer   | yes | yes  |    0.50 | PASS |
| E07 | answer     | answer   | yes | yes  |    1.00 | PASS |
| E08 | answer     | answer   | yes | yes  |    0.50 | PASS |
| E09 | answer     | answer   | yes | yes  |    1.00 | PASS |
| E10 | refuse     | answer   | -   | -    |    0.00 | FAIL |
| E11 | refuse     | answer   | -   | -    |    0.00 | FAIL |
| E12 | refuse     | refuse   | -   | -    |       - | PASS |

Završne metrike:

| Metrika                 |    Pre |  Posle |
| ----------------------- | -----: | -----: |
| Passed                  |   6/12 |   9/12 |
| Behaviour accuracy      | 0.8333 | 0.8333 |
| Retrieval hit rate      | 0.7778 | 0.8889 |
| Mean citation precision | 0.4545 | 0.5455 |

SHA-256 eval skupa posle izmene ostao je isti:

```text
afe46db1e75a0b85bb5c9a93b6763e3b96e5cd7b289e12da6d2671e4d8650064
```

Završni rezultat je sačuvan u `results/after.json`.

## 9. Poređenje pre i posle

Komanda:

```powershell
python -m src.eval_compare results/before.json results/after.json
```

Rezultat:

```text
Fixed (fail -> pass): E04, E06, E09
Regressed (pass -> fail): none
Unchanged: E01, E02, E03, E05, E07, E08, E10, E11, E12

Summary:
  passed: 6 -> 9 (+3)
  total: 12 -> 12 (+0)
  behaviour_accuracy: 0.8333 -> 0.8333 (+0.0)
  retrieval_hit_rate: 0.7778 -> 0.8889 (+0.1111)
  mean_citation_precision: 0.4545 -> 0.5455 (+0.091)

Changed settings:
  CHUNK_STRATEGY: 'chars' -> 'sentences'
```

Nije bilo regresija. Promena je popravila tri pitanja: `E04`, `E06` i `E09`.

`E06` je poboljšan kao posledica drugačijih chunk granica, iako `CURRENT_ONLY` nije uključen. Aktuelni dokument `hb-03` sada je prvi rezultat i sadrži činjenicu `3 days per week`, ali se u citatima i dalje pojavljuje i zastareli dokument `hb-02`.

## 10. Ručna provera posle izmene

### E04

Komanda:

```powershell
python -m src.cli --question "How many business days does a laptop replacement take?" --show-retrieved
```

Najvažniji rezultat:

```text
[hb-01-chunk-1] After manager approval, a replacement laptop is set up by IT staff with your files, accounts, email and access rights, and handover can take up to 5 business days.
```

Činjenica `5 business days` sada je cela u jednom citiranom chunku. To potvrđuje rezultat evaluacije `Fact=yes`, `CitPrec=1.00`, `PASS`.

### E09

Ručna provera je vratila dva relevantna chunk-a:

```text
[hb-01-chunk-2] Report lost or stolen devices to security first. Request a replacement laptop in IT portal under Hardware.
[hb-01-chunk-1] After manager approval, a replacement laptop is set up by IT staff with your files, accounts, email and access rights, and handover can take up to 5 business days.
```

Sada su u citatima prisutne i informacija gde se podnosi zahtev i informacija da zamena traje do `5 business days`.

## 11. Preostala ograničenja

Promena nije rešila sve probleme sistema:

1. `E05` i dalje ne razume dovoljno dobro parafrazu „notebook computer“ za „laptop“.
2. `E10` i `E11` i dalje daju odgovor na pitanja koja bi trebalo odbiti. Prag za odbijanje i dalje nije kalibrisan.
3. `E06` sada pronalazi aktuelnu politiku, ali citira i zastarelu verziju jer `CURRENT_ONLY` ostaje `false`.
4. Sentence-aware chunking menja chunk granice i retrieval skorove za ceo korpus.
5. Jednostavno prepoznavanje rečenica može imati ograničenja kod skraćenica ili složenije interpunkcije.

Ograničenja su namerno ostavljena jer je zadatak zahtevao tačno jednu promenu.

## 12. Korišćenje AI asistenta

AI asistent je korišćen za:

- razumevanje zahteva zadatka i strukture projekta,
- analizu početnih eval rezultata,
- izbor/specifikaciju promene M1,
- implementaciju sentence-aware chunkovanja,
- tumačenje rezultata pre i posle izmene.

Predlog asistenta nije tretiran kao dokaz. Implementacija je proverena nezavisno pomoću:

- `pytest` testa,
- pune evaluacije pre i posle,
- poređenja `eval_compare`,
- ručne CLI provere pitanja `E04` i `E09`.

## 13. Doprinos članova para

Ovaj deo treba dopuniti stvarnim imenima i doprinosima:

| Član             | Doprinos                         |
| ---------------- | -------------------------------- |
| Milena Paripović | Testiranje i izmene u kodu       |
| Dušan Nikolić    | Provera koda i praćenje postupka |

Osoba koja vodi demonstraciju na Sesiji 2: Milena Paripović.

## 14. Artefakti

- `results/before.json`
- `results/after.json`
- `src/chunking.py`
- `src/config.py`
- ovaj izveštaj: `evidence.md`
