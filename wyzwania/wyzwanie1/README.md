# Wyzwanie 1: Lokalne uruchomienie z SQLite + FAISS

**Cel:** Zmienić architekturę aplikacji z Google BigQuery na lokalną bazę danych SQLite + FAISS, umożliwiając uruchomienie całego stacka (EmbeddingGemma, Bielik, API) lokalnie bez zależności od chmury.

## 📋 Opis wyzwania

Obecna aplikacja korzysta z:
- **BigQuery Vector Search** - jako wektorowa baza wiedzy
- **Cloud Run** - do hostowania modeli
- **Google Cloud** - jako platforma

Twoim zadaniem jest **usunięcie zależności od Google BigQuery** i zastąpienie go:
- **SQLite** - do przechowywania tekstów dokumentów
- **FAISS** (Facebook AI Similarity Search) - do wektorowego wyszukiwania podobieństw

## 🎯 Wymagania

1. Aplikacja powinna działać **lokalnie** (bez Cloud Run)
2. Modele EmbeddingGemma i Bielik powinny być uruchamiane **lokalnie** (np. przez Ollama)
3. Zamiast BigQuery użyj **SQLite + FAISS**
4. Zachowaj istniejące API (`/ask`, `/ask_direct`, `/ingest`)
5. Czas implementacji: **max 1 godzina**

---

## 🚀 Kroki implementacji

### Krok 1: Zainstaluj Ollama lokalnie

