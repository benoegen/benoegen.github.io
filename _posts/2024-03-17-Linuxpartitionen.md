---
title: Partitionieren unter Linux
author: benoegen
tag: it
---
Seit mehren Tagen bin ich nun auf Linux unterwegs und das austesten verschiedener Distros führte letztendlich dazu, dass ich meine Datenpartition retten  und Windows neu installieren musste. 
Ziel ist ein Dual Boot System, da ich noch mindestens ein Spiel weiterhin spielen möchte, welches nicht unter Linux funktioniert.

Getestet habe ich die Distros Endeavor (läuft bereits problemlos auf dem laptop zum surfen), Nobara und Garuda. Für den Hauptrechner wollte ich etwas mehr vorselektiertes, gerne mit mehr GUI als das "terminal-centric" Endeavor.
Nobara lief zunächst gut, der Wechsel auf Garuda hat dann im Bereich der EFI was zerstört, so dass es in den genannten Neuinstallationen endete. Letztendlich habe ich manuell partitioniert und zwar wie folgt:

200 MB Fat32 /Boot/EFI flag:boot

1gb ext4 /Boot

Rest Btrfs /


Damit fahre ich bisher gut und auch der Dual Boot wurde nicht beschädigt. Nobara überzeugt auch mit der Zugänglichkeit, wobei das Gegenargument one man show auf Fedora von mir zunächst ignoriert wird. Garuda sagt mir optisch leider gar nicht zu. Nach den ersten Gehversuchen wird es vielleicht irgendwann wieder Endeavor und die gaming relevanten Dinge installiere ich dann manuell, man weiß ja nie.
