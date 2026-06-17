# Wyzwanie 2: UI dla tematu warsztatów hotelowych

**Cel:** Dostosować istniejący interfejs użytkownika (UI) pod temat warsztatów hotelowych, zachowując funkcjonalność porównywania odpowiedzi modelu Bielik z i bez RAG.

## 📋 Opis wyzwania

Obecny UI (w `orchestration/static/index.html`) jest ogólny i nie jest dostosowany do konkretnego tematu. Twoim zadaniem jest **przystosowanie go pod temat warsztatów hotelowych**, aby:

1. **Wizualnie** - pasował do tematyki hotelowej
2. **Funkcjonalnie** - lepiej prezentował informacje specyficzne dla hotelu
3. **Użytkownik** - miał lepsze doświadczenie przy testowaniu systemu RAG

## 🎯 Wymagania

1. Zmienić **tytuł, opisy i kolorystykę** na hotelową
2. Dodać **ikony lub grafiki** związane z hotelem (opcjonalnie)
3. Poprawić **prezentację kontekstu** - lepsze formatowanie wyświetlanych dokumentów
4. Zachować **wszystkie istniejące funkcjonalności** (porównanie RAG vs bez RAG)
5. Czas implementacji: **max 1 godzina**

---

## 🚀 Kroki implementacji

### Krok 1: Analiza obecnego UI

**Podpowiedź:**
- Przeczytaj plik `orchestration/static/index.html`
- Zidentyfikuj sekcje, które należy zmienić
- Zrozum jak działa JavaScript do obsługi API

**Do zrobienia:**
1. Otwórz plik `orchestration/static/index.html`
2. Zidentyfikuj:
   - Nagłówek i tytuł
   - Opisy panelów
   - Kolorystykę CSS
   - Sekcję kontekstu
   - JavaScript obsługujący API

**Na co zwrócić uwagę:**
- Nie zmieniaj logiki JavaScript (funkcje `askBielik()`, `callApi()` itp.)
- Skup się na warstwie wizualnej (HTML, CSS)
- Zachowaj wszystkie endpointy API (`/ask`, `/ask_direct`)

---

### Krok 2: Zmiana tytułu i opisu

**Podpowiedź:**
- Tytuł jest w tagu `<title>` i w sekcji `<div class="header">`
- Opis jest w tagu `<p>` wewnątrz header

**Do zrobienia:**

Zmień:
```html
<title>Eskadra Bielik - Porównanie RAG</title>
```
na:
```html
<title>Hotel Bielik - Asystent RAG</title>
```

Zmień:
```html
<h1>Bielik + RAG</h1>
<p>Zadaj pytanie i porównaj odpowiedź samego modelu Bielik z odpowiedzią wspartą kontekstem RAG opartym o hotelową bazę wiedzy w BigQuery.</p>
```
na:
```html
<h1>🏨 Hotel Bielik - Asystent</h1>
<p>Zadaj pytanie dotyczące zasad hotelowych i porównaj odpowiedź modelu Bielik z odpowiedzią wspartą kontekstem RAG opartym o bazę wiedzy hotelu.</p>
```

**Na co zwrócić uwagę:**
- Użyj emoji związanych z hotelem (🏨, 🛎️, 🍽️, 🏊 itp.)
- Opis powinien być klarowny i zwięzły

---

### Krok 3: Zmiana kolorystyki CSS

**Podpowiedź:**
- Kolory są zdefiniowane w sekcji `:root` w CSS
- Obecna paleta: niebieskie akcenty (`--accent-color: #0071e3`)
- Dla hotelu możesz użyć:
  - **Złote/brązowe** - luksusowy hotel
  - **Zielone** - ekologiczny hotel
  - **Ciemnoniebieskie** - profesjonalny hotel

**Do zrobienia:**

Zmień w sekcji `:root`:
```css
:root {
    --bg-color: #f5f5f7;
    --container-bg: rgba(255, 255, 255, 0.85);
    --text-color: #1d1d1f;
    --accent-color: #0071e3;
    --accent-hover: #0077ED;
    --border-color: #d2d2d7;
    --shadow: 0 4px 24px rgba(0,0,0,0.06);
}
```

**Przykładowa paleta dla luksusowego hotelu:**
```css
:root {
    --bg-color: #faf9f7;
    --container-bg: rgba(255, 255, 255, 0.95);
    --text-color: #2c2c2c;
    --accent-color: #b8860b;  /* Złoty */
    --accent-hover: #daa520; /* Ciemniejszy złoty */
    --border-color: #e0d8c0;
    --shadow: 0 4px 24px rgba(184, 134, 11, 0.15);
}
```

