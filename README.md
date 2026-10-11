# SprintFramework

Framework MVC/REST minimal en Java pour Jakarta Servlet.

Cette documentation explique le fonctionnement du framework, son lancement,
les étapes des Sprints 0 à 7, les annotations, le binding des paramètres et
les conventions Java nécessaires pour l'utiliser correctement.

## 1. Ce que contient le dépôt

Le projet est une bibliothèque de framework, pas une application web complète.
Il fournit :

- `FrontControllerListener` : découverte et enregistrement des contrôleurs ;
- `FrontControllerServlet` : réception et dispatch des requêtes HTTP ;
- `Mapping` : clé `(URL, méthode HTTP)` utilisée dans la table des routes ;
- `ModelAndView` : retour d'une vue JSP avec des attributs ;
- `ParamBinder` : conversion des paramètres HTTP vers des arguments Java ;
- `JsonUtil` : sérialisation JSON sans dépendance externe ;
- `src.annotation.*` : annotations de contrôleur, de route et de validation ;
- `lib/jakarta.servlet-api-6.0.0.jar` : API nécessaire à la compilation.

Une application utilisatrice doit encore fournir un serveur Jakarta Servlet
(par exemple Tomcat), ses contrôleurs, son fichier de déploiement et ses vues.

## 2. Prérequis

- JDK 21 recommandé (JDK 17 ou supérieur attendu pour le code moderne utilisé) ;
- un serveur compatible Jakarta Servlet 6, par exemple Tomcat 10.1 ;
- un terminal placé à la racine du projet ;
- les classes de l'application dans un package scannable, par défaut
  `tiavina.main`.

Vérifier Java :

```powershell
java -version
javac -version
```

## 3. Compilation

Sous Windows :

```powershell
.\deploy.bat
```

Sous Linux/macOS/Git Bash :

```bash
chmod +x deploy.sh
./deploy.sh
```

Les scripts :

1. suppriment `build` ;
2. créent `build/classes` ;
3. recherchent tous les fichiers `.java` sous `src` ;
4. compilent avec `lib/*` dans le classpath ;
5. créent `Framework.jar`.

Le script Windows utilise `sources.txt` comme liste temporaire de fichiers.
Le script Unix transmet directement la liste produite par `find`.

## 4. Configuration du scan

Le listener lit le paramètre de contexte `base-package`. Exemple conceptuel
dans la configuration de l'application :

```xml
<context-param>
    <param-name>base-package</param-name>
    <param-value>tiavina.main</param-value>
</context-param>
```

Si ce paramètre est absent ou vide, le framework utilise `tiavina.main`.
Ce nom ne change pas les packages internes `src` et `src.annotation` :
il indique le package de l'application dans lequel le listener cherche les
classes annotées `@Controller`.

Le listener utilise aussi :

- `view-prefix`, par défaut `/WEB-INF/jsp/` ;
- `view-suffix`, par défaut `.jsp`.

## 5. Exemple de contrôleur

```java
package tiavina.main;

import src.ModelAndView;
import src.annotation.Controller;
import src.annotation.GetMapping;
import src.annotation.Param;
import src.annotation.UrlApi;

@Controller
public class HomeController {
    @GetMapping("/")
    public ModelAndView home() {
        return new ModelAndView("home").addAttribute("title", "Accueil");
    }

    @GetMapping("/api/hello")
    @UrlApi
    public String hello(@Param(name = "name") String name) {
        return name;
    }
}
```

La déclaration `package tiavina.main;` doit correspondre au chemin compilé
`tiavina/main/HomeController.class`. Les imports indiquent où se trouvent les
types utilisés : les classes du framework sont dans `src`, les annotations
dans `src.annotation`, et les types HTTP dans `jakarta.servlet.*`.

## 6. Architecture d'une requête

1. Le conteneur Servlet démarre l'application.
2. `FrontControllerListener.contextInitialized` lit la configuration.
3. Le listener convertit le package en chemin (`tiavina.main` devient
   `tiavina/main`) et inspecte les classes compilées.
4. Les classes portant `@Controller` sont instanciées.
5. Les méthodes portant une annotation de route sont ajoutées dans une
   `Map<Mapping, List<MethodInfo>>`.
6. `FrontControllerServlet` reçoit `GET` ou `POST`.
7. L'URL est réduite au chemin relatif au contexte.
8. `ParamBinder` construit les arguments de la méthode.
9. La méthode du contrôleur est appelée par réflexion.
10. Le résultat est rendu en texte, en JSON ou en vue JSP selon les annotations
    et le type retourné.

## 7. Sprints

### Sprint 0 — socle Servlet

