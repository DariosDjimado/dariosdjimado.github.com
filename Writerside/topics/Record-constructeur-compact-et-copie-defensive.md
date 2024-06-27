# Record, constructeur compact et copie défensive

Les records sont une nouvelle fonctionnalité introduite en Java 14 qui permet de définir des classes de données de
manière concise. Ils sont souvent utilisés pour représenter des données immuables.

## Record

```Java
public record Person(String name, int age, List<String> hobbies) {}
```

Les records sont déclarés à l'aide du mot-clé `record` suivi du nom de l'objet et de la liste des champs appelés
composants. Le compilateur génère automatiquement un constructeur dit canonique qui initialise tous les composants,
des méthodes d'accès aux composants et les méthodes `equals()`, `hashCode()` et `toString()`.

### Exemple d'utilisation

```Java
public class RecordExample {
  public static void main(String[] args) {
    var alice = new Person("Alice", 42, List.of("Reading", "Coding"));

    // getters
    System.out.println(alice.name()); // Alice
    System.out.println(alice.age()); // 42
    System.out.println(alice.hobbies()); // [Reading, Coding]

    // toString
    System.out.println(alice); // Person[
    // name=Alice, age=42, hobbies=[Reading, Coding]]

    // equals
    var alice2 = new Person("Alice", 42, List.of("Reading", "Coding"));
    System.out.println(alice.equals(alice2)); // true

    // hashCode
    System.out.println(alice.hashCode() == alice2.hashCode()); // true
  }
}
```

## Constructeur compact : validation des champs

Nous avons intérêt à ce que de tels objets soient valides dès leur création. Sinon, il sera utile de les déboguer en
production, ce qui peut être un choix, j'entends bien :). Pour cela, on peut définir un constructeur compact qui
effectue les vérifications nécessaires avant de créer une instance du record.

```Java
public record Person(String name, int age, List<String> hobbies) {
  public Person {
    if (age < 0) {
      throw new IllegalArgumentException("Age must be positive");
    }
    if (name == null || name.isBlank()) {
      throw new IllegalArgumentException(
              "Name must not be null or blank");
    }
    if (hobbies == null || hobbies.isEmpty()) {
      throw new IllegalArgumentException(
              "Hobbies must not be null or empty");
    }
  }
}
```

Un constructeur compact est une méthode qui porte le même nom que la classe et qui ne prend pas de paramètres. Il est
exécuté avant le constructeur canonique et peut être utilisé pour effectuer des vérifications sur les champs avant de
créer une instance. Ainsi, on garantit que les instances de record sont toujours valides.

Il est possible que la liste des passe-temps contienne des éléments de valeur `null` ou vides. Pour éviter cela, on peut
créer par exemple un record `Hobby` qui contient un seul champ `name` et qui est utilisé dans le record `Person`. Il est
aussi possible de valider les éléments de la liste des passe-temps dans le constructeur compact.

## Copie défensive

Les records sont immuables, ce qui signifie que leurs champs ne peuvent pas être modifiés après la création de
l'instance. Cependant, si un champ est mutable, tel qu'une liste, il est possible de modifier son contenu après la
création de l'instance. Pour éviter cela, il est possible de faire une copie dite défensive des champs mutables dans le
constructeur.

```Java
public record Person(String name, int age, List<String> hobbies) {
  public Person {
    if (age < 0) {
      throw new IllegalArgumentException("Age must be positive");
    }
    if (name == null || name.isBlank()) {
      throw new IllegalArgumentException("Name must not be null or blank");
    }
    if (hobbies == null || hobbies.isEmpty()) {
      throw new IllegalArgumentException("Hobbies must not be null or empty");
    }

    hobbies = List.copyOf(hobbies); // defensive copy
  }
}
```

Dans cet exemple, nous faisons une copie défensive de la liste des passe-temps en utilisant la méthode `List.copyOf()`.
Cela crée une copie immuable de la liste, ce qui garantit que les modifications apportées à la liste d'origine
n'affectent pas l'instance du record. Petite note : si la liste est déjà immuable, la méthode `List.copyOf()` ne crée
pas de nouvelle instance, mais renvoie simplement la liste d'origine.

La copie défensive est une technique qui consiste à créer une copie immuable des champs mutables d'un objet pour éviter
les modifications non autorisées. Cela garantit que les instances de record restent immuables et cohérentes.

## Conclusion

Les records simplifient la définition de classes de données immuables. En utilisant des constructeurs compacts et des
copies défensives, on peut garantir que les instances de record sont toujours valides et cohérentes. Cela rend le
code plus sûr et plus facile à comprendre, en réduisant les risques d'erreurs et de bugs liés à la manipulation des
données.

## Ressources

- [JEP 359: Records (Standard) - OpenJDK](https://openjdk.java.net/jeps/359)
