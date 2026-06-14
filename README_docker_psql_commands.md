# Docker Compose — βασικές εντολές
docker info
docker volume ls
docker ps -a
docker version
docker build backend -t test-backend --progress=plain
docker  compose down -v
postgres → flyway → backend → frontend είναι η σωστή σειρά των services

wsl --shutdown
docker images
docker image ls
docker image rm flyway/flyway

Next.js routing = file‑based routing για σελίδες και API routes  
API Gateway = reverse proxy + traffic control + auth + rate limiting routing_services — δρομολογεί traffic σε πολλά microservices

auth_gateway — JWT, OAuth2, API keys
rate_limiting_gateway — προστασία από overload
caching_gateway — caching responses
monitoring_gateway — logs, metrics, tracing
versioning_gateway — /v1, /v2 routing
load_balancing_gateway — μοιράζει requests σε πολλά instances

Στο Docker Compose σου, το Flyway:
Ξεκινάει ως container
Συνδέεται στη βάση
Τρέχει όλα τα migrations
Τερματίζει

Άρα:
Τρέχει μόνο όταν σηκώνεις το stack (docker compose up)  
Τρέχει μόνο όταν αλλάξεις SQL migration και ξανασηκώσεις Flyway

Ναι, μπορείς να τρέχεις SQL queries χωρίς Flyway.
Όμως το Flyway σου δίνει πράγματα που το “σκέτο Docker” δεν μπορεί να κάνει.

Το μεγάλο πρόβλημα χωρίς Flyway
Αν αλλάξεις το schema σου (π.χ. προσθέσεις column):

το Postgres δεν ξανατρέχει τα init scripts
δεν ξέρεις ποια version της βάσης έχει ο καθένας
δεν υπάρχει ιστορικό αλλαγών
δεν υπάρχει έλεγχος consistency
δεν μπορείς να κάνεις rollback
δεν μπορείς να κάνεις CI/CD migrations
δεν μπορείς να κάνεις production deployments με ασφάλεια

Με λίγα λόγια:
Χωρίς Flyway → δεν έχεις database lifecycle management.

Τα init scripts του Docker τρέχουν ΜΟΝΟ την πρώτη φορά
Το Postgres στο Docker τρέχει SQL scripts μόνο όταν δημιουργείται το volume.
Μετά:
αλλάζεις schema
αλλάζεις columns
αλλάζεις constraints
Το Docker δεν τα ξανατρέχει ποτέ.
Άρα η βάση μένει “κολλημένη” στην παλιά μορφή.

- **Εκκίνηση containers**  
  ```
  docker compose up --build
  ```

- **Σταμάτημα containers**  
  ```
  docker compose down
  ```

- **Σταμάτημα + διαγραφή volumes (reset DB)**  
  ```
  docker compose down -v
  ```

---

# 2. Docker Volumes — διαχείριση

- **Λίστα όλων των volumes**  
  ```
  docker volume ls
  ```

- **Διαγραφή συγκεκριμένου volume**  
  ```
  docker volume rm <volume-name>
  ```

- **Διαγραφή όλων των unused volumes**  
  ```
  docker volume prune
  ```

---

# 3. Postgres μέσα στο Docker

- **Μπες στο Postgres container**  
  ```
  docker exec -it genai-postgres psql -U genai -d genai
  ```

---

# 4. psql — βασικές εντολές

- **Λίστα πινάκων**  
  ```
  \dt
  ```

- **Λίστα schemas**  
  ```
  \dn
  ```

- **Εκτέλεση SQL query**  
  ```
  SELECT * FROM app_user;
  ```

- **Ακύρωση εντολής που δεν έκλεισε**  
  ```
  Ctrl + C
  ```

- **Έξοδος από psql**  
  ```
  \q
  ```

---

# 5. Reset της βάσης (ώστε να ξανατρέξουν migrations)

- **Σβήσε containers + volumes**  
  ```
  docker compose down -v
  ```

