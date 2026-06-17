# Wyzwanie 4: Nowy endpoint + UI dla trybu podsumowania źródeł

**Cel:** Zaimplementować nowy endpoint `/ask_summary`, który zwraca podsumowanie podlinkowanych źródeł zamiast pełnej odpowiedzi, oraz zaktualizować UI aby obsługiwał ten nowy tryb.

## 📋 Opis wyzwania

Obecna aplikacja ma dwa tryby:
- `/ask` - RAG z pełną odpowiedzią i kontekstem
- `/ask_direct` - Bezpośrednia odpowiedź modelu bez RAG

Twoim zadaniem jest **dodanie trzeciego trybu** `/ask_summary`, który:
1. **Podsumowuje** znalezione dokumenty zamiast generowania pełnej odpowiedzi
2. **Zwraca** zwięzłe podsumowanie każdego źródła
3. **UI** powinien mieć nowy panel lub opcję do wyświetlenia podsumowań

## 🎯 Wymagania

1. Zaimplementować endpoint `/ask_summary`
2. Endpoint powinien:
   - Wyszukiwać dokumenty (jak `/ask`)
   - Zamiast generować odpowiedź, **podsumować** znalezione dokumenty
   - Zwracać listę podsumowań z identyfikatorami źródeł
3. Zaktualizować UI aby obsługiwał nowy endpoint
4. Czas implementacji: **max 1 godzina**

---

## 🚀 Kroki implementacji

### Krok 1: Zdefiniuj nowy model requestu

**Podpowiedź:**
- Możesz użyć tego samego modelu co `/ask` lub stworzyć nowy
- Sprawdź `AskRequest` w `orchestration/main.py`

**Do zrobienia:**

Dodaj nowy model (opcjonalnie, możesz użyć istniejącego):
```python
class AskSummaryRequest(BaseModel):
    query: str
    k: int = 3  # Liczba dokumentów do podsumowania
    max_length: int = 200  # Maksymalna długość podsumowania (opcjonalnie)
```

**Na co zwrócić uwagę:**
- Możesz dodać parametr `max_length` do kontroli długości podsumowania
- Zachowaj kompatybilność z istniejącym `AskRequest`

---

### Krok 2: Zaimplementuj endpoint `/ask_summary`

**Podpowiedź:**
- Endpoint powinien być podobny do `/ask`, ale zamiast generować odpowiedź, podsumowuje dokumenty
- Możesz użyć modelu LLM do generowania podsumowań

**Do zrobienia:**

Dodaj nowy endpoint:
```python
@app.post("/ask_summary")
async def ask_summary(request_data: AskSummaryRequest):
    query = request_data.query
    k = request_data.k
    max_length = request_data.max_length
    
    # 1. Generuj embedding dla zapytania
    query_embedding = get_embedding(query)
    
    # 2. Wyszukaj dokumenty (tak jak w /ask)
    # ... (kod wyszukiwania)
    context_docs = [row.content for row in results]
    
    # 3. Wygeneruj podsumowania dla każdego dokumentu
    summaries = []
    for i, doc in enumerate(context_docs):
        summary = generate_summary(doc, max_length)
        summaries.append({
            "id": i + 1,
            "source": f"document_{i}",  # lub prawdziwe ID
            "content": doc,
            "summary": summary
        })
    
    return {
        "query": query,
        "summaries": summaries,
        "count": len(summaries)
    }
```

**Na co zwrócić uwagę:**
- Wyszukiwanie dokumentów powinno być takie samo jak w `/ask`
- Podsumowania powinny być zwięzłe i informacyjne
- Możesz dodać informację o źródle (ID, nazwa pliku itp.)

---

### Krok 3: Zaimplementuj funkcję `generate_summary`

**Podpowiedź:**
- Możesz użyć modelu LLM do generowania podsumowań
- Albo użyć prostszej metody (np. pierwsze N zdań)

**Do zrobienia (z LLM):**

```python
def generate_summary(text: str, max_length: int = 200) -> str:
    """
    Generuj podsumowanie tekstu używając modelu LLM.
    
    Args:
        text: Tekst do podsumowania
        max_length: Maksymalna długość podsumowania (opcjonalnie)
        
    Returns:
        str: Podsumowanie
    """
    prompt = f"""
    Podsumuj poniższy tekst w maksymalnie {max_length} znakach.
    Bądź zwięzły i precyzyjny. Wyłącznie po polsku.
    
    TEKST:
    {text}
    
    PODSUMOWANIE:
    """
    
    try:
        response = call_llm(prompt)
        return response
    except Exception as e:
        print(f"Błąd generowania podsumowania: {e}")
        # Fallback: pierwsze zdanie
        sentences = text.split('.')
        return sentences[0] + "." if sentences else text[:max_length]
```

