# Wed&Joy


## Structure de l'app

```text
Wed&Joy tree
├── app ➜ Résultat de la compilation du serveur
├── database ➜ Dossier pour le container mariadb
│   ├── config ➜ Contient les fichiers de configuration de mariadb
│   ├── volume ➜ Contient les données de la base de donnée. Permet la persistance
├── public ➜ Le dossier public du serveur.
			  Contient tout ce qui sera utilisé par le navigateur(images, HTML/css, javascript)
│   └── Vue ➜ Le dossier de notre application Vue
│       ├── dist ➜ Resultat de la compilation du client
│       │   ├── css
│       │   └── js
│       └── src ➜ Code de la vue
│           ├── assets ➜ Assets statiques de la vue
│           ├── components ➜ Components vue.js
│           ├── layouts ➜ Layouts utilisés avec children de vue router
│           └── views ➜ Contient nos "pages"
│               ├── adminUser
│               ├── customProviders
│               ├── messages
│               └── todo
├── sessions ➜ Contient les fichiers de sessions
├── src ➜ Code du serveur
│   ├── App ➜ Contient toutes les fonctions de base comme le lancement du serveur ou la configuration des stratégies de passport
│   │   └── Abstracts ➜ Contient les classes abstraites comme AbstractEntity
│   ├── Controllers ➜ Contient les controlleurs (classes)
│   ├── Entities ➜ Contient les entités (classes)
│   └── Middlewares ➜ Contient les middlewares (classes)
```


