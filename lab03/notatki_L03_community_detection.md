# Notatki – Wykrywanie społeczności w sieciach złożonych (L03)

---

## 1. Podstawowe pojęcia

### Społeczność (Community)
- **Społeczność** to lokalnie gęsto połączony podgraf wewnątrz większej sieci.
- Węzły wewnątrz społeczności są ze sobą silnie połączone, a połączeń między różnymi społecznościami jest relatywnie mało.
- Wykrywanie społeczności (*community detection*) pozwala odkryć ukrytą strukturę grupową sieci.

### Klika (Clique)
- **Klika** to w pełni połączony podgraf – każdy węzeł jest połączony z każdym innym węzłem w tym podgrafie.
- Kliki o rozmiarze $k$ to podgrafy $K_k$.

### Silna społeczność (Strong community)
- Każdy węzeł ma **więcej krawędzi wewnątrz** grupy niż na zewnątrz.
- Warunek silniejszy – dotyczy każdego węzła z osobna.

### Słaba społeczność (Weak community)
- **Łączny stopień wewnętrzny** grupy przekracza łączny stopień zewnętrzny.
- Warunek słabszy – dotyczy całej grupy jako całości, nie poszczególnych węzłów.

### Partycja grafu
- Podział zbioru węzłów na rozłączne, niepuste podzbiory (społeczności).
- Każdy węzeł należy dokładnie do jednej społeczności.
- W NetworkX sprawdzamy poprawność partycji: `nx.community.is_partition(G, partition)`.

---

## 2. Modularność (Modularity)

### Intuicja
Modularność mierzy, **o ile gęstsza jest sieć wewnątrz społeczności** w porównaniu do losowej sieci o takim samym rozkładzie stopni węzłów.

> Losowe sieci nie mają struktury grupowej – są punktem odniesienia.

### Wzór

$$Q = \frac{1}{L}\sum_C\left(L_C - \frac{k_C^2}{4L}\right)$$

| Symbol | Znaczenie |
|--------|-----------|
| $L$ | całkowita liczba krawędzi w grafie |
| $L_C$ | liczba krawędzi **wewnątrz** klastra $C$ |
| $\frac{k_C^2}{4L}$ | oczekiwana liczba krawędzi wewnątrz $C$ w sieci losowej |
| $k_C$ | suma stopni węzłów w klastrze $C$ |

### Interpretacja wartości $Q$
| Wartość $Q$ | Interpretacja |
|-------------|---------------|
| $Q \approx 0$ | brak struktury grupowej (jak sieć losowa) |
| $Q > 0.3$ | umiarkowana struktura społecznościowa |
| $Q > 0.7$ | silna struktura społecznościowa |
| $Q = 1$ | teoretyczne maksimum (nieosiągalne w praktyce) |

### Zastosowanie w NetworkX
```python
nx.community.quality.modularity(G, partition)
```

### Ważna właściwość
Zmiana partycji zmienia wartość $Q$. Partycja, w której węzły z różnych naturalnych grup są razem, będzie miała niższą modularność niż partycja respektująca strukturę sieci.

---

## 3. Używane sieci testowe

| Sieć | Funkcja NetworkX | Opis |
|------|-----------------|------|
| Connected caveman graph | `nx.connected_caveman_graph(4, 8)` | Kliki połączone mostami – wyraźna struktura grupowa |
| Small-world network | `nx.watts_strogatz_graph(34, 4, 0.15)` | Model Wattsa-Strogatza – sieć małego świata |
| Windmill graph | `nx.windmill_graph(5, 7)` | Klikowate "wiatraczki" – silne grupy |
| Gaussian random partition graph | `nx.gaussian_random_partition_graph(...)` | Losowa partycja z gaussowskim szumem |
| Random partition graph | `nx.random_partition_graph(...)` | Losowa partycja z ~80% krawędzi wewnętrznych |
| Karate club network | `nx.karate_club_graph()` | Klasyczna sieć realna – 34 węzły, 2 społeczności |

---

## 4. Algorytm Girvan–Newman (klastrowanie hierarchiczne)

### Idea
Algorytm oparty na **pośrednictwie krawędziowym** (*edge betweenness centrality*). Krawędzie łączące różne społeczności mają wysokie pośrednictwo – przez nie przechodzi wiele najkrótszych ścieżek.

