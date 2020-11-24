# tp-rest

Dépôt des fichiers pour le TP REST
GARCIA Florian

## Partie 1 : Utiliser la méthode REST

2. On peut piloter notre objet en allant sur l'URL suivante : http://10.202.255.252/
On obtient une interface Web nous permettant de controller notre objet : 

<img src="https://raw.githubusercontent.com/floriangarciasoto/tp-rest/main/images/Capture%20du%202020-11-24%2011-00-24.png"/>

On peut allumer et éteindre notre Shelly avec le bouton power.

3. Pour récuperer la consomation en Watt, on peut d'abord récuperer la page web  suivante : 
```bash
test@202-13:~/tp-rest$ curl http://10.202.255.252/meter/0 --output page.json
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   120  100   120    0     0   7500      0 --:--:-- --:--:-- --:--:--  7500

```
Puis prendre uniquement dans le fichier la conso : 
```bash
test@202-13:~/tp-rest$ cat page.json | grep -oE '\{"power":[^,]*,' | sed 's/{"power"://' | sed 's/,//'
49.15
```

Pour allumer la LED : 
```bash
test@202-13:~/tp-rest$ curl http://10.202.255.252/relay/0?turn=on
{"ison":true,"has_timer":false,"timer_started":0,"timer_duration":0,"timer_remaining":0,"overpower":false,"source":"http"}
```

Pour l'éteindre : 
```bash
test@202-13:~/tp-rest$ curl http://10.202.255.252/relay/0?turn=off
{"ison":false,"has_timer":false,"timer_started":0,"timer_duration":0,"timer_remaining":0,"overpower":false,"source":"http"}
```

4. On peut maintenant récuperer le résultat d'une façon bien plus propre : 
```bash
test@202-13:~/tp-rest$ curl --silent http://10.202.255.252/meter/0 | jq
{
  "power": 50.79,
  "overpower": 0,
  "is_valid": true,
  "timestamp": 1606218357,
  "counters": [
    47.139,
    40.369,
    0
  ],
  "total": 87
}
```
On peut filtrer le résultat, en pouvant par exemple récupérer la conso : 
```bash
test@202-13:~/tp-rest$ curl --silent http://10.202.255.252/meter/0 | jq .power
50.8
```

5. On peut créer notre script et lui donner les droits d'exécution : 
```bash
test@202-13:~/tp-rest$ touch recupConso
test@202-13:~/tp-rest$ chmod u+x recupConso 
```
On pourra simplement faire marcher le script comme ceci : 
```bash
test@202-13:~/tp-rest$ ./recupConso
```
Dans le script : 
```bash
#!/bin/bash

echo -n $(curl --silent http://10.202.255.252/meter/0 | jq .power)
echo " Watt"
```
On indique le shebang permettant de spécifier l'interpréteur avec lequel interpréter le script, dans notre cas bash.
On affiche ensuite le retour de la commande permettant de prendre uniquement la conso avec un $().
On peut enfin afficher d'éventuels messages avec echo.

Le script allumerOuEteindreLED permet donc d'allumer ou d'éteindre la LED en fonction de son état, et affiche sa consommation. Ce script utilise les autres scripts afin de fonctionner, ce qui fait le jeu de scripts.


