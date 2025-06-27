# Module 2 / Chapitre 6 : Gestion des Utilisateurs en Base de Données

## L'essentiel

### Objectifs pédagogiques

À la fin de ce chapitre, vous serez capable de :

* **Remplacer** la gestion des utilisateurs en mémoire par une source de données persistante (JPA).
* **Implémenter** l'interface `UserDetailsService` pour connecter Spring Security à votre base de données.
* **Adapter** votre entité utilisateur pour qu'elle respecte le contrat `UserDetails` de Spring Security.
* **Stocker** des mots de passe hachés avec BCrypt dans la base de données et les vérifier.
* **Mettre en place** une stratégie pour initialiser la base de données avec des utilisateurs de test.

### Introduction : Du Registre Papier au Système d'Information

Jusqu'à présent, notre application TaskMaster s'appuyait sur un `InMemoryUserDetailsManager`. C'est comme si la
réceptionniste de notre immeuble avait la liste des employés sur un post-it. C'est rapide et pratique pour quelques
tests, mais totalement irréaliste. Que se passe-t-il si on redémarre le serveur ? Tous les utilisateurs disparaissent !
Et comment gérer des milliers d'utilisateurs ?

Dans ce chapitre, nous allons professionnaliser notre système d'authentification. Nous allons connecter Spring Security
au véritable système d'information de notre application : sa base de données. Nous allons apprendre à dire à Spring : "
Ne cherche plus les utilisateurs sur un bout de papier. Va les chercher dans la table `users` de la base de données, qui
est la source de vérité."

C'est l'étape la plus cruciale pour rendre votre système de sécurité robuste, persistant et évolutif.

### Le pont entre Spring Security et vos données : `UserDetailsService`

Le cœur de cette intégration est une interface très simple de Spring Security : `UserDetailsService`. Elle n'a qu'une
seule méthode :

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

