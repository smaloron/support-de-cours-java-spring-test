# Chapitre 10 : Le Test dans une Autre Dimension : Code Asynchrone et Réactif (L'essentiel)

### Objectifs pédagogiques

À la fin de cette partie, vous serez en mesure de :

- **Identifier** les défis spécifiques posés par le test de code asynchrone (`@Async`).
- **Utiliser** la bibliothèque Awaitility pour écrire des tests asynchrones stables et lisibles.
- **Comprendre** les bases de la programmation réactive avec Project Reactor (`Mono`, `Flux`).
- **Tester** des endpoints réactifs (WebFlux) avec `WebTestClient`.
- **Valider** le comportement de `Mono` et `Flux` avec `StepVerifier`.

### Introduction : La cuisson du gâteau à retardement

Imaginez que vous testez une recette de cuisine. Jusqu'à présent, toutes nos recettes étaient instantanées : "mélanger
les ingrédients" (`méthode synchrone`) -> "goûter le mélange" (`assertion`). C'est simple, l'action et la vérification
sont immédiates.

Maintenant, vous devez tester une recette de gâteau. L'une des étapes est : "mettre le gâteau au four pendant 30
minutes" (`@Async`). C'est une opération **asynchrone**. Vous ne pouvez pas goûter le gâteau immédiatement après l'avoir
mis au four, il ne serait pas cuit ! Vous devez **attendre**. Mais attendre combien de temps ? Attendre 30 minutes dans
un test est inacceptable. Et si la cuisson prend 31 minutes ? Ou 29 ?

Tester du code asynchrone, c'est comme tester cette recette. On ne peut pas simplement appeler la méthode et faire une
assertion juste après. La méthode de test se terminerait avant que le traitement asynchrone n'ait eu lieu. Nous avons
besoin d'outils spéciaux pour :

1. **Attendre** intelligemment qu'une condition soit remplie (le gâteau est cuit).
2. **Valider** le résultat une fois que le traitement est terminé.

Ce chapitre vous donnera les outils pour tester ces processus "à retardement", que ce soit avec le classique `@Async` de
Spring ou avec les flux de données modernes de la programmation réactive.

### Tester les Méthodes `@Async` avec Awaitility

Spring permet de rendre une méthode asynchrone très facilement avec l'annotation `@Async`. Lorsqu'elle est appelée, elle
s'exécute dans un thread séparé, et le thread appelant continue son chemin sans attendre.

**Le service à tester :**

```java

@Service
public class NotificationService {
    private final Map<UUID, String> notificationStatus = new ConcurrentHashMap<>();

    @Async // Cette méthode ne bloquera pas l'appelant
    public void sendNotification(UUID notificationId, String message) {
        // Simule un travail long (appel à un service externe, etc.)
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        notificationStatus.put(notificationId, "SENT: " + message);
    }

    public Optional<String> getStatus(UUID notificationId) {
        return Optional.ofNullable(notificationStatus.get(notificationId));
    }
}
```

**Le test naïf (qui échouera) :**

```java

@Test
void testNotification_fails() {
    // Act
    notificationService.sendNotification(id, "Hello");
    // Assert
    Optional<String> status = notificationService.getStatus(id);
    assertThat(status).isPresent(); // ==> ECHEC ! Le test arrive ici AVANT la fin du sleep.
}
```

**La solution : Awaitility**
Awaitility est une petite bibliothèque très puissante qui permet d'exprimer des attentes asynchrones de manière lisible.
Spring Boot ne l'inclut pas par défaut, il faut l'ajouter :

```xml

<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.1</version>
    <scope>test</scope>
</dependency>
```

**Le test correct avec Awaitility :**

```java
import static org.awaitility.Awaitility.await;
import static java.util.concurrent.TimeUnit.SECONDS;

@SpringBootTest
@EnableAsync // Nécessaire pour que @Async fonctionne dans les tests
class NotificationServiceTest {

    @Autowired
    private NotificationService notificationService;

    @Test
    void sendNotification_shouldUpdateStatus_eventually() {
        // Arrange
        UUID notificationId = UUID.randomUUID();

        // Act
        notificationService.sendNotification(notificationId, "Test message");

        // Assert avec Awaitility
        await()
                .atMost(5, SECONDS) // On attend au maximum 5 secondes
                .untilAsserted(() -> { // Et on réessaie cette assertion jusqu'à ce qu'elle passe
                    Optional<String> status = notificationService.getStatus(notificationId);
                    assertThat(status)
                            .isPresent()
                            .hasValue("SENT: Test message");
                });
    }
}
```

