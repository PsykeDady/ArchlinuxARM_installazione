 

# Installazione Archlinux ARM 64bit su PI4

**Autore**: *Davide Galati (in arte PsykeDady)*

**Versione**: `0.1 metempsicosis`



<pre>
Copyright (C)  Davide Galati (aka PsykeDady).
Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.3
or any later version published by the Free Software Foundation;
with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.
A copy of the license is included in the section entitled "GNU
Free Documentation License". 
</pre>

La versione completa della licenza si può trovare nella sezione: [Licenza per intero](##Licenza-per-intero)



## intro

Questa guida raccoglierà quelle che sono le mie esperienze in merito all'installazione e configurazione di Archlinux a 64bit su RaspBerry PI4

> **<u>ATTENZIONE</u>**: 
>
> ==Al momento esistono davvero pochi pacchetti su Archlinux arm a 64bit, ci son diversi problemi in fase di installazione e non è un installazione per nulla stabile. Consiglio, per chi necessita di un installazione funzionante o pronta, di usare **la versione 32bit** o meglio ancora **Manjaro a 64bit** che è molto più fornito.==

## Download e Flash su SD

Assicuratevi di avere un SD abbastanza capiente a quelle che sono le vostre necessità. Considerate che, a sistema pulito, ci son poco meno di 2 Gb già occupati.

Download e flash vanno fatti da un pc funzionante, nella guida supporremo con un sistema operativo linux.

### partizionamento e mount

Inserite la vostra schedina sd nel pc, quindi date fdisk per modificarne le partizioni. 

> *<u>SUGGERIMENTO</u>*:
> se si ha poca confidenza, si può provare ad usare `cfdisk` che propone una comoda interfaccia grafica

Innanzitutto dare blkid per sapere su quale device operare:

`sudo blkid`

Con l'identificativo (supponiamo `/dev/mmcblk0` ), utilizzare quindi `fdisk`:

```bash
sudo fdisk /dev/mmcblk0

o
n
p
1
#premi invio
+200M
t
c

n
p
2
#premi invio
#premi invio
w
yes
```

Quindi formattiamo le due partizioni appena create

```bash
# la prima in vfat
sudo mkfs.vfat /dev/mmcblk0p1 -n "ARCH_BOOT" 

# la seconda in ext4
sudo mkfs.ext4 /dev/mmcblk0p2 -L "ARCH_ROOT"

#ovviamente potete personalizzare (o non scrivere) i nomi ARCH_ROOT e ARCH_BOOT, ma non scrivete neanche i comandi n e L
```

Creiamo due cartelle nel nostro file system e montiamo le due partizioni appena create:

```bash
mkdir root boot
sudo mount /dev/mmcblk0p1 boot
sudo mount /dev/mmcblk0p2 root
```

### download e flash

Quindi scarichiamo l'immagine, e trasferiamone tutto il contenuto nella root

```bash
wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz

sudo bsdtar -xpf ArchLinuxARM-rpi-aarch64-latest.tar.gz -C root
```

Spostiamo il contenuto della cartella boot della root nella partizione di boot

```bash
sudo mv root/boot/* boot
```

Quindi andiamo ad applicare alcune modifiche ai file di boot.

### modificare i file di boot

Innanzitutto, dando `blkid`, segnamoci gli **UUID** delle due partizioni. Per isolare gli output:
```bash
sudo blkid /dev/mmcblk0p1
sudo blkid /dev/mmcblk0p2
```

Una volta segnati, apriamo con il nostro editor di testo preferito ( e con i permessi di amministratore ) il file `root/etc/fstab` e modifichiamolo come segue:

```bash
UUID=<uuid della partizione di boot>  /boot vfat defaults  0 0

UUID=<uuid partizione di root>  /  ext4  defaults 0 0
```



Modifica più importante, andiamo a modificare il file  `boot/boot.txt` esattamente la riga che inizia con `setenv`. Modifichiamola come segue:

```bash
setenv bootargs console=ttyS1,115200 console=tty0 root=UUID=<inserite il vostro UUID della partizione 2> rw rootwait smsc95xx.macaddr="${usbethaddr}"

```

> <u>NOTE</u>:
>
> la modifica consiste nell'inserire staticamente la UUID della root, la parte modificata è proprio quella che inizia con `root=...` che passa da `PARTUUID=...` a `UUID=...`



Ora installiamo sul nostro sistema i tools uboot.

```bash
# su ubuntu e derivate
sudo apt install u-boot-tools

# su arch 
sudo pacman -S uboot-tools

# per altre distro a voi l'onore ...
```

e diamo il comando (mi raccomando il punto iniziale):  

```bash 
cd boot
sudo ./mksrc
```



### il file config.txt 

Il file di configurazione `boot/config.txt` contiene tutte le impostazioni che vengono applicate fin dall'avvio.  Queste includono audio/video/accelerazione etc...

Se il file non c'è, **createlo**.

#### lo schermo

Se avete uno schermo HDMI potrebbe essere necessario attivare la safe mode. 

Aggiungete quindi la riga:
`hdmi_safe=1`

Salvate e chiudete.



<u>Le impostazioni possono cambiare ovviamente da monitor a monitor</u>, nella sezione [Link Utili](#Link-Utili) trovate vari link con template e documentazioni su come cambiarle!

Per impostare le dimensioni di un monitor specifico di inserisce un particolare **group** e quindi una **hdmi_mode**. 
Seguendo le tabelle di uno dei link citati, si può notare ad esempio che il <u>gruppo 2</u> e la <u>modalità 82</u> portano ad una risoluzione **1920x1080**

```properties
hdmi_group=2
hdmi_mode=83
```



#### config d'esempio

Inoltre vi condivido le impostazioni del mio **config** (risoluzione a 1600x900 con 24 pixel di overscan):

```properties
enable_uart=1

dtparam=i2c_arm=on

gpu_mem=256
dtoverlay=vc4-fkms-v3d
hdmi_force_hotplug=1


hdmi_ignore_edid=0xa5000080
hdmi_group=2
hdmi_mode=83
disable_overscan=0
overscan_left=24
overscan_right=24
overscan_top=24
overscan_bottom=24
```



#### config manjaro

alternativamente ecco il config di manjaro

```properties
# See /boot/overlays/README for all available options

gpu_mem=64
initramfs initramfs-linux.img followkernel
kernel=kernel8.img
arm_64bit=1
enable_gic=1
disable_overscan=1

#enable sound
dtparam=audio=on
hdmi_drive=2

#enable vc4
dtoverlay=vc4-fkms-v3d
max_framebuffers=2
```



### troubleshoot: RAM > 4G

Se la memoria del raspberry è superiore ai 4Gb, potreste avere un problema legato al malfunzionamento delle porte usb ( in realtà non funzionano proprio ) 

Due sono le possibili soluzioni.

#### limitare la ram

Una soluzione potrebbe essere quella di limitare la dimensione della ram utilizzata scrivendo questa riga nel file di **config.txt**

```properties
total_mem=4000 # limita la RAM a 3.7 GiB
```

#### soluzione consigliata

Esistono in realtà delle patch che potete scaricare e inserire nella partizione di boot per evitare la limitazione di ram. Andate su [questo repository](https://github.com/raspberrypi/firmware) e scaricate questi file dalla cartella **boot** :

- [bcm2711-rpi-4-b.dtb](https://github.com/raspberrypi/firmware/blob/master/boot/bcm2711-rpi-4-b.dtb)
- [fixup4.dat](https://github.com/raspberrypi/firmware/blob/master/boot/fixup4.dat)
- [fixup4cd.dat](https://github.com/raspberrypi/firmware/blob/master/boot/fixup4cd.dat)
- [fixup4db.dat](https://github.com/raspberrypi/firmware/blob/master/boot/fixup4db.dat)
- [fixup4x.dat](https://github.com/raspberrypi/firmware/blob/master/boot/fixup4x.dat)
- [start4.elf](https://github.com/raspberrypi/firmware/blob/master/boot/start4.elf)
- [start4cd.elf](https://github.com/raspberrypi/firmware/blob/master/boot/start4cd.elf)
- [start4db.elf](https://github.com/raspberrypi/firmware/blob/master/boot/start4db.elf)
- [start4x.elf](https://github.com/raspberrypi/firmware/blob/master/boot/start4x.elf)

Quindi sostituiteli ai corrispettivi della vostra cartella di boot.
Ora dovrebbe funzionare tutto, se così non dovesse essere, attuate la [prima soluzione](####limitare-la-ram)!



### fine


Ok, la scheda SD è pronta! assicuriamoci solo di smontarla correttamente così:

```bash
sudo sync
sudo umount root,boot
```

*inseriamola nel nostro raspberry!*



## Primo Avvio

Per accedere al nostro sistema, inseriamo nome utente e password:

> **nome utente** : root
>
> **password** : root

Se si ha una tastiera italiana:

`loadkeys it`

se si ha uno schermo hidpi digitate anche:

`setfont ``/usr/share/kbd/consolefonts/sun12x22.psfu.gz`



Quindi prima cosa da fare, cambiare password e sceglierne una più appropriata!  
`passwd`

### connessione e aggiornamento

Siamo connessi? se la risposta è no, usiamo 

`wifi-menu` 

per connetterci e 

`dhpcd` 

per forzare l'assegnamento dell'ip. 

><u>NOTE</u>:
>
> Se siamo connessi tramite cavo, possiamo mandare direttamente il secondo comando.



Quindi aggiorniamo il keyring di pacman:

```bash
pacman-key --init
pacman-key --populate archlinuxarm
```



e aggiorniamo eventualmente la nostra distro con:

```bash
pacman -Syyu
```

### tmp file system

Nel nostro file system una cartella speciale, */tmp*, contiene file e cartelle temporaneamente creati dai programmi per il loro corretto funzionamento. Questa cartella viene montata da systemd automaticamente in memoria **RAM** in modo da essere svuotata ogni qual volta il calcolatore viene spento.

È comunque possibile, per quei pc che non hanno un gran quantitativo di memoria disponibile, montare la cartella nello spazio di archiviazione piuttosto.

> *<u>SUGGERIMENTO</u>*:
> Se installate frequentemente programmi da AUR con un AUR-helper è possibile che quest’ultimo, facendo tante scritture su /tmp, vi saturi facilmente la memoria. Consiglio particolarmente questo procedimento ai pc con meno di 8GB di RAM che installeranno quindi molti pacchetti da AUR

Per disabilitare il montaggio automatico basta dare:

`sudo systemctl mask tmp.mount`

Bisogna comunque tener conto che disabilitando questo comportamento, il contenuto della cartella dei file temporanei non verrà più cancellato allo spegnimento del dispositivo, va quindi eliminato manualmente.
Si può tornare al comportamento di default specificando manualmente il montaggio in `/etc/fstab`, aggiungendo questa riga al file:

`tmpfs /tmp tmpfs nodev,nosuid 0 0`

Si può anche specificare una size massima per evitare che la ram venga monopolizzata dalla sola partizione di file temporanei, facendo un esempio con il limite di 2GB avremmo:

`tmpfs /tmp tmpfs nodev,nosuid,size=2G 0 0`

Altre interessanti informazioni su **tmpfs** si trovano sulla guida di arch.

### configurazioni base

Si scelga un nome personalizzato per vedere in rete il raspberry:

`echo "NOMEIMPORTANTE" > /etc/hostname`


Si modifichi la lingua del sistema (si usi il proprio editor preferito, nell'esempio sarà **nano**)

`nano /etc/locale.gen`

decommentiamo le righe che iniziano con **it_IT**

quindi diamo il comando:

`locale-gen`



In seguito bisogna generare un buon */etc/locale.conf*, aprire quindi con un editor il file 

`nano /etc/locale.conf`

E quindi si digiti all'interno:

```bash
LANG=it_IT.UTF-8
LC_COLLATE="C"
LC_TIME="it_IT.UTF-8"
LANGUAGE="it_IT:en_EN:en"
```



Si imposti la mappatura della tastiera del tty ( se italiana ) con:
`echo "KEYMAP=it" > /etc/vconsole.conf`

e quindi il fuso orario di sistema con:
`ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime`

Dopo le configurazioni sulla lingua, vi conviene riavviare:

`reboot`



## Configurazioni e package manager

### Installazione di pacchetti di base

Mancano alcuni pacchetti di base del sistema, suggerisco di dare:

```bash
pacman -S base base-devel git bash-completion man-db man-pages git xdg-user-dirs linux-aarch64-headers xorg-server xorg-xrefresh xorg-xinit
```



### Driver video

Esistono due driver video che può sfruttare il raspberry: 

- `xf86-video-fbdev`
- `xf86-video-fbturbo-git`

Per installare il primo:

`pacman -S xf86-video-fbdev`



### configurazione utente

Quindi cancelliamo l'utente già creato nel nostro sistema (utente **alarm**) e mettiamone uno che piace di più a noi

```bash 
# cancella vecchio utente
userdel -rf alarm

# aggiungi un nuovo utente
useradd -m -g users -G
wheel,video,audio,sys,lp,storage,scanner,games,network,disk,input -s /bin/bash
<nome utente>

```

Oppure (*sconsigliato*) cambiamo il nome al precedente:

```bash
usermod -l <nuovo utente> alarm

mv /home/alarm /home/<nuova home>

usermod -d <nuova home> <nuovo utente>
```



indichiamo al programma sudo ( che ci permette di effettuare operazioni in modalità amministratore) che il nostro utente fa parte del gruppo amministratore, questo tramite visudo:

```bash
#impostiamo prima un editor di testo a noi più amichevole, di default è vi
export EDITOR=<nome editor>
visudo
```

a questo punto esistono due modi di impostare i permessi di amministratore: con richiesta della password (consigliato) e senza richiesta.

Nel primo caso decommentare la riga:

`wheel ALL=(ALL) ALL`

nel secondo caso decommentare:

`wheel ALL=(ALL) NOPASSWD: ALL`



Facciamo accesso con il nostro utente con su:

```bash
su <nomeutente>
cd
```

e creiamo le cartelle utente con:

`xdg-user-dirs-update`

> <u>NOTE</u>:
>
> Ricordati che puoi cambiare il nome delle cartelle di base (Immagini, Video ...etc) modificando il corrispettivo in `/home/<nome utente>/.config/user-dirs.dirs` inserendo ad uno ad uno i nomi che desideriamo sostituire.
>
> I nomi predefiniti dovrebbero essere quelli della lingua impostata

### Configurare pacman

Perché archlinux? perchè complicarsi la vita con questa tortura che ti porta a perdere una giornata per l'installazione di un sistema operativo? Le risposte sono tante, ma la prima in assoluto è il gestore di pacchetti **pacman** e tutto ciò che ne deriva, compreso il famoso **AUR: Arch User Repository**.

Prima di tutto abbelliamo il nostro gestore! 
Sempre con il nostro editor preferito modifichiamo il file `/etc/pacman.conf`: andiamo a decommentare la riga con scritto **Color** e aggiungiamo sotto **ILoveCandy**. 

### AUR-helper

Un AUR-helper cerca per voi e aggiorna (o segnala aggiornamenti) di pacchetti su AUR. Ne esistono diversi tipi, da quelli più semplici che fanno solo la ricerca a quelli più complessi che cercano, gestiscono e aggiornano i pacchetti e anche il sistema al posto di **pacman**. Quest’ultimi son detti anche **pacman-wrapper** ed usano la stessa sintassi di **pacman** in genere. 

>**<u>ATTENZIONE</u>**:
>
>==Tenete sempre conto che siamo su un architettura ARM e tantissimi pacchetti di AUR non hanno assolutamente alcun supporto. Fate attenzione==

Supponiamo di volerne installare uno di nome `<nomeaurhelper>` e di cui abbiamo il link del repository ( che si può facilmente trovare con una ricerca su internet). Generalmente si installa così:

```bash
git clone https://aur.archlinux.org/<nomeaurhelper>.git
cd <nomeaurhelper>
makepkg -si
```

Installiamone quindi uno: **yay** ! Per lo step successivo è <u>**fortemente consigliato**</u> l'accesso con l'utente e <u>non con root</u>.

Per accedere come utente:

`su nomeutente`

Quindi:

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

D'ora in poi potete sostituirlo in tutto e per tutto al package manager, l'unica differenza sta nel fatto che cerca i pacchetti anche su **AUR**, questo <u>immenso repository di pacchetti</u> offerti dalla comunità di Archlinux, ci trovate davvero di tutto dentro (*perciò fate comunque attenzione*).

> <u>NOTE</u>:
>
> yay, così come la maggior parte degli AUR-helper, non vanno usati come utenti privilegiati (con **sudo** o **l’account root** per intenderci)

Tramite yay potete visualizzare la differenza tra versione presente e quella aggiornata del software, oppure visionare e modificare il **PKGBUILD** (una sorta di file che contiene le istruzioni di compilazione del pacchetto, con dipendenze, siti da dove scaricare i binari ed eventualmente potete anche selezionare delle opzioni), questo lo rende uno strumento molto potente!

> <u>NOTE</u>:
>
> Altri AUR-helper interessanti li trovate nella guida, perciò se per un motivo o per l’altro quello segnalato in guida non dovesse funzionare, vi invito a visitarne la pagina con la lista 


## Desktop Environment, Display Manager, Network Manager e servizi systemd

Premessa, diceva il mio maestro di basso: <u>ad ognuno la sua pistola</u>. Su linux chiunque ha piena libertà di scegliere l’ambiente grafico (o di non sceglierlo proprio) che più gli garba, sia per leggerezza, per bellezza, per complessità d’uso o per la sua malleabilità. Ce n’è per tutti i gusti! La wiki in questo può essere molto più completa di qualunque cosa io possa scrivere qui sotto, vi elencherò in un primo momento i DE più conosciuti e le loro peculiarità.

Per sistemi poco prestanti consiglio **XFCE**, **LXDE** o (per chi ha la pazienza di configurarlo) **I3WM**, queste sono tre validissime alternative che con qualche tocco di eleganza hanno fatto la storia delle configurazioni e dei temi <u>più belli e apprezzati</u> del mondo Linux.

> <u>NOTE</u>:
>
> D’ora in poi potrete operare tranquillamente con l’account root così come quello utente (consigliato). Quando agirete come root non sarà necessario specificare **sudo** all’inizio dei comandi.
>
> Durante però l’esecuzione di un **aur-helper** ricordo che il root potrebbe dare problemi. 


Per chi ha uno schermo HIDPI consiglio **plasma**, **Cinnamon**, **Mate**, **DDE** o **GNOME**. Andiamo con ordine: 

- reputo **plasma** l’ambiente più avanzato dal punto di vista delle animazioni, delle personalizzazioni e dei servizi automatici, inoltre nel suo approccio minimale occupa meno delle sue alternative (intorno ai 300Mb per esperienza personale).  
- **Cinnamon** è un ambiente molto conosciuto dagli utenti di *LinuxMint*, infatti è la scelta predefinita ed è sviluppata e mantenuta dal team che sostiene la distribuzione. Lo considero l’alternativa più avanzata ma anche più tra le più pesanti.
- **Mate** invece è una buona via di mezzo, non molto avanzato con i tempi, da poco ha introdotto il supporto all’hidpi e comunque ho riscontrato qualche problemino nell’usarlo, ma è leggero ed è altamente modulare, permette senza problemi di usare componenti di altri **WM** o **DE**.
- **XFCE** probabilmente il primo DE in assoluto. Semplice, modulare e leggero. Un po' vecchiotto a primo impatto ma con un buon tema..
- **Gnome** è un ambiente conosciuto per la sua pesantezza in termini sia di memoria occupata che di animazioni, nella versione di *Fedora Linux* è straordinariamente leggera e funzionante, ma si trasforma totalmente su altre distribuzioni. Resta uno degli ambienti più apprezzati per via delle estensioni (che comunque lo appesantiscono) e della “omogeneità” dei suoi temi.

Ci sarebbe infine **Enlightment**, un DE con tiling manager che ho trovato difettoso in alcune applicazioni (come il noto I.M. **Telegram**). È comunque un alternativa che cito poiché gestisce nativamente l’HIDPI. <u>In ogni caso consultate la guida ufficiale per ogni chiarimento su DE che non sono qui trattati</u>.

>*<u>SUGGERIMENTO</u>*:
>Normalmente consiglierei plasma. Ma le risorse limitate del raspberry non permettono di godere a pieno dell'esperienza.
>
>In quest'ottica, la mia guida tratterà un ambiente più leggero: **xfce4** 



### XFCE4

Uno dei primissimi **Desktop Environment** per linux, è modulare, facilmente estendibile, leggero e facilmente configurabile.

<u>Non aspettatevi però miracoli</u> dal punto di vista delle animazioni, anzi non ve ne aspettate proprio di animazioni.

Per installarlo con tutti i suoi plugin (consigliato) :

`sudo pacman -S xfce4 xfce4-goodies`

Potrebbe poi essere carino installare un software che vi permetta di usare un effeto stile exposé di Mac, consiglio con AUR di installare [skippy-xd](https://aur.archlinux.org/packages/skippy-xd-git/):
```yay -S skippy-xd-git```


O perché no, compilatelo a mano dal suo repo git:

```bash
git clone https://github.com/dreamcat4/skippy-xd.git
cd skippy-xd
make
make install
```

Per farlo partire poi:

```bash
skippy-xd-runner --toggle
```



### Display Manager

>**<u>ATTENZIONE</u>**:
>
>==attualmente (forse a causa dell'accelerazione grafica) non mi funziona alcun DM, consiglio l'avvio con `xorg-xinit`==

Questo genere di software è banalmente quello che vi si presenta alla schermata di avvio chiedendovi la password per farvi accedere all’ambiente desktop avviando per altro il server X (o qualunque server grafico voi usiate).  
In genere è possibile selezionare anche un utente e un desktop diverso nel caso in cui ne abbiate più di uno installato.

Ad esempio installiamo **lightdm**:  
`sudo pacman -S lightdm`  

Per utilizzare un DM dovrete abilitarlo come servizio di systemd, nel caso di `lightdm` ad esempio avremo:  
`sudo systemctl enable lightdm`  

Al riavvio potrete godere della vostra interfaccia d’accesso.

#### xorg-xinit

Normalmente, se l’avete installato, anche `xorg-xinit` può avviare il vostro ambiente grafico, questa soluzione potrebbe accontentare quell’utenza che vuole fare del proprio portatile una vera freccia ad avviarsi. Per usufruire di xinit dobbiamo prima di tutto scrivere il comando di avvio del nostro DE all’interno del file *~/.xinitrc* , il file si trova nella home e varierà quindi per ogni utente.
Abbiamo dunque:

- per kde :  

  `exec startplasma-x11`

- per GNOME con X Server:

  `export GDK_BACKEND=x11`

  `exec gnome-session`

- per xfce  

  `exec startxfce4`

- per mate

  `exec mate-session`

- per dde

  `exec startdde`

- per cinnamon

  `exec cinnamon-session`

Per altri DE consulare le relative wiki!



### NetworkManager e servizi di rete

Dietro le quinte, quando nela guida si è fatto uso di `wifi-menu`, il servizio che permetteva di accedere ad internet nient’altro era che **netctl**. Se avete seguito la guida dall’inizio, sia *netctl* che *wifi-menu* continueranno a funzionare egregiamente. Tuttavia spesso è utile avere a che fare con servizi più user-friendly, che vi notifichino ad esempio quando cade la connessione, vi mostrino la potenza di segnale continuamente e facilitino l’inserimento di ogni tipo di credenziali, proteggendone anche il contenuto se siete in pubblico. 

Il servizio di connessione per eccellenza sui sistemi linux è **NetworkManager**.

> <u>NOTE</u>:
>
> Usando NetworkManager disabiliterai netctl. Tentando di usare netctl quando nm è attivo causerai errori. Utilizza systemd per gestire il passaggio da un servizio all’altro se dovessi preferirne uno ad un altro in base alle situazioni

Quindi per installare networkmanager:

` sudo pacman -S networkmanager`

Se eventualmente vi servono particolari tipi di connessione, come vpn o point-to-point protocol, potrebbero servirvi pacchetti aggiuntivi:
`sudo pacman -S networkmanager-pptp networkmanager-vpnc`
Alcuni DE si portano nell’installazione completa un applet che interagisce con NetworkManager, tuttavia se non fosse così la wiki spiega quali pacchetti installare per connettersi comodamente con un click.
Nel più comune dei casi comunque viene usato quello di gnome:
`sudo pacman -S network-manager-applet`
è necessario abilitare NetworkManager con **systemd**, altrimenti sarà necessario ogni riavvio richiamare manualmente il servizio. Quindi
`sudo systemctl enable NetworkManager`



### Systemd: la bestia nera

Systemd è un insieme di tool che gestiscono servizi e avvio del vostro sistema su base Linux. Non entrerò nel dettaglio spiegando perché molti lo odiano, perché altri lo amano e perché altri ancora, come il sottoscritto, se ne fregano altamente, basta che funzioni.

Vi spiegherò invece come interfacciarvi al gestore facilmente, elencano una serie di comandi e spiegandone l’uso

|       sudo?        | comando                                       | spiegazione                                                  |
| :----------------: | :-------------------------------------------- | :----------------------------------------------------------- |
| :white_check_mark: | `systemctl enable <servizio>`                 | abilita il servizio all’avvio, che viene quindi attivato ogni qualvolta accedete |
| :white_check_mark: | `systemctl start <servizio>`                  | avvia immediatamente il servizio                             |
| :white_check_mark: | `systemctl restart <servizio>`                | spegne e riavvia il servizio                                 |
| :white_check_mark: | `systemctl stop <servizio>`                   | spegne il servizio, contrario di start                       |
| :white_check_mark: | `systemctl disable <servizio>`                | disabilita il servizio, contrario di enable                  |
| :white_check_mark: | `systemctl status <servizio>`                 | controlla lo stato del servizio, se è attivo, in errore o spento |
| :heavy_minus_sign: | --------------------------------------------- | -------------------------------------------------------------------------------------------- |
|                    | `systemctl poweroff`                          | spegne il sistema                                            |
|                    | `systemctl reboot`                            | riavvia il sistema                                           |
|                    | `systemctl hibernate`                         | iberna il sistema, da usare solo se avete  attivato l’ibernazione in modo corretto |
|                    | `systemctl suspend`                           | sospende il sistema                                          |
|                    | `systemctl suspend-then-hibernate`            | sospende per un certo periodo di tempo.  Poi iberna          |
|                    | `systemctl hybrid-sleep`                      | Sospende e iberna il sistema. Così che se la batteria si scarica, il pc è comunque ibernato |

Prendiamo ad esempio che vogliate usare nuoamente *netctl* anzichè *NetworkManager*, la procedura corretta sarebbe: 

```bash
sudo systemctl stop NetworkManager
sudo systemctl start netctl
wifi-menu
sudo dhcpcd
```

Al contrario invece

```bash
sudo dhcpcd -x
sudo systemctl stop netctl
sudo systemctl start NetworkManager
```

> <u>NOTE</u>:
>
> Spesso **NetworkManager**, anche a servizio spento, prende il sopravvento perchè il plugin ( in background tramite il DE ) resta attivo. In tal caso bisogna ripetere la procedura più volte se si vuole usare **netctl**.

## Post-personalizzazioni di sistema by PsykeDady

Seguiranno alcune personalizzazioni tipiche dei miei sistemi. Questo capitolo è assolutamente opzionale e non necessario al funzionamento del sistema

### I sette 'nano'

da linea di comando, l’avrete capito, il mio editor preferito è **nano**. Premesso che io penso che la linea di comando serva giusto per piccole modifiche, non uso editor che consentono, anche da terminale, di progettare, sviluppare e scrivere documenti complessi come potrebbe essere *VIM*.

Detto ciò, piccole modifiche non significa che non si debba stare comodi no? quindi vediamo come fare a rendere più piacevole l'uso di nano.

Creiamo con il nostro editor preferito il file *~/.nanorc* :

```bash
set softwrap
set autoindent
set linenumbers
	
include "/usr/share/nano/*.nanorc"
```

- **softwrap**: fa andare a capo le righe che superano la lunghezza del terminale, senza però generare un nuovo fine linea nel testo, così non vi dovrete spostare per vedere cosa contengono quelle lunghe infinite linee che scrivete

- **autoindent**: se programmate con nano, questo può essere davvero un aiuto importante. Abilitare questa opzione vi permette, una volta indentata una linea di codice, di mantenere sulle righe di sotto la stessa tabulazione precedente. <u>Se programmate e non indentate, o siete figli di satana o vi volete davvero male o ancora siete alle prime armi</u> (in questo caso vi perdono, ma <u>rimediate</u>)

- **linenumbers**: una volta che avete abilitato softwrap non capirete quando inizia una riga e quando invece sta continuando quella precedente. Questa opzione mostra i numeri riga, in termini di numeri cardinali, ogni suo inizio

- **include** /bla/bla/bla: permette l’evidenziamento della sintassi dei linguaggi di programmazione supportati da nano. Il percorso indicato è dove normalmente sono salvati i template che descrivono come evidenziare la sintassi di quel linguaggio. Su sistemi diversi da arch potrebbe non essere lo stesso path.


### oh zsh, mio amore :heart:

*l terminale è vita, il terminale è amore.* Spesso per l’utilizzatore linux il terminale è davvero tutto. Ecco perchè abbellirlo e renderlo funzionare dovrebbe essere una priorità.

Allora ecco che entrano in gioco le *‘alternative’* alla shell per eccellenza, sua maestà `bash`.

La proposta più popolare è sicuramente `zsh`. In realtà l’avete già usata e non lo sapete (?), infatti la live di archlinux ne è equipaggiata.

Quindi installiamo `zsh` e anche il suo famoso gestore di plugin: **oh-my-zsh**. Qui entra in gioco il famoso AUR con l’aur helper scelto, quindi non eseguite il prossimo comando da un account root.

`yay -S zsh oh-my-zsh-git zsh-theme-powerlevel10k`

Il terzo pacchetto è uno dei temi più famosi di zsh, la configurazione che vi propongo è proprio quella che si basa su questo pacchetto.

In realtà solo oh-my-zsh si trova su AUR, gli altri si dovrebbero trovare sui repository standard, ma `yay` come già detto, installerà anche i pacchetti normali ( è come fosse un estensione di `pacman`)

Copiamoci nella home innanzitutto il file rc di oh-my-zsh:

`cp /usr/share/oh-my-zsh/zshrc ~/.zshrc`

Modifichiamo quindi il file appena creato con il nostro editor preferito, andiamo alla linea `ZSH_THEME` e scriviamo 

`ZSH_THEME="powerlevel10k/powerlevel10k"`

per funzionare comunque bisogna anche copiare il tema nella cartella temi di zsh:

`sudo ln -sf /usr/share/zsh-theme-powerlevel10k /usr/share/oh-my-zsh/themes/powerlevel10k`

Il nostro `zsh` dovrebbe essere pronto, possiamo testarlo digitando zsh ed eventualmente impostarlo come shell predefinita tramite `chsh`:

`chsh -s /usr/bin/zsh`


#### nerd font
Per far funzionare correttamente powerlevel10k, dobbiamo installare un font font che ne supporti i caratteri.
Scarichiamo Fura Code ad esempio:
```bash
#!/usr/bin/env bash

fonts_dir="${HOME}/.local/share/fonts"
if [ ! -d "${fonts_dir}" ]; then
    echo "mkdir -p $fonts_dir"
    mkdir -p "${fonts_dir}"
else
    echo "Found fonts dir $fonts_dir"
fi

for type in Bold Light Medium Regular Retina; do
    file_path="${HOME}/.local/share/fonts/FiraCode-${type}.ttf"
    file_url="https://github.com/tonsky/FiraCode/blob/master/distr/ttf/FiraCode-${type}.ttf?raw=true"
    if [ ! -e "${file_path}" ]; then
        echo "wget -O $file_path $file_url"
        wget -O "${file_path}" "${file_url}"
    else
	echo "Found existing file $file_path"
    fi;
done

echo "fc-cache -f"
fc-cache -f
```

Alternativamente, usate yay:  
`yay -S nerd-fonts-fira-code`

### fstab montare ~~la panna~~ partizioni all'avvio

È possibile, volendo, montare altre partizioni all’avvio. 

Innanzitutto individuiamo le partizioni interessate:

`sudo blkid`

ci apparirà una lista di file del tipo **/dev/sdXY** dove X è una lettera (parte da a in genere, e indica il disco) e Y è un numero (indica la partizione, parte da 1). Individuiamo (basandoci sulla Label, se ne abbiamo impostata una, oppure sul tipo di partizione) la riga corrispondente al disco che ci interessa e annotiamoci la **UUID**

Poi creiamo una cartella dove vogliamo trovare il nostro disco ad ogni avvio, ad esempio **/media/SharingVolume**, il nome e il percorso possono essere di pura fantasia, potreste avere la necessità di usare i permessi di amministratore per questo:

`sudo mkdir -p /media/Sharing``Volume`

Poi, assicurandoci che la partizione non sia montata ( `sudo umount /dev/sdXY `nel caso contrario) scriviamo questa riga nel nostro `/etc/fstab`

`UUID=<uuid annotato> /percorso/creato/prima tipo_partizione defaults 0 0`

Ad esempio supponiamo di voler montare **/dev/sdc3** con **UUID=123-123-123-123** sotto il nome di **Sharing****Volume**, considerando che è una partizione di tipo **ntfs**. Allora scriveremo:

`UUID=123-123-123-123 /media/SharingVolume ``ntfs-3g`` defaults 0 0`

Salviamo e chiudiamo il file. Poi proviamo:

`sudo mount -a`

Se tutto va bene, la partizione è montata, e questo significa che sarà montata con successo ogni riavvio. Altrimenti cancellate immediatamente la riga inserita, e documentatevi sulla wiki per capire cosa avete sbagliato.

> **<u>ATTENZIONE</u>**:
>
> ==la pena per un fstab mal fatto è l’impossibilità di avvio del sistema. Quindi assicuriamoci che sia tutto apposto prima di riavviare==

### fish, un ulteriore avanzatissima shell

Ancora più avanzata è la shell `fish`, che possiede per altro un proprio linguaggio di scripting, diverso da quello di bash.

Lo possiamo installare digitando

`sudo pacman -S fish`

Fish è un po’ particolare da tutti i punti di vista, ad esempio il suo file rc lo trovate in *~/.config/config.fish*.

Per personalizzare la shell dovrete usare `fish_config`, che aprirà un server web sul vostro browser dove potrete scegliere facilmente il tema, il prompt e altre funzionalità.

Un altra particolare funzione di `fish` è il suo saluto di benvenuto, che può piacere come no. Per ridefinirlo scrivere nell’ rc queste righe: 

```bash
function fish_greeting
	#...scrivere qui il codice o lasciare vuoto se si vuole disattivare
end
```

Per usarlo come shell predefinita si può digitare:

`chsh -s /usr/bin/fish`

### neofetch

Agli utenti linux piace avere tutto sotto controllo, e sopratutto piace che questo continuo monitorare sia esteticamente appagante.
 Ecco perché, chi per noi, ha creato dei tool che mostrano in modo sgargiante quello che tutte le informazioni di cui si ha bisogno. Uno di questi è `neofetch`, che possiamo semplicemente installare con 

`sudo pacman -S neofetch`

Possiamo abilitare varie funzionalità installando i pacchetti aggiunti:

`pacman -S catimg feh imagemagick jp2a libcaca nitrogen pacman-contrib w3m`

Avviamolo almeno una volta scrivendo neofetch in modo da fargli creare i file di configurazione e poi andiamo a modificarli
`nano ~/.config/neofetch/config.conf`
Ecco la mia semplicissima configurazione della sezione info, che è quella poi che determina cosa uscirà stampato sul terminale:

```bash
print_info() {
    info title
    info underline

    info "OS" distro
    info "Host" model
    info "Kernel" kernel
    info "Uptime" uptime
    info "Packages" packages
    info "Shell" shell
    info "Resolution" resolution
    info "DE" de
    info "WM" wm
    info "WM Theme" wm_theme
    info "Theme" theme
    info "Icons" icons
    info "Terminal" term
    info "Terminal Font" term_font
    info "CPU" cpu
    info "GPU" gpu
    info "Memory" memory
    info underline
    # info "GPU Driver" gpu_driver  # Linux/macOS only
    # info "CPU Usage" cpu_usage
    info "Disk" disk
    info "Battery" battery
    # info "Font" font
    info "Song" song
    # info "Local IP" local_ip
    # info "Public IP" public_ip
    # info "Users" users
    # info "Install Date" install_date
    # info "Locale" locale  # This only works on glibc systems.

    info line_break
    info cols
    info line_break
}

```

Se studiate bene la wiki e le documentazioni sul web potrete sfruttare al meglio il vostro neofetch.

Poniamo di voler far partire neofetch con l’apertura del terminale (altra cosa che piace molto in genere) potete inserire `neofetch` nel file rc del vostro interprete preferito ( *~/.bashrc* per **bash**, *~/.zshrc* per **zsh**, etc...)

### Sotto effetto di LSD

Se avete aperto questa sezione sperando che si parli di erbe, sarete abbastanza delusi. Nonostante questo vi assicuro che `lsd` è un piccolo software che merita di essere approfondito. 
 Grazie ad esso quando vorrete vedere la lista dei file da terminale, saranno visualizzate anche delle icone simbolo (tipo quella della cartella, oppure la nota per i filemusicali..etc)

per installarlo basta dare:

`sudo pacman -S lsd`

Potete anche decidere di sostituirlo al normale ls ( solito strumento che da terminale fa vedere i file nella cartella) scrivendo nel file rc della shell questa riga:

`alias "ls=lsd"`

> <u>NOTE</u>:
>
> Anche in questo caso potrebbe necessitare di un font in grado di visualizzare alcuni caratteri particolari. consiglio sempre i [nerd-font](####nerd-font) 







## Licenza per intero

**GNU Free Documentation License**

Version 1.3, 3 November 2008

Copyright (C) 2000, 2001, 2002, 2007, 2008 Free Software Foundation, Inc. https://fsf.org/

Everyone is permitted to copy and distribute verbatim copies of this license document, but changing it is not allowed.

**0. PREAMBLE**

The purpose of this License is to make a manual, textbook, or other functional and useful document "free" in the sense of freedom: to assure everyone the effective freedom to copy and redistribute it, with or without modifying it, either commercially or noncommercially. Secondarily, this License preserves for the author and publisher a way to get credit for their work, while not being considered responsible for modifications made by others.

This License is a kind of "copyleft", which means that derivative works of the document must themselves be free in the same sense. It complements the GNU General Public License, which is a copyleft license designed for free software.

We have designed this License in order to use it for manuals for free software, because free software needs free documentation: a free program should come with manuals providing the same freedoms that the software does. But this License is not limited to software manuals; it can be used for any textual work, regardless of subject matter or whether it is published as a printed book. We recommend this License principally for works whose purpose is instruction or reference.

**1. APPLICABILITY AND DEFINITIONS**

This License applies to any manual or other work, in any medium, that contains a notice placed by the copyright holder saying it can be distributed under the terms of this License. Such a notice grants a world-wide, royalty-free license, unlimited in duration, to use that work under the conditions stated herein. The "Document", below, refers to any such manual or work. Any member of the public is a licensee, and is addressed as "you". You accept the license if you copy, modify or distribute the work in a way requiring permission under copyright law.

A "Modified Version" of the Document means any work containing the Document or a portion of it, either copied verbatim, or with modifications and/or translated into another language.

A "Secondary Section" is a named appendix or a front-matter section of the Document that deals exclusively with the relationship of the publishers or authors of the Document to the Document's overall subject (or to related matters) and contains nothing that could fall directly within that overall subject. (Thus, if the Document is in part a textbook of mathematics, a Secondary Section may not explain any mathematics.) The relationship could be a matter of historical connection with the subject or with related matters, or of legal, commercial, philosophical, ethical or political position regarding them.

The "Invariant Sections" are certain Secondary Sections whose titles are designated, as being those of Invariant Sections, in the notice that says that the Document is released under this License. If a section does not fit the above definition of Secondary then it is not allowed to be designated as Invariant. The Document may contain zero Invariant Sections. If the Document does not identify any Invariant Sections then there are none.

The "Cover Texts" are certain short passages of text that are listed, as Front-Cover Texts or Back-Cover Texts, in the notice that says that the Document is released under this License. A Front-Cover Text may be at most 5 words, and a Back-Cover Text may be at most 25 words.

A "Transparent" copy of the Document means a machine-readable copy, represented in a format whose specification is available to the general public, that is suitable for revising the document straightforwardly with generic text editors or (for images composed of pixels) generic paint programs or (for drawings) some widely available drawing editor, and that is suitable for input to text formatters or for automatic translation to a variety of formats suitable for input to text formatters. A copy made in an otherwise Transparent file format whose markup, or absence of markup, has been arranged to thwart or discourage subsequent modification by readers is not Transparent. An image format is not Transparent if used for any substantial amount of text. A copy that is not "Transparent" is called "Opaque".

Examples of suitable formats for Transparent copies include plain ASCII without markup, Texinfo input format, LaTeX input format, SGML or XML using a publicly available DTD, and standard-conforming simple HTML, PostScript or PDF designed for human modification. Examples of transparent image formats include PNG, XCF and JPG. Opaque formats include proprietary formats that can be read and edited only by proprietary word processors, SGML or XML for which the DTD and/or processing tools are not generally available, and the machine-generated HTML, PostScript or PDF produced by some word processors for output purposes only.

The "Title Page" means, for a printed book, the title page itself, plus such following pages as are needed to hold, legibly, the material this License requires to appear in the title page. For works in formats which do not have any title page as such, "Title Page" means the text near the most prominent appearance of the work's title, preceding the beginning of the body of the text.

The "publisher" means any person or entity that distributes copies of the Document to the public.

A section "Entitled XYZ" means a named subunit of the Document whose title either is precisely XYZ or contains XYZ in parentheses following text that translates XYZ in another language. (Here XYZ stands for a specific section name mentioned below, such as "Acknowledgements", "Dedications", "Endorsements", or "History".) To "Preserve the Title" of such a section when you modify the Document means that it remains a section "Entitled XYZ" according to this definition.

The Document may include Warranty Disclaimers next to the notice which states that this License applies to the Document. These Warranty Disclaimers are considered to be included by reference in this License, but only as regards disclaiming warranties: any other implication that these Warranty Disclaimers may have is void and has no effect on the meaning of this License.

**2. VERBATIM COPYING**

You may copy and distribute the Document in any medium, either commercially or noncommercially, provided that this License, the copyright notices, and the license notice saying this License applies to the Document are reproduced in all copies, and that you add no other conditions whatsoever to those of this License. You may not use technical measures to obstruct or control the reading or further copying of the copies you make or distribute. However, you may accept compensation in exchange for copies. If you distribute a large enough number of copies you must also follow the conditions in section 3.

You may also lend copies, under the same conditions stated above, and you may publicly display copies.

**3. COPYING IN QUANTITY**

If you publish printed copies (or copies in media that commonly have printed covers) of the Document, numbering more than 100, and the Document's license notice requires Cover Texts, you must enclose the copies in covers that carry, clearly and legibly, all these Cover Texts: Front-Cover Texts on the front cover, and Back-Cover Texts on the back cover. Both covers must also clearly and legibly identify you as the publisher of these copies. The front cover must present the full title with all words of the title equally prominent and visible. You may add other material on the covers in addition. Copying with changes limited to the covers, as long as they preserve the title of the Document and satisfy these conditions, can be treated as verbatim copying in other respects.

If the required texts for either cover are too voluminous to fit legibly, you should put the first ones listed (as many as fit reasonably) on the actual cover, and continue the rest onto adjacent pages.

If you publish or distribute Opaque copies of the Document numbering more than 100, you must either include a machine-readable Transparent copy along with each Opaque copy, or state in or with each Opaque copy a computer-network location from which the general network-using public has access to download using public-standard network protocols a complete Transparent copy of the Document, free of added material. If you use the latter option, you must take reasonably prudent steps, when you begin distribution of Opaque copies in quantity, to ensure that this Transparent copy will remain thus accessible at the stated location until at least one year after the last time you distribute an Opaque copy (directly or through your agents or retailers) of that edition to the public.

It is requested, but not required, that you contact the authors of the Document well before redistributing any large number of copies, to give them a chance to provide you with an updated version of the Document.

**4. MODIFICATIONS**

You may copy and distribute a Modified Version of the Document under the conditions of sections 2 and 3 above, provided that you release the Modified Version under precisely this License, with the Modified Version filling the role of the Document, thus licensing distribution and modification of the Modified Version to whoever possesses a copy of it. In addition, you must do these things in the Modified Version:

​    • A. Use in the Title Page (and on the covers, if any) a title distinct from that of the Document, and from those of previous versions (which should, if there were any, be listed in the History section of the Document). You may use the same title as a previous version if the original publisher of that version gives permission.

​    • B. List on the Title Page, as authors, one or more persons or entities responsible for authorship of the modifications in the Modified Version, together with at least five of the principal authors of the Document (all of its principal authors, if it has fewer than five), unless they release you from this requirement.

​    • C. State on the Title page the name of the publisher of the Modified Version, as the publisher.

​    • D. Preserve all the copyright notices of the Document.

​    • E. Add an appropriate copyright notice for your modifications adjacent to the other copyright notices.

​    • F. Include, immediately after the copyright notices, a license notice giving the public permission to use the Modified Version under the terms of this License, in the form shown in the Addendum below.

​    • G. Preserve in that license notice the full lists of Invariant Sections and required Cover Texts given in the Document's license notice.

​    • H. Include an unaltered copy of this License.

​    • I. Preserve the section Entitled "History", Preserve its Title, and add to it an item stating at least the title, year, new authors, and publisher of the Modified Version as given on the Title Page. If there is no section Entitled "History" in the Document, create one stating the title, year, authors, and publisher of the Document as given on its Title Page, then add an item describing the Modified Version as stated in the previous sentence.

​    • J. Preserve the network location, if any, given in the Document for public access to a Transparent copy of the Document, and likewise the network locations given in the Document for previous versions it was based on. These may be placed in the "History" section. You may omit a network location for a work that was published at least four years before the Document itself, or if the original publisher of the version it refers to gives permission.

​    • K. For any section Entitled "Acknowledgements" or "Dedications", Preserve the Title of the section, and preserve in the section all the substance and tone of each of the contributor acknowledgements and/or dedications given therein.

​    • L. Preserve all the Invariant Sections of the Document, unaltered in their text and in their titles. Section numbers or the equivalent are not considered part of the section titles.

​    • M. Delete any section Entitled "Endorsements". Such a section may not be included in the Modified Version.

​    • N. Do not retitle any existing section to be Entitled "Endorsements" or to conflict in title with any Invariant Section.

​    • O. Preserve any Warranty Disclaimers.

If the Modified Version includes new front-matter sections or appendices that qualify as Secondary Sections and contain no material copied from the Document, you may at your option designate some or all of these sections as invariant. To do this, add their titles to the list of Invariant Sections in the Modified Version's license notice. These titles must be distinct from any other section titles.

You may add a section Entitled "Endorsements", provided it contains nothing but endorsements of your Modified Version by various parties—for example, statements of peer review or that the text has been approved by an organization as the authoritative definition of a standard.

You may add a passage of up to five words as a Front-Cover Text, and a passage of up to 25 words as a Back-Cover Text, to the end of the list of Cover Texts in the Modified Version. Only one passage of Front-Cover Text and one of Back-Cover Text may be added by (or through arrangements made by) any one entity. If the Document already includes a cover text for the same cover, previously added by you or by arrangement made by the same entity you are acting on behalf of, you may not add another; but you may replace the old one, on explicit permission from the previous publisher that added the old one.

The author(s) and publisher(s) of the Document do not by this License give permission to use their names for publicity for or to assert or imply endorsement of any Modified Version.

**5. COMBINING DOCUMENTS**

You may combine the Document with other documents released under this License, under the terms defined in section 4 above for modified versions, provided that you include in the combination all of the Invariant Sections of all of the original documents, unmodified, and list them all as Invariant Sections of your combined work in its license notice, and that you preserve all their Warranty Disclaimers.

The combined work need only contain one copy of this License, and multiple identical Invariant Sections may be replaced with a single copy. If there are multiple Invariant Sections with the same name but different contents, make the title of each such section unique by adding at the end of it, in parentheses, the name of the original author or publisher of that section if known, or else a unique number. Make the same adjustment to the section titles in the list of Invariant Sections in the license notice of the combined work.

In the combination, you must combine any sections Entitled "History" in the various original documents, forming one section Entitled "History"; likewise combine any sections Entitled "Acknowledgements", and any sections Entitled "Dedications". You must delete all sections Entitled "Endorsements".

**6. COLLECTIONS OF DOCUMENTS**

You may make a collection consisting of the Document and other documents released under this License, and replace the individual copies of this License in the various documents with a single copy that is included in the collection, provided that you follow the rules of this License for verbatim copying of each of the documents in all other respects.

You may extract a single document from such a collection, and distribute it individually under this License, provided you insert a copy of this License into the extracted document, and follow this License in all other respects regarding verbatim copying of that document.

**7. AGGREGATION WITH INDEPENDENT WORKS**

A compilation of the Document or its derivatives with other separate and independent documents or works, in or on a volume of a storage or distribution medium, is called an "aggregate" if the copyright resulting from the compilation is not used to limit the legal rights of the compilation's users beyond what the individual works permit. When the Document is included in an aggregate, this License does not apply to the other works in the aggregate which are not themselves derivative works of the Document.

If the Cover Text requirement of section 3 is applicable to these copies of the Document, then if the Document is less than one half of the entire aggregate, the Document's Cover Texts may be placed on covers that bracket the Document within the aggregate, or the electronic equivalent of covers if the Document is in electronic form. Otherwise they must appear on printed covers that bracket the whole aggregate.

**8. TRANSLATION**

Translation is considered a kind of modification, so you may distribute translations of the Document under the terms of section 4. Replacing Invariant Sections with translations requires special permission from their copyright holders, but you may include translations of some or all Invariant Sections in addition to the original versions of these Invariant Sections. You may include a translation of this License, and all the license notices in the Document, and any Warranty Disclaimers, provided that you also include the original English version of this License and the original versions of those notices and disclaimers. In case of a disagreement between the translation and the original version of this License or a notice or disclaimer, the original version will prevail.

If a section in the Document is Entitled "Acknowledgements", "Dedications", or "History", the requirement (section 4) to Preserve its Title (section 1) will typically require changing the actual title.

**9. TERMINATION**

You may not copy, modify, sublicense, or distribute the Document except as expressly provided under this License. Any attempt otherwise to copy, modify, sublicense, or distribute it is void, and will automatically terminate your rights under this License.

However, if you cease all violation of this License, then your license from a particular copyright holder is reinstated (a) provisionally, unless and until the copyright holder explicitly and finally terminates your license, and (b) permanently, if the copyright holder fails to notify you of the violation by some reasonable means prior to 60 days after the cessation.

Moreover, your license from a particular copyright holder is reinstated permanently if the copyright holder notifies you of the violation by some reasonable means, this is the first time you have received notice of violation of this License (for any work) from that copyright holder, and you cure the violation prior to 30 days after your receipt of the notice.

Termination of your rights under this section does not terminate the licenses of parties who have received copies or rights from you under this License. If your rights have been terminated and not permanently reinstated, receipt of a copy of some or all of the same material does not give you any rights to use it.

\10. FUTURE REVISIONS OF THIS LICENSE

The Free Software Foundation may publish new, revised versions of the GNU Free Documentation License from time to time. Such new versions will be similar in spirit to the present version, but may differ in detail to address new problems or concerns. See https://www.gnu.org/licenses/.

Each version of the License is given a distinguishing version number. If the Document specifies that a particular numbered version of this License "or any later version" applies to it, you have the option of following the terms and conditions either of that specified version or of any later version that has been published (not as a draft) by the Free Software Foundation. If the Document does not specify a version number of this License, you may choose any version ever published (not as a draft) by the Free Software Foundation. If the Document specifies that a proxy can decide which future versions of this License can be used, that proxy's public statement of acceptance of a version permanently authorizes you to choose that version for the Document.

**11. RELICENSING**

"Massive Multiauthor Collaboration Site" (or "MMC Site") means any World Wide Web server that publishes copyrightable works and also provides prominent facilities for anybody to edit those works. A public wiki that anybody can edit is an example of such a server. A "Massive Multiauthor Collaboration" (or "MMC") contained in the site means any set of copyrightable works thus published on the MMC site.

"CC-BY-SA" means the Creative Commons Attribution-Share Alike 3.0 license published by Creative Commons Corporation, a not-for-profit corporation with a principal place of business in San Francisco, California, as well as future copyleft versions of that license published by that same organization.

"Incorporate" means to publish or republish a Document, in whole or in part, as part of another Document.

An MMC is "eligible for relicensing" if it is licensed under this License, and if all works that were first published under this License somewhere other than this MMC, and subsequently incorporated in whole or in part into the MMC, (1) had no cover texts or invariant sections, and (2) were thus incorporated prior to November 1, 2008.

The operator of an MMC Site may republish an MMC contained in the site under CC-BY-SA on the same site at any time before August 1, 2009, provided the MMC is eligible for relicensing.

**ADDENDUM: How to use this License for your documents**

To use this License in a document you have written, include a copy of the License in the document and put the following copyright and license notices just after the title page:

​    Copyright (C)  YEAR  YOUR NAME.

​    Permission is granted to copy, distribute and/or modify this document

​    under the terms of the GNU Free Documentation License, Version 1.3

​    or any later version published by the Free Software Foundation;

​    with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.

​    A copy of the license is included in the section entitled "GNU

​    Free Documentation License".

If you have Invariant Sections, Front-Cover Texts and Back-Cover Texts, replace the "with … Texts." line with this:

​    with the Invariant Sections being LIST THEIR TITLES, with the

​    Front-Cover Texts being LIST, and with the Back-Cover Texts being LIST.

If you have Invariant Sections without Cover Texts, or some other combination of the three, merge those two alternatives to suit the situation.

If your document contains nontrivial examples of program code, we recommend releasing these examples in parallel under your choice of free software license, such as the GNU General Public License, to permit their use in free software.



# Link utili

- [Documentazioni **elinux**](https://elinux.org/RPiconfig) : sul file config.txt
- [Template completo di EvilPaul](https://raw.githubusercontent.com/Evilpaul/RPi-config/master/config.txt) del config.txt
- [Wiki ufficiale](https://www.raspberrypi.org/documentation/configuration/config-txt/video.md) sul config.txt video


# Template e legenda

Seguono i template utilizzati per il file Markdown

`codice inline o singola linea di codice`

```bash
# codice bash 
a blocchi o con sintassi da evidenziare
```

Nei blocchi di codice l'introduzione di un cancelletto (#) è da intendersi come commento.

Tra parentesi angolate (&lt;&gt;) trovate il nome dei valori che dovreste sostituire voi (ad esempio `<nomeutente>` se il vostro nome è *Mario*, è da sostituire con **Mario**, eliminando le parentesi)

Tutti i comandi dati con sudo è da sottintendere che necessitino dei permessi di amministratore, perciò se li operate da account root può essere evitato il sudo

> <u>NOTE</u>:
>
> note 

> **<u>ATTENZIONE</u>**:
>
> ==Un avvertimento. Il tratto appena spiegato non è stato testato oppure può fallire in alcuni casi ed è quindi sconsigliato da usare se non si sa esattamente quello che si fa. L’autore declina ogni responsabilità per danni che si possano verificare sul proprio sistema o dispositivo==

> *<u>SUGGERIMENTO</u>*:
> Un suggerimento orientato soprattutto ai meno esperti