Le rôle de cette interface est de servir de **pont** (ou d'adaptateur) entre le monde de Spring Security et votre schéma
de données. Quand un utilisateur tente de se connecter, Spring Security appelle cette méthode avec le nom d'utilisateur
saisi. Votre travail est d'implémenter cette méthode pour :

1. Chercher l'utilisateur dans votre base de données (via son repository).
2. Si l'utilisateur n'est pas trouvé, lancer une `UsernameNotFoundException`.
3. Si l'utilisateur est trouvé, le retourner sous la forme d'un objet qui implémente l'interface `UserDetails`.

### Étape 1 : Adapter notre entité `User` pour être un `UserDetails`

Spring Security a besoin d'un objet qui respecte un certain contrat : l'interface `UserDetails`. Cette interface définit
les méthodes pour obtenir le nom d'utilisateur, le mot de passe (haché !), les rôles (autorités), et le statut du
compte (expiré, verrouillé...).

La façon la plus directe est de faire en sorte que notre entité `User` implémente `UserDetails`.

**`fr/formation/spring/taskmaster/model/User.java` (mise à jour)**

```java
package fr.formation.spring.taskmaster.model;

import jakarta.persistence.*;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.Collections;

@Entity
@Table(name = "users")
public class User implements UserDetails { // On implémente UserDetails

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    private String role; // Ex: "ROLE_USER", "ROLE_ADMIN"

    // ... autres champs (email, etc.) ...

    // --- Méthodes de l'interface UserDetails ---

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // Transforme notre simple String "role" en une collection d'autorités
        return Collections.singletonList(
                new SimpleGrantedAuthority(this.role)
        );
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    // Pour la démo, on considère que les comptes n'expirent jamais
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }

    // --- Getters et Setters pour les autres champs ---
}
```

### Étape 2 : Mettre à jour le `UserRepository`

Nous avons besoin d'une méthode pour trouver un utilisateur par son `username`.

**`fr/formation/spring/taskmaster/repository/UserRepository.java` (mise à jour)**

```java
package fr.formation.spring.taskmaster.repository;

import fr.formation.spring.taskmaster.model.User;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {

    // Spring Data JPA va automatiquement générer l'implémentation
    Optional<User> findByUsername(String username);
}
```

### Étape 3 : Créer notre implémentation de `UserDetailsService`

C'est ici que nous construisons le pont.

**`fr/formation/spring/taskmaster/service/JpaUserDetailsService.java` (nouveau fichier)**

```java
package fr.formation.spring.taskmaster.service;

import fr.formation.spring.taskmaster.repository.UserRepository;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
public class JpaUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    // Injection du repository via le constructeur
    public JpaUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        // On cherche l'utilisateur dans la base de données
        return userRepository.findByUsername(username)
                // Si non trouvé, on lève une exception que Spring Security gèrera
                .orElseThrow(() ->
                        new UsernameNotFoundException(
                                "User " + username + " not found"
                        )
                );
    }
}
```

```plantuml
@startuml
!theme vibrant
title Fl flot de recherche d'utilisateur

skinparam componentStyle rectangle

package "Spring Security" {
    component "AuthenticationManager" as AuthManager
}

package "Notre Application" {
    component "JpaUserDetailsService" as JpaService
    database "Base de Données" as DB
    
    package "Repository" {
        component "UserRepository" as UserRepo
    }
    
    package "Model" {
        entity "User (implémente UserDetails)" as UserEntity
    }
}

AuthManager -> JpaService : 1. loadUserByUsername("john.doe")
JpaService -> UserRepo : 2. findByUsername("john.doe")
UserRepo -> DB : 3. SELECT * FROM users WHERE username = ?
DB --> UserRepo : 4. Ligne de l'utilisateur
UserRepo --> JpaService : 5. Optional<User>
JpaService --> AuthManager : 6. Retourne l'objet User (UserDetails)

note on link of JpaService and DB
    L'objet retourné contient le mot de passe HACHÉ.
    Spring Security va ensuite le comparer avec le
    mot de passe (haché aussi) fourni par l'utilisateur.
end note
@enduml
```

### Étape 4 : Nettoyer `SecurityConfig`

La beauté du système est que notre `SecurityConfig` devient encore plus simple. Spring Boot est assez intelligent pour
détecter qu'il existe un bean de type `UserDetailsService` dans le contexte de l'application et l'utiliser
automatiquement.

**`fr/formation/spring/taskmaster/config/SecurityConfig.java` (mise à jour)**

```java

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    // ... bean SecurityFilterChain ...
    // ... bean PasswordEncoder ...

    // SUPPRIMEZ le bean UserDetailsService qui utilisait InMemoryUserDetailsManager !
    /*
    @Bean
    public UserDetailsService userDetailsService(PasswordEncoder encoder) {
        // ... code à supprimer ...
    }
    */
}
```

C'est tout ! En retirant le bean en mémoire, Spring va automatiquement se brancher sur notre `JpaUserDetailsService`.

---

## Pour aller plus loin

### Initialiser la base de données (Data Seeding)

Notre système est prêt, mais notre base de données est vide ! Comment se connecter ? Nous devons insérer nos
utilisateurs de test au démarrage de l'application. La meilleure façon de le faire dans Spring Boot est d'utiliser un
`CommandLineRunner`. C'est un bean dont la méthode `run()` est exécutée une seule fois, juste après le démarrage de
l'application.

**`fr/formation/spring/taskmaster/config/DataSeeder.java` (nouveau fichier)**

```java
package fr.formation.spring.taskmaster.config;

import fr.formation.spring.taskmaster.model.User;
import fr.formation.spring.taskmaster.repository.UserRepository;
import org.springframework.boot.CommandLineRunner;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;

@Component
public class DataSeeder implements CommandLineRunner {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public DataSeeder(UserRepository userRepository,
                      PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public void run(String... args) throws Exception {
        // On ne crée les utilisateurs que si la base est vide
        if (userRepository.count() == 0) {
            // Création de l'utilisateur standard
            User user = new User();
            user.setUsername("john.doe");
            // Le mot de passe doit être encodé avant d'être sauvegardé !
            user.setPassword(passwordEncoder.encode("pass1"));
            user.setRole("ROLE_USER");
            // ... setter d'autres champs si nécessaire
            userRepository.save(user);

            // Création de l'administrateur
            User admin = new User();
            admin.setUsername("jane.smith");
            admin.setPassword(passwordEncoder.encode("pass2"));
            admin.setRole("ROLE_ADMIN");
            userRepository.save(admin);

            System.out.println(">>> Utilisateurs de test créés !");
        }
    }
}
```

### Découplage : Le Pattern "Wrapper"

Faire implémenter `UserDetails` par notre entité JPA est pratique, mais crée un **couplage** entre notre modèle de
domaine (`User`) et un framework de sécurité. Certains architectes préfèrent garder les entités "pures" (simples POJOs).

Une solution plus élégante est d'utiliser un pattern "Wrapper" ou "Adapter". On crée une classe dédiée qui implémente
`UserDetails` et qui "enveloppe" notre entité `User`.

**1. La classe `SecurityUser` (le wrapper)**

```java
public class SecurityUser implements UserDetails {
    private final User user; // Notre entité "pure"

    public SecurityUser(User user) {
        this.user = user;
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    // ... on délègue tous les appels à l'objet user ...
}
```

**2. Adapter le `JpaUserDetailsService`**

```java

@Service
public class JpaUserDetailsService implements UserDetailsService {
    // ...
    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
                .orElseThrow(/* ... */);

        // On retourne le wrapper, pas l'entité directement
        return new SecurityUser(user);
    }
}
```

Avec cette approche, votre entité `User` n'a plus besoin d'implémenter `UserDetails`. Elle reste un simple objet de
données, et toute la logique de sécurité est contenue dans des classes spécifiques à la sécurité. C'est une excellente
pratique pour les projets de grande envergure.

### Exercice 7 : Migration vers une base de données pour TaskMaster

<procedure title="Migration complète de l'authentification vers JPA" id="exercice-2-1">
    <p>
    L'objectif est de migrer entièrement le système d'authentification de notre projet <strong>TaskMaster</strong> de <code>InMemoryUserDetailsManager</code> vers une base de données H2 persistée via JPA.
    </p>
    <ol>
        <li>Modifiez l'entité <code>fr.formation.spring.taskmaster.model.User</code> pour qu'elle implémente l'interface <code>UserDetails</code>. Assurez-vous d'implémenter toutes les méthodes requises. Le champ `role` (String) devra être transformé en <code>Collection&lt;GrantedAuthority&gt;</code> dans la méthode <code>getAuthorities()</code>.</li>
        <li>Mettez à jour l'interface <code>UserRepository</code> pour y inclure la méthode <code>Optional&lt;User&gt; findByUsername(String username)</code>.</li>
        <li>Créez la classe <code>JpaUserDetailsService</code> dans le package <code>service</code>. Elle doit implémenter <code>UserDetailsService</code>, injecter <code>UserRepository</code> et implémenter correctement la méthode <code>loadUserByUsername</code>.</li>
        <li>Modifiez votre classe <code>SecurityConfig</code> en supprimant complètement le bean <code>UserDetailsService</code> qui créait les utilisateurs en mémoire.</li>
        <li>Créez la classe <code>DataSeeder</code> dans le package <code>config</code>. Elle doit implémenter <code>CommandLineRunner</code> et injecter <code>UserRepository</code> et <code>PasswordEncoder</code>. Dans sa méthode <code>run</code>, elle doit créer et sauvegarder les utilisateurs <code>john.doe</code> (ROLE_USER) et <code>jane.smith</code> (ROLE_ADMIN) avec des mots de passe encodés, uniquement si la base de données est vide.</li>
        <li>Lancez l'application. Vérifiez les logs pour vous assurer que vos utilisateurs sont bien créés. Tentez de vous connecter avec <code>john.doe</code> / <code>pass1</code> et <code>jane.smith</code> / <code>pass2</code>. Vérifiez que les règles d'autorisation fonctionnent toujours.</li>
    </ol>
</procedure>

### Correction exercice 7 {collapsible="true"}

<p>Si vous avez bien suivi toutes les étapes, voici les fichiers clés de votre application dans leur état final pour cet exercice.</p>

**1. `User.java` (modifiée)**

```java
// Contenu identique à celui montré dans la section "L'essentiel".
// Il est crucial d'implémenter UserDetails et toutes ses méthodes.
```

**2. `UserRepository.java` (modifiée)**

```java
// Contenu identique à celui montré dans la section "L'essentiel".
```

**3. `JpaUserDetailsService.java` (nouvelle)**

```java
// Contenu identique à celui montré dans la section "L'essentiel".
```

**4. `SecurityConfig.java` (modifiée/nettoyée)**

```java
package fr.formation.spring.taskmaster.config;
// ... imports ...

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        // La configuration du filter chain ne change pas
        http
                .authorizeHttpRequests(/* ... */)
                .formLogin(/* ... */)
                .logout(/* ... */)
                .csrf(/* ... */);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // Le bean UserDetailsService a été supprimé. Spring utilisera le nôtre.
}
```

**5. `DataSeeder.java` (nouvelle)**

```java
// Contenu identique à celui montré dans la section "Pour aller plus loin".
```

<p>Au lancement, vous devriez voir le message ">>> Utilisateurs de test créés !" dans la console. Vous pouvez ensuite vous connecter avec les identifiants définis, et tout fonctionnera comme avant, mais cette fois, les données proviennent d'une base de données ! Vous pouvez même vous connecter à la console H2 (`http://localhost:8080/h2-console`) et exécuter `SELECT * FROM USERS` pour voir vos utilisateurs, avec leurs mots de passe bien hachés.</p>

### Auto-évaluation

1. **Question ouverte :** Quel est le rôle de l'interface `UserDetailsService` et pourquoi est-elle si fondamentale pour
   connecter Spring Security à une source de données personnalisée ?
2. **QCM :** Quelle exception devez-vous lancer dans `loadUserByUsername` si l'utilisateur n'est pas trouvé ?
    * a) `IllegalStateException`
    * b) `UserNotFoundException`
    * c) `UsernameNotFoundException`
    * d) `AccessDeniedException`