**Do zrobienia (prosta metoda):**

```python
def generate_summary(text: str, max_length: int = 200) -> str:
    """
    Generuj podsumowanie biorąc pierwsze zdania.
    """
    sentences = text.split('.')
    summary = []
    current_length = 0
    
    for sentence in sentences:
        if current_length + len(sentence) + 1 <= max_length:
            summary.append(sentence)
            current_length += len(sentence) + 1
        else:
            break
    
    result = '. '.join(summary) + ('.' if summary else '')
    return result if result else text[:max_length]
```

**Na co zwrócić uwagę:**
- Podsumowania powinny być zwięzłe (maksymalnie `max_length` znaków)
- Użyj polskiego języka w promptach
- Dodaj fallback na wypadek błędu LLM

---

### Krok 4: Zaktualizuj UI - dodaj nowy panel

**Podpowiedź:**
- Obecny UI ma dwa panele: Bielik i Bielik (RAG)
- Musisz dodać trzeci panel dla podsumowań
- Sprawdź `orchestration/static/index.html`

**Do zrobienia:**

1. **Dodaj nowy panel w HTML:**
```html
<div class="comparision-container">
    <!-- Istniejące panele -->
    <div class="panel">
        <h2>Bielik</h2>
        <p class="desc">Odpowiedź wygenerowana wyłącznie w oparciu o wagę modelu.</p>
        <div id="directContent" class="content"><i>Czekam na Twoje pytanie...</i></div>
    </div>
    
    <div class="panel">
        <h2>Bielik (RAG)</h2>
        <p class="desc">Odpowiedź wygenerowana w oparciu o odszukane dokumenty.</p>
        <div id="ragContent" class="content"><i>Czekam na Twoje pytanie...</i></div>
        <div id="ragContext" class="context-box" style="display: none;"></div>
    </div>
    
    <!-- NOWY PANEL -->
    <div class="panel">
        <h2>📝 Podsumowania</h2>
        <p class="desc">Podsumowania znalezionych dokumentów źródłowych.</p>
        <div id="summaryContent" class="content"><i>Czekam na Twoje pytanie...</i></div>
    </div>
</div>
```

2. **Dodaj obsługę w JavaScript:**
```javascript
const summaryContent = document.getElementById('summaryContent');

// Zmodyfikuj funkcję askBielik
async function askBielik() {
    const query = promptInput.value.trim();
    if(!query) return;

    submitBtn.disabled = true;
    promptInput.disabled = true;

    directContent.innerHTML = getSkeletonHTML();
    ragContent.innerHTML = getSkeletonHTML();
    summaryContent.innerHTML = getSkeletonHTML();
    ragContext.style.display = 'none';
    ragContext.innerHTML = '';

    const callApi = async (url) => {
        try {
            const response = await fetch(url, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ query: query })
            });
            
            if(!response.ok) throw new Error("Błąd serwera.");
            return await response.json();
        } catch (error) {
            return { error: error.message };
        }
    };

    const directPromise = callApi('/ask_direct');
    const ragPromise = callApi('/ask');
    const summaryPromise = callApi('/ask_summary');

    // Handle Direct response
    directPromise.then(res => {
        if(res.error) directContent.innerHTML = `<span style="color:#ff3b30;">${res.error}</span>`;
        else directContent.innerText = res.answer;
    });

    // Handle RAG response
    ragPromise.then(res => {
        if(res.error) {
            ragContent.innerHTML = `<span style="color:#ff3b30;">${res.error}</span>`;
        } else {
            ragContent.innerText = res.answer;
            if(res.context_used && res.context_used.length > 0) {
                ragContext.style.display = 'block';
                ragContext.innerHTML = `<h4>💡 Wykorzystany kontekst:</h4>`;
                res.context_used.forEach(ctx => {
                    ragContext.innerHTML += `<div class="context-item">${ctx}</div>`;
                });
            }
        }
    });

    // Handle Summary response
    summaryPromise.then(res => {
        if(res.error) {
            summaryContent.innerHTML = `<span style="color:#ff3b30;">${res.error}</span>`;
        } else {
            if(res.summaries && res.summaries.length > 0) {
                summaryContent.innerHTML = `<h4>Znaleziono ${res.count} dokumentów:</h4>`;
                res.summaries.forEach(summary => {
                    summaryContent.innerHTML += `
                        <div class="summary-item" style="margin-bottom: 16px; padding: 12px; background: #f5f5f7; border-radius: 8px;">
                            <strong>Źródło ${summary.id}:</strong>
                            <p style="margin-top: 8px;">${summary.summary}</p>
                        </div>
                    `;
                });
            } else {
                summaryContent.innerHTML = `<i>Nie znaleziono dokumentów.</i>`;
            }
        }
    });

    await Promise.allSettled([directPromise, ragPromise, summaryPromise]);
    submitBtn.disabled = false;
    promptInput.disabled = false;
    promptInput.focus();
}
```