6. Le filtre de capture nécessaire afin de capturer ces trames est le suivant : 
```
((host 10.202.255.252 and dst 10.202.13.1) or (host 10.202.13.1 and dst 10.202.255.252)) and port 80
```
Cela permet d'avoir uniquement les échanges HTTP entre notre machine et le Shelly.
On obtient donc la capture suivante : 
```
No.     Time           Source                Destination           Protocol Length Info
      1 0.000000000    10.202.13.1           10.202.255.252        TCP      74     34320 → 80 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=2230054381 TSecr=0 WS=1024
      2 0.002874360    10.202.255.252        10.202.13.1           TCP      62     80 → 34320 [SYN, ACK] Seq=0 Ack=1 Win=2144 Len=0 MSS=536 SACK_PERM=1
      3 0.002932155    10.202.13.1           10.202.255.252        TCP      54     34320 → 80 [ACK] Seq=1 Ack=1 Win=64240 Len=0
      4 0.003013212    10.202.13.1           10.202.255.252        HTTP     147    GET /relay/0?turn=on HTTP/1.1 
      5 0.017582986    10.202.255.252        10.202.13.1           HTTP     290    HTTP/1.1 200 OK  (application/json)
      6 0.017627295    10.202.13.1           10.202.255.252        TCP      54     34320 → 80 [ACK] Seq=94 Ack=237 Win=64004 Len=0
      7 0.017713911    10.202.13.1           10.202.255.252        TCP      54     34320 → 80 [FIN, ACK] Seq=94 Ack=237 Win=64004 Len=0
      8 0.020667064    10.202.255.252        10.202.13.1           TCP      60     80 → 34320 [ACK] Seq=237 Ack=95 Win=2050 Len=0
      9 0.022049057    10.202.255.252        10.202.13.1           TCP      60     80 → 34320 [FIN, ACK] Seq=237 Ack=95 Win=2050 Len=0
     10 0.022083004    10.202.13.1           10.202.255.252        TCP      54     34320 → 80 [ACK] Seq=95 Ack=238 Win=64004 Len=0
```
Comme n'importe quel échange en TCP, il y a d'abord les trois premières trames de connexion.
La trame 4 correspond à l'envoi de la requête GET permettant d'éteindre la LED : 
```
Frame 4: 147 bytes on wire (1176 bits), 147 bytes captured (1176 bits) on interface 0
Ethernet II, Src: Dell_96:3e:56 (d4:be:d9:96:3e:56), Dst: Espressi_6a:64:db (ec:fa:bc:6a:64:db)
Internet Protocol Version 4, Src: 10.202.13.1, Dst: 10.202.255.252
Transmission Control Protocol, Src Port: 34320, Dst Port: 80, Seq: 1, Ack: 1, Len: 93
Hypertext Transfer Protocol
```
Comme on peut le voir, la trame envoyée est de 147 octets.
Il faut donc 147 octets pour seulement allumer la LED, ce qui est beaucoup d'octets pour seulement réaliser une petite action, on pourrait se passer de beaucoup d'informations ...

## Partie 2 : MQTT

Notre objet est un Shelly de type Plug-s 6A6534, on va pouvoir donc publish à partir de : 
```
shellies/shellyplug-s-6A6534/
```
A partir de là, il s'agit quasiment de la même arborescance.
On peut donc allumer la LED comme ceci : 
```bash
test@202-13:~/tp-rest$ mosquitto_pub -h 10.202.0.107 -t shellies/shellyplug-s-6A6534/relay/0/command -m "on"
```
Et l'éteindre avec : 
```bash
test@202-13:~/tp-rest$ mosquitto_pub -h 10.202.0.107 -t shellies/shellyplug-s-6A6534/relay/0/command -m "off"
```
Le fait de publier un message sur le sujet shellies/shellyplug-s-6A6534/relay/0/command va permettre à ce que le Shelly prenne en charge la commande 'on' ou 'off'.
Pour obtenir la consommation de la LED : 
```bash
test@202-13:~/tp-rest$ mosquitto_sub -h 10.202.0.107 -t shellies/shellyplug-s-6A6534/relay/0/power -C 1
52.87
```
On précisque que l'on se désabonne dés que l'on a reçu 1 message, la consommation en question.
Pour obtenir l'état de la LED : 
```bash
test@202-13:~/tp-rest$ mosquitto_sub -h 10.202.0.107 -t shellies/shellyplug-s-6A6534/relay/0 -C 1
```
Cela renvoi 'on' ou 'off'.
On peut donc modifier les scripts en question.

## Partie 3 : Serveur REST

Avant d'installer nginx, il faut désinstaller complètement apache : 
```bash
test@202-13:~/tp-rest$ sudo apt remove apache2 --purge
```
On peut ensuite l'installer : 
```bash
test@202-13:~/tp-rest$ sudo apt install nginx
```
On peut donc se rendre sur http://localhost (ne pas oublier de recharger complètement la page avec CTRL + F5) : 

<img src="https://raw.githubusercontent.com/floriangarciasoto/tp-rest/main/images/Capture%20du%202020-11-24%2015-47-03.png"/>

Notre serveur nginx est opérationel.
