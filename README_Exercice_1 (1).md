# TP1 — Exercice 1 : GPIO et interruptions

## 1. Objectif

Cet exercice consiste à lire l'état d'un bouton-poussoir relié à une entrée numérique de l'Arduino Nano 33 IoT, puis à comparer deux méthodes de détection d'un appui :

- la **scrutation** (*polling*) : le programme interroge périodiquement la broche ;
- l'**interruption** : le microcontrôleur réagit à un événement électrique sans attendre le prochain passage dans `loop()`.

L'exercice permet aussi d'observer :

- l'effet du mode d'entrée (`INPUT` ou `INPUT_PULLUP`) ;
- l'effet du temps passé dans `loop()` (`delay`) ;
- les rebonds mécaniques du bouton.

## 2. Matériel et branchements

- Arduino Nano 33 IoT et câble micro-USB ;
- un bouton-poussoir ;
- fils de connexion.

| Bouton | Arduino Nano 33 IoT |
|--------|---------------------|
| Borne 1 | D2 |
| Borne 2 | GND |

Aucune résistance externe n'est nécessaire : avec `INPUT_PULLUP`, la résistance de rappel est interne à la carte.

> **Attention :** la carte fonctionne en logique 3,3 V. Aucune broche ne doit recevoir une tension supérieure à 3,3 V. Le câblage doit être vérifié avant de brancher l'USB.

## 3. Principe de fonctionnement

### 3.1 Entrée numérique et résistance de rappel

Une entrée numérique prend deux états logiques : `HIGH` (niveau haut) et `LOW` (niveau bas). La fonction `digitalRead()` retourne cet état.

Le bouton étant câblé entre D2 et GND, la broche n'est reliée à rien lorsque le bouton est relâché. Le mode d'entrée détermine alors ce qu'elle lit :

| Mode | Bouton relâché | Bouton appuyé |
|------|----------------|---------------|
| `INPUT` | Entrée **flottante** : niveau indéterminé, instable | `LOW` |
| `INPUT_PULLUP` | `HIGH` (stable) | `LOW` |

Avec `INPUT_PULLUP`, une **résistance de rappel interne** relie la broche à la tension logique haute (3,3 V). Quand le bouton est ouvert, la broche est donc maintenue à un niveau `HIGH` bien défini, ce qui rend le bruit électrique négligeable. Quand le bouton est fermé, la broche est reliée à GND et lit `LOW`.

Avec `INPUT` seul, la broche est flottante bouton relâché. Elle capte alors le bruit électrique ambiant (approche de la main, déplacement de la carte) et peut changer d'état sans aucun appui.

**Conséquence :** avec `INPUT_PULLUP`, un appui donne `LOW` et non `HIGH`. Un appui correspond à un **front descendant** (passage de `HIGH` à `LOW`).

### 3.2 Scrutation

Le programme lit la broche à chaque passage dans `loop()` et la compare à l'état mémorisé au passage précédent. Un appui est compté lorsque l'état passe de `HIGH` à `LOW`.

Un appui plus court que l'intervalle entre deux lectures peut passer entre deux lectures et ne jamais être vu. Plus `loop()` est occupée ou ralentie (par exemple par un `delay(200)` qui simule un programme occupé), plus le risque de manquer des appuis augmente.

### 3.3 Interruption

Avec `attachInterrupt()`, le microcontrôleur interrompt le programme dès qu'un front descendant apparaît sur D2 et exécute une fonction dédiée, la **routine de service d'interruption** (ISR). L'événement est détecté même si `loop()` est bloquée dans un `delay()`.

Une ISR doit rester la plus courte possible : elle signale l'événement, et les traitements plus longs (affichage série par exemple) sont faits dans `loop()`.

### 3.4 Rebonds mécaniques

Un bouton mécanique ne produit pas une transition parfaitement nette : à l'appui comme au relâchement, ses contacts s'ouvrent et se ferment plusieurs fois très rapidement. Ce phénomène est le **rebond** (*bouncing*).

Chaque front descendant déclenchant l'interruption, un seul appui physique peut provoquer plusieurs incrémentations. L'interruption évite de manquer des événements, mais elle ne supprime pas les rebonds.

## 4. Implémentation

### 4.1 Programme 1 : scrutation

- `BOUTON = 2` : broche D2.
- `nbAppuis` : compteur d'appuis détectés.
- `etatPrecedent` : état lu au passage précédent, initialisé à `HIGH` (bouton relâché).
- `setup()` : initialise la liaison série à 115 200 bauds, attend le moniteur série pendant 3 s au plus (`millis() < 3000`), configure D2 en entrée et affiche un message de démarrage.
- `loop()` : lit l'état courant avec `digitalRead()`. La condition `etatPrecedent == HIGH && etat == LOW` détecte le front descendant. L'état courant est ensuite mémorisé dans `etatPrecedent`.
- `delay(2)` : suspend brièvement la boucle. L'énoncé utilisait `delay(200)` pour simuler un programme occupé.

Configurations essayées pendant la séance :

| Mode d'entrée | Délai dans `loop()` | Objectif de l'essai |
|---------------|---------------------|---------------------|
| `INPUT_PULLUP` | 200 ms (énoncé) | Observer les appuis manqués |
| `INPUT` | 2 ms | Observer l'entrée flottante |
| `INPUT_PULLUP` | 2 ms | Comparer avec `INPUT` à délai identique |

### 4.2 Programme 2 : interruption