## Table des matières
1. [Introduction](#intro)
2. [Fonctionnement](#fonctionnement)
	1. [Exemple](#fonctionnement.exemple)
3. [Le javascript async moderne](#modernjs)
4. [Créer une route express, chemin complet](#routing)
5. [L'authentification avec passport et vuex](#passport)
6. [Point sur l'ajax](#ajax)
7. [Installer et utiliser TypeORM, babel avec NodeJS](#tybano)
	1. [Babel](#babel)
	2. [Les décorateurs](#decorators)
		1. [Exemples](#exemples)
		2. [Liste des décorateurs](#decolist)
			1. [Serveur](#decoserv)
			2. [Client](#decoclient)
	3. [TypeORM](#typeorm)
		1. [Créer une entité](#typeorm.create)
		1. [Insérer une entité](#typeorm.insert)
			1. [Methode 1](#method1)
			1. [Methode 2](#method2)
		2. [Trouver une entité](#typeorm.find)
## Introduction <a name="intro"></a>
Wed&Joy utilise les technologies principales suivantes :

- NodeJS et expressJS pour le backend avec babel
- VueJS pour le frontend avec Vue Loader (webpack + babel spécial Vue)
- MariaDB pour la base de donnée (dans un container Docker pour le developpement)

## Fonctionnement <a name="fonctionnement"></a>
Pour permettre à l'application, que ce soit le serveur ou le frontend, de fonctionner correctement il est nécessaire d'utiliser le transpilleur babel.

Babel est utilisé en tant que commande dans le script "start" présent dans le package.json. Une bonne pratique serait de remplacer la longue commande par un script bash. Une fois babel ayant fait son travail, un dossier "app" va apparaitre à ses côtés contenant le code compilé de notre serveur, et un fichier launcher.bundle.js va apparaitre.

Ce fichier est en fait le résultat de la compilation de launcher.js (qui n'est pas à éxecuter, seulement à modifier).


Le role de babel est de parcourir nos fichiers javascript qui utilisent des fonctionnalités modernes en les transformant en fichier javascript "classique".

> Ce qu'il faut retenir de ce schema, c'est qu'il ne faut pas écrire dans les fichiers issues d'une compilation de babel. Il s'agit des "résultats" pas des "sources". Les sources sont dans un dossier "src" ;-) (le dossier peut s'appeller app, bin ou dist, mais il est assez commun de l'appeller dist. Je l'ai appellé app car dist est deja le dossier resultat de VueJS. C'est juste pour simplifier la gestion dans l'editeur)

### Exemple <a name="fonctionnement.exemple"></a>
Ainsi grace à babel le code suivant :
```javascript
@Get('/', [HtmlRes])
indexAction(req, res) {
	return res.render("index.html");
}
// ... D'autres declaration de fonctions "Action" dans ce fichier
```
est traduit et executé sous cette forme (reformaté car le résultat est sur une seule ligne) :
```javascript
return _createClass(a, [{
	key: "indexAction", value: function c(a, b) {
		return b.render("index.html")
	}
},
 // ... D'autres declaration de fonctions "Action" dans ce tableau
```
Cette version est plus complexe à écrire pour un developpeur mais est bien plus rapide pour une machine.
Un point faible du javascript moderne est :

- Les fonctionnalitées sont experimentales et causent souvent des ralentissements du language
- Leur fonctionnement devient très dépendant de la version de node qui va executer le projet, ou le navigateur qui va executer l'application vue. En effet pour le second cas, seul les version les plus récentes des navigateurs permettent une execution des fonctionnalités ES2016 natives.

Babel permet donc d'utiliser ces fonctionnalités sans toutes ces contraintes au prix d'un temps de compilation. Il est possible grace à cette phase supplémentaire d'étendre les possiblités de notre application, comme mettre un minifier de code *(deja fait)* ou un linter par exemple, pour émettre des erreurs ou avertissements si le code ne réspecte pas la norme ou encore des tests unitaires.

La configuration de babel se fait grace au fichier .babelrc ou babel.config.js, qui est une variante utilisant du javascript (utile pour WebPack)

> **Exception**
La minification du code des entités de TypeORM est à tout prix à éviter. En effet si babel minify le code des classes modèles, il va remplacer tous les noms de variables par des lettres uniques ou double pour gagner de la place. C'est très bien pour notre code "controlleur" mais pour l'ORM c'est catastrophique (il va créer une table "a" "b" "c" etc.)
	**C'est pour cette raison qu'il y a une double compilation dans le package.json**

## Le javascript asynchrone moderne
### L'asynchrone historiquement

Javascript est un langage très fortement asynchrone. A l'origine, il est possible de créer un certain type de fonction en javascript "classique" qui sont dites "asynchrones". Prenons le code suivant :

```javascript
/*
 * Imaginons que cette fonction soit async. Ce n'est pas le cas, en js "classique" il est
 * assez compliqué d'en créer
 */
function maFunctionAsync() {
	console.log("MA ASYNC")
}

console.log("Avant")
maFunctionAsync();
console.log("Après")
```
Nous produit le résultat :
```javascript
Avant
Après
MA ASYNC
```

L'avantage est que si le resultat de l'opération n'est pas nécessaire pour la suite de l'éxecution du programme,
il est possible de continuer sans en attendre la fin. C'est également cette capacité qui va permettre aux developpeurs d'optimiser au maximum leur application.

Néanmoins, il est souvent nécessaire d'effectuer des opérations après l'opération (pour traiter son résultat par exemple) ainsi, notre code devient :

```javascript
/*
 * Imaginons que cette fonction soit async. Ce n'est pas le cas, en js "classique" il est
 * assez compliqué d'en créer
 */
function maFunctionAsync(callback) {
	console.log("MA ASYNC")
	// La fonction fait son travail, puis appelle la fonction "callback" passée en paramètre
	callback("je suis une data");
}

console.log("Avant")
maFunctionAsync(function(data) {
	console.log(data);
	console.log("Programme terminé !");
});
console.log("Après")
```

Affichera :
```javascript
	Avant
	Après
	MA ASYNC
	Je suis une data
	Programme terminé !
```

Mais cela a également posé un gros problème d'architecture de code pendant plusieurs années : le callback hell. Voici un exemple fictif :
```javascript
	var query = db->("SELECT * FROM USERS");
	query.on("data", function(data) {
		data.on("result", function(result) {

			var final = []
			result.forEach(function() {



				if (result.id) {
					final.push(result.id)
					result.isOk = true;

					var newQuery = db->("UPDATE USER ...");
					newQuery.on('end', function(error) {
						if (!error)
							res.send("ok !");

					});
				}
			});


		});
	});
```

Ca devient vite compliqué ! Car on a accès aux données de l'appel asynchrone QUE dans sa callback. De facto tout ce qui utilise la donnée initiale doit être dans une callback, ce qui fait des callback dans des callback etc.

> Anecdote : Un ami avait fait un énorme code JS pris par le "callback hell" et en était venu à réduire son niveau d'indentation jusqu'à 1 espace pour limiter le scrolling horizontal de son editeur. Lui faire découvrir les promises et l'async await a surement changé sa vie pour toujours :)

### Les promises
Les promises, c'est la fonctionnalité ES6 de javascript la plus hot et qui était la plus nécessaire. Elle vient apporter un peut d'ordre à ces brutes de callback.
Les promises sont incontournables dans ce projet. Elles sont omniprésentes dans TypeORM et Axios (qui permet de faire des requêtes HTTP "ajax" depuis la Vue.js)

Utiliser une promise :
```javascript
	maFonctionAsync(param).then(function() {
		console.log("Je suis dans ma callback !")
	});
```
> Note : parfois, surtout pour axios, il peut être utile d'ajouter .catch en cas d'erreur
```javascript
axios.post(...)
	.then(function)
	.catch(function)
```

Avec un peut de travail, notre callback hell de tout à l'heure deviendrait en gros :
```javascript

	query
	.then(function(data) {
		data.then(...)
	})
	...

```

Pour créer une promise, c'est finalement très simple :
```javascript
   function monAppelSql() {
		return new Promise(function(resolve, reject) {
			db->query.on('result', resolve);
		});
   }

const maPromiseSQl = monAppelSql();
console.log(typeof maPromiseSQL) // "Promise"
maPromiseSQL.then(function(data) {
	// ...
});
```

C'est une modification bienvenue, car elle apporte une certaine constance à l'asynchrone et un type spécial "Promise" pour le reconnaitre.
Ca ne résout par contre pas le problème du callback hell, les callback sont controllées mais sont toujours présentes.

### Async & Await

Async Await est une grosse évolution du comportement de l'asynchronise javascript.
Commencons par async.
Le mot clé async permet de simplifier la création de fonction asynchrone. Ainsi :
```javascript
	function async maFunctionAsync() {
		console.log("Après")
		// ...
	}

	maFunctionAsync()
	console.log("Avant")

	// Affiche :
	// Avant
	// Après
```

> Note : Async peut aussi se mettre sur les fonction anonymes passées à then, c'est très interessant quand on le couple à await.

Parlons d'await, le compagnon d'async.
Await permet d'absorber le promise en quelque sorte. Grace aux promises les callbacks ont toujours la même forme : une fonction passée à then. Si le mot clé "await" est placé devant une instruction qui devrait avoir un .then, alors le code suivant :

```javascript
asyncFunc().then(data) {
	console.log(data);
}
```
devient :
```javascript
const data = await asyncFunc();
console.log(data);
```

On tient donc notre anti callback hell :)

Première règle, l'await ne sort jamais sans son async. L'await ne peut être utilisé QUE dans une fonction async. Ce n'est pas souvent un problème, en general, par exemple pour une route express, il suffit d'ajouter "async" devant la fonction parente car express se fiche pas mal que la fonction soit async ou pas.

Exemple :
```javascript
// Dans un controlleur ...
@Get("/maHome")
homeAction(req, res) {
	this.userRepository.find({where: { id: 4 }).then(function(user) {
		res.send(JSON.stringify(user));
	});
}

```
Devient :
```javascript
// Dans un controlleur ...
@Get("/maHome")
async homeAction(req, res) {
	const user = await this.userRepository.find({where: { id: 4 });
	res.send(JSON.stringify(user));
}

```
Le code est plus facile à comprendre, et ne comprend plus de lignes inutiles. Evidement c'est à utiliser que quand l'asynchrone n'est pas nécessaire, car il s'agit néanmoins d'un comportement très utile selon les cas.

## Créer une route express, chemin complet
On commence par créer un controlleur si il n'existe pas dans src/Controllers :
```javascript

"use strict";

import {Controller, Get, Request, Response} from "@decorators/express";
import HtmlRes from "../Middlewares/htmlRes";

@Controller('/') // Le prefixe de l'URL de ce controlleur
export default class FrontController {

    @Get('/', [HtmlRes]) // Le decorateur pour déclarer une GET avec un middleware
    indexAction(req, res) {
        return res.render("index.html");
    }

    @Get('/test') // Tous les decorateurs @Post, @Put etc. fonctionnent de la même manière
    testAction(req, res) {
			  /*
				 * On recoit req et res.
				 * Pour récupérer le contenu d'une requête, on utilise
				 * req.body qui est un objet clé valeur de nos données
				 * On peut utiliser req.headers, nottament dans les Middlewares
				 * pour vérifier l'auth ou le type de requête
				 * Pour récupérer un paramètre d'URL dans le cas d'un get on utilise
				 * req.params comme body.
				 */
        return res.json({test: 'ok'});

				/*
				 * pour res on peut préciser le status de la requête de cette manière
				 * res.status(200).json...
				 * Ca permet de retourner une erreur 403 par exemple si l'user n'est pas autorisé, ce genre de chose (404 pour ressources non trouvée etc.)
				 */
    }

}

```

Il est ensuite nécessaire d'enregister ce controlleur dans src/App/router.js :
```javascript

// On créer un router express classique, commun à toutes nos routes d'API (ajax)
// Il est deja créé dans le fichier router.js, il ne reste qu'à le compléter comme décris en dessous

import FrontController from '../Controllers/FrontController';

const apiRouter = new express.Router();

attachControllers(apiRouter, [

		/*
		 * On passe dans ce tableau toutes les classes de nos controlleurs
		 */
		 FrontController
		 // Désormais front controller est enregistré dans notre routeur express

]);

// Enfin on passe notre router express à express lui même.
// On peut passer des middlewares qui seront executés à chaque requête du router. Par exemple pour api router on oblige l'utilisateur à être connecté, ce qui n'est pas le cas du public router par exemple.
// On peut créer autant de routeur que l'on a besoin, mais je pense que 2 suffisent.
app.use('/api', checkAuthMiddleWare, jsonResMiddleware.use, apiRouter);

```

## L'authentification avec passport.js
Passport fonctionne sur un système appellé les "Stategie".
Il en existe plusieurs, les deux que nous utilisons sont
"local strategy" qui est prévue pour une connexion classique, elle prend un field username et password
Nous utisons egalement "jwt strategy" pour vérifier le token de l'utilisateur (la stratégie le fait toute seule, elle nous passe le payload en cas de succès pour que l'on puisse vérifier que l'utilisateur existe)

Les stratégies sont dans src/App/passport.

Le fonctionnement global de l'auth est le suivant :
L'utilisateur se connecte en envoyant en POST son nom de compte et mot de passe à la route UserController@login.
La route declenche la stratégie passport local-login, ce qui génère un payload, le transforme en json web token et le retourne à la vue.

Dans la vue, la fonction de connexion qui effectue l'ajax POST est dans un equivalent de passport, du moins qui peut être utilisé comme, Vuex.
Le fichier est /public/Vue/src/store.js
A la reception de la confirmation de connexion, le store va :
1. Configurer axios pour lui ajouter le header d'authentification sous cette forme : "Bearer le_token"
2. Sauvegarder dans "l'état de l'application" (le state dans le code) l'user id, le role et si il est connecté ou non.
3. Sauvegarder dans le localstorage le token
(3.) Evidement, à l'initialisation du store on vérifie si le token est dans le local storage pour restaurer la session de l'utilisateur, rendant le state persistant au rechargement de page, fermeture du navigateur temporaire etc.
4. Le store est reactif, autrement dis, dans le html :
```html
<template v-if="store.isLoggedIn">
	Est connecté
</template>
<template v-else>Pas connecté</template>
```
Va changer "par magie" à la connexion de l'utilisateur. Pas besoin d'emettre d'events global compliqués à maintenir et à lire.

> Note : Il ne faut pas en abuser non plus, et garder à l'esprit qu'il n'y a qu'un seul store pour une application. Il ne faut pas le surcharger avec des échanges "locaux" (par exemple les échanges entre un parent et son fils. Il est preferable d'utiliser ou d'implémenter v-model pour ces cas)
Les store doivent être utiliser pour "partager des données" de type "paramètres". Par exemple savoir si l'utilisateur est connecté / son role, ou par exemple la langue ou d'autre paramètres concernant "toute l'application".

## Point sur l'ajax

Il n'y a pas grand chose à dire sur l'ajax en soit :
```
axios({ url: config.api.getServerUrl() + 'users', method: "GET" }).then(response => {
	this.list = response.data.data;
});

```
Si l'utilisateur est connecté, il a deja les bons headers avec le token, du coup il n'y a rien de special à gérer par rapport à avant.
La seule difference est l'utilisation du module local "config" qui contient une methode getServerUrl qui permet d'obtenir une URL depuis "un fichier de config".

Pour une mise en production ça permet de remplacer "localhost" par le nom de domaine du serveur de production en ne changeant l'information qu'à seul endroit.

## Vue class component
Le projet implémente désormais Vue class component, le plugin officiel phare de Vue.js.
Tout est résummé en quelques lignes à cette addresse https://github.com/vuejs/vue-class-component
Un exemple sur le projet est disponible sur le composant "LoginBox".

L'idée est de remplacer la strucrure :
```javascript

Vue.component({

	props: [

	],
	data() {
		return {

		}
	},
	methods: {

	},
	computed: {

	}

});

```

par :
```javascript

@Component()
class MyComponent extends Vue {
	@Prop monProp;

	maData = "test";
	maAutre = undefined;

	maMethode() {

	}

	set monComputedSetter() { // btw j'ai jamais réussi à faire marcher ça avec l'ancien style d'écriture avec data() etc. Meme avec ce système ça marche :)

	}

	get monComputedGetter() {

	}
}

```

Le résultat est plus moderne et permet de faire de l'héritage, et donc de plus facilement architecturer son code en vue de le réutiliser.
(par exemple on peut créer un component "AbstractComponent" qui hérite de vue, puis faire heriter MyComponent de AbstractComponent a la place de Vue)
MyComponent sera comme AbstractComponent, mais avec des trucs en plus qui lui sont propre. Ce principe peut être éxploité à l'infinie.

> Bonus : Du coup fini d'oublier des , entre les blocs d'un composant ;)

## TypeORM, les décorateurs et babel <a name="tybano"></a>
Pour installer babel et le rendre compatible avec typeORM, il est nécessaire de vérifier dans le package.json que babel et ses composants sont en version 7.2.0 ou compatible.
 Si le package.json est deja près ou récuperé sur le github du projet Wed&Joy, alors il suffit de faire un npm install pour installer les dépendances de babel.
 Il est également important de reproduire la configuration décrite dans le fichier de configuration de babel : babelrc.
 Ce fichier déclare les "presets" (le modèle de compilation) et "plugins" (fonctionnalité additionelle comme le support des décorateurs).
 Cette configuration doit impérativement être respectée sous peine de perdre la prise en charge de certaines fonctionnalités, utilisées par le serveur, l'orm ou vue.js.

 package.json, dépendances
 ```json
 "dependencies": {
     "@decorators/di": "^1.0.2",
     "@decorators/express": "^2.3.0",
     "babel-core": "^7.0.0-bridge.0",
     "body-parser": "^1.18.3",
     "connect-history-api-fallback": "^1.5.0",
     "crypto": "^1.0.1",
     "express": "~4.16.0",
     "express-session": "^1.15.6",
     "jsonwebtoken": "^8.4.0",
     "morgan": "^1.9.1",
     "mysql": "^2.14.1",
     "nodemon": "^1.18.6",
     "passport": "^0.4.0",
     "passport-jwt": "^4.0.0",
     "passport-local": "^1.0.0",
     "pug": "^2.0.3",
     "session-file-store": "^1.2.0",
     "typeorm": "^0.2.6"
   },
 ```
 package.json, dépendances de developement
 ```json
 "devDependencies": {
     "@babel/cli": "^7.2.0",
     "@babel/core": "^7.2.0",
     "@babel/node": "^7.2.0",
     "@babel/plugin-proposal-class-properties": "^7.2.0",
     "@babel/plugin-proposal-decorators": "^7.2.0",
     "@babel/preset-env": "^7.2.0",
     "babel-plugin-transform-function-parameter-decorators": "^1.2.0",
     "babel-polyfill": "^6.26.0",
     "babel-preset-minify": "^0.5.0"
   }
 ```

 .babelrc
 ```json
 {
   "presets": [
     "@babel/preset-env"
   ],
   "plugins": [
     [ "transform-function-parameter-decorators" ],
     [ "@babel/plugin-proposal-decorators", { "legacy": true } ],
     [ "@babel/plugin-proposal-class-properties", { "loose": true } ]

   ],
   "ignore": [

   ]
 }
 ```

### Babel <a name="babel"></a>
Revenons sur babel, partie importante de l'architecture de ce projet.

Babel a été mis en place sur ce mockup pour utiliser pleinement les fonctionnalités de TypeORM, ORM choisis sur ce projet. TypeORM est comme son nom l'indique, prévu pour une utilisation TypeScript. Le projet est compatible nativement avec javascript, néanmoins il ne propose de fait que très peut de fonctionnalités modernes et limite énormement sa simplicité d'utilisation et sa capacité à évoluer (pas d'héritage par exemple).
	Grace à Babel qui compile le javascript comme le ferait typescript, on a la possibilité d'utiliser dans notre application toutes les fonctionnalités de typeorm, hormis les types de typescript bien sur.
	Nous avons donc accès entités sous formes de classes, aux décorateurs pour déclarer les types dans la base de données, les tailles, si il est nullable, sa valeur par défaut ou pour gérer les relation OneTo ManyTo

### Les décorateurs <a name="decorators"></a>
Les décorateurs sont une nouvelles fonctionnalité de javascript qui n'est pas encore au point en terme de performance. Grace à babel on peut s'en servir en "transpilation" en attendant que la fonctionnalité soit prête au grand public des developpeurs.

Un décorateur, c'est un petit bout de code qui vient se placer au dessus des fonctions et classes. Il est reconnaissable par son "@" en début de nom. Il est accès commun que son nom commence par une majuscule.

Son but et de, en fonction des paramètres qui lui sont passés, "construire" une fonction autours de la fonction ciblée (ou classe), et donc finalement d'en prendre le controle.

#### Exemples <a name="exemples"></a>

L'objectif de la mise en place du support des décorateurs n'est pas particulièrement d'en créer de nouveaux, mais plutôt d'en utiliser des existants, néanmoins pour bien en comprendre le fonctionnement, créons en un :
```javascript
	function MonDecorator(obj) {
		return function(target) {
			target.helloWorld = obj.helloworld;
		}
	}

	@MonDecorator({ helloworld: true })
	class Test {
		helloWorld = false;
	}

	const test = new Test();
	console.log(test.helloWorld) // true
```

Voila basiquement un résultat possible de babel :
```javascript

"use strict";

var _dec, _class;

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function MonDecorator(obj) {
	return function (target) {
		target.helloWorld = obj.helloworld;
	};
}

var Test = (_dec = MonDecorator({ helloworld: true }), _dec(_class = function Test() {
	_classCallCheck(this, Test);

	this.helloWorld = false;
}) || _class);


var test = new Test();
console.log(test.helloWorld); // true

```


#### Liste des décorateurs <a name="decolist"></a>

##### Serveur <a name="decoserv"></a>
Le serveur propose de nombreux decorateurs pour les types de requêtes :
@Post
@Get
etc .
Et @Controller pour déclarer une classe comme un controlleur

Ils prennent tous les mêmes paramètres :
```

@Post(
	'/login', // Il s'agit de la route de la fonction ou du controlleur,
	[ MiddleWare1, MiddleWare2, ... ] // Un tableau de classes contenant une méthode run. Ce sont nos middlewares
)
login(req, res) { ...

```

##### Client <a name="decoclient"></a>
Pour les clients, ce sont les décorateurs de Vue Class Component, à savoir :
@Component pour déclarer une classe qui extends Vue comme un component.
@Prop et @Watch.
Prop est pour déclarer un field de classe comme un prop, et @Watch pour déclarer une fonction comme un watcher
Il prennent tous des paramètres differents, le plus simple est de se referer au github de Vue Class Component.


### TypeORM <a name="typeorm"></a>
TypeORM est un des ORM les plus moderne du monde javascript. Il fonctionne sur le même schema que Doctrine, dont il s'inspire, mais profite de la liberté permise par le javascript pour rendre la manipulation de donnée plus simple. Un exemple simple est la facilité déconcertante que demande une serialization / deserialization des entités en json, la gain de ligne incroyable que permet la création et la mise à jour d'entités via les methodes create et update, et la légereté de ne pas avoir à créer des Repository quand il n'y en a pas le besoin.

L'outil propose une documentation très complète, couvrant toutes les utilisation de la technologie, allant des entités, des relations de tous types et de l'utilisation du query builder au travers de repository customs. Il propose également un utilitaire en ligne de commande
```bash
$ sudo npm install -g typeorm
```
qui permet de créer de nouvelles entités ou de synchroniser le schema de base avec le schema local.

#### Créer une entité <a name="typeorm.create"></a>
```javascript
@Entity("User")
export default class User extends AbstractEntity {

	@Column({name:'firstName', type: "varchar", width: 255 })
	firstName;

	@Column({ type: "varchar", width: 255, nullable: true })
	lastName;

	@Column({ type: "varchar", width: 255, nullable: true })
	mail;

	@Column({ type: "varchar", width: 255, unique: true })
	phone;

	@Column({ type: "text", nullable: true })
	salt;

	@Column({ type: "text", nullable: true })
	hash;
}
```
Cette entité est celle de l'utilisateur. Elle hérite de AbstractEntity, qui est une autre entité ne contenant qu'une ID, une date de création de mise à jour (le tout gérer automatiquement). De fait chaque entité possède ces champs.

#### Insérer une nouvelle entrée <a name="typeorm.insert"></a>
Il existe deux moyens de créer une nouvelle entité, dépendant des cas d'utilisation. Les deux sont utilisés dans l'application actuellement et leurs fonctionnement restent décris sur la documentation de typeorm.

##### Manière 1 <a name="method1"></a>
```javascript

const user = new User();
user.firstName = "Lucas";
user.lastName = "Pointurier";
// etc.

const result = await getRepository(User).save(user);
if (result) {
	// Tout s'est bien passé !

```

##### Manière 2 <a name="method2"></a>
```javascript

const user = getRepository(User).create({
	firstName = "Lucas",
	lastName = "Pointurier",
	// ...
});

getRepository(User).save(user);

```

Cette methode est plus lourde que la précedente, sauf quand on a deja un objet sous la forme {firstName: string, lastName: string, ...}
Dans ce cas
```javascript
	const user = repo.create(object);
	repo.save(user);
```

est bien plus pertinante !

#### Rechercher <a name="typeorm.find"></a>

La recherche de typeorm est assez simple et ressemble à Doctrine.
Elle se produit de cette manière :
```javascript
	const repo = getRepository(User);
	const results = await repo.find({
		where: { name: 'Toto', role: 'Foo' },
		relations: ['wedding']
	})
	console.log(results.length) // 2
```
Il est également possible de trouver à partir de relations et de faire des jonctions, ou de selectionner que les fields interessants (par exemple que l'ID) et même utiliser des opérateurs de recherche complete comme with, ou faire des récuperations en fonction de date ou d'écheances.

## Mise en production
Voici les étapes de mise en production :

1. Compiler la vue : npm run build.
Cela va créer un dossier dist dans public/Vue qui contient la version packagée de notre application.
2. Compiler le serveur : on fais comme si on voulais le lancer, donc npm start, on attend qu'il compile et se lance, ctrl + c : le dossier app a été créé, mais on ne va pas laisser le serveur tourner "en mode dev", sinon il va pas tourner longtemps ...
3. En utilisant le package PM2 (voir ci dessous pour l'installer) on peut lancer un projet nodejs "en serveur mode". Si on se contente de faire un npm start, le serveur node va se couper dès que l'on se deconnectera de notre session SSH :
	```bash
		pm2 start ./launcher.bundle.js
	```

> Info : Pourquoi ne pas utiliser apache ou nginx comme avant avec PHP ?
node a un fonctionnement un peu different de PHP. Node est le serveur, alors qu'en PHP on utilise un serveur comme apache ou nginx pour servir le client. C'est apache et nginx qui vont "transférer la requête à php" et php qui va répondre en utilisant notre code toujours en passant par ngapaxe2
Pour node tout est "inclus" dans le package node.
Ici pm2 est juste un système pour "sandboxer" l'application, et la relancer en cas de crash (ce qui peut arriver en cas d'erreurs fatales non gérés).


*Pour installer pm2 :*
```bash
# En root ou sudo
npm install -g pm2
```

> Note : Il existe beaucoups de solutions pour faire tourner node.js en production. La plus simple est pm2, car lors d'une mise à jour de la version en production, il suffit de remplacer le code et de faire pm2 restart <nom>.

> Note 2 : voir pm2 pour une liste des commandes, certaines sont très utiles comme pm2 list pour voir les applications qui tournent, stop pour arreter, et surtout monit qui ouvre une interface user_friendly pour surveiller le fonctionnement de l'app.