**Podpowiedź:** 
- Ollama to narzędzie do uruchamiania modeli LLM lokalnie
- Dokumentacja: [https://ollama.com/](https://ollama.com/)
- Instalacja na Linux: `curl -fsSL https://ollama.com/install.sh | sh`
- Instalacja na Windows/macOS: zobacz dokumentację

**Do zrobienia:**
```bash
# Zainstaluj Ollama
# Uruchom model Bielik: ollama pull SpeakLeash/bielik-4.5b-v3.0-instruct:Q8_0
# Uruchom model EmbeddingGemma: ollama pull embeddinggemma:latest
```

**Na co zwrócić uwagę:**
- Sprawdź czy masz wystarczająco miejsca na dysku (modele zajmują ~6-8GB: EmbeddingGemma ~1.5-2GB, Bielik ~4-5GB)
- Upewnij się, że masz wystarczająco RAM (minimum 16GB rekomendowane)

---

### Krok 2: Uruchom modele lokalnie

**Podpowiedź:**
- Ollama domyślnie uruchamia serwer na `localhost:11434`
- Możesz uruchomić Ollama w trybie serwera: `ollama serve`
- Sprawdź dokumentację API: [Ollama API Docs](https://github.com/ollama/ollama/blob/main/docs/api.md)

**Do zrobienia:**
```bash
# Uruchom serwer Ollama
ollama serve

# W nowym terminalu sprawdź dostępne modele
curl http://localhost:11434/api/tags
```

**Na co zwrócić uwagę:**
- Upewnij się, że port 11434 jest dostępny
- Sprawdź, czy modele są poprawnie załadowane

---

### Krok 3: Zainstaluj FAISS i SQLite

**Podpowiedź:**
- FAISS to biblioteka do wektorowego wyszukiwania podobieństw
- Dokumentacja: [https://github.com/facebookresearch/faiss](https://github.com/facebookresearch/faiss)
- SQLite to wbudowana baza danych w Pythonie (nie wymaga instalacji)

**Do zrobienia:**
```bash
pip install faiss-cpu  # lub faiss-gpu jeśli masz CUDA
```

**Na co zwrócić uwagę:**
- `faiss-cpu` działa na każdym systemie
- `faiss-gpu` wymaga CUDA i jest szybszy
- Sprawdź dokumentację FAISS dla Python API

---

### Krok 4: Stwórz strukturę bazy SQLite + FAISS

**Podpowiedź:**
- Potrzebujesz dwóch struktur:
  1. **SQLite** - do przechowywania tekstów i metadanych
  2. **FAISS Index** - do przechowywania wektorów i szybkiego wyszukiwania

**Do zrobienia:**

1. **SQLite** - Tabela do przechowywania dokumentów:
```python
import sqlite3

conn = sqlite3.connect('rag_database.db')
cursor = conn.cursor()

# Stwórz tabelę dla dokumentów
cursor.execute('''
CREATE TABLE IF NOT EXISTS documents (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    metadata TEXT
)
''')
conn.commit()
```

2. **FAISS** - Indeks wektorowy:
```python
import faiss
import numpy as np

# Dimensja wektora dla EmbeddingGemma to 768
# Dokumentacja: https://deepmind.google/models/gemma/embeddinggemma/
dimension = 768

# Stwórz indeks FAISS
index = faiss.IndexFlatIP(dimension)  # Inner Product (cosine similarity)

# Zapisz indeks do pliku
faiss.write_index(index, 'faiss_index.bin')
```

**Na co zwrócić uwagę:**
- Sprawdź jaka jest dimensja wektorów dla EmbeddingGemma (domyślnie 768)
- `IndexFlatIP` używa iloczynu skalarnego (odpowiada cosine similarity po normalizacji)
- Pamiętaj o normalizacji wektorów jeśli używasz cosine similarity

---

### Krok 5: Zmodyfikuj `/ingest` endpoint

**Podpowiedź:**
- Obecny endpoint `/ingest` zapisuje do BigQuery
- Musisz zmienić go, aby zapisywał do SQLite + FAISS
- Sprawdź oryginalny kod w `orchestration/main.py`

**Do zrobienia:**

1. Zmień funkcję `ingest_csv` aby:
   - Czytała plik CSV
   - Generowała embeddingi dla każdego dokumentu
   - Zapisywała teksty do SQLite
   - Dodawała wektory do indeksu FAISS

2. **Przykładowa struktura:**
```python
@app.post("/ingest")
async def ingest_csv(file: UploadFile = File(...)):
    content = await file.read()
    csv_reader = csv.DictReader(io.StringIO(content.decode("utf-8")))
    
    # 1. Generuj embeddingi
    embeddings = []
    docs = []
    for row in csv_reader:
        doc_id = row.get("id")
        text = row.get("text")
        embedding = get_embedding(text)  # Użyj lokalnego Ollama
        embeddings.append(embedding)
        docs.append({"id": doc_id, "content": text})
    
    # 2. Zapisz do SQLite
    # ...
    
    # 3. Dodaj do FAISS
    embeddings_array = np.array(embeddings, dtype='float32')
    index.add(embeddings_array)
    
    return {"status": "success", "inserted_count": len(docs)}
```

**Na co zwrócić uwagę:**
- Pamiętaj o konwersji listy wektorów na numpy array
- FAISS oczekuje wektorów jako `float32`
- Zapisz zmodyfikowany indeks FAISS z powrotem do pliku

---

### Krok 6: Zmodyfikuj `/ask` endpoint

**Podpowiedź:**
- Obecny endpoint `/ask` wyszukuje w BigQuery
- Musisz zmienić go, aby wyszukiwał w FAISS
- Sprawdź oryginalny kod w `orchestration/main.py`

**Do zrobienia:**

1. Zmień funkcję `ask_question` aby:
   - Generowała embedding dla zapytania
   - Wyszukiwała podobne wektory w FAISS
   - Pobierała odpowiadające teksty z SQLite
   - Tworzyła kontekst i wysyłała do modelu Bielik

2. **Przykładowa struktura:**
```python
@app.post("/ask")
async def ask_question(request_data: AskRequest):
    query = request_data.query
    
    # 1. Generuj embedding dla zapytania
    query_embedding = get_embedding(query)
    query_vector = np.array([query_embedding], dtype='float32')
    
    # 2. Wyszukaj w FAISS
    k = 3  # Liczba wyników
    distances, indices = index.search(query_vector, k)
    
    # 3. Pobierz dokumenty z SQLite
    context_docs = []
    for idx in indices[0]:
        # Pobierz dokument o danym ID z SQLite
        # ...
        context_docs.append(doc_content)
    
    # 4. Stwórz prompt i wyślij do Bielik
    # ... (tak jak w oryginale)
    
    return {"answer": answer, "context_used": context_docs}
```

**Na co zwrócić uwagę:**
- FAISS `search()` zwraca odległości i indeksy
- Indeksy odnoszą się do pozycji w indeksie FAISS (musisz mapować je na ID dokumentów)
- Pamiętaj o normalizacji wektorów jeśli używasz cosine similarity

---

### Krok 7: Zmodyfikuj `get_embedding()` i połączenie z LLM

**Podpowiedź:**
- Obecnie `get_embedding()` używa Cloud Run
- Musisz zmienić go, aby używał lokalnego Ollama
- Podobnie dla połączenia z modelem Bielik

**Do zrobienia:**

1. **EmbeddingGemma lokalnie:**
```python
def get_embedding(text: str) -> list[float]:
    url = "http://localhost:11434/api/embed"
    payload = {
        "model": "embeddinggemma",
        "input": text
    }
    response = requests.post(url, json=payload)
    response.raise_for_status()
    return response.json().get("embeddings", [[]])[0]
```

2. **Bielik lokalnie:**
```python
def call_llm(prompt: str) -> str:
    url = "http://localhost:11434/api/chat"
    payload = {
        "model": "SpeakLeash/bielik-4.5b-v3.0-instruct:Q8_0",
        "messages": [{"role": "user", "content": prompt}],
        "stream": False
    }
    response = requests.post(url, json=payload)
    response.raise_for_status()
    return response.json().get("message", {}).get("content", "")
```

**Na co zwrócić uwagę:**
- Sprawdź, czy Ollama jest uruchomiony (`ollama serve`)
- Upewnij się, że modele są załadowane
- Port domyślny to 11434

---

### Krok 8: Uruchom aplikację lokalnie

**Podpowiedź:**
- Użyj Uvicorn do uruchomienia FastAPI
- Sprawdź dokumentację FastAPI

**Do zrobienia:**
```bash
# Uruchom aplikację
uvicorn main:app --reload --port 8000

# Testuj endpointy
curl -X POST "http://localhost:8000/ask" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?"}'
```

**Na co zwrócić uwagę:**
- Upewnij się, że wszystkie zależności są zainstalowane
- Sprawdź, czy port 8000 jest dostępny
- Użyj `--reload` podczas rozwoju

---

## 📦 Pliki do stworzenia/zmodyfikowania

| Plik | Opis | Akcja |
|------|------|-------|
| `requirements.txt` | Zależności Python | **Stwórz** |
| `.env.example` | Przykładowe zmienne środowiskowe | **Stwórz** |
| `main.py` | Główna aplikacja FastAPI | **Zmodyfikuj** |

---

## 🔍 Testowanie

**Test 1: Ingest dokumentów**
```bash
curl -X POST "http://localhost:8000/ingest" \
     -F "file=@hotel_rules.csv"
```

**Oczekiwany wynik:**
- Dokumenty powinny zostać zapisane do SQLite
- Indeks FAISS powinien zostać zaktualizowany
- Powinien zwrócić `{"status": "success", "inserted_count": N}`

**Test 2: Zadaj pytanie**
```bash
curl -X POST "http://localhost:8000/ask" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?"}'
```

**Oczekiwany wynik:**
- Powinien zwrócić odpowiedź z modelu Bielik
- Powinien zawierać `context_used` z fragmentami dokumentów

**Test 3: Porównanie z `/ask_direct`**
```bash
curl -X POST "http://localhost:8000/ask_direct" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?"}'
```

**Oczekiwany wynik:**
- Powinien zwrócić odpowiedź bez kontekstu RAG

---

## 💡 Podpowiedzi i rozwiązania typowych problemów

### Problem 1: FAISS nie znajduje wyników
**Przyczyna:** Wektory nie są znormalizowane
**Rozwiązanie:** Znormalizuj wektory przed dodaniem do indeksu:
```python
faiss.normalize_L2(embeddings_array)
```

### Problem 2: Ollama nie odpowiada
**Przyczyna:** Serwer Ollama nie jest uruchomiony
**Rozwiązanie:** Uruchom `ollama serve` w oddzielnym terminalu

### Problem 3: Błąd "model not found"
**Przyczyna:** Model nie jest załadowany
**Rozwiązanie:** `ollama pull SpeakLeash/bielik-4.5b-v3.0-instruct:Q8_0`

### Problem 4: Wolne wyszukiwanie w FAISS
**Przyczyna:** Używasz IndexFlatIP z dużą bazą
**Rozwiązanie:** Rozważ użycie `IndexIVFFlat` dla większych zbiorów

### Problem 5: Błąd konwersji typów w FAISS
**Przyczyna:** Wektory nie są typu float32
**Rozwiązanie:** `embeddings_array = np.array(embeddings, dtype='float32')`

---

## 📚 Dokumentacja do przestudiowania

1. [FAISS Python API](https://github.com/facebookresearch/faiss/wiki/Faiss-for-the-impatient) - Oficjalna dokumentacja FAISS z przykładami w Pythonie
2. [Ollama API Documentation](https://github.com/ollama/ollama/blob/main/docs/api.md) - Pełna dokumentacja API Ollama (embed, chat, generate)
3. [SQLite Python Tutorial](https://docs.python.org/3/library/sqlite3.html) - Oficjalna dokumentacja modułu sqlite3 w Pythonie
4. [FastAPI Documentation](https://fastapi.tiangolo.com/) - Oficjalna dokumentacja frameworka FastAPI
5. [NumPy Documentation](https://numpy.org/doc/stable/) - Dokumentacja biblioteki NumPy (wymagana przez FAISS)
6. [Uvicorn Documentation](https://www.uvicorn.org/) - Dokumentacja serwera ASGI Uvicorn
7. [Requests Documentation](https://docs.python-requests.org/) - Dokumentacja biblioteki requests do obsługi HTTP

---

## ✅ Kryteria akceptacji

- [ ] Aplikacja uruchamia się lokalnie
- [ ] Modele EmbeddingGemma i Bielik działają lokalnie przez Ollama
- [ ] Endpoint `/ingest` zapisuje dokumenty do SQLite + FAISS
- [ ] Endpoint `/ask` wyszukuje w FAISS i zwraca odpowiedź z kontekstem
- [ ] Endpoint `/ask_direct` działa bez RAG
- [ ] Czas odpowiedzi < 5 sekund dla prostych zapytań
- [ ] Kod jest czytelny i skomentowany

---

## 🎉 Bonus (opcjonalne)

1. Dodaj endpoint `/reset` do czyszczenia bazy i indeksu
2. Zaimplementuj automatyczne ładowanie indeksu FAISS przy starcie
3. Dodaj obsługę wielu kolekcji dokumentów
4. Zaimplementuj cache dla embeddingów