Ajout de `FrontControllerServlet`, `@Controller`, des scripts de compilation
et de l'API Servlet. Le framework sait recevoir une requête, mais la gestion
des routes reste minimale.

### Sprint 1 — première correction du contrôleur frontal

Une correction ciblée du servlet frontal stabilise le traitement initial des
requêtes. Le principe reste celui d'un point d'entrée unique.

### Sprint 2 — `@RequestMapping`

`@RequestMapping` cible une méthode (`@Target(ElementType.METHOD)`) et reste
disponible à l'exécution (`@Retention(RetentionPolicy.RUNTIME)`). Ses attributs
`url` et `method` décrivent la route.

### Sprint 3 — table de mapping et routes spécialisées

`Mapping` normalise la méthode HTTP et implémente `equals`/`hashCode`, ce qui
permet son utilisation comme clé de `HashMap`. `@GetMapping` et
`@PostMapping` offrent une syntaxe plus lisible que `@RequestMapping`.

### Sprints 4–5 — cycle de vie et vues

`FrontControllerListener` effectue le scan au démarrage. `ModelAndView`
transporte un nom de vue et une map d'attributs. Le servlet place ces attributs
dans la requête puis délègue à un `RequestDispatcher`.

### Sprint 6 — binding et JSON

`ParamBinder` injecte les objets Servlet, convertit les paramètres simples et
construit des objets. `JsonUtil` sérialise les valeurs primitives, chaînes,
dates, maps, collections, tableaux et POJOs.

### Sprint 7 — validation et annotations de binding

Ajout de `BindingException`, `@Param`, `@ParamObject`, `@Required`,
`@Numeric`, `@Range`, `@DateFormat`, `@Url`, `@Get` et `@Post`. Les erreurs de
binding donnent une réponse HTTP 400 au lieu de laisser passer une valeur
incorrecte.

## 8. Comprendre les symboles Java

- `package` place une classe dans un espace de noms ; le chemin des sources et
  le package doivent être cohérents.
- `import` rend un nom de type disponible sans écrire son nom complet.
- `@Override` demande au compilateur de vérifier qu'une méthode redéfinit bien
  une méthode héritée ou d'une interface.
- `@WebListener` permet au conteneur de découvrir le listener.
- `@Controller`, `@GetMapping` et les autres annotations sont des métadonnées
  lues par réflexion à l'exécution.
- `.` accède à un membre (`sce.getServletContext()`).
- `()` appelle une méthode ou déclare ses paramètres.
- `{}` délimite un bloc de code.
- `<>` porte les paramètres génériques (`Map<Mapping, List<...>>`).
- `? :` est l'opérateur conditionnel : condition, valeur si vrai, valeur si
  faux.
- `!= null` protège contre une référence absente.
- `&&` signifie « et » avec évaluation court-circuitée.
- `;` termine une instruction.
- `//` et `/* ... */` introduisent des commentaires.

Chaque virgule sépare des arguments ou des éléments d'une déclaration ; chaque
point-virgule termine une instruction ; aucun de ces signes n'est décoratif :
leur rôle est défini par la grammaire Java et le compilateur les contrôle.

## 9. Comment déterminer les imports

Méthode professionnelle :

1. identifier le type exact utilisé (`HttpServletRequest`, `Map`, etc.) ;
2. rechercher sa documentation Java/Jakarta ;
3. importer le package propriétaire du type ;
4. supprimer les imports inutilisés avec l'IDE ;
5. compiler immédiatement ;
6. ne pas importer une classe uniquement parce que son nom ressemble au besoin.

Exemples :

```java
import java.util.Map;
import jakarta.servlet.http.HttpServletRequest;
import src.annotation.GetMapping;
```

`java.util.Map` appartient au JDK, `jakarta.servlet.http.HttpServletRequest`
appartient à l'API Servlet fournie par le JAR, et `src.annotation.GetMapping`
appartient à ce projet.

## 10. Limites actuelles

- Le dépôt ne contient pas d'application exemple complète ni de vue JSP.
- Le serveur Tomcat n'est pas inclus.
- Le scan dépend de classes compilées et d'un package correctement configuré.
- La réflexion exige un constructeur sans argument pour les contrôleurs et les
  objets liés.
- La compilation et le déploiement doivent être vérifiés dans un environnement
  disposant d'un JDK et d'un conteneur Jakarta Servlet.

## 11. Historique des Sprints

```powershell
git log --oneline --all --graph --decorate
git switch sprint-3
git show --stat sprint-3
git switch main
```

Les changements d'identité, de dates ou d'auteur GitHub doivent rester
traçables. Une modification légitime est publiée avec un commit et une PR
explicites ; il ne faut pas tenter de dissimuler sa provenance.