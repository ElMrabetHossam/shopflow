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
