# Wyzwanie 5: Zmiana źródła na podany przez użytkownika plik PDF

**Cel:** Zmienić źródło wiedzy z pliku CSV na plik PDF przesyłany przez użytkownika, umożliwiając dynamiczne zasilanie bazy wiedzy dokumentami PDF.

## 📋 Opis wyzwania

Obecna aplikacja korzysta z:
- **Pliku CSV** (`hotel_rules.csv`) jako źródła wiedzy
- **Endpointu `/ingest`** do wgrywania danych z CSV

Twoim zadaniem jest **rozszerzenie aplikacji o obsługę plików PDF**, aby:
1. Użytkownik mógł przesłać **dowolny plik PDF**
2. System **wyekstrahował tekst** z PDF
3. Tekst został **podzielony na fragmenty** (chunking)
4. Fragmenty zostały **zaindeksowane** w bazie wektorowej
5. Użytkownik mógł **zadawać pytania** dotyczące treści PDF

## 🎯 Wymagania

1. Dodać obsługę uploadu plików PDF w endpointzie `/ingest`
2. Zaimplementować ekstrakcję tekstu z PDF
3. Zaimplementować chunking (podział tekstu na fragmenty)
4. Zachować kompatybilność z istniejącym CSV
5. Czas implementacji: **max 1 godzina**

---

## 🚀 Kroki implementacji

### Krok 1: Zainstaluj biblioteki do obsługi PDF

**Podpowiedź:**
- Do ekstrakcji tekstu z PDF potrzebujesz biblioteki
- Popularne opcje:
  - `PyPDF2` - prosta, czysta ekstrakcja tekstu
  - `pdfplumber` - bardziej zaawansowana, lepsza obsługa layoutu
  - `pdfminer.six` - bardzo dokładna, ale wolniejsza

**Do zrobienia:**
```bash
pip install PyPDF2
# lub
pip install pdfplumber
```

**Na co zwrócić uwagę:**
- `PyPDF2` jest prostszy i szybszy
- `pdfplumber` lepiej radzi sobie z złożonymi layoutami
- Sprawdź dokumentację wybranej biblioteki

---

### Krok 2: Zmodyfikuj endpoint `/ingest` - obsługa PDF

**Podpowiedź:**
- Obecny endpoint `/ingest` akceptuje tylko CSV
- Musisz dodać obsługę PDF
- Sprawdź `orchestration/main.py`

**Do zrobienia:**

Zmień funkcję `ingest_csv` na `ingest_file` (lub dodaj nową funkcję):
```python
from fastapi import UploadFile, File
import io

@app.post("/ingest")
async def ingest_file(file: UploadFile = File(...)):
    """
    Ingest file (CSV or PDF) and index its content.
    """
    # Sprawdź typ pliku po rozszerzeniu
    filename = file.filename.lower()
    
    if filename.endswith('.csv'):
        return await ingest_csv(file)
    elif filename.endswith('.pdf'):
        return await ingest_pdf(file)
    else:
        raise HTTPException(
            status_code=400, 
            detail="Niewspierany format pliku. Obsługiwane formaty: CSV, PDF"
        )
```

**Na co zwrócić uwagę:**
- Sprawdź rozszerzenie pliku (`.csv`, `.pdf`)
- Zachowaj kompatybilność z istniejącym CSV
- Zwróć odpowiedni błąd dla niewspieranych formatów

---

### Krok 3: Zaimplementuj ekstrakcję tekstu z PDF

