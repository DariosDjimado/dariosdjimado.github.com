# 20 opérateurs implémentés avec les Gatherer - Java 22

L'API Stream de Java évolue avec l'introduction des `Stream Gatherers` via la proposition `JEP 461`. Cette nouvelle
fonctionnalité permet de créer des opérateurs personnalisés, offrant ainsi une plus grande flexibilité et des
possibilités de traitement de données enrichies. Voici 20 exemples concrets pour illustrer ces nouvelles capacités. Ces
exemples montrent comment les `gatherers` peuvent être intégrés dans des projets Java pour des transformations de
données plus spécifiques et adaptées à divers besoins.

## 1. map

Transformer chaque élément du flux en un autre objet à l'aide de la fonction spécifiée.

```java
public static <T, R> Gatherer<T, ?, R> map(Function<? super T, ? extends R> mapper) {
  Gatherer.Integrator<Void, T, R> integrator = (_, element, downstream) -> {
    downstream.push(mapper.apply(element));
    return true;
  };
  return Gatherer.of(integrator);
}
```

Exemple d'utilisation

```java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(map(n -> n * n))
  .toList();

// output: [1, 4, 9, 16, 25]
```

## 2. filter

Filtrer les éléments du flux en fonction d'un prédicat spécifié.

```java
public static <T> Gatherer<T, ?, T> filter(Predicate<T> predicate) {
  Gatherer.Integrator<Void, T, T> integrator = (_, element, downstream) -> {
    if (predicate.test(element)) {
      downstream.push(element);
    }
    return true;
  };
  return Gatherer.of(integrator);
}
```

Exemple d'utilisation

```java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(filter(n -> n % 2 == 0))
  .toList();
    
// output: [2, 4]
```

## 3. flatMap

Transformer chaque élément du flux puis aplatir le résultat en un seul flux.

```java
public static <T, R> Gatherer<T, ?, R> flatMap(Function<? super T,
    Stream<? extends R>> mapper) {
  Gatherer.Integrator<Void, T, R> integrator = (_, element, downstream) -> {
    mapper.apply(element).forEach(downstream::push);
    return true;
  };
  return Gatherer.of(integrator);
}
```

Exemple d'utilisation

```java
List<List<Integer>> input = List.of(
  List.of(1, 2), List.of(3, 4), List.of(5, 6));
List<Integer> output = input.stream()
  .gather(flatMap(List::stream))
  .toList();
  
// output: [1, 2, 3, 4, 5, 6]
```

## 4. distinct

Éliminer les doublons du flux.

```java
public static <T> Gatherer<T, ?, T> distinct() {
  Supplier<Set<T>> initializer = HashSet::new;
  Gatherer.Integrator<Set<T>, T, T> integrator = (state, element, downstream) -> {
    if (state.add(element)) {
      downstream.push(element);
    }
    return true;
  };

  return Gatherer.ofSequential(initializer, integrator);
}
```

Exemple d'utilisation

```java
List<Integer> input = List.of(1, 2, 3, 4, 5, 5, 4, 3, 2, 1);
List<Integer> output = input.stream()
  .gather(distinct())
  .toList();

// output: [1, 2, 3, 4, 5]
```

## 5. limit

Limiter la taille du flux à n éléments.

```java
public static <T> Gatherer<T, ?, T> limit(long maxSize) {
  Supplier<AtomicLong> initializer = AtomicLong::new;
  Gatherer.Integrator<AtomicLong, T, T> integrator = (state, element, downstream) -> {
    long size = state.getAndIncrement();
    if (size < maxSize) {
      downstream.push(element);
    }
    return size + 1 < maxSize;
  };

  return Gatherer.ofSequential(initializer, integrator);
}
```

Exemple d'utilisation

```java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(limit(3))
  .toList();
  
// output: [1, 2, 3]
```

## 6. skip

Ignorer les n premiers éléments du flux.

```java
public static <T> Gatherer<T, ?, T> skip(long n) {
  Supplier<AtomicLong> initializer = AtomicLong::new;
  Gatherer.Integrator<AtomicLong, T, T> integrator = (state, element, downstream) -> {
    long index = state.getAndIncrement();
    if (index >= n) {
      downstream.push(element);
    }
    return true;
  };

  return Gatherer.ofSequential(initializer, integrator);
}
```

