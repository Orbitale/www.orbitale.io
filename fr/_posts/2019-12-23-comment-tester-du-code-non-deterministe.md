---
layout: post
title:  'Comment tester du code non déterministe ?'
date:   2019-12-24 12:30:01 +0200
lang: fr
lang-ref: how-to-test-non-deterministic-code
---

> **Note:** Cet article est une retranscription de l'[article du calendrier de l'avent de l'AFSY](https://afsy.fr/avent/2019/19-comment-tester-du-code-non-deterministe) publié en 2019.

# Comment tester du code non-déterministe ?

Je ne sais pas si vous avez remarqué, mais il y a certaines équipes de développement qui font des choses étranges avec leurs projets. Des choses dont tout le monde parle, mais on n'en voit pas la couleur.

Pas des fantômes, non.

Je parle des tests.

Et très souvent, quand on parle de tests, on voit ce genre de choses :

```php
class Math
{
    public function add(float $a, float $b): float
    {
        return $a + $b;
    }
}
```

```php
use PHPUnit\Framework\TestCase;

class MathTest extends TestCase
{
    public function testAdd(): void
    {
        $math = new Math();

        static::assertSame(3, $math->add(1, 2));
    }
}
```

Whoa ! 🎉

Super, nous savons tester notre code !

Et ce code a une particularité : il est **déterministe**.

## Un peu d'explications

D'après [Wiktionary](https://fr.wiktionary.org/wiki/d%C3%A9terminisme), voici la définition du déterminisme :
> Déterminisme : (Informatique) Qualité des systèmes, des processus dont l’issue ne dépend que des conditions initiales.

Ce qui veut dire que si vous connaissez les paramètres d'entrée, vous savez prévoir la sortie, quelle que soit la situation.

La plupart des choses que l'on fait sont déterministes car elles dépendent de données "fixes". Un nombre, une chaîne de caractères, une date, etc.

Mais lorsque nos algorithmes ont certains besoins, nous devrons utiliser du code non-déterministe.

C'est le cas par exemple de la gestion des données **aléatoires**.

Générer un identifiant unique dans tout l'univers, ou des données aléatoires en général, est une tâche ardue pour un ordinateur. Ce qui est encore plus ardu, c'est de prévoir le résultat.

