# TP1 — Exercice 4 : Datasheet et bus I2C

## 1. Objectif

Cet exercice vise à découvrir la centrale inertielle **LSM6DS3** intégrée à l'Arduino Nano 33 IoT et à comprendre comment un microcontrôleur communique avec un composant par le bus **I2C**.

Les manipulations portent sur :

- la lecture de la datasheet du LSM6DS3 (alimentation, adresses I2C, plages de mesure) ;
- la détection des périphériques présents sur le bus (scanner I2C) ;
- la lecture des accélérations avec la bibliothèque `Arduino_LSM6DS3` ;
- le calcul de l'inclinaison de la carte à partir des accélérations (question bonus) ;
- une lecture complémentaire des registres du composant par accès I2C direct avec `Wire`.

## 2. Matériel et branchements

Aucun câblage n'est nécessaire : le LSM6DS3 est soudé sur la carte Arduino Nano 33 IoT et relié au microcontrôleur par le bus I2C. Seul le câble micro-USB est utilisé.

La datasheet du LSM6DS3 est téléchargée et placée dans le dossier `docs/` du projet, pour conserver une référence technique associée au code.

## 3. Principe de fonctionnement

### 3.1 Informations relevées dans la datasheet

Les informations sont à relever dans la datasheet de la référence exacte du composant (LSM6DS3), et non dans la documentation de la carte qui l'accueille.

| Information | Valeur |
|-------------|--------|
| (a) Alimentation du composant | Environ 1,71 V à 3,6 V (broche Vdd) |
| (b) Adresses I2C 7 bits possibles | `0x6A` ou `0x6B`, selon l'état de la broche SA0 |
| (c) Plages de l'accéléromètre | ±2 g, ±4 g, ±8 g, ±16 g |

Les valeurs d'alimentation sont à confirmer dans le tableau des caractéristiques électriques de la datasheet placée dans `docs/`, avec la référence de page.

**Peut-on brancher le composant en 5 V ?** Non : la tension maximale admissible du composant est de l'ordre de 3,6 V. Une alimentation de 5 V dépasse cette limite. La carte Nano 33 IoT fonctionne en logique 3,3 V.

### 3.2 Le bus I2C

I2C (*Inter-Integrated Circuit*) est un bus série synchrone à deux lignes :

- **SDA** : ligne de données ;
- **SCL** : ligne d'horloge.

Le microcontrôleur joue le rôle de contrôleur : il interroge les périphériques à l'aide de leur adresse sur 7 bits, et un périphérique répond lorsqu'il reconnaît son adresse. Sur Arduino, la bibliothèque `Wire` fournit les fonctions de base.

### 3.3 Accélération et gravité

L'accéléromètre mesure l'accélération propre selon trois axes (x, y, z), exprimée en g. Elle inclut la gravité : carte à plat et immobile, x et y valent environ 0 et z vaut environ 1 g. La valeur de z est non nulle parce que la pesanteur s'exerce en permanence sur la carte. Le signe de z dépend de l'orientation des axes du composant et de la façon dont la carte est posée.

### 3.4 Calcul de l'inclinaison

Lorsque le boîtier est immobile, le vecteur d'accélération mesuré est essentiellement dû à la gravité. Son orientation donne l'inclinaison de la carte par rapport à l'axe z :

```text
θ = arccos( z / √(x² + y² + z²) )
```

- le dénominateur est la norme du vecteur d'accélération ;
- `acos()` retourne un angle en radians, converti en degrés avec `RAD_TO_DEG` ;
- une alerte est déclenchée si `θ > 30°`.

Carte à plat, θ est proche de 0°.

## 4. Implémentation

### 4.1 Programme 1 : scanner I2C

- `Wire.begin()` initialise le bus.
- La boucle parcourt les adresses 7 bits de 1 à 124.
- Pour chaque adresse, `Wire.beginTransmission(adr)` prépare une transmission et `Wire.endTransmission()` la termine en retournant un code d'état. Le code `0` signifie que le périphérique a acquitté son adresse.
- `Serial.println(adr, HEX)` affiche l'adresse en hexadécimal.