Exemple d'utilisation

```java
  List<Integer> input = List.of(1, 2, 3, 4, 5);
  List<Integer> output = input.stream()
      .gather(skip(2))
      .toList();

// output: [3, 4, 5]
```

## 7. takeWhile

Accepter les éléments tant que la condition spécifiée est vraie.

```Java
public static <T> Gatherer<T, ?, T> takeWhile(Predicate<T> predicate) {
  Gatherer.Integrator<Void, T, T> integrator = (_, element, downstream) -> {
    if (predicate.test(element)) {
      downstream.push(element);
      return true;
    }
    return false;
  };

  return Gatherer.of(integrator);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(takeWhile(n -> n < 3))
  .toList();
  
// output: [1, 2]
```

## 8. dropWhile

Ignorer les éléments tant que la condition spécifiée est vraie puis inclure tous les éléments restants.

```Java
public static <T> Gatherer<T, ?, T> dropWhile(Predicate<T> predicate) {
  Supplier<AtomicBoolean> startKeeping = AtomicBoolean::new;
  Gatherer.Integrator<AtomicBoolean, T, T> integrator = (state, element, downstream) -> {
    boolean keep = state.get();
    if (keep) {
      downstream.push(element);
    } else if (!predicate.test(element)) {
      state.set(true);
      downstream.push(element);
    }

    return true;
  };

  return Gatherer.ofSequential(startKeeping, integrator);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(dropWhile(n -> n < 3))
  .toList();
  
// output: [3, 4, 5]
```

## 9. scan

Appliquer une fonction accumulatrice à chaque élément du flux.

```Java
public static <T, R> Gatherer<T, ?, R> scan(R initialValue, BiFunction<R, T, R> accumulator) {
  Supplier<AtomicReference<R>> initializer = () -> new AtomicReference<>(initialValue);
  Gatherer.Integrator<AtomicReference<R>, T, R> integrator = (state, element, downstream) -> {
    state.set(accumulator.apply(state.get(), element));
    downstream.push(state.get());
    return true;
  };

  return Gatherer.ofSequential(initializer, integrator);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(scan(0, Integer::sum))
  .toList();

// output: [1, 3, 6, 10, 15]
```

## 10. pairwise

Regrouper les éléments consécutifs du flux en paires à partir du deuxième élément.

```Java
public static <T> Gatherer<T, ?, List<T>> pairwise() {
  Supplier<AtomicReference<T>> initializer = AtomicReference::new;
  Gatherer.Integrator<AtomicReference<T>, T, List<T>> integrator = (state, element, downstream) -> {
    T previous = state.getAndSet(element);
    if (previous != null) {
      downstream.push(List.of(previous, element));
    }
    return true;
  };

  return Gatherer.ofSequential(initializer, integrator);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<List<Integer>> output = input.stream()
  .gather(pairwise())
  .toList();

// output: [[1, 2], [2, 3], [3, 4], [4, 5], [5, 6]]
```

## 11. Peek

Appliquer une action à chaque élément du flux sans modifier les éléments eux-mêmes.

```Java
public static <T> Gatherer<T, ?, T> peek(Consumer<T> consumer) {
  Gatherer.Integrator<Void, T, T> integrator = (_, element, downstream) -> {
    consumer.accept(element);
    downstream.push(element);
    return true;
  };

  return Gatherer.of(integrator);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(peek(i -> System.out.println(STR."Processing: \{i}")))
  .toList();

// output:
// Processing: 1
// Processing: 2
// Processing: 3
// Processing: 4
// Processing: 5
// [1, 2, 3, 4, 5]
```

## 12. sorted

Trier les éléments du flux selon un comparateur spécifié.

