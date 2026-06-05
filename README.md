Compte rendu — Réunion « Organisation de l’organized layer » (4 juin 2026)

1. Problèmes identifiés

Process de livraison flou. Le processus de livraison des modèles n’est pas formalisé et sa compréhension diffère selon les parties prenantes, sur le plan fonctionnel comme technique. Personne ne partage la même lecture de ce qui est livré, quand, et sous quelle forme.

Dépendance mal maîtrisée entre équipes. Le reporting a pris une dépendance directe sur l’exposition du socle data. Les BA peuvent anticiper ~80–90 % des spécifications sur la base du jalon « domaine », mais les 10–20 % restants attendent les interfaces du socle. Sans anticipation, le délai global glisse d’environ trois semaines.

Modèle de l’organized layer insuffisamment compris. La place et le rôle de la couche restent flous. Exemple révélateur : certains comprenaient « une autorisation = une transaction », alors que le modèle organisé distingue la transaction (document aplati) de ses mouvements (autorisation, capture, précompensation, compensation).

Documentation du modèle insuffisante. Onglets inutiles à nettoyer, absence de visualisation lisible du modèle et de détail des champs (signification, valeurs admises). Le manque de versioning clair rend les évolutions difficiles à suivre.

Risque d’inefficacité du modèle physique. L’approche actuelle force à passer systématiquement par la transaction pour toute extraction. Or certains besoins ne portent que sur un type de mouvement. Exemple : extraire sur une semaine toutes les autorisations Castorama de type PB obligerait à scanner 50–60 millions de transactions — non viable.

Règles de corrélation non spécifiées. Les clés reliant transaction / remise / fichier / mouvements ne sont pas documentées de façon explicite et partagée. L’expérience passée (pages Confluence éparses) a montré que ces règles deviennent vite intraçables.

Origine du modèle de données questionnée. Le modèle s’appuie sur le modèle de transfert E-Stream (partiellement inspiré d’ISO 20022), et non sur un standard complet type ISO 20022 ou BIAN.

2. Approches actuelles et remédiation

Conception — se fédérer autour de la compréhension. La valeur est dans l’effort collectif de conception en amont, pas dans la production isolée de modèles. Expliciter de bout en bout les flux d’ingestion (contrat d’ingestion → compréhension → mapping → modèle SI), et le même exercice côté consommation : savoir où se trouve la donnée, ce qui évolue, ce qui va vers quoi.

Principes de design retenus.

	•	Plusieurs data products par transaction : transaction complète (document + tous mouvements) et transactions simplifiées (ex. transaction + dernier mouvement d’autorisation).
	•	Sortir du fonctionnement par statuts hérité du socle Vision (accepté → autorisé → réglé), jugé inefficace ; ne pas le reconduire.
	•	Remise = objet indépendant de la transaction. Postulat de travail : lien 1–1 transaction/remise, variantes à réexaminer (cas pressentis de transaction liée à plusieurs remises). L’enrichissement remise→transaction relève du modèle logique.
	•	Confronter le modèle aux besoins réels d’extraction avant de figer le modèle physique (index notamment).

Choix du standard — tranché. Pas de standard type BIAN ni ISO 20022 complet : jugé trop coûteux pour le métier et ITGA. On conserve le modèle dérivé du transfert E-Stream, en assumant ce choix.

Corrélation — deux niveaux. Une partie portée par le modèle (cardinalités remise/transaction, couvertes par Nadir), une partie par des règles de gestion dynamiques de consolidation. Le lien repose sur les références d’estampillage déjà présentes dans les flux du socle Vision, reproduites dans la version E-Stream (Correlation ID reliant autorisation et capture, vérifié par Franck), couvrant transaction / remise / fichier et tout le cycle de vie (chargeback, fraude).

Validation et inclusion des parties prenantes.

	•	Process de livraison à formaliser avec le métier et tous les stakeholders dans la boucle (notifications mail, demandes de relecture explicites).
	•	Équipes reporting intégrées aux ateliers de mapping de bout en bout — pour anticiper et être force de proposition sur la couverture de leurs besoins, sans exploiter directement l’organized layer à ce stade.
	•	La formalisation ne doit pas bloquer le travail en cours : mieux livrer, pas mettre « le crayon en attente ».
	•	Proposition de modèle (clés de corrélation, champs, cardinalités) posée lundi par Franck, puis challengée collectivement.

Mise en œuvre — documentation et versioning.

	•	Versioning du modèle (et non du flux) avec changelog.
	•	Nettoyage des onglets inutiles.
	•	Visualisation du modèle à l’image de InSite, complétée par le détail des champs (signification, valeurs admises).
	•	Espace documentaire unique précisant clés et règles de corrélation, avec un moyen de suivre l’ajout des cas non couverts au fil de la confrontation au réel.

3. Décisions et prochaines étapes

	•	Deux ateliers actés : (1) process de livraison ; (2) atterrissage MCD → modèle logique (relecture du modèle de Franck : quelles données au niveau mouvement vs transaction).
	•	Souad livre la première version côté émission d’ici la fin de semaine prochaine ; mêmes principes que l’acquisition.
	•	Franck envoie le support lundi et cale au minimum deux ateliers de relecture.
	•	El Hadi organise et planifie les ateliers ; chaque équipe fournit les noms des participants par atelier (reporting inclus ; partie MCD/MLD avec Dominique D.).
	•	Compte rendu attendu et diffusé.
