# Projections dynamiques avec Spring Data JPA

Temps de réponse excessifs pour des requêtes simples, même lorsqu'il s'agit de récupérer une ou deux colonnes dans
une table de base de données, voila un problème que vous pouvez rencontrer dans les applications Spring Boot utilisant
JPA/Hibernate.

Quelles peuvent être les causes de ces problèmes de performances, et quelles solutions peuvent être envisagées pour
les améliorer ? Dans cet article, nous allons examiner comment les projections dynamiques peuvent constituer une
alternative parmi d'autres pour optimiser les performances des applications Spring Boot utilisant JPA/Hibernate.

## Utilisation traditionnelle de JPA/Hibernate dans une application Spring boot

Premièrement, nous allons examiner comment les entités sont généralement utilisées dans une application Spring Boot
avec JPA/Hibernate. Les entités sont des classes Java qui sont mappées sur des tables de base de données. Elles
contiennent des champs qui correspondent aux colonnes de la table, et des méthodes qui permettent de manipuler les
données de la table.

### Entité de base

Dans cet exemple, nous avons une entité `CropEntity` qui est la classe de base pour les entités spécialisées
`CerealEntity`, `FruitEntity` et `VegetableEntity`. La classe de base contient des champs communs à toutes les
entités, tandis que les classes spécialisées contiennent des champs spécifiques à chaque type de culture.

```Java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public class CropEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;
  private String color;
  private String size;
  private String origin;
  private String description;
  ...
}
```

### Entités spécialisées

```Java
@Entity
public class CerealEntity extends CropEntity {

  private String grainType;
  ...
}

@Entity
public class FruitEntity extends CropEntity {

  private String variety;
  ...
}

@Entity
public class VegetableEntity extends CropEntity {

  private String harvestSeason;
  ...
}
```

### Repository et contrôleur

```Java
@Repository
public interface CropRepository extends 
        JpaRepository<CropEntity, Long> {}
```

```Java
@RestController
@RequestMapping("/crops")
@RequiredArgsConstructor
public class CropController {
  private final CropRepository cropRepository;

  @GetMapping
  public List<CropNameOutput> getAll() {
    return cropRepository.findAll().stream()
        .map(cropEntity -> new CropNameOutput(
                cropEntity.getId(),
                cropEntity.getName()))
        .toList();
  }
}
```

### Résultat de la requête

Voici le résultat de la requête `GET /crops` :

```json lines
[
  {
    "id": 1,
    "name": "Apple"
  },
  {
    "id": 2,
    "name": "Banana"
  },
  {
    "id": 3,
    "name": "Pineapple"
  },
  ...
]
```

Comme attendu, la requête `GET /crops` renvoie une liste de cultures avec leur nom.

## Problèmes de performances

Maintenant, analysons la requête SQL générée par Hibernate pour récupérer les données de la base de données.
Pour cela, nous allons activer les logs SQL dans le fichier `application.yml` :

```yaml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true
        format_sql: true
```

Voici la requête SQL générée par Hibernate pour récupérer les données :

```Sql
select
     ce1_0.id,
     case 
         when ce1_1.id is not null 
             then 1 
         when ce1_2.id is not null 
             then 2 
         when ce1_3.id is not null 
             then 3 
         when ce1_0.id is not null 
             then 0 
         end,
     ce1_0.color,
     ce1_0.description,
     ce1_0.name,
     ce1_0.origin,
     ce1_0.size,
     ce1_1.grain_type,
     ce1_2.variety,
     ce1_3.harvest_season 
     from
         crop_entity ce1_0 
     left join
         cereal_entity ce1_1 
             on ce1_0.id=ce1_1.id 
     left join
         fruit_entity ce1_2 
             on ce1_0.id=ce1_2.id 
     left join
         vegetable_entity ce1_3 
             on ce1_0.id=ce1_3.id
```

Trois jointures et de nombreux champs sont récupérés, alors que nous n'avons besoin que de l'id et du nom de la culture.

### 1. Taille des données récupérées

Les entités peuvent contenir de nombreux champs, dont certains ne sont pas nécessaires pour une requête particulière.
Cela signifie que nous récupérons plus de données qu'il n'en faut, ce qui peut ralentir les performances de
l'application. Surtout lorsque les entités contiennent des champs volumineux tels que de longues descriptions.

### 2. Complexité des entités

Les entités peuvent être complexes, avec des relations entre elles et des champs imbriqués. Cela peut rendre les
requêtes plus complexes et impacter significativement les performances de la base de données.

### 3. Requêtes SQL multiples

Lorsque vous utilisez une entité dans une requête, Hibernate génère une requête SQL pour chaque entité. Cela peut
entraîner un grand nombre de requêtes SQL, ce qui peut ralentir les performances de l'application.

## Projections dynamiques

### Fonctionnement

Les projections dynamiques permettent de récupérer uniquement les données nécessaires à partir de la base de données,
plutôt que de charger toutes les données de l'entité. Cela permet de réduire le nombre de requêtes SQL générées et
la taille des données récupérées, ce qui améliore les performances de l'application.

### Exemple

Il existe plusieurs façons de faire des projections dynamiques avec Spring Data JPA. L'une d'entre elles consiste à
utiliser des interfaces de projection pour définir les champs que nous souhaitons récupérer.

### Interface de projection

```Java
public interface CropName {
  String getId();

  String getName();
}
```

### Repository avec projection dynamique

```Java
@Repository
public interface CropRepository extends 
        JpaRepository<CropEntity, Long> {
  <T> List<T> findBy(Class<T> type);
}
```

### Contrôleur

```Java
@RestController
@RequestMapping("/crops")
@RequiredArgsConstructor
public class CropController {
  private final CropRepository cropRepository;

  @GetMapping("/names-only")
  public List<CropNameOutput> getNamesOnly() {
    return cropRepository.findBy(CropNameOutput.class);
  }
}
```

## Alternatives avancées : Querydsl, JOOQ et GraphQL

Outre les projections dynamiques, des alternatives avancées comme Querydsl, JOOQ et GraphQL offrent des fonctionnalités
intéressantes pour optimiser davantage les requêtes SQL impliquées dans la communication avec la base de données.

- [Querydsl](http://querydsl.com/) permet de créer des requêtes de manière programmatique et type-safe en utilisant des
  classes Java générées à partir du modèle de données de la base de données. Cela permet de réduire les erreurs de
  syntaxe et d'optimiser les requêtes SQL.

- [JOOQ](https://www.jooq.org/) offre une approche basée sur SQL pour la construction de requêtes, ce qui permet un
  contrôle fin sur la génération de requêtes SQL et une meilleure intégration avec la base de données.

- [GraphQL](https://graphql.org/) fournit un langage de requête flexible qui permet aux clients de demander exactement
  les données dont ils ont besoin, ce qui peut réduire la surcharge réseau et améliorer les performances pour les
  applications avec des besoins complexes de récupération de données.

## Conclusion

Les projections dynamiques sont une solution de premier plan pour améliorer les performances des applications Spring
Boot utilisant JPA/Hibernate. En réduisant le nombre de requêtes SQL générées et la taille des données récupérées, les
projections dynamiques permettent d'optimiser les performances de l'application et d'offrir une meilleure expérience
utilisateur.

## Ressources

- [Spring Data JPA Projections](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#projections)
- [Spring Data JPA](https://spring.io/projects/spring-data-jpa)