```Java
public static <T> Gatherer<T, ?, T> sorted(Comparator<T> comparator) {
  Supplier<List<T>> initializer = ArrayList::new;
  Gatherer.Integrator<List<T>, T, T> integrator = (state, element, _) -> {
    state.add(element);
    return true;
  };

  BiConsumer<List<T>, Gatherer.Downstream<? super T>> finisher = (state, downstream) -> {
    state.sort(comparator);
    state.forEach(downstream::push);
  };

  return Gatherer.ofSequential(initializer, integrator, finisher);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(5, 3, 2, 1, 4);
List<Integer> output = input.stream()
  .gather(sorted(Comparator.<Integer>naturalOrder()))
  .toList();

// output: [1, 2, 3, 4, 5]
```

## 13. reverse

Inverser l'ordre des éléments du flux.

```Java
public static <T> Gatherer<T, ?, T> reverse() {
  Supplier<List<T>> initializer = ArrayList::new;
  Gatherer.Integrator<List<T>, T, T> integrator = (state, element, _) -> {
    state.add(element);
    return true;
  };

  BiConsumer<List<T>, Gatherer.Downstream<? super T>> finisher = (state, downstream) -> {
    for (int i = state.size() - 1; i >= 0; i--) {
      downstream.push(state.get(i));
    }
  };

  return Gatherer.ofSequential(initializer, integrator, finisher);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(reverse())
  .toList();
  
// output: [5, 4, 3, 2, 1]
```

## 14.window

Regrouper les éléments du flux en fenêtres de taille n.

```Java
public static <T> Gatherer<T, ?, List<T>> window(int size) {
  assert size > 0 : "Size must be greater than 0";

  Supplier<List<T>> initializer = () -> new ArrayList<>(size);
  Gatherer.Integrator<List<T>, T, List<T>> integrator = (state, element, downstream) -> {
    state.add(element);
    if (state.size() == size) {
      downstream.push(List.copyOf(state));
      state.clear();
    }
    return true;
  };

  BiConsumer<List<T>, Gatherer.Downstream<? super List<T>>> finisher = (state, downstream) -> {
    if (!state.isEmpty()) {
      downstream.push(List.copyOf(state));
    }
  };

  return Gatherer.ofSequential(initializer, integrator, finisher);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<List<Integer>> output = input.stream()
  .gather(window(2))
  .toList();

// output: [[1, 2], [3, 4], [5]]
```

## 15. slidingWindow

Regrouper les éléments du flux en fenêtres glissantes de taille n.

```Java
public static <T> Gatherer<T, ?, List<T>> slidingWindow(int size) {
  assert size > 0 : "Size must be greater than 0";

  Supplier<Deque<T>> initializer = () -> new ArrayDeque<>(size);
  Gatherer.Integrator<Deque<T>, T, List<T>> integrator = (state, element, downstream) -> {
    state.add(element);
    if (state.size() == size) {
      downstream.push(List.copyOf(state));
      state.removeFirst();
    }
    return true;
  };

  BiConsumer<Deque<T>, Gatherer.Downstream<? super List<T>>> finisher = (state, downstream) -> {
    if (!state.isEmpty()) {
      downstream.push(List.copyOf(state));
    }
  };

  return Gatherer.ofSequential(initializer, integrator, finisher);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<List<Integer>> output = input.stream()
  .gather(slidingWindow(2))
  .toList();
  
// output: [[1, 2], [2, 3], [3, 4], [4, 5], [5]]
```

## 16. interleave

Entrelacer les éléments de deux flux.