### Pośrednictwo krawędziowe (Edge betweenness centrality)
$$C_B(e) = \sum_{s \neq t} \frac{\sigma(s,t|e)}{\sigma(s,t)}$$

gdzie $\sigma(s,t)$ – liczba najkrótszych ścieżek między $s$ i $t$, a $\sigma(s,t|e)$ – liczba tych ścieżek przechodzących przez krawędź $e$.

### Kroki algorytmu
1. Oblicz pośrednictwo wszystkich krawędzi w sieci.
2. Usuń krawędź (lub krawędzie) o **najwyższym pośrednictwie**.
3. Przelicz pośrednictwo krawędzi, na które wpłynęło usunięcie.
4. Powtarzaj kroki 2–3 aż wszystkie krawędzie zostaną usunięte.

### Wynik – dendrogram
- Algorytm produkuje **hierarchię partycji** (dendrogram).
- Na każdym poziomie hierarchii mamy inny podział na społeczności.
- Najlepszy podział wybieramy, **maksymalizując modularność $Q$**.

### Zastosowanie w NetworkX
```python
comp = nx.algorithms.community.girvan_newman(G)
# comp to generator – każda iteracja to kolejny poziom podziału
for communities in itertools.islice(comp, k):
    print(sorted(map(sorted, communities)))
```

### Wady algorytmu
- Złożoność obliczeniowa: **$O(m^2 n)$** (gdzie $m$ – liczba krawędzi, $n$ – węzłów).
- Dla dużych sieci bardzo wolny.
- Przy każdym usunięciu krawędzi przelicza się całe pośrednictwo od nowa.

---

## 5. Algorytm Louvain (optymalizacja modularności)

### Idea
Louvain to **zachłanny algorytm optymalizacji modularności**. Jest ekstremalnie szybki – nadaje się do sieci z milionami węzłów i krawędzi.

### Kroki algorytmu

#### Faza 1 – lokalna optymalizacja
1. Na początku każdy węzeł tworzy własną społeczność (tyle społeczności ile węzłów).
2. Dla każdego węzła $i$:
   - Oblicz przyrost modularności $\Delta Q$ przy przeniesieniu $i$ do każdej sąsiedniej społeczności.
   - Przenieś węzeł do społeczności dającej **największy przyrost** $\Delta Q$.
   - Jeśli żadne przeniesienie nie zwiększa $Q$ – węzeł pozostaje w swojej społeczności.
3. Powtarzaj przejście po wszystkich węzłach, aż $Q$ przestanie rosnąć.

#### Faza 2 – kompresja sieci (budowa supersieci)
1. Każda wykryta społeczność staje się **superwęzłem**.
2. Krawędzie między superwęzłami = sumaryczna liczba połączeń między społecznościami.
3. Krawędzie wewnątrz superwęzłów = krawędzie wewnątrz społeczności (pętle własne).

#### Iteracja faz
- Fazy 1 i 2 są powtarzane na coraz mniejszej (skompresowanej) sieci.
- Kończymy, gdy żadna zmiana nie zwiększa modularności.

### Zastosowanie w NetworkX
```python
from networkx.algorithms.community import louvain_communities
partition = louvain_communities(G, seed=42)
```

### Zalety Louvain
- Złożoność: **$O(n \log n)$** – praktycznie liniowa.
- Działa na bardzo dużych sieciach.
- Nie wymaga podania liczby społeczności z góry.

### Wady Louvain
- Niedeterministyczny (zależy od losowej kolejności węzłów) → parametr `seed`.
- Może wpaść w lokalne maksimum modularności.
- Znany problem: "rozdzielanie" dużych społeczności.

---

## 6. Porównanie algorytmów

| Cecha | Girvan–Newman | Louvain |
|-------|--------------|---------|
| Podstawa | pośrednictwo krawędzi | optymalizacja modularności |
| Złożoność | $O(m^2 n)$ – wolny | $O(n \log n)$ – szybki |
| Skalowalność | małe sieci | bardzo duże sieci |
| Hierarchia | tak (dendrogram) | nie (flat) |
| Deterministyczność | tak | nie (zależy od seed) |
| Wybór liczby grup | po fakcie (wg $Q$) | automatyczny |

---

