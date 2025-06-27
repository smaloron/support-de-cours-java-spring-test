# Chapitre 7 : L'Épreuve du Feu : Tests d'Intégration Complets (L'essentiel)

### Objectifs pédagogiques

À la fin de cette partie, vous serez en mesure de :

- **Identifier** les scénarios où un test d'intégration complet avec `@SpringBootTest` est pertinent.
- **Écrire** un test qui simule un flux complet, du contrôleur à la base de données.
- **Utiliser** `TestRestTemplate` pour effectuer de réels appels HTTP sur votre application en cours d'exécution.
- **Comparer** l'approche `MockMvc` avec l'approche `TestRestTemplate`.
- **Apprécier** les avantages et les inconvénients de ce type de test (réalisme vs. lenteur).

### Introduction : Le grand final de la répétition générale

Imaginez que notre application est une troupe de théâtre.

- Chapitre 4 : Nous avons testé chaque acteur (`Service`) individuellement pour s'assurer qu'il connaissait son texte.
- Chapitre 5 : Nous avons testé le metteur en scène (`Controller`) avec des doublures pour les acteurs, pour vérifier
  qu'il donnait les bonnes instructions.
- Chapitre 6 : Nous avons testé les machinistes (`Repository`) pour être sûrs qu'ils pouvaient changer les décors (
  `BDD`) correctement.

Chaque test était isolé, focalisé. Mais maintenant, c'est l'heure de la **répétition générale**. On allume les
projecteurs, on met les vrais acteurs sur scène avec les vrais décors, et on joue une scène entière, de l'ouverture du
rideau au salut final. On veut s'assurer que tout s'enchaîne parfaitement, que les acteurs interagissent bien entre eux
et avec les décors.

Un test d'intégration complet avec `@SpringBootTest` et un serveur web réel, c'est exactement cela. On démarre (presque)
toute l'application et on teste un flux métier complet, de la requête HTTP entrante jusqu'à la modification des données
en base, et vice-versa.

### Quand utiliser `@SpringBootTest` pour un test complet ?

