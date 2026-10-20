# Nedeljni izveštaj — NACRT

## 1. Osnovne informacije

| Polje | Odgovor |
| --- | --- |
| Ime i prezime | Milena Paripović |
| Adresa e-pošte | paripovicmilena@gmail.com |
| Discord korisničko ime | Milena Paripović |
| Nedelja | 7 — Jedno kontrolisano poboljšanje RAG sistema |
| Par / tim | Milena Paripović i Dušan Nikolić |
| Moj konkretan doprinos / uloga | Radila sam testiranje i izmene u kodu. Dušan Nikolić je proverio kod i pratio postupak. |
| Datum predaje | 2026-10-20 |
| Reference na rad i dokaze | `evidence.md`, `results/before.json`, `results/after.json`, izlaz komande `eval_compare`, izmene u `src/chunking.py` i `src/config.py` |

## 2. Moj status

**Status:** Završeno

Završila sam početnu proveru, analizu problema, implementaciju jedne promene, ponovno merenje i poređenje rezultata. Ostaje da se dopuni potvrda drugog para u pratećem dokazu i da se pripremi demonstracija.

## 3. Rad ove nedelje

Zadatak je bio da na slabom lokalnom RAG sistemu pronađemo konkretne probleme, primenimo tačno jednu promenu i dokažemo njen efekat merenjem na fiksnom eval skupu od 12 pitanja. Najpre sam pokrenula testove i svih 18 testova je prošlo. Zatim sam pokrenula početnu evaluaciju, koja je pokazala 6/12 prolaznih pitanja. Početni retrieval hit rate bio je 0.7778, a mean citation precision 0.4545.

U početnom rezultatu sam identifikovala probleme kod pitanja E04 i E09. Kod oba pitanja dokument o zameni laptopa jeste pronađen, ali je činjenica `5 business days` bila presečena između chunkova. Dodatno, E05 nije pronašao relevantan dokument za parafrazu „notebook computer“, E06 je vraćao zastarelu politiku rada na daljinu, a E10 i E11 su davali odgovor iako je trebalo odbiti nepodržana pitanja.

Izabrala sam M1, sentence-aware chunking. Implementirala sam `split_text_by_sentences` u `src/chunking.py` i uključila `CHUNK_STRATEGY = "sentences"` u `src/config.py`. Rečenice se sada pakuju u chunkove do `CHUNK_SIZE`, a preduga rečenica ostaje samostalni chunk. Nisam menjala eval skup, kod za metrike ni druga podešavanja.

Posle izmene svih 18 testova je ponovo prošlo. Završna evaluacija je pokazala 9/12 prolaznih pitanja. Popravljena su E04, E06 i E09, bez regresija. Retrieval hit rate je porastao sa 0.7778 na 0.8889, a mean citation precision sa 0.4545 na 0.5455. SHA-256 eval skupa je ostao isti. Poređenje je pokazalo da je jedino podešavanje koje se promenilo `CHUNK_STRATEGY: 'chars' -> 'sentences'`.

Ručnom CLI proverom E04 potvrđeno je da se cela činjenica `5 business days` sada nalazi u jednom citiranom chunku. Kod E09 su citirani i chunk sa mestom podnošenja zahteva i chunk sa trajanjem zamene. Preostala ograničenja su da E05 i dalje ne rešava parafrazu, E10 i E11 se i dalje ne odbijaju pravilno, a E06 i dalje uključuje i zastareli dokument jer `CURRENT_ONLY` nije uključen.

AI asistent je korišćen za razumevanje zadatka i koda, izbor i implementaciju M1, kao i uređivanje dokaza. Rezultate sam proverila testovima, punom evaluacijom pre i posle, alatom `eval_compare` i ručnim CLI proverama. Dušan Nikolić je proverio kod i pratio postupak.

Pomoć tutora trenutno nije potrebna; rezultate i ograničenja imamo dokumentovane u `evidence.md`.

## 4. Sledeći korak

Pre druge sesije potvrdiću komentar drugog para u izveštaju i pripremiti kratku demonstraciju: početni problem E04, specifikaciju M1, relevantni `diff`, rezultate pre/posle i preostala ograničenja.

## 5. Poverljiva napomena za tutora

Nema dodatne napomene.

