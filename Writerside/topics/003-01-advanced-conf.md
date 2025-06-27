# Chapitre 3 : Configurations Avancées et Contrôle de l'Environnement de Test (Pour aller plus loin)

### Objectifs pédagogiques

À la fin de cette partie, vous serez en mesure de :

- **Maîtriser** différentes stratégies de chargement du contexte avec `@SpringBootTest`.
- **Personnaliser** le contexte d'application chargé pour un test avec `classes` et `@ContextConfiguration`.
- **Injecter** des valeurs de configuration dans vos tests en utilisant `@TestPropertySource`.
- **Comprendre** la différence entre `@TestConfiguration` et `@Configuration` dans un contexte de test.
- **Initialiser** votre base de données de test de manière déclarative avec `@Sql`.

### Introduction : Le metteur en scène exigeant

Dans la partie précédente, vous avez appris à utiliser `@SpringBootTest` pour que votre "équipe de stand" prépare la
voiture pour la course. C'était simple et efficace : l'équipe préparait systématiquement la voiture de la même manière,
avec tous les systèmes activés.

Mais que se passe-t-il si, en tant que metteur en scène, vous voulez tester une scène très spécifique ? Une scène de
nuit sous la pluie ? Une scène où seul le moteur est testé, sans la carrosserie ? Vous n'avez pas besoin de toute l'
équipe ni de toute la préparation. Vous avez besoin d'un contrôle plus fin sur l'environnement, les accessoires et les
acteurs présents.

Cette partie vous donne les outils du metteur en scène. Vous apprendrez à ne charger que les parties de l'application
dont vous avez besoin, à injecter des "accessoires" de configuration spécifiques (des propriétés), et à préparer le "
décor" (votre base de données) pour chaque scène de test. Vous passerez du rôle de pilote à celui de réalisateur de vos
tests.

### `@SpringBootTest` : Plus qu'une simple annotation

L'annotation `@SpringBootTest` est plus polyvalente qu'il n'y paraît. Son attribut `webEnvironment` vous permet de
contrôler précisément si et comment la partie web de votre application doit être démarrée.

<tabs>
<tab title="MOCK (défaut)">
    <p><code>@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)</code></p>
    <p>C'est le comportement par défaut. Le contexte de l'application est chargé, mais <b>le serveur web (Tomcat, Netty) n'est pas démarré</b>. Un environnement de servlet "mock" est fourni. C'est parfait pour les tests d'intégration qui n'ont pas besoin de faire de vrais appels HTTP, mais qui nécessitent les beans de la couche web.</p>
    <p><b>Quand l'utiliser ?</b> Pour la plupart des tests d'intégration qui ne ciblent pas spécifiquement les contrôleurs via des appels réseau. C'est plus rapide.</p>
</tab>
<tab title="RANDOM_PORT">
    <p><code>@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)</code></p>
    <p>Un serveur web <b>réel est démarré et écoute sur un port aléatoire</b> disponible. C'est idéal pour des tests d'intégration complets où vous voulez faire de vrais appels réseau à votre propre application, sans risque de conflit de port si plusieurs processus de test tournent en parallèle.</p>
    <p><b>Quand l'utiliser ?</b> Pour les tests d'intégration "de bout en bout" au sein d'un microservice.</p>
</tab>
<tab title="DEFINED_PORT">
    <p><code>@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)</code></p>
    <p>Similaire à `RANDOM_PORT`, mais le serveur démarre sur le port défini dans votre `application.properties` (par exemple, `server.port=8080`).</p>
    <p><b>Quand l'utiliser ?</b> Moins courant, utile si vous avez un client de test externe qui est configuré pour appeler une URL fixe.</p>
</tab>
<tab title="NONE">
    <p><code>@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)</code></p>
    <p>Le contexte de l'application est chargé, mais <b>aucun environnement web</b> (ni réel, ni mock) n'est créé. L'application ne s'exécute pas comme une application web.</p>
    <p><b>Quand l'utiliser ?</b> Pour tester des applications Spring qui ne sont pas des applications web (par exemple, des batchs, des applications en ligne de commande).</p>
</tab>
</tabs>

### Personnaliser le Contexte de Test

Charger toute l'application avec `@SpringBootTest` peut être lent. Parfois, vous ne voulez tester qu'un petit
sous-ensemble de votre application. L'attribut `classes` de `@SpringBootTest` est votre allié. Il vous permet de
spécifier manuellement quelles classes de configuration (`@Configuration`) ou quels composants (`@Component`,
`@Service`...) doivent être chargés dans le contexte.

