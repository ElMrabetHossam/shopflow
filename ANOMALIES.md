# Fiche d'anomalies — Panier ShopFlow (TP1)

## Anomalie 1 : livraison facturée pour un sous-total de exactement 500 DH

| Rubrique | Détail |
|---|---|
| **Règle concernée** | R5 : la livraison coûte 30 DH ; elle est offerte dès que le sous-total **atteint** 500 DH. |
| **Test révélateur** | `PanierTest > R5 : frais de livraison > laLivraisonEstOfferteAPartirDe500DH`, cas `[3]` « sous-total 500.00 DH -> livraison 0.00 DH ». Résultat : `Montant attendu : 0.00 DH, obtenu : 30.00 DH`. |
| **Cause** | Dans `Panier.fraisLivraison()`, la comparaison `sousTotal().compareTo(SEUIL_LIVRAISON_OFFERTE) > 0` signifie « strictement supérieur à 500 ». Erreur de type *off-by-one* : visible uniquement à la valeur limite exacte. |
| **Correctif** | Remplacer `> 0` par `>= 0`. Le cas `[3]` reste dans la suite comme test de non-régression. |

## Anomalie 2 : la remise du code promo s'applique aussi sur la livraison

| Rubrique | Détail |
|---|---|
| **Règle concernée** | R6 : la remise porte sur le sous-total uniquement, jamais sur la livraison. R7 : total = sous-total − remise + livraison. |
| **Test révélateur** | `PanierTest > R6 et R7 : remise et total > laRemiseNePortePasSurLaLivraison` (une souris à 120 DH, code BIENVENUE10). Résultat : `Montant attendu : 138.00 DH, obtenu : 135.00 DH`. |
| **Cause** | `Panier.total()` calculait `(sous-total + livraison)` puis retirait le pourcentage de ce montant : 10 % de 150 DH = 15 DH au lieu de 10 % de 120 DH = 12 DH. ShopFlow perdait 3 DH par commande avec code promo et livraison payante. Le test à 570 DH ne le détectait pas, car avec une livraison offerte les deux calculs coïncident. |
| **Correctif** | Réécrire `total()` comme la traduction directe de R7 : `return sousTotal().subtract(remise()).add(fraisLivraison());` |

---

# Fiche d'anomalies — Service de commande (TP2)

## Anomalie 3 : le stock réservé n'est pas libéré après un paiement refusé

| Rubrique | Détail |
|---|---|
| **Règle concernée** | C5 : si le paiement est refusé, le stock réservé est libéré, `PaiementRefuseException` est levée, rien n'est enregistré et aucun e-mail n'est envoyé. |
| **Test révélateur** | `ServiceCommandeTest > unPaiementRefuseLibereLeStock`. Mockito : `Wanted but not invoked: stock.liberer("SOUR-02", 2)` ; seuls `estDisponible` et `reserver` avaient été appelés. |
| **Cause** | Dans `ServiceCommande.passerCommande`, la branche « paiement refusé » levait l'exception sans rendre les quantités réservées juste avant. Chaque carte refusée bloquait du stock définitivement. Le bug est invisible dans le résultat (l'exception est bien levée) : seule la vérification d'un appel par un mock pouvait le détecter. |
| **Correctif** | Avant de lever `PaiementRefuseException`, parcourir les lignes du panier et appeler `stock.liberer(reference, quantite)` pour chacune. |

## Anomalie 4 : une panne d'e-mail fait échouer une commande déjà payée

| Rubrique | Détail |
|---|---|
| **Règle concernée** | C6 : une panne de l'envoi d'e-mail n'annule pas une commande payée. |
| **Test révélateur** | `ServiceCommandeTest > unePanneDeMailNAnnulePasLaCommande` (notification programmée avec `doThrow(new IllegalStateException("Serveur SMTP injoignable"))`). Résultat : `Errors: 1` — l'exception traverse le service jusqu'à l'appelant. |
| **Cause** | L'appel `notifications.envoyerConfirmation(...)` n'était pas protégé : une exception du serveur d'e-mails remontait au client alors que la commande était déjà payée et enregistrée. Le client, voyant une erreur, risquait de repasser commande et de payer deux fois. |
| **Correctif** | Entourer l'envoi d'un `try / catch (RuntimeException e)` qui journalise la panne avec `System.Logger` au niveau `WARNING` (sans l'ignorer silencieusement), puis renvoyer la commande confirmée. |