Ce type de test est le plus haut niveau de la pyramide des tests (avant les tests E2E qui incluent l'UI). Il est lent,
lourd, mais il offre le plus haut niveau de confiance. Utilisez-le avec parcimonie pour vos **scénarios les plus
critiques**, les "happy paths" qui définissent la valeur de votre application.

**Bons candidats pour un test d'intégration complet :**

- Le flux d'inscription d'un utilisateur.
- Le processus de passage d'une commande.
- La publication d'un article de blog.
- Un flux qui implique plusieurs services qui collaborent.

Il ne s'agit pas de tester tous les cas d'erreur ici. Ces cas sont mieux couverts par des tests de tranches, plus
rapides. Ici, on veut la confirmation que le "chemin heureux" fonctionne de A à Z.

### Mise en Place de l'Environnement

Pour réaliser un test d'intégration complet, nous allons combiner plusieurs éléments :

1. `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)`: Cette annotation démarre un contexte
   Spring complet **ET** un vrai serveur web (Tomcat) sur un port aléatoire.
2. `@Autowired private TestRestTemplate restTemplate;`: `TestRestTemplate` est un client HTTP fourni par Spring Boot,
   spécialement conçu pour les tests. Contrairement à `MockMvc`, il effectue de **vrais appels réseau** (sur
   `localhost`). Il est parfait pour dialoguer avec notre serveur démarré sur un port aléatoire.
3. `@ActiveProfiles("test")`: Pour s'assurer que nous utilisons une configuration de test, notamment pour la base de
   données (H2, par exemple).
4. `@Transactional`: Nous pouvons toujours l'utiliser pour que la base de données soit nettoyée après chaque test.

```plantuml
@startuml
!theme plain
title "Test d'Intégration Complet"

node "Machine de Test" {
  
  package "Classe de Test" {
    [MonIntegrationTest]
  }

  package "Client HTTP" {
    [TestRestTemplate]
  }

  package "Application Spring Boot" #LightGreen {
    [Controller] <--> [Service] <--> [Repository]
  }

  package "Serveur Web (Tomcat)" {
  }

  package "Base de Données (H2)" {
  }

  [MonIntegrationTest] ..> [TestRestTemplate] : utilise
  [TestRestTemplate] ..> [Serveur Web (Tomcat)] : Appel HTTP réel (localhost:random_port)
  [Serveur Web (Tomcat)] ..> [Controller] : Transmet la requête
  [Repository] ..> [Base de Données (H2)] : Lit/Écrit
}
@enduml
```

### `MockMvc` vs. `TestRestTemplate` : Le Duel

| Caractéristique     | `MockMvc` (`@WebMvcTest`)                                               | `TestRestTemplate` (`@SpringBootTest(webEnvironment=...)`) |
|---------------------|-------------------------------------------------------------------------|------------------------------------------------------------|
| **Type d'appel**    | Simulé, dans le processus. Pas de socket réseau.                        | **Appel HTTP réel** sur `localhost`.                       |
| **Serveur Web**     | Le serveur (Tomcat) n'est pas démarré.                                  | Un **vrai serveur** est démarré.                           |
| **Contexte Spring** | Partiel (couche web uniquement).                                        | **Complet** (ou presque).                                  |
| **Vitesse**         | **Rapide**.                                                             | **Lent**.                                                  |
| **Test de...**      | Principalement le contrôleur et la couche web (filtres, sérialisation). | Le **flux complet** à travers toutes les couches.          |
| **Syntaxe**         | Fluide, avec des `perform` et des `andExpect`.                          | Plus "brute", similaire à `RestTemplate`.                  |
| **Idéal pour...**   | Tester les contrôleurs en isolation.                                    | Tester les scénarios critiques de bout en bout.            |

### Exemple : Tester le flux de création de livre

Reprenons notre exemple de création de livre. Nous allons tester le flux complet :

1. Un client envoie une requête `POST` sur `/api/books`.
2. Le `BookController` la reçoit.
3. Il la transmet au `BookService`.
4. Le `BookService` la valide et l'envoie au `BookRepository`.
5. Le `BookRepository` l'insère dans la base de données H2.

```java
// src/test/java/fr/formation/spring/integration/BookFlowIntegrationTest.java
package fr.formation.spring.integration;

import fr.formation.spring.api.dto.CreateBookDto;
import fr.formation.spring.entity.Book;
import fr.formation.spring.repository.BookRepository;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class BookFlowIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private BookRepository bookRepository; // On l'injecte pour vérifier la BDD

    // On nettoie la BDD après chaque test car nous n'utilisons pas @Transactional
    // pour que les données soient vraiment commitées.
    @AfterEach
    void tearDown() {
        bookRepository.deleteAll();
    }

    @Test
    void createBook_flow_shouldCreateBookInDatabase() {
        // Arrange
        CreateBookDto newBookDto = new CreateBookDto("Le Fléau", "Stephen King");
        long countBefore = bookRepository.count();

        // Act
        // On fait un vrai appel POST
        ResponseEntity<Book> response = restTemplate.postForEntity(
                "/api/books",
                newBookDto,
                Book.class
        );

        // Assert
        // 1. Vérifier la réponse HTTP
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getTitle()).isEqualTo("Le Fléau");
        assertThat(response.getBody().getId()).isNotNull();

        // 2. Vérifier directement en base de données (la preuve ultime)
        long countAfter = bookRepository.count();
        assertThat(countAfter).isEqualTo(countBefore + 1);

        Book bookInDb = bookRepository.findById(response.getBody().getId()).orElseThrow();
        assertThat(bookInDb.getAuthor()).isEqualTo("Stephen King");
    }
}
```

<warning>
**`@Transactional` et `TestRestTemplate`**
Attention, le `@Transactional` avec son rollback par défaut pose problème ici. La transaction du test et la transaction de l'application (déclenchée par l'appel HTTP) sont deux transactions différentes. Les données insérées par l'application ne seraient pas visibles par le test. Une solution simple est de ne pas utiliser `@Transactional` et de nettoyer la base manuellement avec `@AfterEach` comme dans l'exemple.
</warning>

### Exercice 10 : Tester un flux de recherche

Vous allez tester le flux complet de la recherche d'un livre par son ID.

