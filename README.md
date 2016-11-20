# Objectif

Dans ce `hands-on-lab` nous allons nous intéresser aux architectures de microservices et à la problématique de leur réactivité aux évènements d'un système commun.

Un microservice est dédié à une tâche atomatique, permettant de rendre nos architectures plus faciles à `scaler` et à superviser.
Toutefois, la multiplication des processus mène à une exécution de nos cas d'utilisation au sein de transactions distribuées, soulevant la question de leur gestion.

Une solution serait de créer des microservices composites, ou bien, pour ceux qui n'ont pas peur des `ESB`, afin de gérer la transaction.
Sinon, on peut aussi piloter nos microservices par les évènements, en les rendant réactifs aux résultats des processus en amont de la transaction distribuée dont ils font partie.

	+------------+
	| Service A  |
	|------------|
	|  ETAPE A   |
	+------------+
		  ^
		  | Ecoute
		  |
	+------------+
	| Service B  |
	|            |
	|  ETAPE B   |
	+-----+------+
		  ^
		  | Ecoute
		  |
	+-----+------+
	|  Service C |
	|------------|
	|   ETAPE C  |
	+------------+

C'est à ce type de solution que nous allons nous intéresser en s'appuyant sur l'écosystème `Java`.
L'idée est de se pencher sur les technologies innovantes qui seront stables à l'horizon 2017 plutôt que de travailler avec des fonctionnalités `production ready`.  

# Technologies

Nous allons créer notre application web à partir de `Spring Boot` ainsi que le module experimental `Reactive Web` de `Spring 5`.
Pour créer notre application, on pourra utiliser http://start.spring.io avec le module `Reactive Web` (pensez bien à choisir `Spring Boot 2.0.0`).
Pour le support des `Reactive Streams`, l'équipe `Spring` à décider d'utiliser `Reactor`: https://github.com/reactor/reactor-core

On s'appuiera sur une base de données orientée document: `Couchbase`.
Je vous fournirai l'adresse d'une instance hébergée chez Amazon.

Enfin, si on a le temps, on essayera de faire transiter nos évènements via un `MOM` (Message Oriented Middleware): `Kafka`.
Il suffira de la télécharger ici (`0.10.0.1`): https://kafka.apache.org/downloads.html

`Java 8` sera nécessaire: http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

# Sujet

Nous allons implémenter un chat basique où plusieurs personnes peuvent s'échanger des messages.
On souhaite stocker tous nos message en base de données.
Le code sera structuré de façon à pouvoir très facilement isoler chaque étape dans une méthode spécifique, qui pourraient être hébergées dans des microservices dédiés :

* Ecriture et envoi d'un message
* Enregistrement du message
* Réception du message par les autres utilisateur

# Exercice 1

Créer un microservice spring boot en charge d'exposer 3 `endpoints`:

* `/` qui affichera la page `HTML` du chat
* `/message` (`GET`) pour ouvrir une connexion `SSE` suspendue par `Spring` utilisée pour recevoir les messages
* `/message` (`POST`) pour ouvrir un envoyer un message à tous les autres utilisateurs connectés

## Aide côté front

Pour initialiser une connexion `SSE`, vous pouvez vous appuyer sur l'example suivant :

    var eventSource = new EventSource("/message");
    eventSource.onmessage = function(e) {
        console.log(e.data);
	};
	
## Aide côté Back

Afin de créer un flux de message, on peut s'appuyer sur l'élément [Flux](http://projectreactor.io/core/docs/api/reactor/core/publisher/Flux.html) de `Reactor`.
Si notre `endpoint` retourne un `Flux<ServerSentEvent>`, `Spring` va correctement interprêter la connexion `SSE` et la suspendre.
On souhaite modéliser un méchanisme de `pubsub` (Publish/Subscribe) où plusieurs clients, les connexions `SSE`, font un `susbcribe` sur un `publisher` commun (le WS `/message`).
Le [TopicProcessor](http://projectreactor.io/core/docs/api/reactor/core/publisher/TopicProcessor.html) est un `Flux` qui permet de créer ce type de comportement.
Inspirez-vous d'un exemple ici: https://github.com/reactor/reactor-core#async-pub-sub--topicprocessor

# Exercice 2

Lorsque l'on poste un message, nous allons maintenant le sauvegarder dans `Couchbase` avant de le soumettre au `TopicProcessor`.
Couchbase propose un `driver` asynchrone permettant de réaliser des opérations selon le stanndard des `Reactive Streams`.
`RxJava` est utilisé par `Couchbase` plutôt que `Reactor`.
Cela n'a pas d'importance, on peut manipuler un `Observable` auquel on peut souscrire un `handler` notifié lorsque notre message est suvegardé.
L'idée est d'envoyer le message au `TopicProcessor` qu'une fois que notre message a été sauvegardé.

Un exemple se trouve ici: http://developer.couchbase.com/documentation/server/4.0/sdks/java-2.2/documents-creating.html

# Exercice 3

Dans l'exercice précédent nous, avons créé un couplage fort entre la sauvegarde d'un document et l'envoi d'un message aux connexions `SSE`.
Cela est contraire aux architecture de microservice.
Nous allons adopter une méthode différente: enregistrer le document et démarrer un processus annexe qui sera notifié par `Couchbase` lorsque le document est enregistré.
Pour cela, on va s'appuyer sur le `DCP` (database change protocol) supporté par `Couchbase`.

Un exemple se trouve ici : https://github.com/couchbaselabs/java-dcp-client#basic-usage

Je vous donnerai l'adresse d'un serveur `Couchbase` à utiliser.
C'est que les choses deviendront intéressantes, car cela nous permettra de s'envoyer des messages les uns aux autres :-)

# Exercice 4

TODO (Kafka)