```Java
public static <T> Gatherer<T, ?, T> interleave(Stream<T> other) {
  Supplier<Iterator<T>> initializer = other::iterator;
  Gatherer.Integrator<Iterator<T>, T, T> integrator = (state, element, downstream) -> {
    downstream.push(element);
    if (state.hasNext()) {
      downstream.push(state.next());
    }
    return true;
  };

  BiConsumer<Iterator<T>, Gatherer.Downstream<? super T>> finisher = (state, downstream) -> {
    while (state.hasNext()) {
      downstream.push(state.next());
    }
  };
  
  return Gatherer.ofSequential(initializer, integrator, finisher);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 3, 5, 7, 9);
List<Integer> output = input.stream()
  .gather(interleave(Stream.of(2, 4, 6, 8, 10)))
  .toList();

// output: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

## 17. concat

Concaténer plusieurs flux en un seul flux.

```Java
@SafeVarargs
public static <T> Gatherer<T, ?, T> concat(Stream<T>... other) {
  Gatherer.Integrator<Void, T, T> integrator = (_, element, downstream) -> {
    downstream.push(element);
    return true;
  };

  BiConsumer<Void, Gatherer.Downstream<? super T>> finisher = (_, downstream) -> {
    for (Stream<T> stream : other) {
      stream.forEach(downstream::push);
    }
  };

  return Gatherer.of(integrator, finisher);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(
      concat(Stream.of(6, 7, 8, 9, 10), Stream.of(11, 12, 13, 14, 15)))
  .toList();

// output: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15]
```

## 18. repeat

Répéter les éléments du flux n fois.

```Java
public static <T> Gatherer<T, ?, T> repeat(int n) {
  Gatherer.Integrator<Void, T, T> integrator = (_, element, downstream) -> {
    for (int i = 0; i < n; i++) {
      downstream.push(element);
    }
    return true;
  };

  return Gatherer.of(integrator);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5);
List<Integer> output = input.stream()
  .gather(repeat(3))
  .toList();

// output: [1, 1, 1, 2, 2, 2, 3, 3, 3, 4, 4, 4, 5, 5, 5]
```

## 19. distinctUntilChanged

Éliminer les éléments consécutifs en double du flux.

```Java
public static <T> Gatherer<T, ?, T> distinctUntilChanged() {
  Supplier<AtomicReference<T>> initializer = AtomicReference::new;
  Gatherer.Integrator<AtomicReference<T>, T, T> integrator = (state, element, downstream) -> {
    T lastValue = state.getAndSet(element);
    if (!element.equals(lastValue)) {
      downstream.push(element);
    }
    return true;
  };

  return Gatherer.ofSequential(initializer, integrator);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 1, 2, 2, 3, 3, 1, 1, 4, 4, 5, 5, 2, 2);
List<Integer> output = input.stream()
  .gather(distinctUntilChanged())
  .toList();

// output: [1, 2, 3, 1, 4, 5, 2]
```

## 20. groupBy

Regrouper les éléments du flux en fonction d'une fonction de classification.

```Java
public static <T, K> Gatherer<T, ?, List<T>> groupBy(Function<? super T, ? extends K> classifier) {
  Supplier<Map<K, List<T>>> initializer = HashMap::new;
  Gatherer.Integrator<Map<K, List<T>>, T, List<T>> integrator = (state, element, _) -> {
    K key = classifier.apply(element);
    state.computeIfAbsent(key, _ -> new ArrayList<>()).add(element);
    return true;
  };

  BiConsumer<Map<K, List<T>>, Gatherer.Downstream<? super List<T>>> finisher = (state, downstream) -> {
    state.forEach((_, values) -> downstream.push(List.copyOf(values)));
  };

  return Gatherer.ofSequential(initializer, integrator, finisher);
}
```

Exemple d'utilisation

```Java
List<Integer> input = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
List<List<Integer>> output = input.stream()
  .gather(groupBy(i -> i % 2 == 0))
  .toList();

// output: [[1, 3, 5, 7, 9], [2, 4, 6, 8, 10]]
```

## Conclusion

L'introduction des `Stream Gatherers` dans JEP 461 marque une avancée significative pour le développement
Java. Ces opérateurs personnalisés enrichissent les capacités de traitement des flux de données, permettant des
transformations plus fines et adaptées aux besoins spécifiques de chaque projet. En exploitant ces nouvelles
possibilités, il est possible d'écrire un code plus clair, plus performant, et mieux aligné sur les exigences métiers.
Les exemples illustrés montrent la polyvalence et la puissance des Stream Gatherers, et ouvrent la voie à une
utilisation plus appropriée des flux de données dans les applications Java modernes.

## Ressources

- [Code source de l'exemple](https://github.com/DariosDjimado/gatherer-preview)
- [JEP 461: Stream Gatherers (Preview)](https://openjdk.org/jeps/461)
