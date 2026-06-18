# Wyzwanie 3: Parametry wyszukiwania (k, czas, źródło, pewność)

**Cel:** Rozszerzyć endpoint `/ask` o dodatkowe parametry, które pozwolą na lepszą kontrolę nad procesem RAG: liczba wyników (k), czas wyszukiwania, informacja o źródle i procent pewności odpowiedzi.

## 📋 Opis wyzwania

Obecny endpoint `/ask` ma ograniczone możliwości konfiguracji. Twoim zadaniem jest **dodanie nowych parametrów**, które pozwolą na:

1. **k** - Kontrolę liczby pobieranych dokumentów z bazy wektorowej
2. **Czas wyszukiwania** - Pomiar i zwracanie czasu wykonania poszczególnych operacji
3. **Źródło** - Informacja skąd pochodzi każdy fragment kontekstu
4. **Pewność** - Oszacowanie pewności, że model czerpał z odpowiednich źródeł

## 🎯 Wymagania

1. Rozszerzyć `AskRequest` o nowy parametr `k` (liczba dokumentów do pobrania)
2. Dodać pomiar czasu dla:
   - Generowania embeddingu zapytania
   - Wyszukiwania w bazie wektorowej
   - Generowania odpowiedzi przez LLM
3. Zwracać informację o źródle każdego dokumentu w kontekście
4. Oszacować i zwrócić procent pewności, że odpowiedź jest poprawna
5. Czas implementacji: **max 1 godzina**

---

## 🚀 Kroki implementacji

### Krok 1: Zmodyfikuj model `AskRequest`

**Podpowiedź:**
- Obecny model `AskRequest` jest zdefiniowany w `orchestration/main.py`
- Używa `pydantic.BaseModel`
- Musisz dodać nowy parametr `k` z domyślną wartością

**Do zrobienia:**

Zmień:
```python
class AskRequest(BaseModel):
    query: str
```

na:
```python
class AskRequest(BaseModel):
    query: str
    k: int = 3  # Domyślnie 3 dokumenty
```

**Na co zwrócić uwagę:**
- Użyj domyślnej wartości `3` aby zachować kompatybilność wstecz
- `k` powinien być typem `int`
- Możesz dodać walidację (np. `k` między 1 a 10)

---

### Krok 2: Zmodyfikuj funkcję `ask_question` - pomiar czasu