- `volatile uint32_t nbAppuis` : le mot-clé `volatile` indique au compilateur que la variable peut être modifiée à tout moment par l'ISR. Il l'empêche de supposer que sa valeur ne change pas entre deux lectures dans `loop()`.
- `surAppui()` : ISR, se limite à `nbAppuis++`.
- `attachInterrupt(digitalPinToInterrupt(BOUTON), surAppui, FALLING)` : associe la broche D2 à l'ISR et déclenche sur front descendant. `digitalPinToInterrupt()` convertit le numéro de broche en numéro d'interruption.
- `loop()` : compare `nbAppuis` à `affiche` (dernière valeur envoyée) et n'affiche que si le compteur a changé. `delay(200)` est conservé pour montrer que les appuis sont détectés malgré une boucle lente.

### 4.3 Anti-rebond (méthode décrite dans l'énoncé)

Le programme 2 ci-dessus **n'intègre pas** d'anti-rebond. Deux méthodes sont décrites dans l'énoncé.

**Méthode 1 : filtrage temporel dans l'ISR.** On mémorise l'instant du dernier front avec `millis()` et on ignore toute transition survenant moins de 50 ms après la précédente. Il faut pour cela une variable `volatile` mémorisant l'instant du dernier front et la fonction `millis()`.

**Méthode 2 : détection du front dans l'ISR, validation dans `loop()`.** Elle traite aussi les rebonds au relâchement, qui peuvent encore produire un double comptage avec la méthode 1 :

```text
déclaration :
    etatStable <- HIGH

ISR appuiDetecte :
    dernierFront <- capture du temps
    frontDetecte <- vrai

loop :
    si frontDetecte ET (temps écoulé depuis dernierFront > 20 ms)
        frontDetecte <- faux
        etat <- lecture du bouton
        si etat différent de etatStable
            etatStable <- etat
            si etat = BAS
                nbAppuis <- nbAppuis + 1
                afficher nbAppuis
```

L'ISR signale seulement qu'un front a eu lieu ; `loop()` lit la broche une fois qu'elle est stable depuis environ 20 ms et ne compte que le passage à l'état stable `LOW`.

## 5. Code source

### Programme 1 : scrutation

```cpp
#include <Arduino.h>

const int BOUTON = 2;
uint32_t nbAppuis = 0;
bool etatPrecedent = HIGH;

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}   // attend le moniteur, 3 s au plus
  pinMode(BOUTON, INPUT_PULLUP);          // essai comparatif : INPUT
  Serial.println("=== AgriSense TP1 ===");
}

void loop() {
  bool etat = digitalRead(BOUTON);

  if (etatPrecedent == HIGH && etat == LOW) {  // passage de 1 à 0 : appui
    nbAppuis++;
    Serial.print("appuis : ");
    Serial.println(nbAppuis);
  }

  etatPrecedent = etat;
  delay(2);                               // énoncé : delay(200)
}
```

### Programme 2 : interruption

```cpp
#include <Arduino.h>

const int BOUTON = 2;
volatile uint32_t nbAppuis = 0;

void surAppui() { nbAppuis++; }           // ISR : la plus courte possible

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}
  pinMode(BOUTON, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(BOUTON), surAppui, FALLING);
}

void loop() {
  static uint32_t affiche = 0;

  if (nbAppuis != affiche) {
    affiche = nbAppuis;
    Serial.print("appuis : ");
    Serial.println(affiche);
  }

  delay(200);
}
```

## 6. Résultats et observations

### 6.1 Comportements observés selon la manipulation

| Manipulation | Comportement attendu |
|--------------|----------------------|
| Appuis très brefs en scrutation avec `delay(200)` | Certains appuis ne sont pas comptés : l'appui peut survenir entre deux lectures. |
| `INPUT` à la place de `INPUT_PULLUP` | L'entrée est flottante bouton relâché : des changements d'état apparaissent sans appui. |
| Main approchée du montage, carte déplacée (en `INPUT`) | Le bruit électrique perturbe l'entrée flottante et peut produire des comptages parasites. |
| Même essai avec `INPUT_PULLUP` | Le niveau `HIGH` est maintenu par la résistance de rappel : le comportement est stable. |
| Bouton maintenu enfoncé | Un seul front descendant est détecté ; le niveau reste ensuite `LOW`. |
| Appuis répétés avec interruption | Les appuis sont détectés malgré `delay(200)`, mais les rebonds peuvent donner plusieurs incrémentations pour un seul appui. |

### 6.2 Réponses aux questions du TP

- **Appuis comptés en scrutation (dix appuis très brefs) :** moins de dix, car un appui plus court que l'intervalle entre deux lectures n'est jamais lu.
- **Pourquoi un appui donne `LOW` :** le bouton relie D2 à GND, et la résistance de rappel maintient D2 à `HIGH` quand il est relâché. L'appui impose donc `LOW`.
- **Ce que lirait `INPUT` :** un niveau indéterminé, car la broche est flottante bouton relâché.
- **Compte avec interruption :** il peut être supérieur à dix. Le phénomène est le **rebond** du bouton.
- **Fonctions et déclarations pour l'anti-rebond :** `millis()` et une variable `volatile` mémorisant l'instant du dernier front. Dans l'ISR, toute transition survenant moins de 50 ms après la précédente est ignorée.

Un double comptage peut subsister avec cette méthode à cause du rebond au relâchement. La méthode 2 (section 4.3) lit l'état une fois qu'il est stabilisé.

Les valeurs exactes dépendent du bouton, du câblage et du code téléversé.

## 7. Lien avec AgriSense

- **EX8 — Signaler tout déplacement ou basculement du boîtier :** l'exercice 1 ne détecte pas directement un déplacement. Il introduit la lecture d'événements sur une broche (GPIO) et le mécanisme d'interruption, deux briques utiles pour réagir à un événement matériel sans bloquer le programme. La détection effective du basculement est traitée avec la centrale inertielle dans l'exercice 4.