- **Ξανασήκωσε το project**  
  ```
  docker compose up --build
  ```

Αυτό θα εκτελέσει:

- `V1__initial_schema.sql`  
- `V2__seed_demo_data.sql`  

και θα δημιουργήσει ξανά όλους τους πίνακες + demo data.

---

# 6. Έλεγχος ότι τα migrations έτρεξαν

- **Δες αν υπάρχουν οι πίνακες**  
  ```
  \dt
  ```

- **Δες αν μπήκαν τα seed data**  
  ```
  SELECT * FROM account;
  SELECT * FROM app_user;
  SELECT * FROM agent;
  SELECT * FROM document;
  ```

---

# V3__vector_indexes.sql** για pgvector

Η **pgvector** χρειάζεται indexes για έναν βασικό λόγο:
> **Χωρίς index, κάθε similarity search κάνει full table scan — άρα είναι αργό.**
Αν έχεις 100 rows, δεν πειράζει.  
Αν έχεις 10.000, 100.000 ή 1.000.000 embeddings → το σύστημα θα σέρνεται.
Γι’ αυτό οι indexes είναι *απαραίτητοι*.

---

# Τι κάνει ο pgvector index

Ο pgvector index:
- επιταχύνει similarity search (`<->`, cosine, L2, inner product)
- μειώνει latency από **δευτερόλεπτα → χιλιοστά του δευτερολέπτου**
- επιτρέπει ANN (Approximate Nearest Neighbor)
- αποτρέπει full table scans

Χωρίς index, κάθε query:

```sql
SELECT * FROM document ORDER BY embedding <-> $1 LIMIT 5;
```

θα διαβάζει **όλο τον πίνακα**.

---

# Ποιοι indexes υπάρχουν στον pgvector

Υπάρχουν δύο τύποι:

## ivfflat index (ο πιο γρήγορος — ANN)
Χρησιμοποιείται για production.

```sql
CREATE INDEX idx_document_embedding
ON document
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

- χρειάζεται **VECTOR(n)** με dimension  
- χρειάζεται **ANALYZE** μετά το build  
- είναι approximate (όχι 100% exact)

---

## 2HNSW index (νέος, πιο ακριβής)
Αν έχεις pgvector ≥ 0.7:

```sql
CREATE INDEX idx_document_embedding_hnsw
ON document
USING hnsw (embedding vector_cosine_ops);
```

- πιο ακριβής από ivfflat  
- πιο αργός στο build  
- πιο γρήγορος στο search  

---

## 3**Brute-force (χωρίς index) 
Δεν χρειάζεται index, αλλά:

- είναι **πολύ αργό**
- δεν χρησιμοποιείται σε production

---

# Πότε χρειάζεσαι index

Χρειάζεσαι index όταν:
- κάνεις semantic search  
- κάνεις document retrieval  
- κάνεις RAG  
- κάνεις agent memory  
- κάνεις embeddings για chat history  
- έχεις πάνω από 1.000 rows
Αν έχεις 10.000+ rows → ο index είναι **υποχρεωτικός**.

---

# Τι πρέπει να κάνεις στο schema σου

Στον πίνακα:

```sql
embedding VECTOR
```

πρέπει να βάλεις dimension:

```sql
embedding VECTOR(1536)
```

Μετά φτιάχνεις index:

```sql
CREATE INDEX idx_document_embedding
ON document
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

Και τέλος:

```sql
ANALYZE document;
```

---

# V3__vector_indexes.sql έτοιμο για migrations

```
V3__vector_indexes.sql
```

και το βάζεις στο:

```
genai-db/migrations/
```

---

# V3__vector_indexes.sql (έτοιμο για χρήση)

