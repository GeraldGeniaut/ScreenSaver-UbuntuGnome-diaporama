# diaporama-veille

Écran de veille en **diaporama** pour GNOME sous Wayland, avec **verrouillage de la session à la sortie**.

GNOME ne propose plus d'économiseur d'écran : seulement « écran noir puis verrouillage », et l'écran de verrouillage de GNOME 50 affiche le fond d'écran **flouté**. Ce petit outil comble le manque :

- après **3 minutes** d'inactivité, un diaporama plein écran des images d'un dossier s'affiche, **net**, sur **tous les écrans** (une image différente par écran) ;
- au moindre geste (souris, clavier, clic, molette, touche Super, Alt+Tab…), la session est **verrouillée** et le mot de passe est demandé ;
- au bout d'**une heure**, GNOME éteint l'écran et verrouille, comme d'habitude ;
- pendant une visio ou une vidéo (une application qui demande à GNOME de rester éveillé), le diaporama ne se lance pas.

Testé sur **Ubuntu 26.04**, **GNOME Shell 50.1**, **GTK 4.22**, session Wayland, deux écrans.

---

## Sommaire

1. [Fonctionnement](#fonctionnement)
2. [Prérequis](#prérequis)
3. [Installation](#installation)
4. [Réglages GNOME](#réglages-gnome)
5. [Tests](#tests)
6. [Exploitation](#exploitation)
7. [Limites connues](#limites-connues)
8. [Désinstallation](#désinstallation)

---

## Fonctionnement

Le script `diaporama-veille` a deux modes.

| Mode | Rôle |
|---|---|
| `surveiller` | Tourne en tâche de fond (service systemd utilisateur). Toutes les 5 s, demande à GNOME depuis combien de temps l'utilisateur n'a rien touché. Au-delà de `DELAI_VEILLE`, lance le mode `afficher`, sauf si la session est déjà verrouillée ou si la veille est inhibée. |
| `afficher` | Ouvre une fenêtre plein écran par écran (fond noir, sans curseur, images en ordre aléatoire). À la première action, **verrouille d'abord** la session, puis ferme les fenêtres : le bureau n'apparaît jamais déverrouillé. |

Tout passe par **D-Bus**, le canal de communication de la session GNOME :

| Interface D-Bus | Méthode | Usage |
|---|---|---|
| `org.gnome.Mutter.IdleMonitor` | `GetIdletime` | Temps écoulé depuis la dernière action (ms) |
| `org.gnome.ScreenSaver` | `GetActive` | La session est-elle déjà verrouillée ? |
| `org.gnome.ScreenSaver` | `Lock` | Verrouiller la session (repli : `loginctl lock-session`) |
| `org.gnome.SessionManager` | `IsInhibited(8)` | Une application demande-t-elle de ne pas passer en veille ? |

Points de conception :

- **Touches interceptées par GNOME** : Super, Alt+Tab, etc. n'atteignent jamais la fenêtre. Le mode `afficher` vérifie donc aussi l'inactivité 4 fois par seconde : toute action, même invisible pour la fenêtre, déclenche le verrouillage.
- **Démarrage** : la première seconde et demie est ignorée, pour qu'un léger mouvement de souris au lancement ne verrouille pas immédiatement.
- **Souris multi-écran** : la position de départ est mémorisée par fenêtre. Le premier mouvement sur un écran sert de référence, un déplacement de plus de 15 px verrouille.
- **Écran noir à 1 h** : quand GNOME verrouille de lui-même, le diaporama se ferme sans redemander de verrouillage (pas de double saisie du mot de passe).
- **Dossier réseau indisponible** (montage absent, hors VPN) : `afficher` s'arrête avec un message et `surveiller` attend 5 minutes avant de réessayer, au lieu de boucler.

## Prérequis

- GNOME sous **Wayland** (les interfaces D-Bus utilisées sont celles de GNOME Shell/Mutter).
- Python 3 avec les liaisons GTK 4 (`python3-gi`, `gir1.2-gtk-4.0`). GTK **4.12 minimum** (`CssProvider.load_from_string`).

Vérification :

```bash
python3 -c "import gi; gi.require_version('Gtk','4.0'); from gi.repository import Gtk; print('GTK', Gtk.get_major_version(), Gtk.get_minor_version())"
```

Si la commande échoue :

```bash
sudo apt install python3-gi gir1.2-gtk-4.0
```

## Installation

Tout s'installe dans le compte de l'utilisateur, **sans droits root**.

```bash
git clone https://github.com/GeraldGeniaut/ScreenSaver-UbuntuGnome-diaporama.git
cd diaporama-veille

install -Dm755 diaporama-veille              ~/.local/bin/diaporama-veille
install -Dm644 diaporama-veille.conf.example ~/.config/diaporama-veille.conf
install -Dm644 diaporama-veille.service      ~/.config/systemd/user/diaporama-veille.service
```

`install -D` crée les dossiers manquants ; `-m755` rend le script exécutable, `-m644` donne des fichiers en lecture seule pour les autres utilisateurs.

### Configuration

Éditer `~/.config/diaporama-veille.conf` :

```ini
# Dossier des images (parcouru récursivement : jpg, jpeg, png, webp)
DOSSIER=~/Images/wallpaper/
# Inactivité avant le diaporama, en secondes
DELAI_VEILLE=180
# Durée d'affichage de chaque image, en secondes
DUREE_IMAGE=15
```

Le dossier peut être un dossier synchronisé (Nextcloud…) ou un montage réseau. Il peut être le même que celui d'un changement automatique de fond d'écran : les deux fonctionnent indépendamment.

### Activation du service

```bash
systemctl --user daemon-reload
systemctl --user enable --now diaporama-veille.service
systemctl --user status diaporama-veille.service --no-pager
```

- C'est un **service utilisateur** : il tourne dans la session, sans droits root.
- `graphical-session.target` le démarre quand le bureau GNOME est prêt et l'arrête à la déconnexion.
- `Restart=on-failure` le relance s'il plante.
- `status` doit afficher `active (running)`.

## Réglages GNOME

Écran noir et verrouillage au bout d'une heure :

```bash
gsettings set org.gnome.desktop.session idle-delay 3600
gsettings set org.gnome.desktop.screensaver lock-enabled true
gsettings set org.gnome.desktop.screensaver lock-delay 0
```

| Clé | Valeur | Effet |
|---|---|---|
| `idle-delay` | 3600 | Écran noir après 1 h d'inactivité |
| `lock-enabled` | true | Verrouillage quand l'écran s'éteint |
| `lock-delay` | 0 | Verrouillage immédiat, sans délai de grâce |

Optionnel, sur un **portable** : par défaut, GNOME met la machine en veille au bout de 15 minutes sur batterie, ce qui coupe le VPN et les applications réseau. Pour l'empêcher :

```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

Vérifier l'ensemble :

```bash
gsettings get org.gnome.desktop.session idle-delay
gsettings get org.gnome.desktop.screensaver lock-enabled
gsettings get org.gnome.desktop.screensaver lock-delay
gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type
gsettings get org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type
```

## Tests

**Test manuel** (lance le diaporama immédiatement) :

```bash
~/.local/bin/diaporama-veille afficher
```

Tous les écrans passent au diaporama. Attendre 2 secondes, puis bouger la souris : la session se verrouille et le mot de passe est demandé. C'est le comportement voulu.

En cas de message `Aucune image` ou `Dossier inaccessible`, vérifier le dossier :

```bash
ls ~/Images/wallpaper/ | head
```

**Test réel** : ne plus rien toucher pendant `DELAI_VEILLE` secondes. Le diaporama doit se lancer ; un mouvement de souris doit demander le mot de passe.

## Exploitation

| Action | Commande |
|---|---|
| État du service | `systemctl --user status diaporama-veille` |
| Derniers messages | `journalctl --user -u diaporama-veille -n 20` |
| Prendre en compte une modification de la configuration ou du script | `systemctl --user restart diaporama-veille` |
| Suspendre temporairement | `systemctl --user stop diaporama-veille` |
| Désactiver au démarrage | `systemctl --user disable diaporama-veille` |

## Limites connues

- **Pendant le diaporama, la session n'est pas encore verrouillée** : c'est la première action qui la verrouille, en un quart de seconde au plus. Pour un verrouillage dès le lancement, il faudrait verrouiller à l'ouverture du diaporama, mais l'écran de verrouillage de GNOME (flouté) passerait alors devant les images.
- **L'écran de verrouillage reste celui de GNOME**, avec son flou. Ce flou peut être retiré avec l'extension *Blur my Shell* (section *Lockscreen*).
- **Écrans branchés ou débranchés pendant le diaporama** : non pris en compte. La liste des écrans est lue au lancement.
- **Wayland/GNOME uniquement** : sous X11 ou un autre bureau, les interfaces D-Bus de Mutter et de GNOME Shell ne sont pas disponibles.

## Désinstallation

```bash
systemctl --user disable --now diaporama-veille.service
rm ~/.config/systemd/user/diaporama-veille.service
rm ~/.local/bin/diaporama-veille
rm ~/.config/diaporama-veille.conf
systemctl --user daemon-reload
```

Pour revenir au délai GNOME par défaut (5 minutes) :

```bash
gsettings reset org.gnome.desktop.session idle-delay
```

## Arborescence du dépôt

```
ScreenSaver-UbuntuGnome-diaporama/
├── README.md
├── diaporama-veille                 # script Python (modes surveiller / afficher)
├── diaporama-veille.conf.example    # configuration à copier dans ~/.config/
└── diaporama-veille.service         # service systemd utilisateur
```

## Licence

MIT
