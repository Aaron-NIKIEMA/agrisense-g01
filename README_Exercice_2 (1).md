# TP1 — Exercice 2 : ADC, calibration et qualification des mesures

## 1. Objectif

Cet exercice porte sur l'acquisition d'une grandeur analogique avec l'Arduino Nano 33 IoT. Une sonde capacitive d'humidité du sol (ou, pour les essais, un potentiomètre) fournit une tension que le convertisseur analogique-numérique (ADC) transforme en valeur entière.

Le travail consiste à :

- lire la valeur brute de l'ADC et comprendre l'effet de sa résolution ;
- calibrer la mesure à partir de deux références expérimentales ;
- convertir la valeur brute en pourcentage d'humidité ;
- filtrer les variations parasites (médiane et moyenne) ;
- détecter et signaler les mesures invalides.

## 2. Matériel et branchements

- Arduino Nano 33 IoT et câble micro-USB ;
- sonde capacitive d'humidité du sol, ou potentiomètre ;
- fils de connexion ;
- verre d'eau pour le point de référence humide.

### 2.1 Branchement de la sonde

| Sonde | Arduino Nano 33 IoT |
|-------|---------------------|
| VCC | 3V3 |
| GND | GND |
| AOUT | A0 |

Dans l'eau, seule la partie sensible est immergée, jusqu'au repère. L'électronique ne doit jamais être plongée.

### 2.2 Branchement du potentiomètre (pour remplacer la sonde)

Un potentiomètre possède trois bornes : deux extrémités et un curseur (borne centrale).

| Borne du potentiomètre | Arduino Nano 33 IoT |
|------------------------|---------------------|
| Une extrémité | 3V3 |
| L'autre extrémité | GND |
| Curseur (borne centrale) | A0 |

Les deux extrémités jouent le même rôle : l'inversion de 3V3 et GND ne change pas le principe du montage, elle inverse seulement le sens de rotation (la valeur lue augmente alors dans l'autre sens). Le curseur doit être sur A0 : c'est lui qui délivre la tension variable entre 0 V et 3,3 V.

Avec un potentiomètre, la conversion en pourcentage d'humidité n'a pas de sens physique. Il sert seulement à faire varier progressivement la tension pour tester la chaîne de traitement.

> **Attention :** la carte fonctionne en 3,3 V. Une tension supérieure sur A0 peut détruire la carte. Le câblage doit être vérifié avant de brancher l'USB.

## 3. Principe de fonctionnement

### 3.1 Conversion analogique-numérique et pas de quantification

`analogRead(A0)` retourne un entier proportionnel à la tension sur A0. Sa plage dépend de la résolution configurée avec `analogReadResolution()` :

| Résolution | Niveaux | Plage | Pas (3,3 V / niveaux) |
|------------|---------|-------|-----------------------|
| 10 bits (défaut de la plateforme) | 1024 | 0 à 1023 | ≈ 3,22 mV |
| 12 bits | 4096 | 0 à 4095 | ≈ 0,806 mV |

Une résolution plus élevée permet de distinguer des variations de tension plus petites. Elle ne garantit pas à elle seule une meilleure précision : le bruit, la qualité du capteur et la référence de tension interviennent aussi.

### 3.2 Calibration

La valeur brute n'est pas un pourcentage d'humidité : elle dépend de la sonde. Deux points de référence sont relevés :

| Référence | Conditions | Valeur ADC (12 bits) |
|-----------|------------|----------------------|
| Point sec | Sonde à l'air libre | 2755 |
| Point humide | Sonde dans l'eau jusqu'au repère | 1250 |

La sonde étant capacitive, la valeur brute **diminue** quand l'humidité augmente. Une interpolation linéaire entre les deux points donne le pourcentage :

```text
H (%) = (V_sec − V_brut) / (V_sec − V_humide) × 100
      = (2755 − V_brut) / (2755 − 1250) × 100
```

Le résultat est borné entre 0 et 100 %.

Les valeurs relevées par différents binômes ne sont pas identiques d'une sonde à l'autre. Pour un parc de 200 capteurs, il faut donc calibrer chaque capteur (ou au minimum chaque lot, avec vérification) : des constantes uniques fausseraient le pourcentage de chaque sonde.

### 3.3 Filtrage

Le fil de signal peut capter des perturbations qui produisent des valeurs isolées très éloignées des autres. Deux filtres sur 9 échantillons sont comparés :

- **Moyenne** : somme des 9 valeurs divisée par 9. Elle atténue le bruit réparti, mais une valeur extrême déplace directement le résultat.
- **Médiane** : les 9 valeurs sont triées et la valeur centrale (indice 4) est retenue. Elle est peu sensible à une valeur isolée.

Exemple avec un pic isolé :

```text
Mesures : 1500, 1510, 1490, 1505, 1515, 1495, 1500, 1510, 3000
Moyenne : 15025 / 9 ≈ 1669   (déplacée d'environ 160 par le pic)
Médiane : 1505               (valeurs triées : 1490 1495 1500 1500 1505 1510 1510 1515 3000)
```