```sql
-- ============================================
-- V3: Vector indexes for semantic search
-- ============================================

-- Ensure pgvector extension exists
CREATE EXTENSION IF NOT EXISTS vector;

-- IMPORTANT:
-- Your document table must define embedding as VECTOR(n)
-- Example: embedding VECTOR(1536)

-- Analyze table before building ivfflat index
ANALYZE document;

-- Create ivfflat index for cosine similarity
CREATE INDEX IF NOT EXISTS idx_document_embedding_ivfflat
ON document
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Re-analyze after index creation
ANALYZE document;
```

---

# Τι κάνει αυτό το migration

### Ενεργοποιεί το extension (αν δεν υπάρχει)
```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### Κάνει ANALYZE πριν το index  
Αυτό είναι **απαραίτητο** για ivfflat.

### Δημιουργεί ivfflat index για cosine similarity  
Ο πιο γρήγορος index για semantic search:

```sql
CREATE INDEX idx_document_embedding_ivfflat
ON document
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### Κάνει ANALYZE ξανά  
Για να βελτιστοποιηθεί το query planner.

---

# ΠΡΟΣΟΧΗ — πρέπει να έχεις dimension στο schema

Στο schema σου τώρα έχεις:

```sql
embedding VECTOR
```

Πρέπει να γίνει:

```sql
embedding VECTOR(1536)
```

ή όποιο dimension χρησιμοποιεί το embeddings model σου.

Αν δεν το βάλεις, το ivfflat index **ΔΕΝ θα δουλέψει**.

---

# Θες και HNSW index (πιο ακριβής, πιο σύγχρονος);

Μπορώ να σου φτιάξω και:

```
V4__vector_hnsw_index.sql
```

με:

```sql
CREATE INDEX idx_document_embedding_hnsw
ON document
USING hnsw (embedding vector_cosine_ops);
```

**έτοιμο migration file** για HNSW index (τον πιο σύγχρονο και ακριβή index του pgvector)  
Το ονομάζεις:

```
V4__vector_hnsw_index.sql
```

και το βάζεις στον φάκελο migrations.

---

# V4__vector_hnsw_index.sql (production‑ready)

```sql
-- ============================================
-- V4: HNSW index for high-accuracy vector search
-- ============================================

-- Ensure pgvector extension exists
CREATE EXTENSION IF NOT EXISTS vector;

-- IMPORTANT:
-- embedding must be VECTOR(n), e.g. VECTOR(1536)

-- Analyze table before index creation
ANALYZE document;

-- Create HNSW index for cosine similarity
CREATE INDEX IF NOT EXISTS idx_document_embedding_hnsw
ON document
USING hnsw (embedding vector_cosine_ops)
WITH (
    m = 16,          -- graph degree (default 16)
    ef_construction = 200  -- accuracy during build
);

-- Re-analyze after index creation
ANALYZE document;
```

---

# Τι κάνει αυτό το migration

### **1. Χτίζει HNSW index**
Ο πιο σύγχρονος index του pgvector:
- πιο ακριβής από ivfflat  
- πιο γρήγορος στο search  
- ιδανικός για RAG, agents, semantic search  

### **2. Χρησιμοποιεί cosine similarity**
Ταιριάζει με OpenAI embeddings.

### **3. Έχει tuning options**
- `m = 16` → πόσο “πυκνό” είναι το γράφημα  
- `ef_construction = 200` → πόσο ακριβής είναι η κατασκευή  

### **4. Κάνει ANALYZE πριν και μετά**
Αυτό βοηθάει τον Postgres query planner.

---

# Σημαντικό: πρέπει να έχεις dimension στο schema

Αν το schema σου έχει:

```sql
embedding VECTOR
```

πρέπει να γίνει:

```sql
embedding VECTOR(1536)
```

ή όποιο dimension έχει το embeddings model σου.

Χωρίς dimension → HNSW **δεν δουλεύει**.

---

- **V5__updated_at_trigger.sql** (ώστε updated_at να ενημερώνεται αυτόματα)  
- **V6__full_text_search_indexes.sql** (για hybrid RAG)  
- **V7__agent_memory_indexes.sql** (για fast chat history retrieval) 