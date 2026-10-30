# Alexandria : architecture modulaire et module Manga/Webtoon

- **Date :** 2026-10-14
- **Statut :** en attente de relecture
- **Portée :** refactoring du bot en modules + module Manga/Webtoon (source MangaDex) + destination « dossier ». Mangadraft et les *arr sont hors périmètre (voir § 10).

---

## 1. Contexte et objectifs

Alexandria est un bot Telegram mono-utilisateur, auto-hébergé sur un serveur, un NAS ou un VPS. Il sert de **passerelle / télécommande** : on cherche un contenu, on le trouve, puis on le lit ou on le télécharge. Aujourd'hui il ne gère que les ebooks (Anna's Archive + Prowlarr), et son parcours est pensé pour les liseuses (EPUB, Kindle).

Le repo est public et documente comment reproduire le setup. Le public visé sait installer un service sur un serveur. On privilégie le gratuit et l'open source.

**Objectifs**
1. Restructurer le bot en **modules par couche**, sur le principe d'un « botshop » : chaque type de média est un module qu'on active ou non.
2. Ajouter les **mangas** et les **webtoons**, en français en priorité, depuis **MangaDex**.
3. Ne plus privilégier les liseuses. Un manga se lit **en ligne**, en **PDF** directement dans Telegram (mobile ou desktop) ou en **CBZ** dans une appli de lecture.
4. Ajouter une destination **dossier serveur**, commune aux ebooks et aux mangas.

**Contrainte :** aucun comportement ebook existant n'est retiré. Seule exception : on passe à des commandes explicites (§ 3).

## 2. Architecture

Il y a trois couches. Chacune ne dépend que de celle du dessous.

```
bot.py                     bootstrap : .env, Application, registre des modules
core/                      socle partagé, sans aucune notion de type de média
  access.py                whitelist ALLOWED_USER_IDS (ex-_is_allowed)
  prefs.py                 déplacé tel quel
  mailer.py                déplacé tel quel
  delivery.py              envoi d'un fichier : telegram | email | kindle | folder
  progress.py              animation « Préparation… » + barre ▰▱ (extraits de bot.py)
  menu.py                  /start, /help, /settings, onboarding, routage du texte libre
modules/
  __init__.py              protocole Module + registre (ENABLED_MODULES)
  ebook/                   module Ebook = code existant déplacé
    handlers.py            /ebook + callbacks dl_*, dlfmt_*, dest_* (ex-bot.py)
    anna_archive.py, prowlarr.py, downloader.py, watcher.py,
    converter.py, virustotal.py
  manga/                   module Manga, qui sert /manga et /webtoon
    mangadex.py            client API MangaDex
    comic_pack.py          images → CBZ | PDF, découpage par taille
    ranges.py              parsing des plages « 1-5 », « 6-10 »
    handlers.py            commandes, choix série/chapitre/plage, « suivant »
tests/                     pytest (n'existe pas aujourd'hui)
```

### 2.1 Contrat d'un module

```python
class Module(Protocol):
    name: str                       # "ebook", "manga"
    commands: list[tuple[str, str]] # [("manga", "Chercher un manga"), ("webtoon", "…")]
    destinations: set[str]          # destinations proposées, ex. {"telegram","email","folder"}
    def register(self, app: Application) -> None: ...
```

- `modules/__init__.py` contient un dictionnaire `{nom: module}`. `bot.py` n'enregistre que les modules listés dans `ENABLED_MODULES`. La valeur par défaut est `ebook,manga`.
- Chaque module préfixe ses `callback_data` par son nom (`ebook:dl:3`, `manga:ch:12`). Deux modules ne peuvent donc jamais entrer en collision. Ce préfixe reste court, à cause de la limite de 64 octets de Telegram : les données lourdes restent dans `context.user_data[<module>]`, comme aujourd'hui avec `results`.
- `/help` et la liste des commandes Telegram (`set_my_commands`) sont générés à partir des modules actifs.

### 2.2 Couche d'envoi (`core/delivery.py`)

La partie « envoi » de l'actuel `_do_download` est extraite. Le téléchargement, la conversion et le retry restent dans le module ebook.

```python
async def deliver(file_path: str, filename: str, destination: str, *,
                  bot, chat_id: int, user_id: int, caption: str = "",
                  subfolder: str = "") -> DeliveryResult
def max_size(destination: str) -> int | None   # None = pas de limite
```

