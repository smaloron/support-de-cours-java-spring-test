# Corrections des Auto-évaluations

### Chapitre 1 : Introduction aux Tests Logiciels

#### Partie 1 : L'essentiel {id="partie-1-l-essentiel_1"}

1. **Qu'est-ce qu'une "régression" ?**
    * **Réponse :** Une régression est un bug qui apparaît dans une fonctionnalité existante et qui fonctionnait
      correctement, suite à une modification du code (ajout d'une nouvelle fonctionnalité, correction d'un autre bug,
      etc.). Les tests automatisés aident à la prévenir en constituant un filet de sécurité qui re-valide l'ensemble des
      fonctionnalités de l'application après chaque changement, détectant ainsi immédiatement les effets de bord
      indésirables.

2. **'I' dans F.I.R.S.T. ?**
    * **Réponse :** c) Indépendant. Chaque test doit pouvoir s'exécuter seul, sans dépendre de l'état laissé par un test
      précédent.

3. **Différence entre test unitaire et test d'intégration ?**
    * **Réponse :** Le test unitaire vérifie une seule unité de code (une classe, une méthode) de manière totalement
      isolée de ses dépendances (qui sont mockées). Le test d'intégration, lui, vérifie la collaboration et
      l'interaction entre plusieurs composants (par exemple, un contrôleur, un service et un repository qui dialogue
      avec une base de données).

4. **Bug le plus coûteux à corriger ?**
    * **Réponse :** c) Après le déploiement en production, lorsqu'il est trouvé par un client.

5. **But du "Red-Green-Refactor" ?**
    * **Réponse :** b) Laisser les tests guider la conception et l'écriture du code.

#### Partie 2 : Pour aller plus loin {id="partie-2-pour-aller-plus-loin_1"}

1. **Qu'est-ce que le "Shift Left Testing" ?**
    * **Réponse :** C'est la pratique qui consiste à intégrer les activités de test le plus tôt possible dans le cycle
      de développement. Un exemple concret est la tenue d'un atelier BDD (`Given-When-Then`) avec le Product Owner avant
      même d'écrire la première ligne de code.

2. **Modèle favorisant les tests d'intégration ?**
    * **Réponse :** c) Le Trophée des Tests (Testing Trophy).

3. **Différence de "public" entre TDD et BDD ?**
    * **Réponse :** Le TDD est une discipline pour le développeur (une conversation technique sur "comment bien
      construire le code"). Le BDD est un processus de collaboration pour toute l'équipe, y compris les
      non-techniciens (une conversation métier sur "construisons-nous la bonne chose ?").

4. **Exemple de test non fonctionnel ?**
    * **Réponse :** c) Un test qui mesure le temps de réponse de la page d'accueil avec 500 utilisateurs connectés (
      c'est un test de performance).

5. **Syntaxe `Given-When-Then` ?**
    * **Réponse :** c) BDD (Behavior-Driven Development).

### Chapitre 2 : Outils de Base pour les Tests en Java

#### Partie 1 : L'essentiel {id="partie-1-l-essentiel_2"}

1. **À quoi sert `@BeforeEach` ?**
    * **Réponse :** C'est une annotation JUnit 5 qui marque une méthode devant être exécutée **avant chaque** test (
      `@Test`) de la classe. C'est idéal pour initialiser ou réinitialiser un état commun avant chaque test.

2. **Qu'est-ce qu'un "mock" ?**
    * **Réponse :** b) Un objet factice qui simule le comportement d'une dépendance pour isoler le code testé.

3. **Avantage d'AssertJ ?**
    * **Réponse :** Son principal avantage est sa syntaxe "fluide" qui permet d'écrire des assertions très lisibles (
      proches d'une phrase en anglais) et de les chaîner, rendant les tests plus expressifs et plus puissants.

4. **Annotation pour des tests avec plusieurs jeux de données ?**
    * **Réponse :** d) `@ParameterizedTest`.

5. **Syntaxe de "stubbing" Mockito ?**
    * **Réponse :** b) `when(mock.method()).thenReturn("value")`.

