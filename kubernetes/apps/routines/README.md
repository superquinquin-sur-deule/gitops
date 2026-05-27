# Routines

## Adjust FTOP Seats

Tous les jours à 2h du matin, ajuste les compteurs volants sur les services des 7 prochains jours pour avoir Places ABCD - Places Reservées + 1.

## Reset FTOP Counter

Tous les jours à 3h du matin, remet à zéro les compteurs FTOP des coopérateurs en marquant `ignored=true` sur tous les `shift.counter.event` de type `ftop` non encore ignorés (motif « Services avant ouverture »).