**Podpowiedź:**
- Użyj `time.time()` lub `time.perf_counter()` do pomiaru czasu
- Dokumentacja: [Python time module](https://docs.python.org/3/library/time.html)
- Pomiary powinny być dla poszczególnych etapów

**Do zrobienia:**

Dodaj pomiary czasu w funkcji `ask_question`:
```python
import time

@app.post("/ask")
async def ask_question(request_data: AskRequest):
    query = request_data.query
    k = request_data.k
    
    # Inicjalizacja timerów
    timings = {}
    start_total = time.perf_counter()
    
    # 1. Generowanie embeddingu
    start_embedding = time.perf_counter()
    try:
        query_embedding = get_embedding(query)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Błąd generowania wektora zapytania: {e}")
    timings["embedding_time"] = time.perf_counter() - start_embedding
    
    # 2. Wyszukiwanie w bazie wektorowej
    start_search = time.perf_counter()
    # ... (kod wyszukiwania)
    timings["search_time"] = time.perf_counter() - start_search
    
    # 3. Generowanie odpowiedzi przez LLM
    start_llm = time.perf_counter()
    # ... (kod wywoływania LLM)
    timings["llm_time"] = time.perf_counter() - start_llm
    
    timings["total_time"] = time.perf_counter() - start_total
```

**Na co zwrócić uwagę:**
- `time.perf_counter()` jest bardziej precyzyjny niż `time.time()`
- Pomiary powinny być w sekundach
- Zapisz czasy w słowniku, aby zwrócić je w odpowiedzi

---

### Krok 3: Zmodyfikuj wyszukiwanie - użyj parametru `k`

**Podpowiedź:**
- Obecnie wyszukiwanie używa `top_k => 3` (w BigQuery) lub `k=3` (w FAISS)
- Musisz użyć wartości `k` z requestu

**Do zrobienia (dla BigQuery):**

Zmień:
```python
bq_query = f"""
SELECT base.content, distance
FROM VECTOR_SEARCH(
  TABLE `{table_ref}`,
  'embedding',
  (SELECT {query_embedding} as embedding),
  top_k => 3,
  distance_type => 'COSINE'
)
"""
```

na:
```python
bq_query = f"""
SELECT base.content, distance
FROM VECTOR_SEARCH(
  TABLE `{table_ref}`,
  'embedding',
  (SELECT {query_embedding} as embedding),
  top_k => {k},
  distance_type => 'COSINE'
)
"""
```

**Do zrobienia (dla FAISS):**

Zmień:
```python
k = 3  # Liczba wyników
distances, indices = index.search(query_vector, k)
```

na:
```python
distances, indices = index.search(query_vector, k)
```

**Na co zwrócić uwagę:**
- Upewnij się, że `k` jest używany we wszystkich miejscach
- Waliduj, że `k` nie jest zbyt duży (np. > 100)

---

### Krok 4: Dodaj informację o źródle dokumentów

**Podpowiedź:**
- Obecnie zwracany jest tylko tekst dokumentów w `context_used`
- Musisz dodać informację o źródle (np. ID dokumentu, nazwa pliku)
- Możesz zmienić strukturę zwracanych danych

**Do zrobienia:**

Zmień strukturę zwracanych dokumentów:
```python
# Zamiast:
context_docs = [row.content for row in results]

# Użyj:
context_docs = []
for row in results:
    context_docs.append({
        "content": row.content,
        "source": row.id if hasattr(row, 'id') else "unknown",
        "distance": row.distance if hasattr(row, 'distance') else 0.0
    })
```

**Dla FAISS + SQLite:**
```python
context_docs = []
for idx in indices[0]:
    # Pobierz dokument z SQLite
    cursor.execute("SELECT id, content FROM documents WHERE rowid = ?", (idx + 1,))
    doc = cursor.fetchone()
    if doc:
        context_docs.append({
            "content": doc[1],
            "source": doc[0],
            "distance": distances[0][idx]
        })
```

**Na co zwrócić uwagę:**
- `source` powinien identyfikować dokument (ID, nazwa pliku itp.)
- `distance` to odległość wektorowa (im mniejsza, tym lepsze dopasowanie)
- Możesz dodać więcej metadanych (np. typ dokumentu, data dodania)

---

### Krok 5: Oszacuj pewność odpowiedzi

**Podpowiedź:**
- Pewność można oszacować na podstawie:
  - Odległości wektorowych (im mniejsze, tym większa pewność)
  - Podobieństwa między zapytaniem a dokumentami
  - Długości i jakości znalezionych dokumentów
- Możesz użyć prostej formuły lub modelu

**Do zrobienia:**

Dodaj funkcję do oszacowania pewności:
```python
def calculate_confidence(distances: list[float], documents: list[dict]) -> float:
    """
    Oszacuj pewność odpowiedzi na podstawie odległości wektorowych.
    
    Args:
        distances: Lista odległości (im mniejsza, tym lepsze dopasowanie)
        documents: Lista dokumentów
        
    Returns:
        float: Pewność w procentach (0-100)
    """
    if not distances or not documents:
        return 0.0
    
    # Normalizuj odległości do zakresu 0-1
    # Im mniejsza odległość, tym większa pewność
    max_distance = 2.0  # Maksymalna odległość cosine (0-2)
    
    # Średnia odległość
    avg_distance = sum(distances) / len(distances)
    
    # Pewność = (max_distance - avg_distance) / max_distance * 100
    confidence = max(0, min(100, ((max_distance - avg_distance) / max_distance) * 100))
    
    return round(confidence, 2)
```

**Na co zwrócić uwagę:**
- Dla cosine similarity, odległość jest w zakresie -1 do 1 (im wyższa, tym lepsze dopasowanie)
- Dla cosine distance, odległość jest w zakresie 0 do 2 (im niższa, tym lepsze dopasowanie)
- Możesz dostosować formułę w zależności od używanego indeksu

---

### Krok 6: Zmodyfikuj zwracaną odpowiedź

**Podpowiedź:**
- Obecnie zwracane są: `answer` i `context_used`
- Musisz dodać: `timings`, `sources`, `confidence`

**Do zrobienia:**

Zmień:
```python
return {
    "answer": answer,
    "context_used": context_docs
}
```

na:
```python
return {
    "answer": answer,
    "context_used": [doc["content"] for doc in context_docs],  # Zachowaj kompatybilność
    "context_details": context_docs,  # Pełne informacje o kontekście
    "timings": timings,
    "confidence": calculate_confidence(distances, context_docs),
    "k": k
}
```

**Na co zwrócić uwagę:**
- Zachowaj `context_used` dla kompatybilności wstecz
- Dodaj `context_details` z pełnymi informacjami
- `timings` powinien zawierać czasy dla każdego etapu
- `confidence` powinien być w procentach (0-100)

---

### Krok 7: Zaktualizuj UI (opcjonalne)

**Podpowiedź:**
- UI w `index.html` wyświetla `context_used`
- Możesz zaktualizować UI, aby wyświetlał nowe informacje

**Do zrobienia:**

Zmień w JavaScript:
```javascript
// Handle RAG response
ragPromise.then(res => {
    if(res.error) {
        ragContent.innerHTML = `<span style="color:#ff3b30;">${res.error}</span>`;
    } else {
        ragContent.innerText = res.answer;
        
        // Wyświetl nowe informacje
        if(res.context_details && res.context_details.length > 0) {
            ragContext.style.display = 'block';
            ragContext.innerHTML = `<h4>📋 Źródła informacji (pewność: ${res.confidence}%):</h4>`;
            res.context_details.forEach((ctx, index) => {
                ragContext.innerHTML += `
                    <div class="context-item">
                        <strong>Źródło ${index + 1} (ID: ${ctx.source}, odległość: ${ctx.distance.toFixed(4)}):</strong>
                        <p>${ctx.content}</p>
                    </div>
                `;
            });
        }
        
        // Wyświetl czasy
        if(res.timings) {
            console.log("Czasy wykonania:", res.timings);
            // Możesz dodać sekcję do wyświetlania czasów
        }
    }
});
```

**Na co zwrócić uwagę:**
- UI powinien wyświetlać nową pewność
- Możesz dodać wizualizację czasów (np. pasek postępu)
- Zachowaj kompatybilność z istniejącym UI

---

## 📦 Pliki do zmodyfikowania

| Plik | Opis | Akcja |
|------|------|-------|
| `main.py` | Główna aplikacja FastAPI | **Zmodyfikuj** |
| `index.html` | UI (opcjonalnie) | **Zmodyfikuj** |

---

## 🔍 Testowanie

**Test 1: Wywołanie z domyślnym k**
```bash
curl -X POST "http://localhost:8000/ask" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?"}'
```

**Oczekiwany wynik:**
- Powinien zwrócić odpowiedź z `k=3` (domyślnie)
- Powinien zawierać `timings`, `confidence`, `context_details`

**Test 2: Wywołanie z k=5**
```bash
curl -X POST "http://localhost:8000/ask" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?", "k": 5}'
```

**Oczekiwany wynik:**
- Powinien zwrócić 5 dokumentów w kontekście
- Powinien zawierać zaktualizowane `timings` i `confidence`

**Test 3: Wywołanie z k=1**
```bash
curl -X POST "http://localhost:8000/ask" \
     -H "Content-Type: application/json" \
     -d '{"query": "Ile kosztuje parking?", "k": 1}'
```

**Oczekiwany wynik:**
- Powinien zwrócić 1 dokument w kontekście
- Pewność może być niższa (mniej kontekstu)

---

## 💡 Podpowiedzi i rozwiązania typowych problemów

### Problem 1: Błąd walidacji k
**Przyczyna:** `k` nie jest integer lub jest poza zakresem
**Rozwiązanie:** Dodaj walidację w modelu:
```python
from pydantic import Field, validator

class AskRequest(BaseModel):
    query: str
    k: int = Field(default=3, ge=1, le=100)  # k między 1 a 100
```

### Problem 2: Czasy są zbyt małe
**Przyczyna:** `time.time()` ma niską rozdzielczość
**Rozwiązanie:** Użyj `time.perf_counter()` dla większej precyzji

### Problem 3: Pewność jest zawsze 100%
**Przyczyna:** Formuła oszacowania jest zbyt optymistyczna
**Rozwiązanie:** Dostosuj formułę, np.:
```python
def calculate_confidence(distances: list[float]) -> float:
    # Użyj średniej odległości i normalizuj
    avg_dist = sum(distances) / len(distances)
    # Dla cosine similarity (im wyższa, tym lepsze)
    return min(100, max(0, avg_dist * 50 + 50))  # Skala 0-100
```

### Problem 4: Błąd przy dostępie do row.id
**Przyczyna:** BigQuery nie zwraca pola id w wynikach
**Rozwiązanie:** Upewnij się, że zapytanie SELECT zawiera pole id:
```python
bq_query = f"""
SELECT base.id, base.content, distance
FROM VECTOR_SEARCH(
  TABLE `{table_ref}`,
  'embedding',
  (SELECT {query_embedding} as embedding),
  top_k => {k},
  distance_type => 'COSINE'
)
"""
```

### Problem 5: Wolne działanie przy dużym k
**Przyczyna:** Wyszukiwanie wielu dokumentów jest wolne
**Rozwiązanie:** Ograniczyć maksymalne k (np. do 20) lub użyć przybliżonego wyszukiwania

---

## 📚 Dokumentacja do przestudiowania

1. [Pydantic Documentation](https://docs.pydantic.dev/latest/) - Oficjalna dokumentacja Pydantic v2
2. [Python time module](https://docs.python.org/3/library/time.html) - Dokumentacja modułu time (time(), perf_counter())
3. [BigQuery Vector Search](https://cloud.google.com/bigquery/docs/vector-search) - Oficjalna dokumentacja Vector Search w BigQuery
4. [FAISS Index Parameters](https://github.com/facebookresearch/faiss/wiki/Faiss-for-the-impatient#index-parameters) - Przewodnik po parametrach indeksów FAISS
5. [Cosine Similarity](https://en.wikipedia.org/wiki/Cosine_similarity) - Wyjaśnienie cosine similarity na Wikipedii
6. [FastAPI Documentation](https://fastapi.tiangolo.com/) - Dokumentacja frameworka FastAPI
7. [Requests Documentation](https://docs.python-requests.org/) - Dokumentacja biblioteki requests

---

## ✅ Kryteria akceptacji

- [ ] Endpoint `/ask` akceptuje parametr `k`
- [ ] Zwracane są czasy wykonania dla każdego etapu
- [ ] Zwracane są informacje o źródle każdego dokumentu
- [ ] Zwracana jest oszacowana pewność odpowiedzi
- [ ] Domyślne zachowanie (k=3) jest zachowane
- [ ] Kod jest czytelny i skomentowany
- [ ] Walidacja parametrów działa poprawnie

---

## 🎉 Bonus (opcjonalne)

1. Dodaj endpoint `/ask_with_details` z jeszcze więcej statystyk
2. Zaimplementuj zaawansowaną metodę oszacowania pewności (np. z użyciem modelu)
3. Dodaj cache dla embeddingów, aby przyspieszyć powtarzające się zapytania
4. Zaimplementuj logging czasów wykonania do pliku
5. Dodaj endpoint `/stats` zwracający statystyki użycia
6. Zaimplementuj adaptacyjne k (automatyczne dostosowywanie k na podstawie pewności)