#### Partie 2 : Pour aller plus loin {id="partie-2-pour-aller-plus-loin_2"}

1. **Quand utiliser un `@Spy` ?**
    * **Réponse :** Un `@Spy` s'utilise (rarement) quand on a besoin de tester une méthode d'un objet réel tout en
      conservant le comportement original de ses autres méthodes. Le risque est que l'appel à une méthode non-stubbée
      exécute du code réel potentiellement complexe ou avec des effets de bord. C'est souvent le signe d'une classe qui
      mériterait d'être refactorisée.

2. **Utilité de `ArgumentCaptor` ?**
    * **Réponse :** b) Capturer et inspecter les valeurs des arguments passés à une méthode d'un mock.

3. **Avantage des "soft assertions" ?**
    * **Réponse :** Elles permettent de rapporter toutes les erreurs d'assertion au sein d'un bloc de test, au lieu de
      s'arrêter à la première. C'est très utile pour valider plusieurs propriétés d'un même objet en une seule fois.

4. **Annotation pour organiser les tests en groupes ?**
    * **Réponse :** d) L'annotation `@Nested` sur des classes internes.

5. **Tester une exception sur une méthode `void` ?**
    * **Réponse :** c) `doThrow(new MyException()).when(mock).voidMethod();`.

### Chapitre 3 : Introduction aux Tests avec Spring Boot

#### Partie 1 : L'essentiel {id="partie-1-l-essentiel_3"}

1. **Différence entre `@Mock` et `@MockBean` ?**
    * **Réponse :** `@Mock` (de Mockito) crée un simple mock que vous devez gérer et injecter manuellement.
      `@MockBean` (de Spring) crée un mock et remplace le bean correspondant directement dans le contexte d'application
      Spring pour la durée du test.

2. **`spring-boot-starter-test` inclut-il... ?**
    * **Réponse :** c) Un ensemble de bibliothèques incluant Spring Test, JUnit 5, Mockito et AssertJ.

3. **Comment utiliser une base H2 pour les tests ?**
    * **Réponse :** La manière la plus simple est de créer un fichier `src/test/resources/application-test.properties` (
      ou `.yml`) et d'y définir la configuration de la datasource pour qu'elle pointe vers H2.

4. **L'annotation `@SpringBootTest`...**
    * **Réponse :** b) Charge un contexte d'application Spring complet, rendant les beans disponibles pour l'injection.

5. **Que fait `@TestConfiguration` ?**
    * **Réponse :** b) Elle ajoute des beans au contexte de test, qui peuvent être utilisés en plus des beans de
      l'application principale.

#### Partie 2 : Pour aller plus loin {id="partie-2-pour-aller-plus-loin_3"}

1. **Quel `webEnvironment` pour un appel HTTP réel ?**
    * **Réponse :** `@SpringBootTest.WebEnvironment.RANDOM_PORT`. Il démarre un vrai serveur web sur un port aléatoire,
      ce qui permet de faire de vrais appels réseau avec `TestRestTemplate` sans risque de conflit de port.

2. **Utilité de `@Sql` ?**
    * **Réponse :** b) Exécuter des scripts SQL pour initialiser et nettoyer l'état de la base de données pour un test.

3. **`@TestPropertySource` vs. `application-test.properties` ?**
    * **Réponse :** `application-test.properties` définit une configuration de base pour tous les tests.
      `@TestPropertySource` permet de surcharger une ou plusieurs propriétés spécifiquement pour une classe de test ou
      même une seule méthode, ce qui est idéal pour tester des comportements qui dépendent d'une configuration
      particulière.

4. **Effet de `@SpringBootTest(classes = ...)` ?**
    * **Réponse :** d) Spring créera un contexte de test contenant uniquement le bean `MyService` (et tentera d'injecter
      ses dépendances). C'est une simplification de la réponse correcte : Spring va charger les classes spécifiées et
      leurs dépendances nécessaires, mais pas toute l'application. La réponse d) est la plus proche de la réalité parmi
      les choix.