Le scanner confirme qu'un périphérique répond à une adresse, mais pas sa nature : l'adresse trouvée doit être comparée à celles de la datasheet (réponse à la question (b)).

### 4.2 Programme 2 : lecture avec la bibliothèque `Arduino_LSM6DS3`

La bibliothèque `Arduino_LSM6DS3` fournit une interface de haut niveau et évite de manipuler les registres directement.

- `IMU.begin()` initialise la centrale inertielle. En cas d'échec, le message « centrale inertielle absente » est envoyé sur le moniteur série.
- `IMU.accelerationAvailable()` indique si une nouvelle mesure est disponible.
- `IMU.readAcceleration(x, y, z)` lit les accélérations des trois axes, en g.
- `delay(200)` espace les lectures d'environ 200 ms.

### 4.3 Programme 3 : accès direct par `Wire` et calcul de l'inclinaison

Ce programme est une seconde manière d'exploiter le LSM6DS3, **complémentaire** de la bibliothèque `Arduino_LSM6DS3` utilisée au programme 2 : il écrit et lit directement les registres du composant avec `Wire`.

- `LSM6DS3_ADDRESS 0x6A` : adresse I2C du composant, celle trouvée par le scanner.
- **Configuration** : écriture de `0x40` dans le registre `CTRL1_XL` (`0x10`). Cette valeur active l'accéléromètre à 104 Hz avec une plage de ±2 g.
- **Lecture** : sélection du registre de départ `OUTX_L_XL` (`0x28`) avec `Wire.write(0x28)` et `Wire.endTransmission(false)` (le bus n'est pas libéré), puis `Wire.requestFrom(LSM6DS3_ADDRESS, 6)` pour lire 6 octets.
- **Recombinaison** : chaque axe est codé sur 16 bits en complément à deux, avec l'octet de poids faible en premier.

| Axe | Registres |
|-----|-----------|
| X | `OUTX_L_XL` (`0x28`) et `OUTX_H_XL` (`0x29`) |
| Y | `OUTY_L_XL` (`0x2A`) et `OUTY_H_XL` (`0x2B`) |
| Z | `OUTZ_L_XL` (`0x2C`) et `OUTZ_H_XL` (`0x2D`) |

- **Calcul de l'angle** : `acos(z / denominateur) * RAD_TO_DEG`. Les trois axes ayant la même sensibilité, le facteur de conversion en g s'annule dans le rapport : les valeurs brutes suffisent pour calculer l'angle. Elles ne doivent en revanche pas être présentées comme des accélérations en g.
- **Sécurité** : le test `denominateur > 0.0` évite une division par zéro.
- **Alerte** : si l'angle dépasse 30°, un message d'alerte est affiché.

Remarque : dans une expression comme `Wire.read() | (Wire.read() << 8)`, le langage C++ ne garantit pas l'ordre d'évaluation des deux appels à `Wire.read()`. Lire explicitement l'octet de poids faible, puis l'octet de poids fort, dans deux instructions séparées supprime toute ambiguïté.

### 4.4 Comparaison des deux méthodes

| Critère | Bibliothèque `Arduino_LSM6DS3` | Accès direct avec `Wire` |
|---------|--------------------------------|--------------------------|
| Niveau d'abstraction | Élevé | Faible |
| Initialisation | `IMU.begin()` | Écriture explicite dans les registres |
| Lecture | `IMU.readAcceleration()` | Lecture des registres de sortie |
| Conversion des données | Gérée par la bibliothèque (en g) | À réaliser ou à interpréter dans le programme |
| Usage pédagogique | Exploiter rapidement les mesures | Comprendre le protocole et les registres |

## 5. Code source

### Programme 1 : scanner I2C

```cpp
#include <Arduino.h>
#include <Wire.h>  // bibliothèque pilotant le bus I2C

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}

  Wire.begin();

  for (uint8_t adr = 1; adr < 125; adr++) {  // adresses I2C codées sur 7 bits
    Wire.beginTransmission(adr);
    if (Wire.endTransmission() == 0) {       // si l'esclave répond, il retourne 0
      Serial.print("peripherique a l'adresse 0x");
      Serial.println(adr, HEX);
    }
  }
}

void loop() {}
```

### Programme 2 : lecture de l'accélération avec `Arduino_LSM6DS3`

```cpp
#include <Arduino.h>
#include <Arduino_LSM6DS3.h>

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}
  if (!IMU.begin()) Serial.println("centrale inertielle absente");
}

void loop() {
  float x, y, z;

  if (IMU.accelerationAvailable()) {
    IMU.readAcceleration(x, y, z);  // en g
    Serial.print(x); Serial.print(" ");
    Serial.print(y); Serial.print(" ");
    Serial.println(z);
  }

  delay(200);
}
```

### Programme 3 : accès direct par `Wire`, inclinaison et alerte (bonus)

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <math.h>

// Adresse trouvée lors du scan (0x6A ou 0x6B)
#define LSM6DS3_ADDRESS 0x6A

void setup() {
  Serial.begin(115200);
  while (!Serial && millis() < 3000) {}
  Wire.begin();

  // CTRL1_XL (0x10) : accéléromètre actif, 104 Hz, plage ±2 g
  Wire.beginTransmission(LSM6DS3_ADDRESS);
  Wire.write(0x10);
  Wire.write(0x40);
  Wire.endTransmission();
}

void loop() {
  // Registre de départ : OUTX_L_XL (0x28)
  Wire.beginTransmission(LSM6DS3_ADDRESS);
  Wire.write(0x28);
  Wire.endTransmission(false);

  // Lecture des 6 octets : X, Y, Z
  Wire.requestFrom(LSM6DS3_ADDRESS, 6);

  if (Wire.available() == 6) {
    int16_t raw_x = Wire.read() | (Wire.read() << 8);
    int16_t raw_y = Wire.read() | (Wire.read() << 8);
    int16_t raw_z = Wire.read() | (Wire.read() << 8);

    float x = (float)raw_x;
    float y = (float)raw_y;
    float z = (float)raw_z;

    // Inclinaison : arccos(z / norme)
    float denominateur = sqrt(x * x + y * y + z * z);

    if (denominateur > 0.0) {  // évite la division par zéro
      float inclinaison = acos(z / denominateur) * RAD_TO_DEG;

      Serial.print("Inclinaison : ");
      Serial.print(inclinaison);
      Serial.println(" °");

      if (inclinaison > 30.0) {
        Serial.println("ALERTE : Inclinaison supérieure à 30° !");
      }
    }
  }

  delay(200);
}
```

## 6. Résultats et observations

| Manipulation | Résultat attendu |
|--------------|------------------|
| Scanner I2C | Chaque périphérique qui répond est affiché avec son adresse. L'adresse du LSM6DS3 doit être `0x6A` ou `0x6B` (réponse à la question (b)) ; `0x6A` est celle utilisée dans le programme 3. |
| Lecture avec la bibliothèque, carte à plat et immobile | x ≈ 0, y ≈ 0, z ≈ 1 g (le signe dépend de l'orientation des axes). |
| Pourquoi z ≈ 1 ? | L'accéléromètre mesure aussi la gravité : la pesanteur de 1 g s'exerce sur l'axe vertical, même immobile. |
| Programme 3, carte à plat | Inclinaison proche de 0° ; pas d'alerte. |
| Programme 3, carte inclinée au-delà de 30° | Le message « ALERTE : Inclinaison supérieure à 30° ! » s'affiche. |

La formule fournie mesure l'angle par rapport à l'axe z positif. Pour détecter un basculement sur une installation réelle, il faut définir l'orientation de référence du boîtier et comparer l'orientation courante à cette référence.

## 7. Lien avec AgriSense

- **EX8 — Signaler tout déplacement ou basculement du boîtier :** la centrale inertielle fournit les accélérations sur trois axes. Elles permettent de calculer l'angle d'inclinaison et de déclencher une alerte lorsque le seuil de 30° est dépassé (question bonus de l'exercice).

L'exercice met ainsi en place les bases de la détection d'un basculement du boîtier à partir d'un capteur inertiel relié en I2C.
