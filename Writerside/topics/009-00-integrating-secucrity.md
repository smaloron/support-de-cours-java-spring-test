# Chapitre 9 : Tester Sous Haute Surveillance : Intégrer Spring Security (L'essentiel)

### Objectifs pédagogiques

À la fin de cette partie, vous serez en mesure de :

- **Comprendre** les défis posés par Spring Security lors des tests d'endpoints.
- **Utiliser** l'annotation `@WithMockUser` pour simuler un utilisateur authentifié de manière simple.
- **Tester** des règles d'autorisation basées sur les rôles (ex: `hasRole('ADMIN')`).
- **Simuler** une absence d'authentification pour tester les accès non autorisés.
- **Intégrer** les annotations de test de sécurité dans un test `@WebMvcTest`.

### Introduction : Le badge d'accès de l'agent secret

Imaginez que votre API REST est un bâtiment gouvernemental hautement sécurisé. Jusqu'à présent, pour nos tests, nous
avons fait comme si les portes étaient grandes ouvertes. Nous pouvions entrer et sortir à notre guise pour vérifier que
les bureaux (`endpoints`) fonctionnaient.

Mais maintenant, le système de sécurité (`Spring Security`) est activé. Si vous essayez d'accéder à un bureau sans le
bon badge, l'accès vous est refusé. Un simple `GET /api/admin/reports` se solde par un `401 Unauthorized` ou un
`403 Forbidden`. Comment tester nos bureaux si nous ne pouvons même pas passer la porte d'entrée ?

C'est là qu'intervient le module de test de Spring Security. Il nous fournit une série de "faux badges" (
`@WithMockUser`) que nous pouvons présenter lors de nos tests. Ces badges nous permettent de dire au système de
sécurité : "Pour ce test-ci, fais comme si j'étais l'utilisateur 'alex' avec le rôle 'ADMIN'". Cela nous permet de
tester nos endpoints dans des conditions de sécurité réalistes, sans avoir à simuler tout le processus complexe
d'authentification (login/mot de passe).

### Le Défi : Spring Security et les Tests

Lorsque vous ajoutez `spring-boot-starter-security` à votre projet, par défaut, tous les endpoints de votre application
deviennent protégés. Un test qui fonctionnait parfaitement auparavant...

```java
mockMvc.perform(get("/api/users"))
        .ndExpect(status().isOk());
```

...commencera soudainement à échouer avec un statut `401 Unauthorized` ou une redirection vers une page de login. C'est
normal ! Spring Security fait son travail. Notre mission est d'apprendre à "dire" à nos tests avec quelle identité ils
doivent s'exécuter.

### `@WithMockUser` : Votre Faux Badge Universel

