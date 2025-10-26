# Guida Corretta: Importazione Dati AirBnB in Neo4j

Questa guida fornisce i passaggi e gli script Cypher corretti per scaricare e importare il dataset AirBnB in un database Neo4j. Gli script sono stati aggiornati per utilizzare la sintassi moderna di Neo4j e correggono diversi errori presenti nella guida originale.

## Passaggio 1: Scarica e Posiziona i File

1.  **Scarica i Dati:** Scarica i file `listings.csv` e `reviews.csv` dal link fornito nella tua guida originale.
2.  **Trova la Cartella `import`:**
    * Apri l'applicazione Neo4j Desktop.
    * Trova il tuo database **attivo** (quello con il pallino verde "Active").
    * Clicca sui tre puntini (`...`) accanto al nome del database.
    * Seleziona **Open folder** > **import**.
3.  **Sposta i File:** Sposta entrambi i file `listings.csv` e `reviews.csv` che hai scaricato *dentro* questa cartella `import`.
4.  **Controlla i Nomi:** Assicurati che i nomi dei file siano *esattamente* `listings.csv` e `reviews.csv` (tutto in minuscolo e senza numeri o testo aggiuntivo).
5.  **RIAVVIA IL DATABASE (Cruciale):**
    * Torna all'app Neo4j Desktop.
    * Premi **Stop** sul tuo database.
    * Attendi che si sia fermato completely, poi premi **Start**.

Questo passaggio è fondamentale, altrimenti Neo4j non "vedrà" i file che hai appena aggiunto.

## Passaggio 2: Esegui SCRIPT #1 (Creazione Constraint)

Copia e incolla l'intero blocco di codice nel browser Neo4j ed eseguilo. Questo script usa la sintassi moderna `REQUIRE` (invece di `ASSERT`) e corregge gli errori di battitura (come `l.lising_id` in `l.listing_id`).

```cypher
CREATE CONSTRAINT constraint_host_id IF NOT EXISTS FOR (h:Host) REQUIRE h.host_id IS UNIQUE;
CREATE CONSTRAINT constraint_listing_id IF NOT EXISTS FOR (l:Listing) REQUIRE l.listing_id IS UNIQUE;
CREATE CONSTRAINT constraint_user_id IF NOT EXISTS FOR (u:User) REQUIRE u.user_id IS UNIQUE;
CREATE CONSTRAINT constraint_amenity_name IF NOT EXISTS FOR (a:Amenity) REQUIRE a.name IS UNIQUE;
CREATE CONSTRAINT constraint_city_citystate IF NOT EXISTS FOR (c:City) REQUIRE c.citystate IS UNIQUE;
CREATE CONSTRAINT constraint_state_code IF NOT EXISTS FOR (s:State) REQUIRE s.code IS UNIQUE;
CREATE CONSTRAINT constraint_country_code IF NOT EXISTS FOR (c:Country) REQUIRE c.code IS UNIQUE;
CREATE CONSTRAINT constraint_review_id IF NOT EXISTS FOR (r:Review) REQUIRE r.review_id IS UNIQUE;
```

## Passaggio 3: Esegui SCRIPT #2 (Importazione 'listings.csv')
Questo script è diviso in due parti. Eseguile una dopo l'altra.

### Parte 1: Caricamento Annunci (Listings), Posizioni e Servizi
Questo script carica i dati principali degli annunci, crea i nodi per la posizione (Quartiere, Città, Stato, Paese) e i servizi (Amenity), e crea le relazioni tra loro.