1. Créez une classe de test `BookSearchIntegrationTest`.
2. Annotez-la pour démarrer un serveur sur un port aléatoire et utiliser le profil "test".
3. Injectez `TestRestTemplate` et `BookRepository`.
4. Dans une méthode de test, commencez par insérer directement un livre dans la base de données en utilisant le
   `BookRepository` (c'est notre état initial).
5. Utilisez `TestRestTemplate` pour faire un appel `GET` à l'endpoint `/api/books/{id}` avec l'ID du livre que vous
   venez de créer.
6. Vérifiez que la réponse a un statut `200 OK`.
7. Vérifiez que le corps de la réponse n'est pas nul et que le titre du livre dans la réponse correspond à celui que
   vous avez inséré.

### Correction exercice 10 {collapsible="true"}

```java
// src/test/java/fr/formation/spring/integration/BookSearchIntegrationTest.java
package fr.formation.spring.integration;

import fr.formation.spring.entity.Book;
import fr.formation.spring.repository.BookRepository;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class BookSearchIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private BookRepository bookRepository;

    private Book testBook;

    // On prépare les données avant chaque test
    @BeforeEach
    void setUp() {
        bookRepository.deleteAll(); // On nettoie au cas où
        Book book = new Book("Ça", "Stephen King");
        testBook = bookRepository.save(book);
    }

    @Test
    void getBookById_flow_shouldReturnExistingBook() {
        // Arrange : le livre est déjà en BDD grâce au setUp

        // Act
        ResponseEntity<Book> response = restTemplate.getForEntity(
                "/api/books/{id}",
                Book.class,
                testBook.getId() // On passe l'ID comme variable d'URL
        );

        // Assert
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getId()).isEqualTo(testBook.getId());
        assertThat(response.getBody().getTitle()).isEqualTo("Ça");
    }
}
```

### Auto-évaluation

1. (Question ouverte) Décrivez un scénario d'application pour lequel un test d'intégration complet serait plus pertinent
   qu'un test en tranche.
2. (QCM) Quelle est la principale différence entre `MockMvc` et `TestRestTemplate` ?
    * a) `MockMvc` est plus rapide car il ne démarre pas de serveur web.
    * b) `TestRestTemplate` ne peut pas être utilisé avec `@SpringBootTest`.
    * c) `MockMvc` ne peut tester que des endpoints `GET`.
    * d) `TestRestTemplate` ne peut pas envoyer de corps de requête.
3. (Question ouverte) Pourquoi l'annotation
   `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)` est-elle particulièrement bien adaptée
   aux tests d'intégration complets ?
4. (QCM) Dans un test d'intégration complet qui effectue une écriture en base de données, pourquoi est-il souvent
   préférable de nettoyer la base avec `@AfterEach` plutôt que de se fier au `@Transactional` par défaut ?
    * a) Parce que `@Transactional` ne fonctionne pas avec H2.
    * b) Parce que l'appel HTTP se fait dans une transaction séparée de celle du test, rendant le rollback inefficace
      pour ce scénario.
    * c) Parce que `@AfterEach` est plus rapide que `@Transactional`.
    * d) Parce que `@Transactional` n'est pas compatible avec `@SpringBootTest`.
5. (QCM) L'objectif principal d'un test d'intégration complet est de :
    * a) Tester tous les cas d'erreur et les branches de code d'un service.
    * b) Vérifier que les composants de l'application (contrôleur, service, repository) collaborent correctement pour un
      flux métier critique.
    * c) Tester l'interface utilisateur.
    * d) Remplacer les tests unitaires.

*(Les corrections de l'auto-évaluation seront fournies à la toute fin du support de cours.)*

### Conclusion de la partie

Vous avez atteint le sommet de la pyramide des tests (au sein de votre service) ! Vous savez maintenant comment mener
la "répétition générale" de votre application en testant des flux complets. Vous avez appris à utiliser
`@SpringBootTest` avec un vrai serveur web et à interagir avec lui via `TestRestTemplate`. Vous comprenez surtout les
compromis : ces tests sont une ressource précieuse car ils sont très réalistes, mais ils sont aussi coûteux à exécuter.
Il faut donc les réserver pour vos scénarios les plus importants.

Cette compétence est essentielle, mais le monde réel nous confronte souvent à un autre défi : comment tester notre
application lorsqu'elle dépend de services externes (une autre base de données, un service de messagerie, une autre
API) ? C'est le sujet de notre prochain chapitre, où nous découvrirons la magie de Testcontainers.