Avec des standards comme [UUID](https://fr.wikipedia.org/wiki/Universal_Unique_Identifier), on peut connaître la **taille** et le **format** des informations en sortie, mais pas le **contenu**. Les algorithmes sont suffisamment génialement créés pour qu'il soit "quasiment impossible" d'avoir deux fois la même valeur avec deux ordinateurs différents.

Et votre code à vous peut dépendre de ce genre de situation.

## Un exemple

Je vais prendre l'exemple d'un projet que je développe depuis quelques années déjà et qui est lié au [jeu de rôle](https://fr.wikipedia.org/wiki/Jeu_de_r%C3%B4le).

Lorsque l'on crée un personnage dans un jeu de rôle, souvent, on va lancer des dés 🎲 pour déterminer le score d'une caractéristique. C'est donc une valeur **aléatoire**.

Si vous avez l'habitude des jets de dés, on peut représenter un jet de dé au format `2d6+3`, correspondant au jet de 2 dés à 6 faces, dont on ajoute 3 au résultat total. Il nous faut donc 3 paramètres en entrée (pour simplifier évidemment) : le nombre de dés, le nombre de faces pour chaque dé, et un nombre à additionner à la fin.

## Générer de l'aléatoire

Créons donc ce service qui va jeter un dé :

```php
namespace App;

class DiceRoller
{
    public function roll(int $numberOfDice = 1, int $diceSides, int $bonus = 0): int
    {
        $result = $bonus;

        for ($i = 0; $i < $numberOfDice; ++$i) {
            $result += random_int(1, $diceSides);
        }

        return $result;
    }
}
```

> Note : la classe se situe dans le _namespace_ `App`, et ce pour une bonne raison (continuez la lecture de cet article pour comprendre pourquoi).<br>
> De manière générale, dans vos projets, toutes vos classes seront dans des espaces de noms.

Nous utilisons `random_int()`, une fonction native de PHP permettant de générer un nombre entier aléatoire entre deux entiers.

Une fois fait, nous pouvons nous en servir dans nos propres services :

```php// Roll 2d6+3
$diceRoller->roll(2, 6, 3);
```

> Note : tous les nombres devraient être validés pour être des entiers **positifs**, c'est un besoin métier, mais nous n'allons pas revenir là-dessus car nous sommes dans un exemple. Notez simplement que si vous devez implémenter un tel système, il faudra impérativement valider vos variables d'entrée.

Une question subsiste cependant : **Comment tester ce code ?**

La réponse n'est pas simple, mais il existe différents cas :

* Tester directement la méthode `DiceRoller::roll()`
* Tester un service qui _dépend_ du `DiceRoller`

Étrangement, le second cas est bien plus simple à réaliser que le premier.

## Tester le code qui génère une donnée aléatoire

Pour tester le `DiceRoller`, je vais proposer progressivement trois alternatives, dans l'ordre de leur "fiabilité".

Nous partirons du principe que PHPUnit sera utilisé pour tester le code.

## Premier essai : Réaliser un nombre conséquent de tests

Cette solution se présente sous cette forme :

```php
namespace Tests\App;

use App\DiceRoller;
use PHPUnit\Framework\TestCase;

class DiceRollerTest extends TestCase
{
    /**
     * @dataProvider provide dice rolls
     */
    public function test dice roller result is in dice range(int $numberOfDice = 1, int $diceSides, int $bonus = 0): void
    {
        $diceRoller = new DiceRoller();

        $result = $diceRoller->roll($sides, $multiplier, $offset);

        static::assertGreaterThanOrEqual(1 * $multiplier + $offset, $result);
        static::assertLessThanOrEqual($sides * $multiplier + $offset, $result);
    }

    public function provide dice rolls(): \Generator
    {
        $sidesToTest = [4, 6, 8, 12, 20]; // d4, d6, etc.
        $numberOfDicesToTest = range(1, 10); // Up to 10 dices at the same time.
        $bonuses = [1, 2, 3, 4, 5]; // Not too much, that's already a lot.

        foreach ($sidesToTest as $diceSides) {
            foreach ($numberOfDicesToTest as $numberOfDice) {
                foreach ($bonuses as $bonus) {
                    yield "$diceSides-$numberOfDice-$bonus" => [$diceSides, $numberOfDice, $bonus];
                }
            }
        }
    }
}
```

Et à l'exécution, on aura quelque chose de ce style :

```
/var/www/dice_roller $ php phpunit.phar DiceRollerTest.php
PHPUnit 8.4.3 by Sebastian Bergmann and contributors.

...............................................................  63 / 250 ( 25%)
............................................................... 126 / 250 ( 50%)
............................................................... 189 / 250 ( 75%)
.............................................................   250 / 250 (100%)

Time: 130 ms, Memory: 18.00 MB

OK (250 tests, 1000 assertions)
```

Et là… On pourrait se dire _"Super ! J'ai 250 tests pour ma classe, c'est merveilleux !"_.

Ou pas.

En réalité, avec le code ci-dessus, nous avons un problème : chaque exécution de `$diceRoller->roll()` va générer un nombre aléatoire et nous n'avons **aucun moyen de prédire sa valeur**. La seule chose que nous pouvons faire (et qui est faite dans ce test) c'est prédire son **champ de valeurs possibles**. Et évidemment, vu que notre code est bien fait, cela va fonctionner sans problème.

Pour tenter de _"solutionner"_ ce problème, on peut essayer de déterminer des **solutions statistiquement stables**.

En effet, si l'on exécute une batterie de jets de dés, mettons `2d6+3`, en fonction du nombre de jets, la moyenne de résultats n'est pas toujours la même, d'une part car les nombres générés par PHP ne sont pas "réellement aléatoires", on dit qu'ils sont "pseudo-aléatoires", et d'autre part, parce que la distribution des résultats ne peut être "stable" que de façon statistique et théorique. En pratique, c'est rarement le cas (c'est le principe même du concept de _hasard_, finalement…)

Créons un petit script pour effectuer de nombreux jets de dés :

```php
$diceRoller = new App\DiceRoller();

$count = 1000000;
$results = [];

for ($i = 1; $i <= $count; $i++) {
    $results[] = $diceRoller->roll(2, 6, 3);
}

// Average value
echo array_sum($results) / $count, "\n";
```

Exécutons-le plusieurs fois, juste pour voir les différentes moyennes (avec un million de jets à chaque fois) :

```
/var/www/dice_roller $ for i in {1..10}; do php roll.php; done
11.998764
11.997870
12.003618
12.000348
11.999262
11.998068
11.993424
12.000720
12.003618
12.000378
```

Nous voyons bien que la moyenne des résultats totaux tourne toujours autour de 12, mais n'est jamais _égale_ à 12.<br>
Nous ne pouvons donc même pas faire un grand nombre de jets et calculer la moyenne... Ou alors, il faudrait le faire, mais considérer qu'avec un grand nombre de jets vient aussi une petite marge d'erreur sur la moyenne.

Nous avons donc une "solution", mais celle-ci reste approximative.

## Deuxième proposition : surcharger `random_int()`

Merci PHP ! Une fois de plus !

Grâce à PHP, il existe plusieurs façons de pouvoir surcharger une fonction native. Il existait il y a longtemps la fonction `override_function()` mais faisant partie de l'extension APD (Advanced PHP Debugger) qui est abandonnée depuis… 2004.

La meilleure façon c'est la **surcharge dans l'espace de noms**.

En effet, lorsque votre code est situé dans un espace de nom, et que vous exécutez une fonction (n'importe laquelle), PHP va d'abord vérifier si celle-ci existe dans l'espace de noms actuel, et sinon, va se replier sur l'espace de noms global.

Cette surcharge est d'ailleurs celle opérée par les classes `DnsMock` et `ClockMock` du PHPUnit Bridge de Symfony pour permettre de surcharger les fonctions natives de recherche d'enregistrements DNS ou les fonctions de date et de temps.

Voici comment procéder :

Dans votre classe de test `DiceRollerTest`, vous avez la possibilité de déclarer un espace de noms supplémentaire, quel qu'il soit.

L'espace de noms à rajouter **doit être le même que celui du `DiceRoller`**, car c'est lui qui exécute la fonction native à surcharger.

```php

namespace Tests\App;

use App\DiceRoller;
use PHPUnit\Framework\TestCase;

class DiceRollerTest extends TestCase
{
    // ...
}

namespace App;

// Function override
function random_int() { /* */ }
```

Et voilà ! Avec cette méthode, lorsque le `DiceRoller` exécutera la fonction `random_int()`, PHP cherchera d'abord à savoir si elle a été déclarée dans le namespace de celui-ci (`App` dans notre cas), et exécutera la fonction que vous avez créée !

De cette façon, vous pouvez, par exemple, exécuter une fonction d'une classe statique qui vous permettrait de définir dès le départ un résultat à avoir :

```php

namespace Tests\App;

use App\DiceRoller;
use PHPUnit\Framework\TestCase;

class DiceRollerTest extends TestCase
{
    public static int $forcedResult = 0;

    public function test(): void
    {
        $diceRoller = new DiceRoller();

        self::$forcedResult = 1;

        $result = $diceRoller->roll(2, 6, 3);

        static::assertSame(5, $result); // Yay!
    }
    // ...
}

namespace App;

use Tests\App\DiceRollerTest;

// Function override
function random_int(int $min, int $max): int {
    return DiceRollerTest::$forcedResult;
}
```

Cette solution fonctionne bien, **mais elle a un inconvénient** : si un jour le code du `DiceRoller` change et que l'appel à `random_int()` est fait sous la forme `\random_int()` (ou une instruction `use function random_int;` est rajoutée en haut du fichier), c'est fini !<br>
En effet, cette syntaxe va forcer PHP à utiliser uniquement la fonction native, et vous ne pourrez plus jamais surcharger `random_int()`…

Mais ne vous en faites pas, j'ai la solution !

## Troisième solution : l'ultime solution !

La notion "d'aléatoire" comme vous l'avez vu plus haut (et si vous connaissez les problématiques liées au concept même "d'aléatoire dans l'informatique") est assez particulière. C'est comme la récupération de la date, ou du cours de la bourse : ça n'est pas 100% prédictible sans avoir une sorte de "système externe".

Les générateurs aléatoires utilisent tout un tas de techniques, des sortes de _hacks_, pour vous permettre d'obtenir un nombre qui _semble_ aléatoire.

En réalité, **un générateur de nombres aléatoires est un service tiers**.

Vous me voyez venir ?

Et oui : le `DiceRoller` peut très bien fonctionner **sans `random_int()`** ! Par contre il ne peut pas fonctionner sans **générateur de nombres aléatoires**.

Nous allons donc commencer par créer une interface pour représenter notre besoin, qui est assez simple pour notre problématique :

```php
namespace App;

interface RandomIntProviderInterface
{
    public function randomInt(int $min, int $max): int;
}
```

Il nous faudra évidemment changer le code de notre `DiceRoller` :

```php
namespace App;

class DiceRoller
{
    private RandomIntProviderInterface $randomIntProvider;

    public function __construct(RandomIntProviderInterface $randomIntProvider)
    {
        $this->randomIntProvider = $randomIntProvider;
    }

    public function roll(int $numberOfDice = 1, int $diceSides, int $bonus = 0): int
    {
        $result = $bonus;

        for ($i = 0; $i < $numberOfDice; ++$i) {
            $result += $this->randomIntProvider->randomInt(1, $diceSides);
        }

        return $result;
    }
}
```

(Notez avec quelle subtilité j'ai utilisé une propriété typée, merci PHP 7.4 !)

Excellent !

Il n'y a plus qu'à créer deux classes, l'une pour l'application dans son "comportement normal" :

```php
class NativeRandomIntProvider implements RandomIntProviderInterface
{
    public function randomInt(int $min, int $max): int
    {
        return \random_int($min, $max);
    }
}
```

Cette classe sera injectée dans le constructeur du `DiceRoller` avec votre système d'Injection de Dépendances préféré (à tout hasard, celui de Symfony).

Ensuite, dans le cadre de nos tests, nous allons créer une autre implémentation :

```php
class DeterministicRandomIntProvider implements RandomIntProviderInterface
{
    public int $determinedResult = 0;

    public function randomInt(int $min, int $max): int
    {
        return $this->determinedResult;
    }
}
```

Parfait !

Voici donc à quoi pourra ressembler notre test pour la classe `DiceRoller` :

```php
class DiceRollerTest extends TestCase
{
    public function test dice roller result is in dice range(int $sides, int $multiplier, int $offset): void
    {
        $randomIntProvider = new DeterministicRandomIntProvider();

        $diceRoller = new DiceRoller($randomIntProvider);

        $randomIntProvider->determinedResult = 1;

        $result = $diceRoller->roll(2, 6, 3); // 2d6+3

        static::assertSame(5, $result); // Yay!
    }
}
```

Parfait ! Nous pouvons désormais forcer le fournisseur de nombre "aléatoire" à renvoyer un nombre précis, et de cette façon, nous avons un contrôle total sur notre architecture pour pouvoir la tester !

## Conclusion

La génération de données non-déterministes (dates, nombres aléatoires, identifiants uniques, clés secrètes…) est un vrai challenge pour les personnes qui développent ces outils.

Ne pouvant avoir le contrôle sur ces systèmes très avancés utilisant parfois de la cryptographie très poussée ou carrément des données complètement farfelues comme les fluctuations du climat, de la [cryptographie quantique](https://fr.wikipedia.org/wiki/Cryptographie_quantique) ou même des [lampes à lave](https://www.zdnet.com/article/how-lava-lamps-are-used-to-encrypt-the-internet/), nous devons souvent (toujours ?) gérer une multitude de résultats possibles par nous-mêmes.

Partir du principe qu'une donnée non-déterministe est une **donnée venant d'un service tiers** nous permet de mieux structurer notre code et de le rendre plus flexible mais également adaptable à une version déterministe plutôt qu'aléatoire de cette information tierce.

Il existe par exemple la bibliothèque [`nesbot/carbon`](https://github.com/briannesbitt/Carbon), permettant de considérer la date et le temps comme des données venant d'un service externe, et ce service nous permet donc de "falsifier" cette date, pour nos besoins personnels (comparaison de date, passage du temps "fictif" durant un seul et unique test, etc.).
