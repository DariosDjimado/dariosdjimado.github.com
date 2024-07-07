# Angular takeUntilDestroyed : éviter les fuites mémoires

Introduit en Angular 16, l'opérateur `takeUntilDestroyed` est un moyen plus intégré pour gérer les abonnements aux
observables et éviter les fuites mémoires. Cependant, une attention particulière doit être portée à l'ordre des
opérateurs dans les `pipe` pour que `takeUntilDestroyed` fonctionne correctement.

## Fonctionnement

Soit l'exemple suivant avec un composant `QuoteComponent` :

```Typescript
export class QuoteComponent implements OnInit {
  private readonly destroyRef = inject(DestroyRef);
  private readonly tagsService = inject(TagsService);

  public ngOnInit(): void {
    this.tagsService.userSelectedTag$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((tag) => {
        console.log('selected tag', tag);
      });
  }
}
```

Dans cet exemple, `takeUntilDestroyed` est utilisé pour se désabonner de l'observable `userSelectedTag$` lorsque le
composant est détruit. L'opérateur `takeUntilDestroyed` prend un paramètre de type `DestroyRef` qui permet de déclencher
le désabonnement. Ce paramètre est facultatif dans les cas où l'abonnement se fait dans des contextes dits
"d'injection". C'est le cas par exemple dans le constructeur ou dans les champs d'un composant (constructeurs
indirects).

## Ordre des opérateurs dans les `pipe`

Prenons l'exemple suivant :

```Typescript
export class QuoteComponent implements OnInit {
  private readonly destroyRef = inject(DestroyRef);
  private readonly quoteService = inject(QuoteService);
  private readonly tagsService = inject(TagsService);

  public readonly currentQuote = signal<QuoteData | null>(null);

  public ngOnInit(): void {
    this.tagsService.userSelectedTag$
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        switchMap((tag) => this.quoteService.startUpdatingQuotes(tag))
      )
      .subscribe((response) => {
        console.log(response.step, response.data.content);
        this.currentQuote.set(response);
      });
  }
}
```

Ici, nous utilisons `switchMap` pour déclencher une nouvelle requête à chaque fois qu'un tag est
sélectionné. En utilisant `takeUntilDestroyed` dans le `pipe`, nous nous assurons du désabonnement de l'observable
`userSelectedTag$` lorsque le composant est détruit. Malheureusement, nous allons voir sur l'image ci-dessous que
cela fonctionne partiellement.

![Exemple avec fuite mémoire](takeuntildestroyed-before-switch-operator.gif)

Comme on peut le voir, lorsque le composant est détruit, on se désabonne bien de l'observable `userSelectedTag$`, mais
l'observable `quoteService.startUpdatingQuotes(tag)` continue de tourner. Cela est dû à l'ordre des opérateurs dans le
`pipe`. En effet, `takeUntilDestroyed` se désabonne de l'observable `userSelectedTag$` mais pas de l'observable créé par
`switchMap`.

Pour résoudre ce problème, il suffit de déplacer `takeUntilDestroyed` après `switchMap` dans le `pipe` :

```Typescript
export class QuoteComponent implements OnInit {
  ...

  public ngOnInit(): void {
    this.tagsService.userSelectedTag$
      .pipe(
       switchMap((tag) => this.quoteService.startUpdatingQuotes(tag)),
       takeUntilDestroyed(this.destroyRef),
      )
      .subscribe((response) => {
        ...
      });
  }
}
```

Maintenant, lorsque le composant est détruit, l'observable créé par `switchMap` est bien désabonné.

![Exemple sans fuite mémoire](takeuntildestroyed-after-switch-operator.gif)

## Conclusion

`takeUntilDestroyed` est un opérateur très utile pour éviter les fuites mémoires dans une application Angular.
Cependant, il est important de faire attention à l'ordre des opérateurs quand on l'utilise avec d'autres opérateurs
RxJS. En particulier, il est recommandé de placer `takeUntilDestroyed` après les opérateurs qui créent de nouveaux
observables comme `switchMap` ou `mergeMap`.

## Ressources

- [Code source de l'exemple](https://github.com/DariosDjimado/angular-takeuntildestroyed)
- [TakeUntilDestroyed](https://angular.dev/api/core/rxjs-interop/takeUntilDestroyed)
- [Injection context](https://angular.dev/guide/di/dependency-injection-context)