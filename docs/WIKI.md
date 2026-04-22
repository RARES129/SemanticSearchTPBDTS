# Wiki tehnic - Semantic Search peste documente text

## 1. Obiectiv

Aplicația demonstrează căutarea semantică peste documente text cu Oracle AI Vector Search. În loc să caute doar cuvinte exacte, sistemul transformă fragmentele de text și query-ul utilizatorului în vectori și caută fragmente apropiate ca sens.

## 2. Componente

| Componentă | Tehnologie | Rol |
|---|---|---|
| Frontend | Angular 17 | Dashboard pentru upload, căutare, statistici și inspectare chunk-uri |
| Backend | Spring Boot 4 | REST API, extracție documente, chunking, embeddings, ranking |
| Vector Store | Oracle Database Free 23ai | Persistă documente, chunk-uri și vectori |
| Embedding model | AllMiniLmL6V2EmbeddingModel | Generează vectori de 384 dimensiuni |
| PDF parser | Apache PDFBox 3.0.4 | Extrage text și metadate din PDF |

## 3. Ingestie Documente

Endpoint-uri:

- `POST /api/documents/upload-file`
- `POST /api/documents/upload`
- `POST /api/documents/upload-json`

Pași:

1. Se primește fișierul sau textul manual.
2. Se determină tipul sursei: `pdf`, `txt`, `md`, `manual`.
3. Pentru PDF, textul este extras pagină cu pagină.
4. Se salvează metadatele disponibile: titlu PDF, autor, dimensiune fișier.
5. Textul este împărțit în chunk-uri.
6. Fiecare chunk primește embedding.
7. Datele sunt salvate în Oracle.

## 4. Chunking

Parametri actuali:

- dimensiune țintă chunk: `900` caractere;
- overlap: `160` caractere;
- fereastră de ajustare pentru început de chunk: `90` caractere.

Chunk-urile nu au lungimi identice deoarece algoritmul încearcă să păstreze propoziții și cuvinte întregi. Overlap-ul păstrează contextul dintre două chunk-uri vecine.

## 5. Indexare Semantică

Pentru fiecare chunk:

```java
embeddingModel.embed(normalizedChunk).content().vector()
```

Vectorul rezultat este serializat într-un format acceptat de Oracle:

```text
[0.012, -0.034, ...]
```

și salvat în:

```sql
document_chunks.embedding VECTOR(384, FLOAT32)
```

## 6. Căutare

### Semantic

Query-ul este transformat în embedding, apoi Oracle calculează:

```sql
VECTOR_DISTANCE(dc.embedding, TO_VECTOR(:searchVector), COSINE)
```

Scorul semantic afișat în UI este:

```text
1 - cosine_distance
```

### Keyword

Caută expresia textuală în `CLOB`:

```sql
DBMS_LOB.INSTR(LOWER(dc.content), LOWER(:query)) > 0
```

### Hybrid

Rulează semantic search și keyword search, unește candidații și calculează scor final. Dacă același chunk apare în ambele rezultate, păstrează scorurile maxime.

## 7. Re-Ranking

Re-ranking-ul este implementat local în `DocumentService`, fără modele externe.

Factori:

- similaritate semantică;
- potrivire keyword;
- frază exactă;
- potrivire în metadate;
- recența documentului;
- lungimea chunk-ului;
- bonus pe document.

Scop: rezultate mai ușor de explicat în demo și o comparație clară între retrieval brut și ranking compus.

## 8. UI

Dashboard-ul Angular include:

- card de încărcare documente;
- card de căutare cu moduri `Semantic`, `Keyword`, `Hybrid`;
- filtru pe document;
- checkbox pentru agregare pe document;
- checkbox pentru re-ranking;
- statistici corpus;
- tabel documente indexate;
- inspector de chunk-uri;
- rezultate cu scoruri și highlight.

## 9. Dovezi De Rulare

Capturi incluse în repository:

- `docs/assets/dashboard.png`
- `docs/assets/search-results.png`

Acestea arată corpusul indexat, statisticile, căutarea hibridă, scorurile și explicația ranking-ului.

## 10. Limitări Tehnice

- PDF-urile scanate necesită OCR.
- Pentru corpusuri foarte mari, upload-ul poate dura deoarece embeddings sunt generate local.
- Pentru volume mari ar trebui creat un index vectorial ANN în Oracle.
- Nu există autentificare; aplicația este construită pentru demo local.

## 11. Pași Pentru Rulare

1. Pornește containerul Oracle.
2. Rulează `backend/sql/init_oracle_vector_db.ps1`.
3. Pornește backend-ul cu `.\mvnw.cmd spring-boot:run`.
4. Pornește frontend-ul cu `npm start`.
5. Accesează `http://localhost:4200`.
6. Încarcă documente PDF/TXT/MD.
7. Rulează căutări în modurile semantic, keyword și hybrid.