5. **Effet de `@Configuration` dans un test ?**
    * **Réponse :** b) Elle sera utilisée pour ajouter ou modifier des beans dans le contexte principal de l'application
      pour ce test. Elle est plus "puissante" qu'une `@TestConfiguration` et peut remplacer des beans existants.

### Chapitre 4 : Tests Unitaires des Composants Spring

1. **Pourquoi utiliser `@SpringBootTest(classes = ...)` pour un test de service ?**
    * **Réponse :** C'est un bon compromis qui permet de tester le service de manière isolée (en mockant ses
      dépendances) tout en bénéficiant de l'injection de dépendances gérée par Spring, ce qui est plus simple que
      l'instanciation manuelle, surtout si le service a de nombreuses dépendances.

2. **À quoi sert `@MockBean` dans un test de service ?**
    * **Réponse :** b) À remplacer les dépendances du service (comme les repositories) par des simulations contrôlées
      par Mockito.

3. **Comment tester une méthode `void` ?**
    * **Réponse :** On ne peut pas tester sa valeur de retour. On teste donc ses **effets de bord** : le plus souvent,
      on vérifie qu'elle a bien interagi avec ses dépendances en utilisant `Mockito.verify()` pour s'assurer qu'une
      méthode sur un mock a été appelée.

4. **Test unitaire "pur" vs. `@SpringBootTest` ?**
    * **Réponse :** c) Le test pur n'utilise pas du tout le framework Spring pour son exécution, il est donc plus
      rapide.

5. **Signification de `verify(..., never())...` ?**
    * **Réponse :** c) La méthode `deleteById` n'a pas été appelée du tout pendant l'exécution du test.

### Chapitre 5 : Tests d'Intégration : La Couche Web

1. **Différence de contexte entre `@SpringBootTest` et `@WebMvcTest` ?**
    * **Réponse :** `@SpringBootTest` charge le contexte applicatif complet (services, repositories, etc.).
      `@WebMvcTest` ne charge qu'une "tranche" de l'application contenant la couche web (contrôleurs, filtres,
      sérialiseurs JSON) mais PAS les services ni les repositories.

2. **Dans un `@WebMvcTest`, les `@Service` sont... ?**
    * **Réponse :** b) Non chargés, et doivent être mockés avec `@MockBean` si le contrôleur en dépend.

3. **À quoi sert `MockMvc` ?**
    * **Réponse :** C'est un client HTTP simulé fourni par Spring Test. Il permet d'exécuter des requêtes vers les
      contrôleurs sans démarrer de vrai serveur web. On l'obtient par injection de dépendances (`@Autowired`) dans une
      classe de test annotée avec `@WebMvcTest`.

4. **Meilleure méthode pour vérifier un contenu JSON ?**
    * **Réponse :** b) `andExpect(jsonPath("$.cle", is("valeur")))`.

5. **Que faire pour simuler un `POST` avec JSON ?**
    * **Réponse :** c) Utiliser la méthode `content()` pour fournir le JSON et définir le `contentType` à
      `MediaType.APPLICATION_JSON`.

### Chapitre 6 : Tests d'Intégration : La Couche d'Accès aux Données

1. **Rôle de `@DataJpaTest` et configurations clés ?**
    * **Réponse :** Son rôle est de tester la couche de persistance JPA en isolation. Par défaut, il configure une base
      de données en mémoire (H2) et exécute chaque test dans une transaction avec rollback automatique.

2. **Utilité du rollback automatique ?**
    * **Réponse :** c) Pour garantir l'isolation des tests en nettoyant la base de données après chaque test.

3. **`persist()` vs. `persistAndFlush()` ?**
    * **Réponse :** `persist()` enregistre l'entité dans le contexte de persistance, mais ne garantit pas l'exécution
      immédiate de la requête `INSERT` en base. `flush()` force cette synchronisation. On utilise `persistAndFlush()`
      quand on a besoin de s'assurer que la donnée est bien en base avant d'exécuter l'étape suivante du test (par
      exemple, une lecture).

4. **Beans chargés par `@DataJpaTest` ?**
    * **Réponse :** c) Uniquement les `@Repository` JPA, les `@Entity` et l'infrastructure de persistance.

