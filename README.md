# Projet OSINT avec OWASP Maryam

Un exercice de reconnaissance passive sur une organisation publique, `owasp.org`, avec le framework **OWASP Maryam** et quelques outils classiques (`dig`, `whois`, `theHarvester`). L'idée : voir ce qu'on peut apprendre d'une organisation uniquement à partir de sources publiques (DNS, sous-domaines, e-mails, profils, réseaux sociaux), sans jamais toucher à ses systèmes.

Le rapport complet, avec toutes les captures commentées, est dans [`Projet_OSINT_Maryam.pdf`](Projet_OSINT_Maryam.pdf).

## Contexte

- Projet académique réalisé en **2024**.
- Je n'ai pas gardé le travail d'origine. J'ai donc **refait la démonstration en octobre 2026**, avec la même méthodologie et les mêmes outils. Toutes les captures et tous les résultats de ce dépôt viennent de cette reproduction.
- La cible, `owasp.org`, est une organisation publique. Aucune donnée personnelle n'est concernée.

## Cadre éthique

- Collecte **passive uniquement** : pas de scan actif, pas de tentative d'intrusion.
- Cibles publiques ou fictives : un domaine d'organisation, un nom d'utilisateur générique et un numéro fictif (`+1 202 555 0100`, la plage 555-01XX est réservée aux exemples).
- Aucune donnée personnelle de particuliers n'est publiée ici. La liste complète des profils trouvés n'est volontairement pas reproduite, et les adresses IP visibles sur les captures sont partiellement masquées.

## Environnement

| Élément | Valeur |
|---|---|
| Système | Kali Linux (VM VMware) |
| Outil principal | OWASP Maryam v2.5.3 |
| Python | 3.14 (venv) |
| Outils complémentaires | dig, whois, theHarvester 4.11.1 |

## Contenu du dépôt

```
osint-maryam/
├── README.md
├── Rapport_OSINT_Maryam.pdf   (rapport complet avec captures commentées)
```

Le clone de Maryam et le `venv` ne sont pas versionnés (voir `.gitignore`).

## 1. Installer Maryam

Maryam n'est plus très récent et Python 3.14 lui a posé quelques problèmes. Voici ce que j'ai fait pour le faire tourner.

| Étape | Pourquoi |
|---|---|
| `git clone https://github.com/saeeddhqan/Maryam.git` | Récupérer le code source |
| `python3 -m venv venv` puis `source venv/bin/activate` | Travailler dans un environnement isolé, sans toucher aux paquets du système |
| `pip install -r requirements-light.txt` | Le fichier `requirements` d'origine plante (paquet `umap` introuvable) et embarque des bibliothèques de machine learning lourdes qui ne servent qu'au module de topic modeling. J'ai donc gardé les 12 paquets utiles à l'OSINT |
| Ajouter `multiprocessing.set_start_method("fork", force=True)` en tête de `maryam/__main__.py` | Sous Python 3.14, les modules échouaient avec `When serializing multiprocessing.context.Process object`. Forcer l'ancien mode `fork` règle le problème |
| `python3 -m maryam` | Lancer le framework (`python3 maryam` échoue avec `ModuleNotFoundError`) |

Au démarrage, Maryam affiche un avertissement `Util class 'topic' has been disabled ... sentence_transformers`. C'est normal : seul le module de topic modeling est désactivé, ça n'a aucun effet sur l'OSINT.

## 2. Prise en main de Maryam

| Commande | Rôle |
|---|---|
| `help` | Liste les commandes du framework (`search`, `set`, `show`, `workspaces`, `report`...) |
| `show modules` | Liste les modules par catégorie : Footprint, Search, Osint, Iris |
| `<module> -h` | Affiche les options d'un module. Elles changent d'un module à l'autre : `-q` pour `crt`, `-d` pour `dns_search`, `-n` pour `phone_number_search` |
| `--output` | Enregistre le résultat dans le workspace, pour générer un rapport ensuite |

Dans cette version, il n'y a pas de commande `use` : on lance directement le module par son nom, avec ses options.

## 3. Collecte et interprétation

### 3.1 DNS et WHOIS