Awaitility interroge la condition à intervalles réguliers jusqu'à ce qu'elle soit vraie ou que le timeout soit atteint.
C'est propre, lisible et robuste.

### Le Monde Réactif : Tester avec `WebTestClient` et `StepVerifier`

La programmation réactive (implémentée dans Spring avec WebFlux et Project Reactor) est une autre forme d'asynchronisme,
basée sur des flux de données (`Flux` pour 0..N éléments, `Mono` pour 0..1 élément). Tester ce paradigme nécessite des
outils spécifiques.

**Le contrôleur réactif à tester :**

```java

@RestController
public class ReactiveBookController {

    // Retourne un seul élément, de manière asynchrone
    @GetMapping("/reactive/books/{id}")
    public Mono<Book> getBookById(@PathVariable Long id) {
        return Mono.just(new Book(id, "Dune", "Frank Herbert"));
    }

    // Retourne un flux d'éléments, de manière asynchrone
    @GetMapping("/reactive/books")
    public Flux<Book> getAllBooks() {
        return Flux.just(
                new Book(1L, "Dune", "F. Herbert"),
                new Book(2L, "Hyperion", "D. Simmons")
        ).delayElements(Duration.ofMillis(100)); // Simule une latence
    }
}
```

#### `WebTestClient` : Le `MockMvc` du monde réactif

Pour tester les endpoints réactifs, on utilise `@WebFluxTest` (l'équivalent de `@WebMvcTest`) qui nous injecte un
`WebTestClient`. Sa syntaxe est très fluide et pensée pour le réactif.

```java
import org.springframework.boot.test.autoconfigure.web.reactive.WebFluxTest;
import org.springframework.test.web.reactive.server.WebTestClient;

@WebFluxTest(ReactiveBookController.class)
class ReactiveBookControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void getBookById_shouldReturnBook() {
        webTestClient.get().uri("/reactive/books/1")
                .exchange() // Exécute la requête
                .expectStatus().isOk() // Assertion sur le statut
                .expectBody(Book.class) // Assertion sur le corps
                .value(book -> {
                    assertThat(book.getTitle()).isEqualTo("Dune");
                });
    }
}
```

#### `StepVerifier` : Le spécialiste des `Mono` et `Flux`

Si vous voulez tester la logique d'un service réactif (qui retourne un `Mono` ou un `Flux`) sans passer par un
contrôleur, `StepVerifier` est l'outil qu'il vous faut. Il permet de "s'abonner" à un flux et de vérifier chaque
événement qui s'y produit (chaque élément émis, l'événement de complétion, une erreur, etc.).

**Le service réactif :**

```java

@Service
public class ReactiveBookService {
    public Flux<String> getBookTitles() {
        return Flux.just("Dune", "Hyperion")
                .map(String::toUpperCase);
    }
}
```

**Le test du service avec `StepVerifier` :**

```java
// Nécessite la dépendance io.projectreactor:reactor-test

import reactor.test.StepVerifier;

class ReactiveBookServiceTest {

    private final ReactiveBookService service = new ReactiveBookService();

    @Test
    void getBookTitles_shouldReturnUppercasedTitles() {
        // Arrange
        Flux<String> titlesFlux = service.getBookTitles();

        // Act & Assert
        StepVerifier.create(titlesFlux)
                .expectNext("DUNE") // On s'attend à recevoir cet élément
                .expectNext("HYPERION") // Puis celui-ci
                .verifyComplete(); // On s'attend à ce que le flux se termine sans erreur
    }
}
```

### Exercice 13 : Tester une notification asynchrone avec Awaitility

Vous avez un `OrderService` qui, après avoir traité une commande, doit envoyer une confirmation par email de manière
asynchrone.

**Code de départ :**

