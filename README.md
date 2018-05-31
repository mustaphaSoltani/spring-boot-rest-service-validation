
---
# Implémentation de la validation pour les services RESTful avec Spring Boot
---
## Objectif:

Dans cette application nous allons implémenter des validations efficaces pour une API / un service REST avec Spring Boot

Nous apprenons alors les taches suivantes:

* Qu'est-ce que la validation?

* Pourquoi on a besoin de validation?

* Qu'est-ce que Hibernate Validator?

* Qu'est-ce que Bean Validation API?

* Quelles sont les capacités de validation par défaut fournies par Spring Boot?

* Comment implémenter Bean Validation API?

* Comment implémenter la validation avec Bean Validation API?

## Structure du code de l'application:

Les fichiers suivants contiennent les composants importants du projet que nous allons créer. Quelques détails:

*  ``` SpringBoot2RestServiceApplication.java  ``` - La classe Spring Boot Application générée avec Spring Initializer. Cette classe execute le point de départ pour l'application.

*  ``` pom.xml  ``` - Contient toutes les dépendances nécessaires pour construire ce projet. Nous utiliserons Spring Boot Starter AOP.

*  ``` Student.java  ``` - Entité Student JPA

*  ``` StudentRepository.java  ``` - Repository JPA Student. Ceci est créé en utilisant Spring Data JpaRepository.

*  ```StudentResource.java  ``` - Spring Rest Controller exposant tous les services sur la ressource Student.

*  ``` CustomizedResponseEntityExceptionHandler.java  ``` - Composant pour implémenter la gestion des exceptions globales et personnaliser la réponse en fonction du type d'exception.

*  ``` ErrorDetails.java  ``` - Bean de réponse à utiliser lorsque des exceptions sont générées par l'API.

*  ```StudentNotFoundException.java  ``` - Exception rejetée des ressources lorsque l'étudiant n'est pas trouvé.

*  ```data.sql  ``` - Données initiales pour la table des étudiants. Spring Boot exécute ce script après la création des tables à partir des entités.

## Outils:

* Maven 3.0+ est l'outil de construction

* l'IDE préféré nous utilisons Intellij.

* JDK 1.8+

## Qu'est-ce que la validation?

On attends un certain format de demande pour notre service RESTful et on doit excepter des éléments de notre requête pour avoir certains types de données, certaines contraintes de domaine.

Alors, lorsque quelque chose dans la requête n'est pas valide, on doit retourner une réponse d'erreur appropriée:

- Un message clair indiquant ce qui n'a pas fonctionné? Quel champ a une erreur et quelles sont les valeurs acceptées? Qu'est-ce que le consommateur peut faire pour corriger l'erreur?

- Statut de réponse correcte Mauvaise demande.

- N'incluer pas d'informations sensibles dans la réponse.

L'état de réponse recommandé pour l'erreur de validation est -> 400 - BAD REQUEST

### Validation par défaut avec Spring Boot

Spring Boot fournit une bonne implémentation par défaut pour la validation des services RESTful. Examinons rapidement les fonctionnalités de gestion des exceptions par défaut fournies par Spring Boot.

#### * Mauvais type de contenu

Si vous utilisez Type de contenu en tant que application/xmlet que ce n'est pas pris en charge par votre application, Spring Boot par défaut renvoie un état de réponse de 415 - Unsupported Media Type

#### * Contenu JSON non valide

Si vous envoyez un contenu JSON invalide à une méthode qui attend un corps, vous obtiendrez une 400 - Bad Request

#### * JSON valide avec des éléments manquants

Cependant, si vous envoyez une structure JSON valide avec des attributs / éléments manquants / invalides, l'application exécutera la requête avec toutes les données disponibles.

##### Exemple :

1- La requête POST  ``` http://localhost:8080/students``` est exécutée avec un statut de -> 201 Created , et un Contenu de la demande vide :
```bash
{  
}
```
2- La requête POST ```http://localhost:8080/students```  est exécutée avec un statut de -> 201 Created et un demande de contenu :
```bash
{
    "name1": null,
    "passportNumber": "A12345678"
}
```
On peut remarquer que la requête ci-dessus a un attribut name1 non valide.