| Destination | Implémentation | Limite |
|---|---|---|
| `telegram` | `send_document` (code actuel) | `MAX_FILE_SIZE` : 50 Mo, 2 Go avec un Bot API local |
| `email` | `mailer.send_file` vers l'adresse de `/settings` | `EMAIL_MAX_SIZE`, 25 Mo par défaut |
| `kindle` | `mailer` avec le sujet `convert` (code actuel). Ebook uniquement | `EMAIL_MAX_SIZE` |
| `folder` | copie atomique (fichier temporaire + `os.replace`) dans `LIBRARY_PATH/<subfolder>/` | aucune |

- `folder` n'apparaît que si `LIBRARY_PATH` est défini. Le chemin vient du `.env`, l'utilisateur ne le saisit jamais, ce qui exclut toute traversée de chemin depuis Telegram. Les noms de fichiers et de dossiers sont nettoyés (caractères interdits retirés, longueur bornée).
- Arborescence : `Ebooks/<Titre>.<ext>`, `Manga/<Série>/<Série> - Ch. 012.cbz`, `Webtoon/<Série>/…`. Elle reste compatible avec Komga ou Kavita le jour où on en ajoute un.
- Le menu de destination existant (Telegram / Email / Kindle) gagne un bouton **📁 Serveur**. La logique actuelle (le menu s'affiche dès qu'une destination autre que Telegram est disponible) reste la même.

## 3. Commandes et parcours

| Commande | Effet |
|---|---|
| `/ebook <titre>` | La recherche ebook actuelle, inchangée |
| `/manga <titre>` | Recherche MangaDex, tous formats |
| `/webtoon <titre>` | Recherche MangaDex filtrée sur le tag « Long Strip » |
| `/settings`, `/start`, `/help` | Comme aujourd'hui, avec en plus la liste des modules |

**Texte libre (changement de comportement) :** un message sans commande ne lance plus de recherche ebook. Le bot répond « Que cherches-tu ? » et propose un bouton par module actif (`📚 Ebook`, `📖 Manga`, `📜 Webtoon`) qui relance la recherche avec ce texte. Personne ne reste bloqué, et la commande s'apprend en passant. Les saisies en attente (email, Kindle) restent prioritaires, exactement comme aujourd'hui.

Le délai minimal entre deux recherches (`RATE_LIMIT_SECONDS`) s'applique à tous les modules.

### 3.1 Parcours Manga / Webtoon

1. **Recherche** : `GET /manga?title=…&availableTranslatedLanguage[]=fr` (plus le tag Long Strip pour `/webtoon`). On affiche jusqu'à 8 séries : titre, année, nombre de chapitres FR, et un 🇫🇷 ou un 🇬🇧.
2. **Langue :** on cherche d'abord en `fr`. Si la recherche FR ne donne rien, on relance sans filtre et on marque les séries 🇬🇧. Pour une série sans chapitre FR, le bot l'indique et propose les chapitres EN.
3. **Fiche série** : couverture, résumé tronqué, puis les boutons `▶ Premier chapitre`, `⏭ Dernier chapitre`, `🔢 Choisir un chapitre`, `📦 Plusieurs chapitres…`.
   - *Choisir un chapitre* : liste paginée de 10 chapitres par page, ou numéro tapé au clavier.
   - *Plusieurs chapitres* : le bot propose des tranches de 5 (`1-5`, `6-10`, …) et accepte une plage tapée (`12-20`).
