## Sécuriser son serveur

## Ne pas utiliser FTP / Telnet

**FTP** et **Telnet** (ainsi que d'autres commandes comme **rsh / rlogin**) ne chiffrent pas leurs données, il est donc facile en sniffant les ports d'un serveur de recuperer le mot de passe.

Le mieux est donc d'utiliser <u>OpenSSH</u>, <u>SFTP</u> ou <u>FTPS</u> (j'ai une préférence pour SFTP pour une utilisation simple d'un serveur FTP).

## Sécuriser OpenSSH

Pour pouvoir se connecter à son serveur via un terminal ou se connecter avec un client FTP il faut passer par <u>OpenSSH</u>  qui est l'implémentation du protocol SSH.

Pour commencer, si jamais ce n'est pas déjà fait il faut installer le package `openssh-server`:

```shell
sudo apt-get install openssh-server
```

Une fois fait il va falloir configurer le fichier `/etc/ssh/sshd_config`

### Configurer OpenSSH pour utiliser le protocole 2 de SSH

Première étape pour eviter les vunérabilités et les attaques de type MITM (Man in the middle) il faut s'assurer que le protocole utilisé par SSH est bien le 2 et pas le 1:

Pour ca vérifiez que vous utilisez bien le protocole 2:

```shell
$ vi /etc/ssh/sshd_config
Protocol 2
```

### Autoriser que certains utilisateurs  / groupes

Etape très importante, il faut que vous définissiez les utilisateurs (ou groupes) qui peuvent utiliser SSH pour se connecter à votre serveur, pour cela il faut rajouter (ou décommenter) la ligne suivante:

```shell
$ vi /etc/ssh/sshd_config
AllowUsers ankirama # On autorise uniquement l'utilisateur ankirama a se connecter
AllowGroups admin # Seul le groupe admin peut se connecter
```

**ATTENTION** : il y a un ordre sur les règles: **DenyUsers**, **AllowUsers**, **DenyGroups**, **AllowGroups** donc si la règle *AllowUsers* et la règle *AllowGroups* sont utilisées en même temps, *AllowGroups* ne fonctionnera pas !

### Configurer un timeout s'il n'y a pas d'activité

Une chose qui peut toujours être intéressante  est d'activer un timeout avec pour conséquence une déconnexion du serveur, cela peut par exemple éviter que quelqu'un aille sur votre machine physique qui est conecté au serveur quand vous êtes en pause.
Il faut donc décommenter les lignes suivantes:

```shell
$ vi /etc/ssh/sshd_config
ClientAliveInterval 300 # 300 secondes = 5 minutes d'inactivité
ClientAliveCountMax 0 # Cela signifie qu'il n'y aura pas d'avertissement avant déconnexion
```

### Ne pas autoriser le compte root a se connecter

Permettre au root de se connecter via SSH n'est pas une bonne idée car s'il y a une faille avec le protocole (ou que vous subissez une attaque MITM) par exemple la personne récupérant le compte root aura tous les droits !!

Il faut donc vérifier que l'on ne peut pas se connecter en tant que root :

```shell
$ vi /etc/ssh/sshd_config
PermitRootLogin no
```

En vrai ce n'est plus forcément nécessaire, le protocole SSH est bien sécurisé mais c'est toujours une bonne chose de faire ça, ça ne coute rien.

### Changer le port par défaut pour se connecter

Il peut être intéressant également de changer le port par défaut (22), mais attention, si vous souhaitez utiliser par exemple git ou d'autres outils comme rsync qui passent par le protocole SSH il faudra surement configurer vos clients pour qu'ils utilisent le bon port.
Il faut donc changer cette ligne sur le serveur:

```shell
$ vi /etc/ssh/sshd_config
Port 4224 # Attention, vérifiez bien que le port n'est pas utilisé donc > 1024
```

### Utilisation de clés SSH

Une bonne façon de protéger son serveur est également d'utiliser une paire de clés (public / private) avec un mot de passe pour la clé privée. 

Pour cela il faut aller du coté du client:

``` shell
ssh-keygen -t rsa -b 4096 # -t permet de choisir de type de clés, -b permet de specifier le nombre de bits (ici 2^12)
```

Pensez à spécifier une passphrase pour plus de sécurité et choisir un nom de clé cohérent par exemple `/home/ankirama/.ssh/id_my_own_server_rsa`. On peut donc comprendre facilement que cette clé utilise le type **RSA** et qu'elle est pour **my own server**

Il faut maintenant envoyer la clé public sur le serveur:

``` shell
ssh-copy-id -i ~/.ssh/id_my_own_server_rsa.pub user@machine
```

Après cela il peut être intéressant de configuer le fichier `~/.ssh/config` sur votre client pour rajouter ce que vous venez de faire:

``` shell
$ vi ~/.ssh/config
Host my_own_server # permet de faire 'ssh my_own_server' au lieu de 'ssh ankirama@XXX.XXX.XXX.XXX'
	HostName XXX.XXX.XXX.XXX # l'ip / DNS de votre serveur
    IdentityFile ~/.ssh/id_my_own_server_rsa
    User ankirama
    Port 4224
```

### Vérifier que les mots de passe vides sont bien désactivés

On retourne coté serveur dans notre fichier de configuration pour s'assurer que l'on autorise pas les mots de passe vides:

```shell
$ vi /etc/ssh/sshd_config
PermitEmptyPasswords no
```

### Bloquer l'utilisateur dans son home quand il utilise SFTP

Il est possible de *bloquer* l'utilisateur dans son home quand il se connecte en SFTP (FTP via SSH), pour se faire il va falloir modifier le fichier de configuration `/etc/ssh/sshd_config`:

```shell
$ vi /etc/ssh/sshd_config
Subsystem sftp internal-sftp # Permet de specifier que l'on veut configurer SFTP
Match group sftponly # Seulement les utilisateurs du groupe sftponly seront chroote (attention aux regles AllowUsers / AllowGroups /...)
# on peut egalement utiliser Match user userftp a la place
         ChrootDirectory /home/%u # Bloque l'utilisateur dans son home (exemple /home/ankirama)
         X11Forwarding no # N'autorise pas l'affichage graphique (en redirection)
         AllowTcpForwarding no # N'autorise pas l'encapsulation via le protocole SSH
         ForceCommand internal-sftp # Force l'utilisation de SFTP uniquement
```

Attention, en faisant ceci l'utilisateur ne pourra plus se connecter en SSH!