```java
// On ne veut tester que l'interaction entre ces deux services,
// sans charger les contrôleurs, la sécurité, etc.
@SpringBootTest(classes = {GreeterService.class, TimeService.class})
class LeanGreeterServiceTest {

    @Autowired
    private GreeterService greeterService;

    // TimeService est aussi dans le contexte, on peut donc le mocker.
    @MockBean
    private TimeService mockTimeService;

    // ... test
}
```

<warning>
**Soyez prudent !** En utilisant `classes`, vous prenez le contrôle. Si `GreeterService` avait eu une autre dépendance non déclarée, le contexte n'aurait pas pu démarrer. C'est une technique puissante mais qui demande de bien connaître les dépendances de ce que l'on teste.
</warning>

### `@TestPropertySource` : Injecter des Configurations à la Volée

Imaginez que vous ayez un service qui se comporte différemment selon une propriété dans votre `application.properties`.

```java

@Service
public class FeatureFlagService {
    @Value("${features.new-algorithm.enabled:false}")
    private boolean newAlgorithmEnabled;

    public String getAlgorithm() {
        return newAlgorithmEnabled ? "Nouveau et Amélioré" : "Ancien et Fiable";
    }
}
```

Comment tester les deux cas sans créer deux fichiers de propriétés différents ? Avec `@TestPropertySource` !

```java

@SpringBootTest
class FeatureFlagServiceTest {

    @Nested
    class WhenFeatureIsDisabled {
        @Autowired
        private FeatureFlagService featureFlagService;

        @Test
        void shouldUseOldAlgorithm() {
            assertThat(featureFlagService.getAlgorithm())
                    .isEqualTo("Ancien et Fiable");
        }
    }

    @Nested
    @TestPropertySource(properties = "features.new-algorithm.enabled=true")
    class WhenFeatureIsEnabled {
        @Autowired
        private FeatureFlagService featureFlagService;

        @Test
        void shouldUseNewAlgorithm() {
            assertThat(featureFlagService.getAlgorithm())
                    .isEqualTo("Nouveau et Amélioré");
        }
    }
}
```

L'annotation `@TestPropertySource` surcharge les valeurs de `application.properties` pour la durée du test. C'est
incroyablement pratique pour tester des "feature flags", des configurations de timeouts, des URL externes, etc.

### Préparer le Décor : L'annotation `@Sql`

Lorsque l'on teste des services qui dépendent de la base de données, un des plus grands défis est de s'assurer que la
base est dans un état connu et prévisible avant chaque test. Plutôt que d'insérer des données manuellement en Java, vous
pouvez utiliser des scripts SQL.

L'annotation `@Sql` de Spring Test vous permet d'exécuter des scripts SQL avant (et après) vos méthodes de test.