| Commande | Résultat | Ce que ça m'apprend |
|---|---|---|
| `dig owasp.org NS +short` | `fay.ns.cloudflare.com`, `west.ns.cloudflare.com` | DNS géré par Cloudflare, donc l'IP réelle des serveurs est probablement masquée |
| `dig owasp.org MX +short` | 5 enregistrements : `aspmx.l.google.com` et `alt1` à `alt4` | Messagerie sur Google Workspace |
| `dig owasp.org TXT +short` | SPF incluant Google, Mailchimp (`mcsv.net`) et Amazon SES ; vérifications Google, Atlassian et Microsoft | On devine les services tiers utilisés (messagerie, marketing, collaboration) |
| `whois owasp.org` | Registrar GoDaddy, créé le 2001-09-21, expire le 2031-09-21, `DNSSEC: unsigned`, contacts masqués | Domaine ancien, données personnelles masquées, mais DNSSEC non activé |

Petite remarque : `whois` a affiché « Temps limite dépassé » en fin de sortie. Les informations utiles étaient déjà là, donc ça n'a rien changé.

### 3.2 Sous-domaines

| Commande | Ce qu'elle fait | Résultat |
|---|---|---|
| `crt -q owasp.org --output` | Cherche dans les certificats publics (crt.sh) | Vide : crt.sh répondait `502 Bad Gateway`. J'ai vérifié avec `curl`, le problème ne vient ni de Maryam ni de ma machine |
| `dns_search -d owasp.org --output` | Interroge des sources ouvertes (OTX, SecurityTrails) | `OTX is missed` et `Sectrails is missed` : OTX refuse l'accès anonyme et SecurityTrails demande une clé API |
| `theHarvester -d owasp.org -b duckduckgo -l 50` | Hôtes, e-mails et IP via DuckDuckGo | 5 hôtes, aucun e-mail, aucune IP. Deux faux positifs (`2Fgenai...`, `2Fscvs...`) à cause de l'encodage d'URL (`%2F`). Il reste 3 vrais sous-domaines : `community`, `genai`, `scvs` |
| `theHarvester -d owasp.org -b rapiddns,hackertarget,urlscan,yahoo,brave -l 100 -q -f results/harvester_owasp` | Collecte multi-sources sans clé API, export JSON/XML | 127 entrées hôte:IP, 11 IP, 3 ASN, 12 URLs intéressantes, aucun e-mail. `brave` a échoué faute de clé API |

**Ce que montrent les résultats de theHarvester**

- **Trois ASN** : AS13335 (Cloudflare), AS15169 (Google) et AS16509 (Amazon). L'infrastructure est répartie chez plusieurs fournisseurs.
- **La plupart des sous-domaines sont derrière Cloudflare** (par exemple `104.20.44.163` et `172.66.157.115`, plus leurs équivalents IPv6). L'adresse réelle des serveurs reste masquée.
- **Quelques sous-domaines résolvent hors Cloudflare** (`cloud`, `wiki`, `lists`, `name-virt-host`, `giving`, `secureflag`). Leur serveur d'origine est donc visible, ce qui agrandit la surface d'attaque.
- **Les noms parlent d'eux-mêmes** (`dev`, `mail`, `wiki`, `lists`, `members`, `gapps`) : un attaquant en déduit les services en place.
- **Aucune adresse e-mail trouvée.** À relativiser : les sources gratuites sont limitées.

Les résultats complets sont dans `results/harvester_owasp.json` et `results/harvester_owasp.xml`.

### 3.3 E-mails, profils publics, réseaux sociaux

| Commande | Résultat |
|---|---|
| `email_search -q owasp.org -e bing,duckduckgo --output` | Section EMAILS : « Without result ». Cohérent avec theHarvester, mais les moteurs gratuits limitent les requêtes automatisées |
| `username_search -q owasp --output` | « Found 82 accounts » (voir les limites juste après) |
| `social_nets -q owasp -e bing,duckduckgo --output` | Lancé deux fois. Seule la ligne `[BING] Searching in 1 page...` s'affiche, puis l'invite revient sans résultat. Les moteurs ont sans doute bloqué ou limité les requêtes |
| `phone_number_search -n "+12025550100" --output` | `VALID: True`, préfixe `+1`, pays US, type de ligne `special_services`. Localisation et opérateur non renseignés |