**Podpowiedź:**
- Użyj `PyPDF2` lub `pdfplumber` do ekstrakcji tekstu
- Dokumentacja PyPDF2: [https://pypdf2.readthedocs.io/](https://pypdf2.readthedocs.io/)

**Do zrobienia (z PyPDF2):**

```python
from PyPDF2 import PdfReader
import io

async def extract_text_from_pdf(file: UploadFile) -> str:
    """
    Wyekstrahuj tekst z pliku PDF.
    
    Args:
        file: UploadFile z FastAPI
        
    Returns:
        str: Wyekstrahowany tekst
    """
    try:
        # Odczytaj zawartość pliku
        content = await file.read()
        
        # Utwórz obiekt PdfReader
        pdf_reader = PdfReader(io.BytesIO(content))
        
        # Wyekstrahuj tekst ze wszystkich stron
        text = ""
        for page in pdf_reader.pages:
            text += page.extract_text() + "\n"
        
        return text
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Błąd podczas ekstrakcji tekstu z PDF: {e}"
        )
```

**Do zrobienia (z pdfplumber):**

```python
import pdfplumber

async def extract_text_from_pdf(file: UploadFile) -> str:
    """
    Wyekstrahuj tekst z pliku PDF używając pdfplumber.
    """
    try:
        content = await file.read()
        
        with pdfplumber.open(io.BytesIO(content)) as pdf:
            text = ""
            for page in pdf.pages:
                text += page.extract_text() + "\n"
        
        return text
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Błąd podczas ekstrakcji tekstu z PDF: {e}"
        )
```

**Na co zwrócić uwagę:**
- PDF może mieć złożony layout (kolumny, tabele)
- Tekst może wymagać czyszczenia (usunięcie nadmiarowych spacji, nowych linii)
- Niektóre PDFy są skanowane (obrazy) - nie da się wyekstrahować tekstu

---

### Krok 4: Zaimplementuj chunking (podział tekstu na fragmenty)

**Podpowiedź:**
- Tekst z PDF może być bardzo długi
- Musisz podzielić go na mniejsze fragmenty (chunks)
- Każdy fragment będzie osobnym dokumentem w bazie wektorowej
- Popularne strategie:
  - Podział po liczbie znaków (fixed size)
  - Podział po zdaniach
  - Podział po akapitach
  - Zaawansowane: sliding window, semantic chunking

**Do zrobienia:**

```python
def chunk_text(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    """
    Podziel tekst na fragmenty (chunks) o określonym rozmiarze.
    
    Args:
        text: Tekst do podziału
        chunk_size: Rozmiar jednego fragmentu (w znakach)
        overlap: Nakładka między fragmentami (w znakach)
        
    Returns:
        list[str]: Lista fragmentów
    """
    if not text:
        return []
    
    # Czyszczenie tekstu
    text = text.replace('\n\n', '\n').replace('\n', ' ').strip()
    
    chunks = []
    start = 0
    
    while start < len(text):
        end = min(start + chunk_size, len(text))
        
        # Jeśli nie jesteśmy na końcu, spróbuj znaleźć koniec zdania
        if end < len(text):
            # Szukaj ostatniego kropka, wykrzyknika lub pytajnika przed end
            last_punct = max(
                text.rfind('.', start, end),
                text.rfind('!', start, end),
                text.rfind('?', start, end)
            )
            if last_punct > start + chunk_size // 2:  # Jeśli znaleziono w drugiej połowie
                end = last_punct + 1
        
        chunk = text[start:end].strip()
        if chunk:
            chunks.append(chunk)
        
        # Przesuń start o (chunk_size - overlap)
        start = end - overlap if end - overlap > start else end
    
    return chunks
```

**Na co zwrócić uwagę:**
- `chunk_size` powinien być dostosowany do modelu (typowo 500-1000 znaków)
- `overlap` pomaga zachować kontekst między fragmentami (typowo 50-200 znaków)
- Fragmenty powinny kończyć się na końcu zdania (jeśli to możliwe)

---

### Krok 5: Zaimplementuj funkcję `ingest_pdf`

**Podpowiedź:**
- Powinna wyekstrahować tekst z PDF
- Podzielić na fragmenty
- Wygenerować embeddingi dla każdego fragmentu
- Zapisać do bazy wektorowej

**Do zrobienia:**

```python
@app.post("/ingest")
async def ingest_pdf(file: UploadFile = File(...)):
    """
    Ingest PDF file and index its content.
    """
    # 1. Wyekstrahuj tekst z PDF
    text = await extract_text_from_pdf(file)
    
    if not text or len(text.strip()) == 0:
        raise HTTPException(
            status_code=400,
            detail="Plik PDF nie zawiera tekstu lub jest pusty"
        )
    
    # 2. Podziel tekst na fragmenty
    chunks = chunk_text(text, chunk_size=500, overlap=50)
    
    if not chunks:
        raise HTTPException(
            status_code=400,
            detail="Nie udało się podzielić tekstu na fragmenty"
        )
    
    # 3. Generuj embeddingi i zapisz do bazy
    rows_to_insert = []
    for i, chunk in enumerate(chunks):
        try:
            embedding = get_embedding(chunk)
            rows_to_insert.append({
                "id": f"pdf_{file.filename}_{i}",
                "content": chunk,
                "embedding": embedding,
                "source": file.filename,  # Zapamiętaj źródło
                "chunk_index": i
            })
        except Exception as e:
            print(f"Błąd w generowaniu osadzenia dla fragmentu {i}: {e}")
            continue
    
    if rows_to_insert:
        # Zapisz do BigQuery lub FAISS + SQLite
        # ... (tak jak w oryginalnym /ingest)
        pass
    
    return {
        "status": "success",
        "filename": file.filename,
        "chunks_count": len(chunks),
        "inserted_count": len(rows_to_insert)
    }
```

**Na co zwrócić uwagę:**
- Każdy fragment powinien mieć unikalne ID
- ID powinno zawierać nazwę pliku i indeks fragmentu
- Zapamiętaj źródło (nazwa pliku) dla każdego fragmentu

---

### Krok 6: Zaktualizuj endpoint `/ask` - informacja o źródle

**Podpowiedź:**
- Obecnie `/ask` zwraca `context_used` z tekstami dokumentów
- Powinien również zwracać informację o źródle (nazwa pliku)
- Sprawdź Wyzwanie 3, które już to implementuje

**Do zrobienia:**

Zmień zwracaną strukturę:
```python
context_docs = []
for row in results:
    context_docs.append({
        "content": row.content,
        "source": row.source if hasattr(row, 'source') else "unknown",
        "id": row.id if hasattr(row, 'id') else "unknown"
    })

return {
    "answer": answer,
    "context_used": [doc["content"] for doc in context_docs],
    "context_details": context_docs  # Pełne informacje
}
```

**Na co zwrócić uwagę:**
- `source` powinien zawierać nazwę pliku PDF
- Zachowaj kompatybilność z istniejącym API

---

### Krok 7: Zaktualizuj UI - informacja o źródle

**Podpowiedź:**
- UI powinien wyświetlać informację o źródle
- Sprawdź Wyzwanie 3 i 4

**Do zrobienia:**

Zmień wyświetlanie kontekstu:
```javascript
if(res.context_details && res.context_details.length > 0) {
    ragContext.style.display = 'block';
    ragContext.innerHTML = `<h4>📋 Źródła informacji:</h4>`;
    res.context_details.forEach(ctx => {
        ragContext.innerHTML += `
            <div class="context-item">
                <strong>📄 ${ctx.source} (ID: ${ctx.id}):</strong>
                <p>${ctx.content}</p>
            </div>
        `;
    });
}
```

**Na co zwrócić uwagę:**
- Wyświetl nazwę pliku PDF jako źródło
- Zachowaj czytelność i estetykę

---

### Krok 8: Dodaj endpoint `/list_sources` (opcjonalne)

**Podpowiedź:**
- Użytkownik może chcieć zobaczyć, jakie pliki zostały wgrane
- Możesz dodać endpoint zwracający listę źródeł

**Do zrobienia:**

```python
@app.get("/list_sources")
async def list_sources():
    """
    Zwróć listę wszystkich źródeł (plików) w bazie wiedzy.
    """
    # Dla BigQuery:
    query = f"""
    SELECT DISTINCT source
    FROM `{PROJECT_ID}.{DATASET_ID}.{TABLE_ID}`
    WHERE source IS NOT NULL
    """
    
    try:
        query_job = bq_client.query(query)
        results = query_job.result()
        sources = [row.source for row in results]
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Błąd podczas pobierania listy źródeł: {e}"
        )
    
    return {
        "sources": sources,
        "count": len(sources)
    }
```

**Dla SQLite + FAISS:**
```python
@app.get("/list_sources")
async def list_sources():
    """
    Zwróć listę wszystkich źródeł w bazie SQLite.
    """
    try:
        cursor.execute("SELECT DISTINCT source FROM documents WHERE source IS NOT NULL")
        sources = [row[0] for row in cursor.fetchall()]
        return {
            "sources": sources,
            "count": len(sources)
        }
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Błąd podczas pobierania listy źródeł: {e}"
        )
```

---

## 📦 Pliki do zmodyfikowania/stworzenia

| Plik | Opis | Akcja |
|------|------|-------|
| `requirements.txt` | Zależności Python | **Zmodyfikuj** |
| `main.py` | Główna aplikacja FastAPI | **Zmodyfikuj** |
| `index.html` | UI (opcjonalnie) | **Zmodyfikuj** |

---

## 🔍 Testowanie

**Test 1: Wgraj plik PDF**
```bash
curl -X POST "http://localhost:8000/ingest" \
     -F "file=@test_document.pdf"
```

**Oczekiwany wynik:**
- Plik PDF powinien zostać wyekstrahowany
- Tekst powinien zostać podzielony na fragmenty
- Fragmenty powinny zostać zaindeksowane
- Powinien zwrócić `{"status": "success", "filename": "...", "chunks_count": N, "inserted_count": M}`

**Test 2: Zadaj pytanie o treści PDF**
```bash
curl -X POST "http://localhost:8000/ask" \
     -H "Content-Type: application/json" \
     -d '{"query": "Jaka jest główna teza dokumentu?"}'
```

**Oczekiwany wynik:**
- Odpowiedź powinna dotyczyć treści wgranego PDF
- `context_details` powinien zawierać informację o źródle (nazwa pliku)

**Test 3: Wgraj kolejny plik PDF**
```bash
curl -X POST "http://localhost:8000/ingest" \
     -F "file=@another_document.pdf"
```

**Oczekiwany wynik:**
- Obydwa pliki powinny być zaindeksowane
- System powinien obsługiwać wiele źródeł

**Test 4: Lista źródeł (opcjonalne)**
```bash
curl "http://localhost:8000/list_sources"
```

**Oczekiwany wynik:**
- Powinien zwrócić listę wgranych plików

---

## 💡 Podpowiedzi i rozwiązania typowych problemów

### Problem 1: Błąd ekstrakcji tekstu z PDF
**Przyczyna:** Plik PDF jest skanowany (obraz) lub zabezpieczony
**Rozwiązanie:**
- Użyj OCR (np. `pytesseract`) dla skanowanych PDFów
- Poinformuj użytkownika, że plik nie zawiera tekstu
- Sprawdź, czy plik nie jest zabezpieczony hasłem

### Problem 2: Tekst z PDF jest nieczytelny
**Przyczyna:** Zły layout, kolumny, tabele
**Rozwiązanie:**
- Użyj `pdfplumber` z opcjami czyszczenia:
```python
text = page.extract_text(x_tolerance=1, y_tolerance=1)
```
- Albo spróbuj innej biblioteki

### Problem 3: Fragmenty są zbyt duże lub zbyt małe
**Przyczyna:** Niewłaściwe parametry chunkingu
**Rozwiązanie:** Dostosuj `chunk_size` i `overlap`:
```python
# Dla długich dokumentów
chunks = chunk_text(text, chunk_size=1000, overlap=100)

# Dla krótkich dokumentów
chunks = chunk_text(text, chunk_size=300, overlap=30)
```

### Problem 4: Powtarzające się fragmenty
**Przyczyna:** Zbyt duży overlap
**Rozwiązanie:** Zmniejsz `overlap` lub zmień strategię chunkingu

### Problem 5: Wolne przetwarzanie dużych PDFów
**Przyczyna:** Duże pliki PDF wymagają dużo czasu na ekstrakcję i chunking
**Rozwiązanie:**
- Ograniczyć rozmiar pliku (np. max 10MB)
- Zaimplementować processing w tle (background task)
- Dodać progress bar w UI

**Przykład ograniczenia rozmiaru:**
```python
@app.post("/ingest")
async def ingest_file(file: UploadFile = File(...)):
    # Sprawdź rozmiar pliku
    content = await file.read()
    if len(content) > 10 * 1024 * 1024:  # 10MB
        raise HTTPException(
            status_code=400,
            detail="Plik jest zbyt duży. Maksymalny rozmiar: 10MB"
        )
    
    # Resetuj file.read() aby móc odczytać ponownie
    file.file.seek(0)
    # ... reszta kodu
```

### Problem 6: Błąd przy podziale na zdania
**Przyczyna:** Tekst nie zawiera standardowych znaków końca zdania
**Rozwiązanie:** Użyj prostszego podziału (po liczbie znaków):
```python
def chunk_text(text: str, chunk_size: int = 500) -> list[str]:
    """Prosty podział po liczbie znaków."""
    chunks = []
    for i in range(0, len(text), chunk_size):
        chunk = text[i:i+chunk_size].strip()
        if chunk:
            chunks.append(chunk)
    return chunks
```

---

## 📚 Dokumentacja do przestudiowania

1. [PyPDF2 Documentation](https://pypdf2.readthedocs.io/)
2. [pdfplumber Documentation](https://github.com/jsvine/pdfplumber)
3. [Text Chunking Strategies](https://www.pinecone.io/learn/chunking-strategies/)
4. [FastAPI File Upload](https://fastapi.tiangolo.com/tutorial/request-files/)
5. [FastAPI Background Tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/)

---

## ✅ Kryteria akceptacji

- [ ] Endpoint `/ingest` akceptuje pliki PDF
- [ ] Tekst jest poprawnie ekstrahowany z PDF
- [ ] Tekst jest dzielony na fragmenty (chunks)
- [ ] Fragmenty są indeksowane w bazie wektorowej
- [ ] Informacja o źródle (nazwa pliku) jest zachowana
- [ ] Użytkownik może zadawać pytania o treści PDF
- [ ] Zachowana jest kompatybilność z CSV
- [ ] Kod jest czytelny i skomentowany

---

## 🎉 Bonus (opcjonalne)

1. Dodaj obsługę plików DOCX (użyj `python-docx`)
2. Dodaj obsługę plików TXT
3. Zaimplementuj OCR dla skanowanych PDFów (użyj `pytesseract`)
4. Dodaj endpoint `/delete_source` do usuwania źródeł
5. Zaimplementuj wersjonowanie dokumentów
6. Dodaj opcję podziału na rozdziały (dla strukturowanych PDFów)
7. Zaimplementuj automatyczne wykrywanie języka tekstu
8. Dodaj opcję ekstrakcji tabel z PDF (użyj `camelot` lub `pdfplumber`)
9. Zaimplementuj cache dla wyekstrahowanych tekstów
10. Dodaj opcję podglądu tekstu przed indeksowaniem