## 7. Modularność a struktura grafu

### Zależności
- **Klaster coefficient** – im wyższy, tym sieć ma bardziej trójkątową strukturę, co może sprzyjać wysokiej modularności.
- **Średni stopień węzła** – zbyt niski (sieć rzadka) lub zbyt wysoki (sieć gęsta) obniżają modularność.
- **Centrality measures** – węzły o wysokim betweenness łączą różne grupy → niskie pośrednictwo oznacza mocne osadzenie w społeczności.
- **Rozmiar sieci** – modularność jest relatywna; ta sama struktura w większej sieci może mieć inne $Q$.

### Rodzaje badanych sieci a modularność
| Typ sieci | Oczekiwana modularność |
|-----------|----------------------|
| Connected caveman graph | bardzo wysoka (wyraźne kliki) |
| Windmill graph | wysoka (klikowe grupy) |
| Random partition graph | wysoka (sztuczne grupy) |
| Small-world (Watts–Strogatz) | umiarkowana |
| Losowa (Erdős–Rényi) | bliska 0 |

---

## 8. Przypadek rzeczywisty – sieć emailowa firmy

### Dane
- Historia komunikacji emailowej pracowników firmy produkcyjnej (9 miesięcy).
- Każdy email (To, CC, BCC) = osobny wiersz w danych.
- Dodatkowe dane: hierarchia raportowania (`reportsto.csv`), node #86 = CEO.

### Budowanie sieci
- **Sieć ważona**: węzły = pracownicy, wagi krawędzi = liczba emaili między parą.
- Usunięcie kont nie-pracowniczych (filtrowanie).

### Wizualizacja hierarchii
- Użycie `pyGraphviz` z layoutem `dot` do wizualizacji drzewa hierarchii firmy.
- Kolory węzłów według poziomu zarządzania (czerwony = top, pomarańczowy = mid, niebieski = pozostali).

### Zastosowania analizy społeczności w firmie
- Identyfikacja **nieformalnych grup** niewidocznych w strukturze organizacyjnej.
- Wykrycie **silosów komunikacyjnych** – grup odizolowanych od reszty.
- Analiza przepływu informacji między działami.
- Identyfikacja kluczowych pośredników komunikacyjnych (*brokerów*).
- Wsparcie decyzji HR (np. restrukturyzacja, optymalizacja zespołów).

---

## 9. Kluczowe wzory – podsumowanie

### Modularność
$$Q = \frac{1}{L}\sum_C\left(L_C - \frac{k_C^2}{4L}\right)$$

### Pośrednictwo krawędzi (Girvan-Newman)
$$C_B(e) = \sum_{s \neq t} \frac{\sigma(s,t|e)}{\sigma(s,t)}$$

### Przyrost modularności w Louvain (Δ$Q$)
$$\Delta Q = \left[\frac{\Sigma_{in} + 2k_{i,in}}{2m} - \left(\frac{\Sigma_{tot} + k_i}{2m}\right)^2\right] - \left[\frac{\Sigma_{in}}{2m} - \left(\frac{\Sigma_{tot}}{2m}\right)^2 - \left(\frac{k_i}{2m}\right)^2\right]$$

gdzie:
- $\Sigma_{in}$ – suma wag krawędzi wewnątrz społeczności
- $\Sigma_{tot}$ – suma wag krawędzi incydentnych do społeczności
- $k_i$ – stopień węzła $i$ (suma wag jego krawędzi)
- $k_{i,in}$ – suma wag krawędzi między $i$ a węzłami w tej społeczności
- $m$ – suma wszystkich wag krawędzi w grafie

---

## 10. Użyteczne funkcje NetworkX

```python
# Sprawdzenie partycji
nx.community.is_partition(G, partition)

# Modularność partycji
nx.community.quality.modularity(G, partition)

# Girvan-Newman
nx.algorithms.community.girvan_newman(G)

# Louvain
nx.algorithms.community.louvain_communities(G, seed=42)

# Kliki
nx.cliques_containing_node(G, node)
nx.graph_number_of_cliques(G)
nx.clique.find_cliques(G)  # wszystkie maksymalne kliki

# Layout hierarchiczny (wymaga pygraphviz)
nx.nx_agraph.graphviz_layout(H, prog='dot')
```
