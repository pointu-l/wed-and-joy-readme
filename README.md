# Wed&Joy

## Table des matières
1. [Introduction](#intro)
2. [Fonctionnement](#fonctionnement)
	1. [Exemple](#fonctionnement.exemple)
3. [Installer et utiliser TypeORM, babel avec NodeJS (sans typescript !)](#tybano)
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

## TypeORM, les décorateurs et babel <a name="tybano"></a>

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
```babel

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