```cypher
LOAD CSV WITH HEADERS FROM "file:///listings.csv" AS row
WITH row WHERE row.id IS NOT NULL
MERGE (l:Listing {listing_id: row.id})
ON CREATE SET l.name  = row.name,
l.latitude  = toFloat(row.latitude),
l.longitude  = toFloat(row.longitude),
l.reviews_per_month  = toFloat(row.reviews_per_month),
l.cancellation_policy  = row.cancellation_policy,
l.instant_bookable  = CASE WHEN row.instant_bookable = "t" THEN true ELSE false END,
l.review_scores_value  = toInteger(row.review_scores_value),
l.review_scores_location  = toInteger(row.review_scores_location),
l.review_scores_communication = toInteger(row.review_scores_communication),
l.review_scores_checkin  = toInteger(row.review_scores_checkin),
l.review_scores_cleanliness  = toInteger(row.review_scores_cleanliness),
l.review_scores_accuracy  = toInteger(row.review_scores_accuracy),
l.review_scores_rating  = toInteger(row.review_scores_rating),
l.availability_365  = toInteger(row.availability_365),
l.availability_90  = toInteger(row.availability_90),
l.availability_60  = toInteger(row.availability_60),
l.availability_30  = toInteger(row.availability_30),
l.price  = toFloat(substring(row.price, 1)),
l.cleaning_fee  = toFloat(substring(row.cleaning_fee, 1)),
l.security_deposit  = toFloat(substring(row.security_deposit, 1)),
l.monthly_price  = toFloat(substring(row.monthly_price, 1)),
l.weekly_price  = toFloat(substring(row.weekly_price, 1)),
l.square_feet  = toInteger(row.square_feet),
l.bed_type  = row.bed_type,
l.beds  = toInteger(row.beds),
l.bedrooms  = toInteger(row.bedrooms),
l.bathrooms  = toFloat(row.bathrooms),
l.accommodates  = toInteger(row.accommodates),
l.room_type  = row.room_type,
l.property_type  = row.property_type
ON MATCH SET l.count = coalesce(l.count, 0) + 1
MERGE (n:Neighborhood {neighborhood_id: coalesce(row.neighbourhood_cleansed, "NA")})
SET n.name = row.neighbourhood
MERGE (c:City {citystate: coalesce(row.city + "-" + row.state, "NA")})
ON CREATE SET c.name = row.city
MERGE (l)-[:IN_NEIGHBORHOOD]->(n)
MERGE (n)-[:LOCATED_IN]->(c)
MERGE (s:State {code: coalesce(row.state, "NA")})
MERGE (c)-[:IN_STATE]->(s)
MERGE (country:Country {code: coalesce(row.country_code, "NA")})
SET country.name = row.country
MERGE (s)-[:IN_COUNTRY]->(country)
WITH l, row, split(replace(replace(replace(row.amenities, "{", ""), "}", ""), "\"", ""), ",") AS amenities
UNWIND amenities AS amenity
MERGE (a:Amenity {name: amenity})
MERGE (l)-[:HAS]->(a);
```
### Parte 2: Caricamento Proprietari (Hosts) e collegamento ai Listings
Questo script ricarica lo stesso file per creare i nodi Host e collegarli ai Listing che hanno appena creato.

```cypher
LOAD CSV WITH HEADERS FROM "file:///listings.csv" AS row
WITH row WHERE row.host_id IS NOT NULL
MERGE (h:Host {host_id: row.host_id})
ON CREATE SET h.name  = row.host_name,
h.about  = row.host_about,
h.verifications  = row.host_verifications,
h.listings_count  = toInteger(row.host_listings_count),
h.acceptance_rate = toFloat(row.host_acceptance_rate),
h.host_since  = row.host_since,
h.url  = row.host_url,
h.response_rate  = row.host_response_rate,
h.superhost  = CASE WHEN row.host_is_super_host = "t" THEN true ELSE false END,
h.location  = row.host_location,
h.verified  = CASE WHEN row.host_identity_verified = "t" THEN true ELSE false END,
h.image  = row.host_picture_url
ON MATCH SET h.count = coalesce(h.count, 0) + 1
MERGE (l:Listing {listing_id: row.id})
MERGE (h)-[:HOSTS]->(l);
```

## Passaggio 4: Esegui SCRIPT #3 (Importazione 'reviews.csv')
Questo è il file di grandi dimensioni. Lo script è stato aggiornato per usare CALL { ... } IN TRANSACTIONS, che è la sintassi moderna per gestire importazioni pesanti (sostituisce il vecchio USING PERIODIC COMMIT). Potrebbe richiedere diversi minuti per essere completato.

```cypher
LOAD CSV WITH HEADERS FROM "file:///reviews.csv" AS row
CALL {
    WITH row
    MERGE (u:User {user_id: row.reviewer_id})
    SET u.name = row.reviewer_name
    MERGE (r:Review {review_id: row.id})
    ON CREATE SET r.date  = row.date,
                  r.comments = row.comments
    WITH row, u, r
    MATCH (l:Listing {listing_id: row.listing_id})
    MERGE (u)-[:WROTE]->(r)
    MERGE (r)-[:REVIEWS]->(l)
} IN TRANSACTIONS OF 1000 ROWS
```
## Passaggio 5: Verifica l'importazione
Dopo che tutti gli script sono stati eseguiti, puoi eseguire questa query per vedere un campione dei tuoi dati e verificare che i nodi e le relazioni siano stati creati correttamente.

```cypher
MATCH (n:Listing)-[r]-(m) 
RETURN n,r,m 
LIMIT 50
```