**Na co zwrócić uwagę:**
- Nowy panel powinien być spójny wizualnie z istniejącymi
- Obsługa błędów powinna być taka sama jak w innych panelach
- Podsumowania powinny być dobrze sformatowane

---

### Krok 5: Dodaj opcję wyboru trybu (opcjonalne)

**Podpowiedź:**
- Możesz dodać przyciski lub select do wyboru trybu
- Albo użyć checkboxów do włączania/wyłączania poszczególnych paneli

**Do zrobienia:**

Dodaj checkboxy nad inputem:
```html
<div style="margin-bottom: 16px; display: flex; gap: 24px; align-items: center;">
    <label style="display: flex; align-items: center; gap: 8px;">
        <input type="checkbox" id="showDirect" checked> 
        <span>Bielik (Bez RAG)</span>
    </label>
    <label style="display: flex; align-items: center; gap: 8px;">
        <input type="checkbox" id="showRag" checked> 
        <span>Bielik + RAG</span>
    </label>
    <label style="display: flex; align-items: center; gap: 8px;">
        <input type="checkbox" id="showSummary" checked> 
        <span>Podsumowania</span>
    </label>
</div>
```

Zmodyfikuj funkcję `askBielik` aby uwzględniała checkboxy:
```javascript
async function askBielik() {
    const query = promptInput.value.trim();
    if(!query) return;

    submitBtn.disabled = true;
    promptInput.disabled = true;

    // Pokaż skeletony tylko dla zaznaczonych opcji
    const showDirect = document.getElementById('showDirect').checked;
    const showRag = document.getElementById('showRag').checked;
    const showSummary = document.getElementById('showSummary').checked;

    if(showDirect) directContent.innerHTML = getSkeletonHTML();
    if(showRag) {
        ragContent.innerHTML = getSkeletonHTML();
        ragContext.style.display = 'none';
    }
    if(showSummary) summaryContent.innerHTML = getSkeletonHTML();

    const promises = [];
    
    if(showDirect) {
        promises.push(callApi('/ask_direct').then(res => {
            if(res.error) directContent.innerHTML = `<span style="color:#ff3b30;">${res.error}</span>`;
            else directContent.innerText = res.answer;
        }));
    }

    if(showRag) {
        promises.push(callApi('/ask').then(res => {
            if(res.error) {
                ragContent.innerHTML = `<span style="color:#ff3b30;">${res.error}</span>`;
            } else {
                ragContent.innerText = res.answer;
                if(res.context_used && res.context_used.length > 0) {
                    ragContext.style.display = 'block';
                    ragContext.innerHTML = `<h4>💡 Wykorzystany kontekst:</h4>`;
                    res.context_used.forEach(ctx => {
                        ragContext.innerHTML += `<div class="context-item">${ctx}</div>`;
                    });
                }
            }
        }));
    }

    if(showSummary) {
        promises.push(callApi('/ask_summary').then(res => {
            if(res.error) {
                summaryContent.innerHTML = `<span style="color:#ff3b30;">${res.error}</span>`;
            } else {
                if(res.summaries && res.summaries.length > 0) {
                    summaryContent.innerHTML = `<h4>Znaleziono ${res.count} dokumentów:</h4>`;
                    res.summaries.forEach(summary => {
                        summaryContent.innerHTML += `
                            <div class="summary-item" style="margin-bottom: 16px; padding: 12px; background: #f5f5f7; border-radius: 8px;">
                                <strong>Źródło ${summary.id}:</strong>
                                <p style="margin-top: 8px;">${summary.summary}</p>
                            </div>
                        `;
                    });
                } else {
                    summaryContent.innerHTML = `<i>Nie znaleziono dokumentów.</i>`;
                }
            }
        }));
    }

    await Promise.allSettled(promises);
    submitBtn.disabled = false;
    promptInput.disabled = false;
    promptInput.focus();
}
```

**Na co zwrócić uwagę:**
- Checkboxy powinny być domyślnie zaznaczone
- Tylko zaznaczone panele powinny być aktualizowane
- UI powinien pamiętać wybory użytkownika

---

## 📦 Pliki do zmodyfikowania/stworzenia

| Plik | Opis | Akcja |
|------|------|-------|
| `main.py` | Główna aplikacja FastAPI | **Zmodyfikuj** |
| `index.html` | UI | **Zmodyfikuj** |

---

## 🔍 Testowanie

**Test 1: Wywołanie endpointu `/ask_summary`**
```bash
curl -X POST "http://localhost:8000/ask_summary" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?"}'
```

