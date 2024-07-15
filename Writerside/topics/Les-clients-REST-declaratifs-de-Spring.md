# Les clients REST déclaratifs de Spring

L'utilisation des APIs RESTful est devenue un standard pour les applications modernes. Spring Framework, avec son
écosystème riche et flexible, offre des solutions élégantes pour consommer ces APIs. En particulier, l'introduction
de `RestClient`, avec Spring 6.1, apporte une simplicité similaire à celles des Repositories pour les bases de données.
Dans cet article, je vais explorer les clients REST déclaratifs avec Spring, en les comparant à l'approche
des `Repositories`.

## Clients REST Déclaratifs avec Spring

Les clients REST déclaratifs de Spring permettent de définir des interfaces Java pour accéder aux services REST. Ces
interfaces sont annotées avec des annotations spécifiques pour décrire les appels aux services REST, telles que
`@GetExchange`, `@PostExchange`, `@PutExchange`, `@PatchExchange` et `@DeleteExchange`. Par exemple :

```Java
public interface UserRestClient {

  @GetExchange()
  List<UserRestClientEntity> findAll();

  @GetExchange("/{id}")
  Optional<UserRestClientEntity> findById(@PathVariable Long id);

  @PostExchange("/")
  void save(UserRestClientEntity entity);

  @DeleteExchange("/{id}")
  void deleteById(@PathVariable Long id);
  
  ...
}
```

Ces annotations permettent de spécifier les opérations HTTP (GET, POST, DELETE) et les chemins d'accès correspondants,
facilitant ainsi la communication avec les services REST dans une architecture de microservices.

## Configuration de l'API

Pour configurer les clients REST déclaratifs dans Spring, on peut définir une classe de configuration comme suit :

```Java
@Configuration
public class RestClientConfiguration {

  @Bean
  public RestClient restClient(RestClient.Builder builder) {
    return builder
        .requestFactory(new JdkClientHttpRequestFactory())
        .baseUrl("https://jsonplaceholder.typicode.com/users")
        .build();
  }

  @Bean
  public UserRestClient userRestClient(RestClient restClient) {
    return HttpServiceProxyFactory.builder()
        .exchangeAdapter(RestClientAdapter.create(restClient))
        .build()
        .createClient(UserRestClient.class);
  }
}
```

### Explication

- Définition du `RestClient` : pour cet exemple, j'utilise comme base l'API
  JSONPlaceholder ([https://jsonplaceholder.typicode.com/users](https://jsonplaceholder.typicode.com/users)). Notez que
  l'utilisation de la factory `JdkClientHttpRequestFactory` me permet d'utiliser le nouveau client HTTP introduit en
  Java 11.

- Création du client REST déclaratif `UserRestClient` : Le bean `userRestClient` permet d'indiquer à Spring comment
  créer une instance de `UserRestClient`. Cela se fait ici en utilisant la classe `HttpServiceProxyFactory` fournie par
  Spring.

## Utilisation du client

Dans mon domaine métier, je définis une interface pour accéder aux utilisateurs indépendamment de la source de données :

```Java
public interface EntityRepository<E extends Entity> {
  List<E> findAll();

  E findById(Long id);

  void save(E entity);

  ...
}

public interface UserRepository extends EntityRepository<User> {
   ...
}
```

Une implémentation de cette interface peut ressembler à ceci :

```Java
@Repository
@RequiredArgsConstructor
public class UserRestClientRepository implements UserRepository {
  private final UserRestClient client;

  @Override
  public List<User> findAll() {
    return convertToDomain(client.findAll());
  }

  @Override
  public User findById(Long id) {
    return client.findById(id).map(this::convertToDomain)
            .orElse(null);
  }

  @Override
  public void save(User entity) {
    client.save(new UserRestClientEntity(
            entity.getId(), entity.getName()));
  }

  private User convertToDomain(UserRestClientEntity entity) {
    return new User(entity.id(), entity.name());
  }

  private List<User> convertToDomain(
          List<UserRestClientEntity> entities) {
    return entities.stream().map(this::convertToDomain).toList();
  }
}
```

Maintenant, je peux utiliser `UserRepository` dans mes contrôleurs :

```Java
@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {

  private final UserRepository userRepository;

  @GetMapping()
  public List<User> findAll() {
    return userRepository.findAll();
  }

  @GetMapping("/{id}")
  public User findById(@PathVariable Long id) {
    return userRepository.findById(id);
  }

  // ...
}
```

## Comparaison avec les Repositories de Spring Data JPA

### Changement de la source de données

Les `Repositories` de Spring Data JPA étant également déclaratifs et basés sur des interfaces annotées. Voici un exemple
de
repository JPA pour une entité User :

```Java
public interface UserH2Dao extends JpaRepository<UserH2Entity, Long> {
  List<UserH2Entity> findByName(String name);
}
```

Comme pour les clients REST déclaratifs, l'interface repository est utilisée pour définir des méthodes d'accès aux
données sans implémentation explicite. La classe `JpaRepository` contenant déjà plusieurs méthodes communes, les autres
méthodes de requête peuvent être définies de manière déclarative en suivant les conventions de Spring Data
comme `findByName`.

### Implémentation

```Java
@Repository
@RequiredArgsConstructor
public class UserH2Repository implements UserRepository {
  private final UserH2Dao dao;

  @Override
  public List<User> findAll() {
    return dao.findAll().stream().map(this::toDomain).toList();
  }

  @Override
  public User findById(Long id) {
    return dao.findById(id).map(this::toDomain).orElse(null);
  }

  @Override
  public void save(User entity) {
    dao.save(new UserH2Entity(entity.getId(), entity.getName()));
  }

  private User toDomain(UserH2Entity entity) {
    return new User(entity.getId(), entity.getName());
  }
}
```

## Symétrie et avantages comparatifs

### Déclaration et lisibilité

Les deux approches offrent une manière déclarative de définir les interactions, que ce soit avec une base de données ou
une API REST. Cette symétrie dans la déclaration rend le code plus lisible et maintenable.

### Réduction du code boilerplate

Tant les clients REST déclaratifs que les repositories JPA réduisent la quantité de code boilerplate. Les développeurs
peuvent se concentrer sur la logique métier plutôt que sur les détails de l'implémentation des accès aux données ou aux
services.

### Cohérence dans l'écosystème Spring

L'utilisation de ces approches déclaratives dans l'écosystème Spring permet une meilleure cohérence et une uniformité
dans la manière de gérer les dépendances et les services, facilitant ainsi la montée en compétence des développeurs.

## Ressources

- [Code source de l'exemple](https://github.com/DariosDjimado/declarative-rest-client)
- [New in Spring 6.1: RestClient](https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient)
- [REST Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
