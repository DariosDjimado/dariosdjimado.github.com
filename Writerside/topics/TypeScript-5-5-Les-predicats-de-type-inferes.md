# TypeScript 5.5 : Les prédicats de type inférés

Avec TypeScript 5.5, l'inférence de type a connu une amélioration significative. Les prédicats de type inférés
permettent de déduire le type d'une variable en fonction de la condition qui la précède.

## Avant TypeScript 5.5

Avant TypeScript 5.5, il était nécessaire de forcer le typage dans certaines conditions pour lui faire comprendre
l'évolution du type de certaines variables.

Prenons un exemple simple :

```typescript
interface Apple {
    type: 'apple';
    variety: string;
    sweetness: number;
}

interface Orange {
    type: 'orange';
    variety: string;
    sourness: number;
}

type Fruit = Apple | Orange;

const fruits: Fruit[] = [
    {type: 'orange', variety: 'Clementine', sourness: 2},
    {type: 'orange', variety: 'Blood', sourness: 7},
    {type: 'orange', variety: 'Mandarin', sourness: 4},
    {type: 'apple', variety: 'Fuji', sweetness: 10},
    {type: 'apple', variety: 'Honeycrisp', sweetness: 9},
    {type: 'apple', variety: 'Granny Smith', sweetness: 6},
];

fruits.filter(fruit => fruit.type === 'apple').forEach(apple => {
    console.log(apple.sweetness); // Error 
});
```

Dans cet exemple, TypeScript ne comprend pas que `apple` est un fruit de type `Apple` et affiche une erreur.

![Erreur d'inférence de type](inferred-type-error.png)

En réalité, TypeScript déduisait que le résultat de la méthode `filter` était un tableau de `Fruit` et non de `Apple`.

La solution de contournement consistait à passer par les prédicats de type pour forcer TypeScript à comprendre le type
de la variable.

```typescript
function isApple(fruit: Fruit): fruit is Apple {
    return fruit.type === 'apple';
}

fruits.filter(isApple).forEach(apple => {
    console.log(apple.sweetness) // No error
});
```

💡Bien que cette solution soit fonctionnelle, elle est un peu trop verbeuse et nécessite de rajouter une fonction
supplémentaire. Oui, c'est bien de donner des noms sympas aux fonctions, mais là, c'est un peu comme essayer de faire du
yoga 🧘‍♂️ avec du TypeScript 💻.

## TypeScript 5.5

À partir de la version 5.5, TypeScript peut inférer le type de la variable `apple` sans avoir besoin de passer par un
prédicat de type explicite.

```typescript
fruits.filter(fruit => fruit.type === 'apple').forEach(apple => {
    console.log(apple.sweetness); // No error
});
```

## Conditions d'inférence de type

Quelques conditions doivent être remplies pour que TypeScript puisse inférer les types :

1. La fonction n'a pas de type de retour explicite ou d'annotation de prédicat de type.
2. La fonction contient une seule instruction return sans retours implicites.
3. La fonction ne modifie pas ses paramètres.
4. La fonction retourne une expression booléenne liée au raffinement du paramètre.

## Conclusion

Les prédicats de type inférés sont une amélioration significative de TypeScript 5.5 qui simplifie la gestion des types
dans les conditions. Cela permet de réduire la verbosité du code et d'améliorer sa lisibilité.

## Ressources

- [Code source de l'exemple](https://github.com/DariosDjimado/inferred-type-predicates)
- [Announcing TypeScript 5.5](https://devblogs.microsoft.com/typescript/announcing-typescript-5-5/)

