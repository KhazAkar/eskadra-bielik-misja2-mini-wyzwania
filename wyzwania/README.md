# Eskadra Bielik - Mini Wyzwania

Repozytorium zawiera 5 mini-wyzwań opartych na istniejącej aplikacji RAG z modelem Bielik i EmbeddingGemma. Każde wyzwanie to samodzielne zadanie, które student może zaimplementować krok po kroku.

## Dostępne Wyzwania

| Nr | Tytuł | Opis | Poziom Trudności |
|----|-------|------|------------------|
| 1 | **Lokalne uruchomienie z SQLite + FAISS** | Zmiana architektury z Google BigQuery na lokalną bazę SQLite + FAISS | ⭐⭐ |
| 2 | **UI dla tematu warsztatów hotelowych** | Dostosowanie interfejsu użytkownika pod temat hotelowy | ⭐ |
| 3 | **Parametry wyszukiwania (k, czas, źródło, pewność)** | Rozszerzenie endpointu `/ask` o dodatkowe parametry | ⭐⭐⭐ |
| 4 | **Endpoint podsumowania źródeł** | Nowy endpoint `/ask_summary` + update UI | ⭐⭐⭐ |
| 5 | **Źródło PDF od użytkownika** | Zmiana źródła wiedzy na plik PDF przesyłany przez użytkownika | ⭐⭐⭐ |

## Jak korzystać?

1. **Przeczytaj dokumentację** - Każde wyzwanie ma swój własny `README.md` z instrukcjami
2. **Przeanalizuj istniejący kod** - Zobacz jak działa oryginalna aplikacja
3. **Zaimplementuj zmiany** - Postępuj zgodnie z podpowiedziami w dokumentacji
4. **Testuj** - Uruchom i sprawdź działanie

## Wspólne zależności

Wszystkie wyzwania (oprócz czysto UI) wymagają:
- Python 3.10+
- pip
- Wirtualne środowisko (rekomendowane)

## Struktura katalogów

```
wyzwania/
├── wyzwanie1/          # SQLite + FAISS
│   ├── README.md       # Instrukcje
│   ├── requirements.txt
│   ├── .env.example
│   └── main.py         # Przykładowa implementacja
├── wyzwanie2/          # UI Hotel
│   ├── README.md
│   └── index.html      # Zmodyfikowany UI
├── wyzwanie3/          # Parametry wyszukiwania
│   ├── README.md
│   ├── requirements.txt
│   └── main.py
├── wyzwanie4/          # /ask_summary endpoint
│   ├── README.md
│   ├── requirements.txt
│   └── main.py
└── wyzwanie5/          # PDF jako źródło
    ├── README.md
    ├── requirements.txt
    └── main.py
```

## Porady ogólne

1. **Czytaj dokumentację modułów** - Zawsze sprawdzaj dokumentację używanych bibliotek (FAISS, SQLite, PyPDF itp.)
2. **Testuj stopniowo** - Implementuj i testuj małe fragmenty kodu
3. **Korzystaj z debugowania** - Użyj `print()` lub debugera, aby śledzić przepływ danych
4. **Zachowaj kompatybilność** - Staraj się, aby zmiany były kompatybilne z istniejącym API

## Autor

Wyzwania przygotowane na podstawie projektu: [eskadra-bielik-misja2](https://github.com/avedave/eskadra-bielik-misja2)