L'outil le plus simple et le plus courant pour tester la sécurité est l'annotation `@WithMockUser`. Elle se place sur
une méthode de test (ou sur la classe pour s'appliquer à tous les tests) et crée un faux `Principal` dans le
`SecurityContext` pour la durée du test.

<procedure title="Scénarios de test avec @WithMockUser">
    <step><b>Simuler un utilisateur simple</b></step>
    <p>Le plus simple : simuler un utilisateur nommé "user" avec un mot de passe "password" et un rôle "USER".</p>
    <code-block lang="java">
    @Test
    @WithMockUser
    void testEndpointForAnyLoggedInUser() throws Exception {
        mockMvc.perform(get("/api/profile"))
               .andExpect(status().isOk());
    }
    </code-block>

    <step><b>Simuler un utilisateur avec un nom spécifique</b></step>
    <code-block lang="java">
    @Test
    @WithMockUser(username = "alex.dev")
    void testEndpointAsSpecificUser() throws Exception {
        // ...
    }
    </code-block>
    
    <step><b>Simuler un utilisateur avec des rôles spécifiques</b></step>
    <p>C'est le cas le plus courant pour tester les autorisations.</p>
    <code-block lang="java">
    @Test
    @WithMockUser(roles = {"ADMIN", "AUDITOR"})
    void testEndpointForAdminAndAuditor() throws Exception {
        mockMvc.perform(get("/api/admin/reports"))
               .andExpect(status().isOk());
    }
    </code-block>

    <step><b>Simuler un utilisateur avec des autorisations ("authorities")</b></step>
    <p>Si votre sécurité n'est pas basée sur des rôles (<code>ROLE_...</code>) mais sur des autorisations plus fines.</p>
    <code-block lang="java">
    @Test
    @WithMockUser(authorities = "book:read")
    void testEndpointForUserWithReadPermission() throws Exception {
        // ...
    }
    </code-block>

</procedure>

<warning>
**`roles` vs. `authorities`**
Lorsque vous utilisez `roles = "ADMIN"`, Spring Security le traduit automatiquement en une autorité `ROLE_ADMIN`. Si vous utilisez `authorities = "ADMIN"`, l'autorité sera simplement `ADMIN`. Assurez-vous d'être cohérent avec la configuration de votre application (ex: `http.authorizeHttpRequests(auth -> auth.requestMatchers(...).hasRole("ADMIN"))`).
</warning>

### Mettre en Pratique : Tester un Contrôleur Sécurisé

Imaginons un `AdminController` qui ne devrait être accessible qu'aux administrateurs.

**La configuration de sécurité (simplifiée) :**

```java

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/admin/**").hasRole("ADMIN") // Seuls les admins
                        .anyRequest().authenticated() // Tout le reste nécessite une authentification
                )
                .httpBasic(Customizer.withDefaults())
                .csrf(AbstractHttpConfigurer::disable);
        return http.build();
    }
    // ... bean pour le UserDetailsService, PasswordEncoder etc.
}
```

**Le contrôleur à tester :**

```java

@RestController
@RequestMapping("/api/admin")
public class AdminController {
    @GetMapping("/dashboard")
    public String getAdminDashboard() {
        return "Welcome to the Admin Dashboard!";
    }
}
```

**Le test avec `@WebMvcTest` :**

```java
// src/test/java/fr/formation/spring/api/AdminControllerTest.java
package fr.formation.spring.api;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.context.annotation.Import;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import fr.formation.spring.config.SecurityConfig; // Important d'importer la config

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(AdminController.class)
@Import(SecurityConfig.class) // On doit importer notre config de sécurité
class AdminControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @DisplayName("Devrait retourner 401 pour un utilisateur non authentifié")
    void getDashboard_shouldReturn401_forUnauthenticatedUser() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
                .andExpect(status().isUnauthorized()); // ou 403 si CSRF est activé autrement
    }

    @Test
    @DisplayName("Devrait retourner 403 pour un utilisateur authentifié sans le bon rôle")
    @WithMockUser(username = "user", roles = "USER")
    void getDashboard_shouldReturn403_forUserWithWrongRole() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
                .andExpect(status().isForbidden());
    }

    @Test
    @DisplayName("Devrait retourner 200 pour un utilisateur avec le rôle ADMIN")
    @WithMockUser(username = "admin", roles = "ADMIN")
    void getDashboard_shouldReturn200_forAdminUser() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
                .andExpect(status().isOk());
    }
}
```

<tip>
Dans un `@WebMvcTest`, la configuration de sécurité n'est pas appliquée automatiquement. Vous devez l'importer explicitement avec `@Import(SecurityConfig.class)` pour que les règles (`hasRole`, etc.) soient prises en compte.
</tip>

### Exercice 12 : Protéger un endpoint de suppression

Vous avez un endpoint `DELETE /api/books/{id}` dans votre `BookController`. Vous devez vous assurer que seuls les
utilisateurs ayant le rôle `LIBRARIAN` (bibliothécaire) peuvent l'utiliser.

**Votre mission :**

1. Modifiez votre `SecurityConfig` (ou créez-en une) pour ajouter la règle :
   `requestMatchers(HttpMethod.DELETE, "/api/books/**").hasRole("LIBRARIAN")`.
2. Créez ou modifiez la classe de test `BookControllerTest`. Assurez-vous qu'elle est annotée avec `@WebMvcTest` et
   qu'elle importe votre `SecurityConfig`.
3. Écrivez trois tests pour l'endpoint `DELETE /api/books/1` :
    - Un test pour un utilisateur non authentifié (doit retourner `401 Unauthorized`).
    - Un test avec `@WithMockUser(roles = "READER")` (doit retourner `403 Forbidden`).
    - Un test avec `@WithMockUser(roles = "LIBRARIAN")` (doit retourner `204 No Content`, le statut de succès pour un
      DELETE sans corps de réponse).
4. N'oubliez pas de mocker le service sous-jacent avec `@MockBean`, même si pour ces tests de sécurité, il ne sera pas
   beaucoup sollicité.

### Correction exercice 12 {collapsible="true"}

**1. La configuration de sécurité (exemple) :**

```java

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(HttpMethod.DELETE, "/api/books/**").hasRole("LIBRARIAN")
                        .anyRequest().permitAll() // On simplifie pour l'exercice
                )
                .csrf(AbstractHttpConfigurer::disable);
        return http.build();
    }
}
```

**2. Le contrôleur (simplifié) :**

```java

@RestController
@RequestMapping("/api/books")
public class BookController {
    private final BookService bookService;
    // ... constructeur ...

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteBook(@PathVariable Long id) {
        bookService.deleteById(id);
    }
}
```

**3. La classe de test :**

```java
// src/test/java/fr/formation/spring/api/BookControllerTest.java
package fr.formation.spring.api;

import fr.formation.spring.config.SecurityConfig;
import fr.formation.spring.service.BookService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.context.annotation.Import;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.delete;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(BookController.class)
@Import(SecurityConfig.class)
class BookControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private BookService mockBookService;

    @Test
    void deleteBook_shouldReturn401_forUnauthenticated() throws Exception {
        mockMvc.perform(delete("/api/books/1"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "READER")
    void deleteBook_shouldReturn403_forUserWithWrongRole() throws Exception {
        mockMvc.perform(delete("/api/books/1"))
                .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "LIBRARIAN")
    void deleteBook_shouldReturn204_forLibrarian() throws Exception {
        mockMvc.perform(delete("/api/books/1"))
                .andExpect(status().isNoContent());
    }
}

```

### Auto-évaluation

1. (Question ouverte) Pourquoi un test d'API REST qui fonctionnait bien échoue-t-il souvent avec un statut 401 après l'
   ajout de Spring Security ?
2. (QCM) Quelle annotation est la plus simple pour simuler un utilisateur "admin" avec le rôle "ADMIN" pour un test ?
    * a) `@WithAdminUser`
    * b) `@Test(user="admin", role="ADMIN")`
    * c) `@WithMockUser(username="admin", roles="ADMIN")`
    * d) `@MockBean(type=User.class, name="admin")`