<procedure title="Mise en place d'un test avec @Sql">
    <step>Créez un fichier SQL dans <code>src/test/resources</code>. Par exemple, <code>/data/books-setup.sql</code>.</step>
    <code-block lang="sql">
    -- src/test/resources/data/books-setup.sql
    DELETE FROM book; -- On nettoie d'abord
    INSERT INTO book (id, title, author, is_best_seller) VALUES
    (1, 'La Pyramide des Tests', 'M. Testeur', false),
    (2, 'Le Trophée des Tests', 'K.C. Dodds', true);
    </code-block>

    <step>Créez un script pour le nettoyage (optionnel mais recommandé).</step>
    <code-block lang="sql">
    -- src/test/resources/data/books-teardown.sql
    DELETE FROM book;
    </code-block>
    
    <step>Annotez votre méthode de test.</step>
    <code-block lang="java">
    @SpringBootTest
    // Pas de @MockBean pour le repository, on veut tester avec une vraie BDD
    class BookServiceIntegrationTest {

        @Autowired
        private BookService bookService;

        @Test
        @Sql(scripts = "/data/books-setup.sql", 
             executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
        @Sql(scripts = "/data/books-teardown.sql", 
             executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
        void shouldFindBestSeller_fromSqlData() {
            String description = bookService.getBookDescription("Le Trophée des Tests");
            assertThat(description).contains("[BEST-SELLER]");
        }
    }
    </code-block>

</procedure>

<tip>
Utiliser `@Sql` rend vos tests beaucoup plus lisibles. L'état initial est décrit de manière déclarative, séparant clairement la préparation des données de la logique de test.
</tip>

### Exercice 6 : Le test du chef d'orchestre

Vous avez un service de notification qui envoie des alertes. Le canal de notification (EMAIL, SMS, PUSH) et le niveau de
priorité (NORMAL, URGENT) sont définis par des propriétés de configuration. Le service doit formater le message
différemment en fonction de ces propriétés.

```java
// fr.formation.spring.service.AlertService.java
@Service
public class AlertService {
    private final String channel;
    private final String priority;

    public AlertService(@Value("${alert.channel:EMAIL}") String channel,
                        @Value("${alert.priority:NORMAL}") String priority) {
        this.channel = channel;
        this.priority = priority;
    }

    public String sendAlert(String message) {
        String finalMessage = String.format("[%s] %s: %s",
                channel, priority, message);
        // ... (logique d'envoi réelle)
        return finalMessage;
    }
}
```

**Votre mission :**
Écrivez une classe de test `AlertServiceTest` avec une méthode qui vérifie un cas de configuration spécifique :

1. Utilisez `@SpringBootTest` pour ne charger que le `AlertService.class`.
2. Utilisez `@TestPropertySource` pour forcer la configuration suivante :
    * `alert.channel=SMS`
    * `alert.priority=URGENT`
3. Injectez le `AlertService` et appelez sa méthode `sendAlert`.
4. Utilisez AssertJ pour vérifier que le message retourné est exactement `[SMS] URGENT: Panne système imminente`.

### Correction exercice 6 {collapsible="true"}

```java
// Fichier : src/test/java/fr/formation/spring/service/AlertServiceTest.java
package fr.formation.spring.service;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(classes = AlertService.class)
@TestPropertySource(properties = {
        "alert.channel=SMS",
        "alert.priority=URGENT"
})
class AlertServiceTest {

    @Autowired
    private AlertService alertService;

    @Test
    @DisplayName("Devrait formater une alerte SMS urgente correctement")
    void shouldFormatUrgentSmsAlert() {
        // Arrange
        String message = "Panne système imminente";
        String expectedFormat = "[SMS] URGENT: Panne système imminente";

        // Act
        String result = alertService.sendAlert(message);

        // Assert
        assertThat(result).isEqualTo(expectedFormat);
    }
}
```

### Auto-évaluation

1. (Question ouverte) Vous écrivez un test d'intégration complet pour un contrôleur REST et vous voulez faire un vrai
   appel HTTP dessus. Quel `webEnvironment` de `@SpringBootTest` est le plus approprié et pourquoi ?
2. (QCM) Quelle est la principale utilité de l'annotation `@Sql` ?
    * a) Mocker la couche d'accès aux données.
    * b) Exécuter des scripts SQL pour initialiser et nettoyer l'état de la base de données pour un test.
    * c) Valider que le code SQL généré par JPA est correct.
    * d) Injecter un `DataSource` dans le test.
3. (Question ouverte) Expliquez la différence entre utiliser `@TestPropertySource` et définir des propriétés dans
   `src/test/resources/application-test.properties`. Dans quel cas utiliseriez-vous l'un plutôt que l'autre ?
4. (QCM) Si vous utilisez `@SpringBootTest(classes = { MyService.class })`, que se passera-t-il ?
    * a) Le test ne compilera pas.
    * b) L'application entière sera chargée, mais seul `MyService` sera testable.
    * c) Seul `MyService` et ses dépendances directes et transitives seront chargés dans le contexte de test.
    * d) Spring créera un contexte de test contenant uniquement le bean `MyService`, et le test échouera si `MyService`
      a des dépendances.
5. (QCM) Vous mettez une annotation `@Configuration` sur une classe interne dans votre test. Quel est l'effet ?
    * a) Elle sera ignorée par Spring.
    * b) Elle sera utilisée pour ajouter ou modifier des beans dans le contexte principal de l'application pour ce test.
    * c) Elle est équivalente à `@TestConfiguration`.
    * d) Elle lancera une exception car `@Configuration` n'est pas permis dans les tests.

*(Les corrections de l'auto-évaluation seront fournies à la toute fin du support de cours.)*

### Conclusion de la partie

Vous êtes maintenant un metteur en scène accompli ! Vous ne vous contentez plus d'accepter l'environnement de test par
défaut. Vous savez comment le **sculpter** à vos besoins : démarrer un vrai serveur web ou non, charger uniquement les
acteurs (`classes`) pertinents pour votre scène, leur donner des accessoires spécifiques (`@TestPropertySource`) et
préparer le décor (`@Sql`).

Cette maîtrise du contexte est cruciale. Elle vous permet d'écrire des tests plus rapides, plus ciblés et plus robustes.
Elle vous donne la confiance nécessaire pour tester des interactions complexes dans des conditions parfaitement
contrôlées.

Armés de cette connaissance approfondie, nous sommes prêts à aborder la spécialisation. Dans les prochains chapitres,
nous allons nous concentrer sur des "tranches" spécifiques de l'application et découvrir les outils que Spring Boot nous
offre pour les tester de manière optimale : d'abord la couche de service et la logique métier, puis la couche web et
enfin la couche de persistance. L'aventure continue 