3- C'est la réponse lorsque on lance un GET pour http://localhost:8080/students
```bash
[{"Id": 1, "name": null, "numéro de passeport": null},
 {"id": 2, "nom": null, "numéro de passeport": "A12345678"},
 {"id": 10001, "Name": "Ranga", "passportNumber": "E1234567"}, 
{"id": 10002, "nom": "Ravi", "passportNumber": "A1234568"}]
```
On peut  voir que les deux ressources qui ont été créées avec les ID 1 et 2 avec des valeurs NULL pour les valeurs qui n'étaient pas disponibles. Les éléments / attributs invalides sont ignorés

## Personnaliser les validations:

Pour personnaliser la validation, nous utiliserons Hibernate Validator, qui est l'une des implémentations de l'api de validation du bean.

Nous obtenons Hibernate Validator gratuitement lorsque nous utilisons Spring Boot Starter Web.

Nous pouvons donc commencer à implémenter les validations.

### Implémentation des validations sur le bean

Ajoutons quelques validations au bean Student. Nous utilisons ```@Size``` pour spécifier la longueur minimale et aussi un message quand une erreur de validation se produit.
```bash
@Entity
public class Student {
  @Id
  @GeneratedValue
  private Long id;
  
  @NotNull
  @Size(min=2, message="Name should have atleast 2 characters")
  private String name;
  
  @NotNull
  @Size(min=7, message="Passport should have atleast 2 characters")
  private String passportNumber;
```
Bean Validation API fournit un certain nombre de ces annotations. La plupart d'entre eux sont explicites comme: DecimalMax - DecimalMin - Digits - Email - Future - FutureOrPresent - Max - Min - egative - NegativeOrZero - NotBlank - NotEmpty - NotNull - Null - Past - PastOrPresent - Pattern - Positive - PositiveOrZero

### Activation de la validation sur la ressource

On va ajouter l'annotation @Valid avant l'annotation @RequestBody.
```bash
public ResponseEntity<Object> createStudent(@Valid @RequestBody Student student) {
```
### Personnalisation de la réponse de validation
Définissons un simple bean de réponse d'erreur.
```bash
public class ErrorDetails {
  private Date timestamp;
  private String message;
  private String details;

  public ErrorDetails(Date timestamp, String message, String details) {
    super();
    this.timestamp = timestamp;
    this.message = message;
    this.details = details;
  }
  ```
Définissons maintenant un ```@ControllerAdvice``` pour gérer les erreurs de validation. Nous faisons cela en utilisant la méthode prioritaire ```handleMethodArgumentNotValid(MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatus status, WebRequest request) ```  dans le ```ResponseEntityExceptionHandler```.
```bash
@ControllerAdvice
@RestController
public class CustomizedResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

  @Override
  protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
      HttpHeaders headers, HttpStatus status, WebRequest request) {
    ErrorDetails errorDetails = new ErrorDetails(new Date(), "Validation Failed",
        ex.getBindingResult().toString());
    return new ResponseEntity(errorDetails, HttpStatus.BAD_REQUEST);
  } 
  ```
Pour utiliser ```ErrorDetails``` pour renvoyer la réponse d'erreur, définissons un ControllerAdvice comme indiqué ci-dessous.
```bash
@ControllerAdvice
@RestController
public class CustomizedResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

  @ExceptionHandler(StudentNotFoundException)
  public final ResponseEntity<ErrorDetails> handleUserNotFoundException(StudentNotFoundException ex, WebRequest request) {
    ErrorDetails errorDetails = new ErrorDetails(new Date(), ex.getMessage(),
        request.getDescription(false));
    return new ResponseEntity<>(errorDetails, HttpStatus.NOT_FOUND);
  }
  ```
Lorsque nous exécutons une requête avec des attributs ne correspondant pas à la contrainte, nous obtenons un état ```404 BAD Request```.

Demande
```bash
{
    "name": "",
    "passportNumber": "A12345678"
  }
  ```
On obtient également un corps de réponse indiquant ce qui ne va pas!
```bash
{
  "timestamp": 1512717715118,
  "message": "Validation Failed",
  "details": "org.springframework.validation.BeanPropertyBindingResult: 1 errors\nField error in object 'student' on field 'name': rejected value []; codes [Size.student.name,Size.name,Size.java.lang.String,Size]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [student.name,name]; arguments []; default message [name],2147483647,2]; default message [Name should have atleast 2 characters]"
}
```

![capture](https://user-images.githubusercontent.com/28655112/40789603-b359b4d2-64ea-11e8-81cf-0f694eea47b3.PNG)