**Oczekiwany wynik:**
- Powinien zwrócić listę podsumowań dokumentów
- Każde podsumowanie powinno mieć `id`, `source`, `content`, `summary`
- Powinien zwrócić `count` (liczbę dokumentów)

**Test 2: Wywołanie z parametrem k**
```bash
curl -X POST "http://localhost:8000/ask_summary" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?", "k": 5}'
```

**Oczekiwany wynik:**
- Powinien zwrócić 5 podsumowań

**Test 3: Wywołanie z parametrem max_length**
```bash
curl -X POST "http://localhost:8000/ask_summary" \
     -H "Content-Type: application/json" \
     -d '{"query": "O której godzinie jest podawane śniadanie?", "max_length": 100}'
```

**Oczekiwany wynik:**
- Podsumowania powinny mieć maksymalnie 100 znaków

**Test 4: UI z nowym panelem**
- Otwórz `http://localhost:8000` w przeglądarce
- Zadaj pytanie

**Oczekiwany wynik:**
- Powinien pojawić się trzeci panel z podsumowaniami
- Podsumowania powinny być dobrze sformatowane

---

## 💡 Podpowiedzi i rozwiązania typowych problemów

### Problem 1: Podsumowania są zbyt długie
**Przyczyna:** Model LLM generuje zbyt długie odpowiedzi
**Rozwiązanie:** Dostosuj prompt lub użyj `max_tokens` w wywołaniu LLM:
```python
payload = {
    "model": "SpeakLeash/bielik-4.5b-v3.0-instruct:Q8_0",
    "messages": [{"role": "user", "content": prompt}],
    "stream": False,
    "options": {"max_tokens": 100}  # Ograniczenie długości
}
```

### Problem 2: Błąd przy wywołaniu LLM dla podsumowań
**Przyczyna:** Zbyt wiele wywołań LLM (po jednym na dokument)
**Rozwiązanie:** 
- Użyj batch processing (jeśli API to obsługuje)
- Albo generuj podsumowania sekwencyjnie
- Albo użyj prostszej metody (pierwsze zdanie)

### Problem 3: UI nie wyświetla nowego panelu
**Przyczyna:** Błąd w JavaScript lub HTML
**Rozwiązanie:**
- Sprawdź, czy nowy panel jest dodany do HTML
- Sprawdź, czy JavaScript obsługuje nowy endpoint
- Sprawdź console przeglądarki (F12) w poszukiwaniu błędów

### Problem 4: Podsumowania są nieczytelne
**Przyczyna:** Zły formatowanie w HTML
**Rozwiązanie:** Dodaj lepsze style CSS:
```css
.summary-item {
    margin-bottom: 16px;
    padding: 12px;
    background: #f5f5f7;
    border-radius: 8px;
    border-left: 3px solid var(--accent-color);
}

.summary-item strong {
    color: var(--accent-color);
    display: block;
    margin-bottom: 4px;
}

.summary-item p {
    margin: 8px 0 0 0;
    line-height: 1.5;
}
```

### Problem 5: Wolne działanie przy wielu dokumentach
**Przyczyna:** Generowanie podsumowań dla wielu dokumentów jest wolne
**Rozwiązanie:**
- Ograniczyć liczbę dokumentów (parametr `k`)
- Użyć prostszej metody podsumowania (pierwsze zdanie)
- Zaimplementować cache dla podsumowań

---

## 📚 Dokumentacja do przestudiowania

1. [FastAPI Path Parameters](https://fastapi.tiangolo.com/tutorial/path-params/)
2. [FastAPI Query Parameters](https://fastapi.tiangolo.com/tutorial/query-params/)
3. [Ollama API - max_tokens](https://github.com/ollama/ollama/blob/main/docs/api.md#generate-a-response)
4. [JavaScript async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
5. [JavaScript Promise.allSettled](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)

---

## ✅ Kryteria akceptacji

- [ ] Endpoint `/ask_summary` działa i zwraca podsumowania
- [ ] Podsumowania są zwięzłe i informacyjne
- [ ] UI ma nowy panel dla podsumowań
- [ ] UI obsługuje nowy endpoint
- [ ] Wszystkie istniejące funkcjonalności działają
- [ ] Kod jest czytelny i skomentowany
- [ ] Obsługa błędów działa poprawnie

---

## 🎉 Bonus (opcjonalne)

1. Dodaj opcję wyboru trybu (checkboxy lub radio buttons)
2. Zaimplementuj podsumowanie wszystkich dokumentów naraz (zamiast pojedynczo)
3. Dodaj opcję wyświetlania pełnego tekstu dokumentu po kliknięciu
4. Zaimplementuj ranking podsumowań według trafności
5. Dodaj opcję eksportu podsumowań do pliku
6. Zaimplementuj podsumowanie w formie listy punktowanej
7. Dodaj opcję dostosowania długości podsumowania przez użytkownika