**Przykładowa paleta dla nowoczesnego hotelu:**
```css
:root {
    --bg-color: #f0f2f5;
    --container-bg: rgba(255, 255, 255, 0.9);
    --text-color: #1a1a1a;
    --accent-color: #2c5f5f;  /* Ciemna zieleń */
    --accent-hover: #3a7a7a;
    --border-color: #d0d0d0;
    --shadow: 0 4px 24px rgba(44, 95, 95, 0.1);
}
```

**Na co zwrócić uwagę:**
- Kolory powinny być spójne i estetyczne
- Sprawdź kontrast między tekstem a tłem (dostępność)
- Możesz użyć narzędzi jak [Coolors](https://coolors.co/) do generowania palet

---

### Krok 4: Zmiana nagłówków paneli

**Podpowiedź:**
- Nagłówki paneli są w tagach `<h2>` wewnątrz `<div class="panel">`
- Opisy paneli są w tagach `<p class="desc">`

**Do zrobienia:**

Zmień:
```html
<h2>Bielik</h2>
<p class="desc">Odpowiedź wygenerowana wyłącznie w oparciu o wagę modelu, bez dostępu do bazy wektorowej BigQuery.</p>
```
na:
```html
<h2>🤖 Bielik (Bez RAG)</h2>
<p class="desc">Odpowiedź wygenerowana wyłącznie w oparciu o wiedzę modelu, bez dostępu do bazy wiedzy hotelu.</p>
```

Zmień:
```html
<h2>Bielik (RAG)</h2>
<p class="desc">Odpowiedź wygenerowana w oparciu o odszukane w locie dokumenty z BigQuery Vector Search.</p>
```
na:
```html
<h2>🏨 Bielik + RAG</h2>
<p class="desc">Odpowiedź wygenerowana w oparciu o odszukane dokumenty z bazy wiedzy hotelu (regulamin, zasady, usługi).</p>
```

**Na co zwrócić uwagę:**
- Użyj emoji, które pomogą zidentyfikować panele
- Opisy powinny być zrozumiałe dla użytkownika końcowego

---

### Krok 5: Ulepszenie sekcji kontekstu

**Podpowiedź:**
- Sekcja kontekstu jest w `<div id="ragContext" class="context-box">`
- Obecnie wyświetla surowy tekst dokumentów
- Możesz dodać numerowanie, ikony, lepsze formatowanie

**Do zrobienia:**

Zmień w JavaScript (sekcja obsługująca odpowiedź RAG):
```javascript
ragContext.innerHTML = `<h4>💡 Wykorzystany kontekst z BigQuery:</h4>`;
res.context_used.forEach(ctx => {
    ragContext.innerHTML += `<div class="context-item">${ctx}</div>`;
});
```

na:
```javascript
ragContext.innerHTML = `<h4>📋 Źródła informacji:</h4>`;
res.context_used.forEach((ctx, index) => {
    ragContext.innerHTML += `
        <div class="context-item">
            <strong>Źródło ${index + 1}:</strong>
            <p>${ctx}</p>
        </div>
    `;
});
```

**Na co zwrócić uwagę:**
- Numerowanie źródeł ułatwia identyfikację
- Lepsze formatowanie (akapity, pogrubienia) poprawia czytelność
- Możesz dodać ikony (📄, 🔍, 📌) przy każdym źródle

---

### Krok 6: Dodanie placeholdera dla input

**Podpowiedź:**
- Placeholder jest w atrybucie `placeholder` tagu `<input>`
- Powinien sugerować użytkownikowi, jakie pytania może zadać

**Do zrobienia:**

Zmień:
```html
<input type="text" id="promptInput" placeholder="Np. Ile kosztuje parking hotelowy?" autocomplete="off" autofocus>
```

na:
```html
<input type="text" id="promptInput" placeholder="Zadaj pytanie o hotelu... Np. O której godzinie jest śniadanie?" autocomplete="off" autofocus>
```

**Na co zwrócić uwagę:**
- Placeholder powinien być konkretny i pomocny
- Możesz dodać kilka przykładów rozdzielonych przecinkami

---

### Krok 7: Dodanie ikon lub grafik (opcjonalne)

**Podpowiedź:**
- Możesz dodać ikony z [Font Awesome](https://fontawesome.com/) lub [Emoji](https://emojipedia.org/)
- Albo dodać proste grafiki CSS (np. tło z wzorem)

**Do zrobienia (przykład z emoji):**

Dodaj ikony do nagłówków:
```html
<h1>🏨 Hotel Bielik - Asystent</h1>
```

Dodaj ikony do przycisku:
```html
<button id="submitBtn">🔍 Zapytaj</button>
```

**Do zrobienia (przykład z CSS background):**

Dodaj subtelne tło do body:
```css
body {
    background: linear-gradient(135deg, #f5f5f7 0%, #e8e8e8 100%);
    /* lub */
    background-image: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100" viewBox="0 0 100 100"><circle cx="50" cy="50" r="1" fill="%23b8860b" opacity="0.1"/></svg>');
}
```

**Na co zwrócić uwagę:**
- Nie przesadzaj z ilością grafik (mogą spowolnić stronę)
- Użyj subtelnych wzorów, które nie rozpraszają uwagi

---

### Krok 8: Dodanie stopki (opcjonalne)

**Podpowiedź:**
- Stopka może zawierać informacje o projekcie
- Możesz dodać link do dokumentacji

**Do zrobienia:**

Dodaj na końcu body:
```html
<footer style="margin-top: 40px; text-align: center; color: #86868b; font-size: 14px;">
    <p>Hotel Bielik - Inteligentny Asystent | Warsztaty RAG</p>
</footer>
```

---

## 📦 Pliki do zmodyfikowania

| Plik | Opis | Akcja |
|------|------|-------|
| `index.html` | Główna strona UI | **Zmodyfikuj** |

---

## 🔍 Testowanie

**Test 1: Otwórz stronę w przeglądarce**
- Uruchom aplikację: `uvicorn main:app --reload`
- Otwórz `http://localhost:8000` w przeglądarce

**Oczekiwany wynik:**
- Strona powinna mieć nowy tytuł "Hotel Bielik"
- Kolory powinny być zmienione na hotelową paletę
- Nagłówki paneli powinny być zaktualizowane

**Test 2: Zadaj pytanie**
- Wpisz pytanie np. "O której godzinie jest śniadanie?"
- Kliknij "Zapytaj"

**Oczekiwany wynik:**
- Obie odpowiedzi (z RAG i bez) powinny się wyświetlić
- Sekcja kontekstu powinna mieć lepsze formatowanie
- Kolory przycisku i input powinny pasować do nowej palety

**Test 3: Sprawdź responsywność**
- Zmień rozmiar okna przeglądarki
- Otwórz na urządzeniu mobilnym (opcjonalnie)

**Oczekiwany wynik:**
- UI powinien dobrze wyglądać na różnych rozmiarach ekranu

---

## 💡 Podpowiedzi i rozwiązania typowych problemów

### Problem 1: CSS nie działa
**Przyczyna:** Błąd składni w CSS
**Rozwiązanie:** Sprawdź, czy wszystkie nawiasy `}` są zamknięte

### Problem 2: Kolory nie zmieniają się
**Przyczyna:** Zmieniłeś zły plik lub cache przeglądarki
**Rozwiązanie:** 
- Upewnij się, że edytujesz `orchestration/static/index.html`
- Wyczyść cache przeglądarki (Ctrl+Shift+R)

### Problem 3: JavaScript przestał działać
**Przyczyna:** Usunąłeś lub zmieniłeś ważny fragment kodu JS
**Rozwiązanie:** Porównaj z oryginalnym plikiem i przywróć usunięte fragmenty

### Problem 4: UI wygląda źle na mobile
**Przyczyna:** Brak media queries w CSS
**Rozwiązanie:** Dodaj media queries dla mniejszych ekranów:
```css
@media (max-width: 768px) {
    .comparision-container {
        flex-direction: column;
    }
    .panel {
        width: 100%;
    }
}
```

---

## 📚 Dokumentacja do przestudiowania

1. [HTML Living Standard](https://html.spec.whatwg.org/)
2. [CSS MDN Documentation](https://developer.mozilla.org/en-US/docs/Web/CSS)
3. [CSS Colors](https://developer.mozilla.org/en-US/docs/Web/CSS/color)
4. [Emoji List](https://emojipedia.org/)
5. [Font Awesome Icons](https://fontawesome.com/icons) (opcjonalne)

---

## ✅ Kryteria akceptacji

- [ ] Tytuł strony zmieniony na hotelowy
- [ ] Kolorystyka dostosowana do tematu hotelowego
- [ ] Nagłówki i opisy paneli zaktualizowane
- [ ] Sekcja kontekstu ma lepsze formatowanie
- [ ] Placeholder inputa jest pomocny
- [ ] Wszystkie funkcjonalności (API, porównanie) działają
- [ ] UI jest estetyczny i spójny
- [ ] Kod HTML/CSS jest czytelny

---

## 🎉 Bonus (opcjonalne)

1. Dodaj animacje CSS (np. płynne przejścia kolorów)
2. Dodaj ikony z Font Awesome
3. Zaimplementuj tryb ciemny/jasny
4. Dodaj sekcję "Przykładowe pytania" z przyciskami
5. Dodaj loading spinner zamiast skeletonów
6. Dodaj obsługę błędów (np. gdy API nie odpowiada)