**À propos des 82 comptes**

- Le module vérifie seulement qu'une page existe au nom « owasp ». Ça ne prouve pas qu'elle appartient à l'organisation : il faudrait vérifier à la main.
- Il y a des doublons (dev.to, Trakt, Championat, Gitee, Designspiration apparaissent plusieurs fois), donc 82 est un chiffre gonflé.
- Certaines lignes ne sont que des URL de recherche, pas des profils confirmés (par exemple l'entrée ProtonMail).
- Je ne publie pas la liste complète.

**À propos du numéro de téléphone** : il est fictif, donc le module ne vérifie que le format et le plan de numérotation. Il ne révèle rien sur un propriétaire.

## 4. Mapping MITRE ATT&CK

Ce qui a été fait relève de la tactique **TA0043 – Reconnaissance** :

| Technique | Lien avec le projet |
|---|---|
| T1590.002 – Gather Victim Network Information : DNS | `dig` NS / MX / TXT, `dns_search` |
| T1596.002 – Search Open Technical Databases : WHOIS | `whois` |
| T1589.002 – Gather Victim Identity Information : Email Addresses | `email_search`, theHarvester |
| T1593.001 – Search Open Websites/Domains : Social Media | `social_nets`, `username_search` |

**Ce qu'un attaquant pourrait en tirer** : la liste des services utilisés (Google Workspace, Mailchimp, Amazon SES, Atlassian, Microsoft), utile pour préparer du phishing ciblé, et le fait que DNSSEC ne soit pas activé.

**Pistes d'amélioration**

- Activer DNSSEC sur le domaine.
- Vérifier la présence et la configuration de SPF, DKIM et DMARC (le script ci-dessous teste SPF et DMARC).
- Placer les sous-domaines exposés derrière la protection Cloudflare, ou restreindre l'accès direct à leur serveur d'origine.
- Supprimer les enregistrements TXT de vérification devenus inutiles.
- Sensibiliser les utilisateurs au phishing.

## 5. Limites et difficultés rencontrées

- `crt.sh` indisponible pendant les tests (502 Bad Gateway).
- OTX AlienVault et SecurityTrails : accès anonyme refusé, clé API nécessaire.
- Source `brave` de theHarvester : clé API manquante.
- Maryam (code ancien) incompatible avec Python 3.14 : correctif appliqué (voir §1).
- Dépendances de machine learning introuvables ou trop lourdes : installation allégée.
- Interface réseau de la VM tombée (`eth0 DOWN`, VM en mode Bridged) : remise en route avec `sudo ip link set eth0 up`.
- Aucun e-mail ni réseau social exploitable : les sources gratuites sont limitées, il faut garder du recul sur ces résultats.

## 6. Bonus : automatiser avec `osint_recon.sh`

Pour ne pas retaper les mêmes commandes à chaque fois, j'ai écrit un petit script Bash. Il récupère les enregistrements DNS (NS, MX, TXT, A) et un extrait WHOIS, ajoute des indicateurs de sécurité (présence de SPF, DMARC et DNSSEC), puis écrit un rapport Markdown horodaté dans `results/`.

```bash
chmod +x osint_recon.sh
./osint_recon.sh owasp.org
```

Prérequis : `dig` et `whois` (paquets `dnsutils` et `whois`).

Si le serveur WHOIS est lent, le script coupe la commande au bout de 30 secondes (`timeout 30`) et le terminal affiche `Interruption par le signal 15`. Ce n'est pas une erreur grave : le rapport est quand même généré.

Idées pour la suite : brancher les clés API (OTX, SecurityTrails), exporter les résultats de Maryam dans le rapport, ajouter theHarvester au script.

## Compétences mobilisées

OSINT, reconnaissance passive, DNS et WHOIS, Linux (Kali), Python (venv, dépannage de dépendances), Bash, MITRE ATT&CK, rédaction de rapport.