```java
// fr.formation.spring.service.EmailService.java
public interface EmailService {
    @Async
    void sendOrderConfirmation(Long orderId);
}

// fr.formation.spring.service.InMemoryEmailService.java (implémentation)
@Service
public class InMemoryEmailService implements EmailService {
    private final Set<Long> sentEmails = new CopyOnWriteArraySet<>();

    @Override
    @Async
    public void sendOrderConfirmation(Long orderId) {
        try {
            Thread.sleep(500); // Simule l'envoi
            sentEmails.add(orderId);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public boolean wasEmailSentForOrder(Long orderId) {
        return sentEmails.contains(orderId);
    }
}
```

**Votre mission :**
Écrivez une classe de test pour `InMemoryEmailService`.

1. Assurez-vous que le test est configuré pour supporter `@Async`.
2. Dans une méthode de test, appelez `sendOrderConfirmation`.
3. Utilisez Awaitility pour attendre que la méthode `wasEmailSentForOrder` retourne `true` pour l'ID de commande que
   vous avez utilisé.

### Correction exercice 13 {collapsible="true"}

```java
// src/test/java/fr/formation/spring/service/InMemoryEmailServiceTest.java
package fr.formation.spring.service;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.scheduling.annotation.EnableAsync;

import static java.util.concurrent.TimeUnit.SECONDS;
import static org.awaitility.Awaitility.await;

@SpringBootTest(classes = InMemoryEmailService.class)
@EnableAsync // Indispensable pour que @Async fonctionne dans le test
class InMemoryEmailServiceTest {

    @Autowired
    private InMemoryEmailService emailService;

    @Test
    void sendOrderConfirmation_shouldEventuallyMarkEmailAsSent() {
        // Arrange
        Long orderId = 123L;

        // Act
        emailService.sendOrderConfirmation(orderId);

        // Assert
        // On attend (au max 2 secondes) jusqu'à ce que la condition soit vraie
        await()
                .atMost(2, SECONDS)
                .until(() -> emailService.wasEmailSentForOrder(orderId));
    }
}
```

### Auto-évaluation

1. (Question ouverte) Quel est le principal problème si l'on tente de tester une méthode `@Async` avec une assertion
   directe juste après son appel ?
2. (QCM) Quelle bibliothèque est spécifiquement conçue pour tester des processus asynchrones en attendant qu'une
   condition devienne vraie ?
    * a) JUnit
    * b) Mockito
    * c) Awaitility
    * d) AssertJ
3. (Question ouverte) Quelle est la différence fondamentale entre `Mono` et `Flux` dans Project Reactor ?
4. (QCM) Pour tester des endpoints WebFlux, on utilise `@WebFluxTest` qui nous injecte un...
    * a) `MockMvc`
    * b) `TestRestTemplate`
    * c) `WebTestClient`
    * d) `StepVerifier`
5. (QCM) `StepVerifier` est un outil utilisé pour :
    * a) Vérifier les étapes d'une transaction de base de données.
    * b) Vérifier chaque événement (émission d'élément, complétion, erreur) dans un `Mono` ou un `Flux`.
    * c) Lancer des requêtes HTTP pas à pas.
    * d) Vérifier les étapes d'un build Maven.

*(Les corrections de l'auto-évaluation seront fournies à la toute fin du support de cours.)*

### Conclusion de la partie

Vous avez apprivoisé le temps ! Vous savez maintenant que le code asynchrone n'est pas une boîte noire impossible à
tester. Grâce à **Awaitility**, vous pouvez écrire des tests robustes et lisibles pour vos méthodes `@Async`, en
attendant patiemment mais intelligemment que le travail soit fait.

Vous avez également fait vos premiers pas dans le monde de la programmation réactive, en apprenant à tester les deux
facettes de ce paradigme : la couche web avec **`WebTestClient`** et la logique métier avec **`StepVerifier`**.

Ces compétences sont de plus en plus recherchées, car les applications modernes se doivent d'être réactives et
efficaces. Vous êtes maintenant prêt à aborder le dernier chapitre de notre formation, où nous synthétiserons tout ce
que nous avons appris en explorant les bonnes pratiques et les stratégies de test avancées.