4. **Chapitre choisi** : trois boutons, **📖 Lire en ligne** (lien `mangadex.org/chapter/<id>`, ou le lien éditeur `externalUrl` s'il existe), **PDF** et **CBZ**. Si le format préféré est enregistré dans `/settings`, on saute ce choix.
5. **Destination** : le menu de destination, limité aux destinations du module (`telegram`, `email`, `folder`).
6. **Envoi** : la légende contient la série, le chapitre, le groupe de scantrad et « via MangaDex », comme l'exige l'AUP MangaDex. Le message contient un bouton **Chapitre suivant ▶**, qui réutilise le même format et la même destination.

**Chapitres externes :** quand `pages == 0` et qu'un `externalUrl` existe (MangaPlus, par exemple), seul *Lire en ligne* est proposé.

**Doublons :** quand plusieurs groupes ont traduit le même chapitre, on garde le groupe du chapitre précédent s'il l'a traduit. Sinon on garde la version la plus récente. Il n'y a pas de choix manuel dans cette version.

### 3.2 Plages

- Une plage donne **un seul fichier** (`<Série> - Ch. 001-005.pdf`) tant qu'il tient sous la limite de la destination.
- Au-delà, `comic_pack` coupe le fichier **entre deux chapitres**, en parties `(1/2)`, `(2/2)`. Un chapitre seul plus gros que la limite est coupé entre deux pages.
- Taille maximale d'une plage : 50 chapitres, pour protéger MangaDex et la machine.
- La progression s'affiche comme pour les ebooks : « Ch. 3/5 — ▰▰▰▱▱ ». Le bouton Annuler reste actif.

## 4. Client MangaDex (`modules/manga/mangadex.py`)

Toutes les fonctions sont `async` et passent par `httpx.AsyncClient`, comme `anna_archive.py`.

```python
async def search(title: str, *, lang: str = "fr", long_strip: bool = False) -> list[Series]
async def chapters(series_id: str, *, lang: str = "fr") -> list[Chapter]   # triés, paginés côté API
async def pages(chapter_id: str) -> list[str]                              # URLs d'images
async def download_pages(chapter_id: str, dest_dir: str, on_progress) -> list[str]
```

- **Endpoints :** `GET /manga`, `GET /manga/{id}/feed?translatedLanguage[]=…&order[chapter]=asc&limit=500` (pagination par `offset`), `GET /at-home/server/{chapterId}`. Les images sont à `{baseUrl}/data/{hash}/{file}`.
- **Tag Long Strip :** son UUID est résolu au premier appel via `GET /manga/tag`, puis mis en cache en mémoire. Rien n'est codé en dur.
- **Règles de l'API :** un `User-Agent: Alexandria/<version> (+<url repo>)` honnête, au plus environ 5 requêtes par seconde sur l'API (sémaphore + pause), au plus 40 par minute sur `/at-home/server`. Sur une 429, on attend `X-RateLimit-Retry-After`, puis on réessaie une fois.
- **Images :** au plus 4 téléchargements simultanés par chapitre. Pour chaque image, on envoie un rapport MangaDex@Home (`POST https://api.mangadex.network/report`) comme le demandent les règles at-home. Si une image échoue, on redemande un nouveau `baseUrl` une fois, puis le chapitre est déclaré en échec.
- On utilise les images `data` (qualité d'origine). `dataSaver` est écarté dans cette version.

## 5. Empaquetage (`modules/manga/comic_pack.py`)

```python
def to_cbz(images: list[str], out_path: str, *, metadata: ComicInfo | None) -> str
def to_pdf(images: list[str], out_path: str) -> str
def split_by_size(chapters: list[list[str]], fmt: str, limit: int | None) -> list[list[str]]
```

- **CBZ :** un zip des images, sans recompression (`ZIP_STORED`). Les fichiers sont nommés `0001.jpg`… pour garantir l'ordre, avec un `ComicInfo.xml` minimal (série, numéro, langue, groupe), que Komga, Kavita et Mihon savent lire.
- **PDF :** avec PyMuPDF, déjà en dépendance. **Une page par image, à la taille exacte de l'image**, avec l'image insérée sans réencodage. Les longues bandes des webtoons défilent ainsi en continu, sans format A4.
- On ne convertit pas d'un format à l'autre : les deux sortent directement des mêmes images.
- Les fichiers temporaires vont dans un dossier par tâche, supprimé dans un `finally`. Le nettoyage des fichiers orphelins existant au démarrage est étendu à ce dossier.

## 6. Préférences et configuration

**Nouvelles clés dans `prefs`** (les clés existantes ne changent pas) :
- `manga_format` : `pdf` | `cbz` | absent (absent = on demande à chaque fois). Réglable dans `/settings`.
- La dernière destination et le dernier format choisis pour un chapitre restent dans `user_data`, pour le bouton « suivant ». Ils ne sont pas persistés.

**Nouvelles variables `.env`** (toutes optionnelles) :

| Variable | Défaut | Rôle |
|---|---|---|
| `ENABLED_MODULES` | `ebook,manga` | Modules actifs |
| `LIBRARY_PATH` | vide (destination désactivée) | Racine de la destination « dossier » |
| `EMAIL_MAX_SIZE` | `25` (Mo) | Limite des pièces jointes |
| `MANGA_LANG` | `fr` | Langue prioritaire |

**Docker :** `docker-compose.yml` gagne un montage `${LIBRARY_PATH}:${LIBRARY_PATH}` en lecture-écriture.

**Hors spec mais à faire :** recréer un `.env.example` (supprimé lors du fork, alors qu'il est encore référencé par `CLAUDE.md` et `LISEZMOI.md`) avec ces variables. `docker-compose.yml` pointe encore sur l'image `ghcr.io/zoeille/maman-books`.

## 7. Gestion des erreurs

| Situation | Comportement |
|---|---|
| MangaDex injoignable ou 5xx | « MangaDex ne répond pas, réessaie dans quelques minutes. » Détails dans les logs |
| 429 répétée | Même message. Pas de nouvel essai en boucle |
| Chapitre externe uniquement | Seul « Lire en ligne » est proposé |
| Image en échec après le nouvel essai | Le chapitre échoue. Dans une plage, on continue avec les autres chapitres et on indique « Ch. 7 ignoré » à la fin |
| Fichier trop gros pour la destination | Découpage automatique (§ 3.2). Il n'y a jamais d'échec silencieux |
| `LIBRARY_PATH` absent ou non inscriptible | Le bouton 📁 n'apparaît pas. Une erreur d'écriture est remontée à l'utilisateur et loguée |
| Annulation | La tâche est annulée et les fichiers temporaires sont supprimés, comme pour les ebooks |

Les messages à l'utilisateur restent génériques (pas de traces, pas d'URL internes), comme pour SMTP aujourd'hui.

## 8. Tests

Il n'y a pas de tests aujourd'hui. On introduit **pytest** (+ `pytest-asyncio`) dans un `requirements-dev.txt`.

- `ranges.py` : `1-5`, `5`, `12-20`, espaces, bornes inversées, au-delà de 50 chapitres, entrées invalides.
- `mangadex.py` : réponses JSON enregistrées dans `tests/fixtures/` et transport `httpx.MockTransport`. On couvre la recherche FR, le repli EN, la pagination du feed, les chapitres externes, le dédoublonnage par groupe, la 429 avec Retry-After, la construction des URLs at-home.
- `comic_pack.py` : images générées par PyMuPDF. On vérifie l'ordre des pages dans le CBZ, la présence de `ComicInfo.xml`, le nombre de pages et leurs dimensions dans le PDF (bande longue comprise), et le découpage par taille entre chapitres.
- `core/delivery.py` : la destination `folder` (atomicité, nettoyage des noms, arborescence) et le choix de la limite selon la destination.
- `modules/__init__.py` : `ENABLED_MODULES` filtre bien les modules, et les préfixes de callback ne se recouvrent pas.
- **Vérification manuelle** (checklist dans le plan) : le parcours ebook complet avant et après le refactoring (non-régression), puis le parcours manga et webtoon de bout en bout sur un vrai bot de test, avec la lecture du PDF dans Telegram sur mobile et sur desktop.

## 9. Découpage en étapes

1. **Socle** : `core/` + `modules/ebook/` en déplaçant le code existant, puis `/ebook` et le routage du texte libre. **Aucun autre changement fonctionnel.** Les tests de non-régression sont manuels.
2. **Destination dossier** dans `delivery.py`, branchée sur les ebooks.
3. **Module manga** : `ranges`, `mangadex`, `comic_pack`, puis `handlers` (`/manga`, `/webtoon`).
4. **Docs** : README, LISEZMOI, `.env.example`, CHANGELOG (v2.0.0, car le texte libre change de comportement).

Chaque étape laisse le bot fonctionnel et peut être livrée seule.

## 10. Hors périmètre

- **Mangadraft** : pas d'API publique, il faudra scraper. Ce sera une spec séparée, qui ajoutera une seconde source au module manga, surtout pour `/webtoon`.
- **rclone** (SFTP, WebDAV, Drive) : seulement si `folder` ne suffit pas.
- **Livres audio, *arr, Jellyfin, Komga/Kavita** : projet distinct, voir `~/cours/repo/docs__rails/ideas.md`.
- **Kindle pour les mangas** (conversion façon KCC), choix manuel du groupe de scantrad, `dataSaver`, notifications de nouveaux chapitres.