5. **`TestEntityManager` est... ?**
    * **Réponse :** c) Une classe utilitaire fournie par Spring Test pour simplifier la manipulation des entités JPA
      dans les tests.

### Chapitre 7 : Tests d'Intégration Complets

1. **Quand un test d'intégration complet est-il pertinent ?**
    * **Réponse :** Pour un scénario métier critique qui traverse plusieurs couches de l'application, comme le processus
      d'inscription d'un utilisateur ou le passage d'une commande.

2. **Différence `MockMvc` et `TestRestTemplate` ?**
    * **Réponse :** a) `MockMvc` est plus rapide car il ne démarre pas de serveur web.

3. **Pourquoi `RANDOM_PORT` est bien adapté ?**
    * **Réponse :** Il démarre un vrai serveur web, ce qui permet des tests très réalistes, mais sur un port aléatoire,
      ce qui évite les conflits si plusieurs processus de test s'exécutent en parallèle (sur un serveur de CI par
      exemple).

4. **Pourquoi `@AfterEach` plutôt que `@Transactional` ?**
    * **Réponse :** b) Parce que l'appel HTTP se fait dans une transaction séparée de celle du test, rendant le rollback
      inefficace pour ce scénario.

5. **Objectif principal d'un test d'intégration complet ?**
    * **Réponse :** b) Vérifier que les composants de l'application (contrôleur, service, repository) collaborent
      correctement pour un flux métier critique.

### Chapitre 8 : Gérer les Dépendances Externes : Testcontainers

1. **Pourquoi Testcontainers est préférable à H2 ?**
    * **Réponse :** 1) Il permet de tester contre la même technologie de base de données que celle utilisée en
      production (haute fidélité), éliminant les bugs dus aux différences de dialecte SQL. 2) Il est polyvalent et peut
      lancer n'importe quel service "dockerisable" (Kafka, Redis, etc.), pas seulement des bases de données.

2. **Annotation pour connecter dynamiquement Spring au conteneur ?**
    * **Réponse :** b) `@DynamicPropertySource`.

3. **Que signifie rendre un `@Container` `static` ?**
    * **Réponse :** Cela signifie que le conteneur Docker sera démarré une seule fois pour toutes les méthodes de test
      de la classe, puis arrêté à la fin. L'avantage est un gain de temps considérable, car le démarrage d'un conteneur
      est une opération coûteuse.

4. **Prérequis pour Testcontainers ?**
    * **Réponse :** b) Avoir Docker installé et en cours d'exécution.

5. **`@AutoConfigureTestDatabase(replace = Replace.NONE)` sert à... ?**
    * **Réponse :** b) Empêcher Spring Boot de configurer une base de données en mémoire à notre place.

### Chapitre 9 : Tester la Sécurité avec Spring Security

1. **Pourquoi un test échoue après ajout de Spring Security ?**
    * **Réponse :** Parce que Spring Security protège par défaut tous les endpoints. La requête de test, n'étant pas
      authentifiée, se voit refuser l'accès avec un statut `401 Unauthorized`.

2. **Annotation pour simuler un utilisateur admin ?**
    * **Réponse :** c) `@WithMockUser(username="admin", roles="ADMIN")`.

3. **Différence entre non authentifié et non autorisé ?**
    * **Réponse :** Un utilisateur **non authentifié** n'a fourni aucune identité valide (résultat :
      `401 Unauthorized`). Un utilisateur **non autorisé** est bien identifié (authentifié), mais son identité (ses
      rôles/permissions) ne lui donne pas le droit d'accéder à la ressource (résultat : `403 Forbidden`).

4. **Pourquoi `@Import(SecurityConfig.class)` ?**
    * **Réponse :** b) Pour charger les règles de sécurité (`authorizeHttpRequests`, etc.) qui ne sont pas chargées par
      défaut dans ce type de test en tranche.

5. **Un statut `403 Forbidden` signifie... ?**
    * **Réponse :** b) Le serveur a compris la requête, mais refuse de l'autoriser car le client n'a pas les droits
      nécessaires.

### Chapitre 10 : Tests Asynchrones et Réactifs