3. (Question ouverte) Quelle est la différence de résultat attendu entre un utilisateur non authentifié et un
   utilisateur authentifié mais n'ayant pas les bons droits pour accéder à une ressource ?
4. (QCM) Dans un `@WebMvcTest`, pourquoi est-il souvent nécessaire d'ajouter `@Import(SecurityConfig.class)` ?
    * a) Pour importer les utilisateurs de test.
    * b) Pour charger les règles de sécurité (`authorizeHttpRequests`, etc.) qui ne sont pas chargées par défaut dans ce
      type de test en tranche.
    * c) Pour désactiver la sécurité pendant le test.
    * d) Pour importer le `MockMvc` bean.
5. (QCM) Un statut HTTP `403 Forbidden` signifie que :
    * a) Le client n'est pas authentifié.
    * b) Le serveur a compris la requête, mais refuse de l'autoriser car le client n'a pas les droits nécessaires.
    * c) La ressource demandée n'existe pas.
    * d) Il y a une erreur dans la requête du client.

*(Les corrections de l'auto-évaluation seront fournies à la toute fin du support de cours.)*

### Conclusion de la partie

Vous avez ajouté une corde essentielle à votre arc de testeur : la capacité de naviguer dans un environnement sécurisé.
Vous savez désormais que tester la sécurité n'est pas une magie noire. Grâce à l'arsenal fourni par
`spring-security-test`, et notamment à l'annotation `@WithMockUser`, vous pouvez simuler n'importe quel type d'
utilisateur et de rôle pour vérifier méticuleusement que vos règles d'autorisation sont bien appliquées.

Vous avez appris à tester les accès autorisés, les accès refusés pour cause de mauvais rôle, et les accès refusés pour
absence d'authentification. C'est une compétence fondamentale pour construire des applications non seulement
fonctionnelles, mais aussi robustes et sûres.

Notre prochain chapitre s'attaquera à un autre aspect parfois délicat : le test du code asynchrone et réactif. Comment
tester quelque chose qui ne se produit pas immédiatement ? C'est ce que nous allons voir.