**Conclusion :** le filtre médian résiste mieux à un pic isolé. La moyenne reste adaptée à un bruit modéré et réparti, sans valeur aberrante.

### 3.4 Qualification de la mesure

Produire une valeur ne signifie pas que la mesure est exploitable : un capteur débranché ou une acquisition perturbée peut fournir une valeur incohérente. L'indicateur de qualité `q` vaut :

- `q = 0` : mesure valide ;
- `q = 1` : mesure invalide.

`q = 1` si l'une des deux conditions est vraie :

1. la valeur filtrée est hors de la plage calibrée `[VAL_MIN ; VAL_MAX]` ;
2. les 9 échantillons sont trop dispersés : `D = x_max − x_min > ECART_MAX`.

Seuils utilisés :

| Seuil | Valeur | Justification |
|-------|--------|---------------|
| `VAL_MIN` | 1250 | Correspond au point humide relevé (sonde dans l'eau) : une valeur plus basse sort de la plage calibrée. |
| `VAL_MAX` | 3750 | Borne haute large, au-dessus du point sec (2755) avec une marge d'environ 1000 unités pour tolérer les écarts de sonde. |
| `ECART_MAX` | 500 | Tolérance supérieure à la dispersion normale d'une rafale de 9 mesures et inférieure à celle provoquée par une perturbation (par exemple toucher le fil de signal). |

Ces seuils sont des paramètres expérimentaux, à justifier à partir des mesures observées. Ils ne sont pas universels et doivent être revus si la sonde ou la calibration change.

Quand le fil de signal est débranché, carte alimentée, l'entrée A0 devient flottante : la valeur lue est instable et ne correspond à aucune mesure du sol. Le contrôle de dispersion et le contrôle de plage détectent ce cas dans la plupart des situations. Un signal flottant peut cependant produire une rafale qui passe les deux contrôles : `q` est un indicateur simple, pas une preuve de bon fonctionnement du capteur.

## 4. Implémentation

### 4.1 Lecture brute (programme 1)

`analogReadResolution(12)` fixe la résolution à 12 bits (0 à 4095). La boucle affiche `analogRead(A0)` toutes les secondes. Sans cette instruction, la plage retombe à la résolution par défaut (10 bits, 0 à 1023).

### 4.2 Conversion en pourcentage (programme 2)

- `SOL_SEC` et `SOL_HUMIDE` : valeurs de calibration relevées.
- `humiditeSol(int brut)` : applique l'interpolation linéaire, en `float`, puis `constrain(pct, 0.0f, 100.0f)` borne le résultat entre 0 et 100.
- `loop()` : lit `analogRead(A0)` **une seule fois** dans `brut`, puis réutilise cette valeur pour l'affichage et pour le calcul. Les deux valeurs affichées correspondent ainsi au même échantillon.

### 4.3 Filtre médian (programme 3)

- `mediane9(int* v)` : trie le tableau de 9 valeurs par insertion (chaque élément est inséré à sa place parmi les éléments déjà triés) et retourne `v[4]`, la valeur centrale.
- `loop()` : acquiert 9 mesures espacées de 10 ms, appelle `mediane9()` puis affiche le résultat.

`mediane9()` trie le tableau passé en argument : après l'appel, `mesures[8]` est **la plus grande valeur** et non la dernière mesure acquise. Le libellé « Brut (derniere) » de l'affichage ne correspond donc pas exactement à ce qui est affiché.

### 4.4 Indicateur de qualité (programme 4)

- Constantes `VAL_MIN`, `VAL_MAX`, `ECART_MAX` : seuils de qualification.
- Acquisition de 9 mesures espacées de 5 ms, puis `mediane9()`.
- Le tableau étant trié, `mesures[0]` est le minimum et `mesures[8]` le maximum de la rafale : `dispersion = mesures[8] − mesures[0]`.
- `q` est mis à 1 si la valeur filtrée sort de `[VAL_MIN ; VAL_MAX]`, ou sinon si la dispersion dépasse `ECART_MAX`.
- L'affichage série donne la valeur filtrée, la dispersion et `q`.

## 5. Code source

### Programme 1 : lecture brute en 12 bits

```cpp
#include <Arduino.h>

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}
  analogReadResolution(12);
}

void loop() {
  Serial.print("ADC brut : ");
  Serial.println(analogRead(A0));
  delay(1000);
}
```

### Programme 2 : conversion en pourcentage d'humidité

```cpp
#include <Arduino.h>

const int SOL_SEC = 2755;     // relevé à l'air libre
const int SOL_HUMIDE = 1250;  // relevé dans l'eau

float humiditeSol(int brut) {
  float pct = (float)(SOL_SEC - brut) * 100.0f
              / (float)(SOL_SEC - SOL_HUMIDE);
  return constrain(pct, 0.0f, 100.0f);
}

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}
  analogReadResolution(12);
}

void loop() {
  int brut = analogRead(A0);

  Serial.print("ADC brut : ");
  Serial.print(brut);
  Serial.print(" - humiditeSol : ");
  Serial.println(humiditeSol(brut));

  delay(1000);
}
```

### Programme 3 : filtre médian sur 9 échantillons

```cpp
#include <Arduino.h>

int mediane9(int* v) {  // trie le tableau, renvoie la valeur du milieu
  for (int i = 1; i < 9; i++) {
    int x = v[i], j = i - 1;
    while (j >= 0 && v[j] > x) { v[j + 1] = v[j]; j--; }
    v[j + 1] = x;
  }
  return v[4];
}

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}
  analogReadResolution(12);
}

void loop() {
  int mesures[9];

  for (int i = 0; i < 9; i++) {
    mesures[i] = analogRead(A0);
    delay(10);  // court délai entre les lectures
  }

  int valeur_filtree = mediane9(mesures);

  Serial.print("Brut (derniere): ");
  Serial.print(mesures[8]);   // après le tri : plus grande valeur
  Serial.print(" - Mediane9: ");
  Serial.println(valeur_filtree);

  delay(1000);
}
```

### Programme 4 : indicateur de qualité `q`

```cpp
#include <Arduino.h>

// Seuils de qualité (à adapter à la sonde)
const int VAL_MIN = 1250;    // valeur minimale calibrée
const int VAL_MAX = 3750;    // valeur maximale calibrée (max 4095 en 12 bits)
const int ECART_MAX = 500;   // écart max toléré entre le min et le max de la rafale

int mediane9(int* v) {  // trie le tableau, renvoie la valeur du milieu
  for (int i = 1; i < 9; i++) {
    int x = v[i], j = i - 1;
    while (j >= 0 && v[j] > x) { v[j + 1] = v[j]; j--; }
    v[j + 1] = x;
  }
  return v[4];
}

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}
  analogReadResolution(12);  // 0 à 4095
}

void loop() {
  int mesures[9];

  // 1. Collecte des 9 mesures
  for (int i = 0; i < 9; i++) {
    mesures[i] = analogRead(A0);
    delay(5);
  }

  // 2. Filtre médian (trie aussi le tableau)
  int valeur_filtree = mediane9(mesures);

  // 3. Indicateur de qualité q
  int q = 0;
  int dispersion = mesures[8] - mesures[0];  // tableau trié : max - min

  if (valeur_filtree < VAL_MIN || valeur_filtree > VAL_MAX) {
    q = 1;  // hors de la plage calibrée
  }
  else if (dispersion > ECART_MAX) {
    q = 1;  // échantillons trop dispersés
  }

  // 4. Affichage
  Serial.print("Valeur: ");
  Serial.print(valeur_filtree);
  Serial.print(" | Dispersion: ");
  Serial.print(dispersion);
  Serial.print(" | Qualite q: ");
  Serial.println(q);

  delay(1000);
}
```

## 6. Résultats et observations

| Essai | Résultat attendu |
|-------|------------------|
| Lecture brute en 12 bits | Valeurs entre 0 et 4095. |
| Sans `analogReadResolution(12)` | Valeurs entre 0 et 1023 (résolution par défaut de 10 bits) ; pas de quantification ≈ 3,22 mV au lieu de ≈ 0,806 mV. |
| Sonde à l'air libre | Valeur proche de 2755, soit environ 0 % d'humidité. |
| Sonde dans l'eau (jusqu'au repère) | Valeur proche de 1250, soit environ 100 % d'humidité. |
| Valeur brute hors des deux références | Pourcentage borné à 0 ou 100 par `constrain()`. |
| Fil de signal touché (pic isolé) | La médiane reste proche de la valeur habituelle ; la moyenne est déplacée par le pic. |
| Fil de signal débranché, carte alimentée | Entrée flottante : valeurs instables et dispersées, généralement signalées par `q = 1`. |

Les valeurs de calibration (2755 et 1250) sont propres à la sonde utilisée. Elles varient d'une sonde à l'autre, d'où la nécessité de calibrer chaque capteur.

## 7. Lien avec AgriSense

- **EX1 — Mesurer l'humidité du sol et la température au moins toutes les 15 min :** l'exercice met en place l'acquisition, la calibration et la conversion de l'humidité du sol. Les programmes lisent la sonde toutes les secondes ou par rafale de 9 mesures : ils démontrent la mesure d'humidité, mais ne mettent pas en œuvre à eux seuls une périodicité de 15 minutes ni la mesure de température.
- **EX2 — Détecter et signaler une mesure invalide :** l'indicateur `q` signale une mesure hors de la plage calibrée ou une rafale trop dispersée. Le résultat est affiché sur le moniteur série.

L'exercice constitue la chaîne de traitement d'une mesure analogique : acquisition, calibration, filtrage et qualification.
