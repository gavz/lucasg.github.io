---
layout: post
title: "Calcule le code d'un autoradio Renault"
date: 2019-08-03
---

Toi aussi tu dois récuperer le "code de sécurité" de ton autoradio Renault parce que tu as debranché la batterie ? Toi non plus tu n'as pas envie d'aller le quémander dans un garage Renault ? 

Dans ce cas là, récupère ton précode (en appuyant sur les touches 1 & 6 puis sur le bouton d'alimentation de l'autoradio) et utilise la routine ci-dessous pour regénérer le code de sécurité :

{% include autoradio_code.html %}


<!--more-->


L'histoire
-----------


Récemment j'ai récupéré une voiture Renault qui venait de subir quelques réparations. Puisque la batterie avait été débranchée pour les manipulations nécessaires, quand j'ai mis le contact et tenter d'allumer l'autoradio, j'ai eu ce message :

![](/assets/renault_enter_code.jpg){:height="80%" width="80%"}


Apparement, Renault place un "code de sécurité" sur les autoradios qu'ils installent dans leurs modèles de voiture. Quel est le modèle d'attaque, j'en sais trop rien (empêcher le vol d'autoradio ? lol) mais toujours est-il que ça existe, que ça m'empêche d'écouter de la musique et que la solution est d'aller demander le code à un garagiste Renault et potentiellement devoir mettre la main à la poche.

Heureusement, c'est également possible de le faire soit même si on connait deux éléments : le "précode" et la routine qui dérive le "précode" en un code d'activation. Il y a plusieurs méthodes pour récupérer le précode est documenté sur plusieurs sites tiers : https://www.lecoindunet.com/recuperer-code-autoradio-renault-280.

Toutefois, la routine de dérivation n'est pas documentée, les seuls alternatives pour réaliser la dérivation est soit [de cliquer sur une pub (qui ne sert à rien pour notre pb)](https://www.lecoindunet.com/https://www.lecoindunet.com/code-autoradio-renault), [lancer un binaire DOS (DOS ? really ?)](https://www.lecoindunet.com/recuperer-code-autoradio-renault-280) ou installer une app Andoid qui a l'air de se payer via de la pub :

![](/assets/renault_garbage_app.jpg){:height="30%" width="30%"}

J'aime pas trop installer des app chelous sur mon téléphone, donc j'ai plutôt téléchargé l'APK à la place et je l'ai ouvert dans Jeb. L'application possède assez peu de code, et il y a des strings de debug dans toutes les fonctions donc je suis tombé rapidement sur la fonction de dérivation du précode :

![](/assets/renault_derivation.PNG)
(il y a aussi moyen de dériver à partir d'un code-barre présent à l'arrière de l'autoradio)

Un peu de nettoyage de code, et une conversion en Python et on obtient la routine suivante :

```python
def derive_precode(precode):
	
	if not len(precode):
		raise ValueError("Could not compute the code with a empty precode !")

	precode = precode.upper()

	x = ord(precode[1]) + (ord(precode[0]) * 5 * 2  - 698)
	y = ord(precode[3]) + (ord(precode[2]) * 5 * 2 + x) - 0x210
	z = ((y << 3) - y) % 100;

	computed_code = (z // 10) + (z % 10)*5*2 + ((0x103 % x) % 100) * 5 * 5 *4
	computed_code = "%04d" % computed_code

	print("%s => %s" % (precode, computed_code))
	return computed_code

if __name__ == '__main__':
	# Les exemples de test proviennent d'un utilisateur d'un forum de lecoindunet
	# qui fait la dérivation pour les gens qui la demande ... sans donner le moyen pour eux 
	# de le faire eux-mêmes.
	# Source : https://www.lecoindunet.com/forums/topic/topic-unique-code-autoradio-renault
	
	assert derive_precode("S053") == "7913"
	assert derive_precode("V188") == "4839"
	assert derive_precode("Q420") == "9588"
```

Je sais pas quels modèles utilisent ce système de "dérivation" de code, mais en tout cas ça a marché pour ma voiture donc balek :p