1. **Problème du test d'une méthode `@Async` ?**
    * **Réponse :** La méthode de test continue son exécution sans attendre la fin du traitement asynchrone. L'assertion
      est donc exécutée trop tôt, avant que le résultat ne soit disponible, provoquant l'échec du test.

2. **Bibliothèque pour tester les processus asynchrones ?**
    * **Réponse :** c) Awaitility.

3. **Différence `Mono` vs. `Flux` ?**
    * **Réponse :** Un `Mono` est un flux réactif qui représente 0 ou 1 élément. Un `Flux` représente 0 à N éléments.

4. **Pour tester des endpoints WebFlux, on utilise... ?**
    * **Réponse :** c) `WebTestClient`.

5. **`StepVerifier` est utilisé pour... ?**
    * **Réponse :** b) Vérifier chaque événement (émission d'élément, complétion, erreur) dans un `Mono` ou un `Flux`.

### Chapitre 11 : Bonnes Pratiques et Stratégies Avancées

#### Partie 1 : L'essentiel

1. **Convention `method_should_when` ?**
    * **Réponse :** C'est une convention de nommage qui rend le test auto-documenté. Elle décrit la méthode testée (
      `method`), le résultat attendu (`should...`) et les conditions dans lesquelles ce résultat se produit (`when...`).

2. **"Test Data Builder" est utilisé pour... ?**
    * **Réponse :** c) Simplifier la création d'objets de test complexes et améliorer la lisibilité des tests.

3. **Couverture de 95% : qu'est-ce que cela signifie/ne signifie pas ?**
    * **Réponse :** Cela signifie que 95% des lignes du code de production ont été exécutées par les tests. Cela **ne
      signifie pas** que le code est correct à 95% ou que les 95% des fonctionnalités sont bien testées (les assertions
      pourraient être faibles ou inexistantes).

4. **Principe DRY ?**
    * **Réponse :** b) Don't Repeat Yourself - Éviter la duplication de code.

5. **Organisation des packages de test ?**
    * **Réponse :** b) Créer une structure de packages qui reflète celle du code source.

#### Partie 2 : Pour aller plus loin

1. **Cycle "Rouge-Vert-Refactor" ?**
    * **Réponse :** C'est le cycle de développement du TDD : 1) **Rouge** : Écrire un test qui échoue. 2) **Vert** :
      Écrire le minimum de code pour que le test passe. 3) **Refactor** : Améliorer le code tout en gardant le test au
      vert.

2. **Énoncé qui décrit le mieux le BDD ?**
    * **Réponse :** b) Une méthodologie de développement axée sur la collaboration et la définition partagée du
      comportement attendu.

3. **Avantage d'une classe de base de test ?**
    * **Réponse :** Elle permet de centraliser la configuration de test commune (comme la déclaration d'un conteneur
      Testcontainers) pour éviter de la dupliquer dans de nombreuses classes de test (principe DRY).

4. **Objectif de l'Intégration Continue (CI) ?**
    * **Réponse :** c) Détecter les problèmes d'intégration et les régressions le plus tôt possible en construisant et
      testant le code fréquemment.

5. **Un "mutant qui survit" signifie... ?**
    * **Réponse :** c) La modification apportée au code n'a été détectée par aucun test, révélant une faiblesse dans la
      suite de tests.

### Chapitre 12 : Conclusion et Aller Plus Loin

1. *(Réponse personnelle)*
2. **Où se situent les tests E2E avec Cypress ?**
    * **Réponse :** c) Au sommet, car ils sont lents, coûteux mais très réalistes.
3. **Bénéfice principal du TDD ?**
    * **Réponse :** Le TDD produit un code naturellement testable et fournit un filet de sécurité complet contre les
      régressions. (Ou : Il utilise les tests pour guider la conception du logiciel).
4. **CI vise à... ?**
    * **Réponse :** b) Intégrer et tester le code de manière fréquente et automatisée pour détecter les problèmes
      rapidement.
5. **Outil pour tester contre une BDD en conteneur ?**
    * **Réponse :** c) Testcontainers.