3. **QCM :** Comment Spring Security trouve-t-il votre implémentation personnalisée de `UserDetailsService` (par exemple
   `JpaUserDetailsService`) ?
    * a) Il faut le déclarer explicitement dans `application.properties`.
    * b) Il scanne le classpath pour toute classe dont le nom finit par `UserDetailsService`.
    * c) Grâce à l'injection de dépendances, il cherche un bean de type `UserDetailsService` dans le contexte de
      l'application.
    * d) Il faut surcharger une méthode `configure(AuthenticationManagerBuilder auth)`. (ancienne méthode)
4. **Question ouverte :** Expliquez pourquoi vous ne devez jamais, au grand jamais, stocker un mot de passe en clair
   dans la base de données et comment le `PasswordEncoder` et le `DataSeeder` travaillent ensemble pour éviter cela.
5. **QCM :** Pour exécuter du code une seule fois au démarrage de l'application (par exemple, pour initialiser une base
   de données), une bonne pratique Spring Boot est de créer un bean qui implémente :
    * a) `ApplicationRunner` ou `CommandLineRunner`
    * b) `InitializingBean`
    * c) `StartupRunner`
    * d) `Runnable`

### Conclusion

Félicitations ! Vous venez de réaliser l'une des intégrations les plus importantes et les plus courantes avec Spring
Security. Votre application dispose désormais d'un système d'authentification de niveau professionnel, basé sur des
données persistantes.

Vous avez appris à façonner vos données pour qu'elles "parlent le langage" de Spring Security via l'interface
`UserDetails`, et à construire le pont entre les deux mondes avec `UserDetailsService`. Vous savez maintenant comment
gérer les mots de passe de manière sécurisée en base de données et comment initialiser votre application avec des
données de test.

Maintenant que la base est solide, nous pouvons commencer à construire des choses plus complexes par-dessus. Et si le
mécanisme d'authentification par formulaire ne suffisait pas ? Et si nous devions implémenter une logique
d'authentification très spécifique ? Le prochain chapitre nous plongera dans la personnalisation avancée de
l'authentification, en créant nos propres `AuthenticationProvider`.