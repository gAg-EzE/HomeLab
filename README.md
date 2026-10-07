# HomeLab HPC — Cluster Slurm sous Proxmox VE (virtualisation imbriquée)

> Un micro-cluster de calcul à la maison : **Windows + Hyper-V → Proxmox VE → 5 VM Debian → Slurm + MPI**, avec un mini-projet de produit matriciel (séquentiel / MPI) exécuté à travers le planificateur.

![Hyperviseur](https://img.shields.io/badge/hyperviseur-Proxmox%20VE%209.1.1-orange)
![Ordonnanceur](https://img.shields.io/badge/ordonnanceur-Slurm%2024.11.5-blue)
![Langage](https://img.shields.io/badge/langage-C%20%7C%20MPI-lightgrey)

## Objectif

Projet personnel mené en parallèle d'un Master 1 CHPS (UPVD). L'ambition n'est pas la performance — le matériel est un PC portable — mais la **maîtrise de la pile complète** : hyperviseur, virtualisation imbriquée, réseau de cluster, authentification, ordonnanceur de jobs et parallélisation de code. Le dépôt documente aussi les pannes réellement rencontrées (réseau, MUNGE, SSH), pas seulement l'état final.

## Architecture

| Couche | Composant | Détail |
|---|---|---|
| 1 — Hôte physique | Windows 11 + **Hyper-V** | hyperviseur de type 2 |
| 2 — VM d'infrastructure | VM `ProxMox VE` | 8 vCPU, 8192 Mo de RAM, virtualisation imbriquée |
| 3 — Hyperviseur | **Proxmox VE 9.1.1** (hôte `pve`) | gère les VM 100 → 105 |
| 4 — Nœuds | 5 VM **Debian 13 (trixie)**, noyau 6.12.86-1 | `master` (VM 101) + `node1`→`node4` (VM 102 → 105) |
| 5 — Ordonnanceur | **Slurm 24.11.5**, authentification MUNGE | `slurmctld` sur `master`, `slurmd` sur les 4 nœuds |
| 6 — Parallélisation | **Open MPI** | répertoire partagé `/shared` (NFS) |

![Topologie réseau](docs/img/topologie-reseau.png)

| Élément | Adresse |
|---|---|
| Hôte Proxmox, `vmbr0` (réseau de cluster) | `192.168.1.200/24` — interface web sur le port 8006 |
| Hôte Proxmox, `eth0` (accès Internet) | DHCP `172.29.x.x/20` via le `Default Switch` Hyper-V |
| `master` | `192.168.1.101` |
| `node1` → `node4` | `192.168.1.102` → `192.168.1.105` |

Chaque nœud de calcul est volontairement minimal (1 CPU, 900 Mo) : le cluster sert à valider des concepts. Détails dans [`docs/architecture.md`](docs/architecture.md).

## Mise en place

Le déroulé complet, commandes comprises, est dans [`docs/installation-slurm.md`](docs/installation-slurm.md). En résumé, dans l'ordre :

1. **Hyper-V** : créer la VM `ProxMox VE` (8 vCPU, 8192 Mo, démarrage sécurisé désactivé, **identification VLAN désactivée** sur la carte du `Default Switch` — incident n°1) et y installer Proxmox VE.
2. **Proxmox** : créer les 5 VM Debian 13 `master` (101) et `node1`→`node4` (102 → 105), IP statiques `192.168.1.101` → `105`.
3. **Prérequis** : SSH par clé de `master` vers toutes les machines, [`/etc/hosts`](configs/hosts.example), `chrony`, utilisateur `slurm` avec le même UID/GID partout.
4. **MUNGE** : clé de `master` copiée sur les nœuds, **puis `munge` redémarré** ([`scripts/setup-munge.sh`](scripts/setup-munge.sh)).
5. **NFS** : [`scripts/setup-nfs-server.sh`](scripts/setup-nfs-server.sh) sur `master`, [`scripts/setup-nfs-client.sh`](scripts/setup-nfs-client.sh) sur les nœuds ([détails et dépannage](docs/setup-nfs.md)).
6. **Slurm** : `apt install slurm-wlm` partout, [`configs/slurm.conf`](configs/slurm.conf) vers `/etc/slurm/`.
7. **Démarrage** : `slurmctld` sur `master`, `slurmd` sur les nœuds, puis validation :

```bash
root@master:~# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      4  idle node[1-4]

root@master:~# srun -N4 hostname
node4
node3
node1
node2
```

Pour valider MPI sur 4 nœuds : [`validation/hello_mpi.c`](validation/hello_mpi.c) (affiche rang et nœud) ou [`validation/hello_mpi_nfs.c`](validation/hello_mpi_nfs.c) (écrit en plus un fichier par processus dans `/shared`, preuve que NFS, Slurm et MPI fonctionnent ensemble — [procédure](docs/setup-nfs.md#test-de-bout-en-bout-nfs--slurm--mpi)).

## Mini-projet : produit matriciel séquentiel vs MPI

[`mini_project/`](mini_project/) contient un générateur de matrices `N × N`, une version séquentielle et une version MPI (`Bcast` de B, `Scatter`/`Gather` des lignes de A et C), plus les scripts Slurm. Les deux versions utilisent le même ordre de boucles (`i-k-j`) ; chacune affiche un *checksum* du résultat pour vérifier qu'elles calculent la même chose.

```bash
cd mini_project
make test              # test local rapide (N=64, 4 processus MPI, sans Slurm)

# Sur le cluster : copier mini_project/ dans /shared (les binaires doivent être
# visibles de tous les nœuds), puis, depuis ce dossier :
make submit            # N=2048 par défaut ; make submit N=1024 pour changer
make clean             # supprime bin/, logs/, matrices.dat et results.csv
```

`make clean` ne touche jamais à [`mini_project/reference/`](mini_project/reference/), qui contient les mesures réelles obtenues sur le cluster.

### Mesures (`reference/results.csv`)

| N | Mode | Processus | Temps (s) |
|---|---|---|---|
| 512 | Séquentiel | 1 | 0,032 |
| 512 | MPI | 4 | 0,011 |
| 2048 | Séquentiel | 1 | 3,777 · 3,847 · 5,041 · 5,179 |
| 2048 | MPI | 4 | 3,616 · 3,758 |

**Lecture des chiffres.** Pour N = 512, le gain est net (≈ 3×). Pour N = 2048, les temps séquentiels varient de 3,8 s à 5,2 s d'une exécution à l'autre : cet écart est du même ordre que le gain apparent de MPI (3,6–3,8 s). Sur ces mesures, MPI n'apporte donc **pas de gain démontré** à N = 2048. Hypothèses à vérifier, non démontrées : charge variable de la machine hôte (les 5 VM partagent un seul PC portable), bande passante mémoire (chaque nœud reçoit la matrice B complète, soit 32 Mo), cache. Le temps MPI mesure uniquement le calcul, hors `Bcast`/`Scatter`/`Gather` ; le temps total serait plus élevé.

## Incidents rencontrés

| # | Symptôme | Cause | Résolution |
|---|---|---|---|
| 1 | Pas de bail DHCP dans la VM Proxmox | identification VLAN activée (ID 2) sur la carte reliée au `Default Switch` | désactiver l'identification LAN virtuelle |
| 2 | `ip route add default` : `File exists` | route par défaut résiduelle vers une passerelle inexistante | `ip route del default`, puis ajout de la bonne route |
| 3 | Nœuds en `unk*`, `Invalid credential`, clé pourtant identique (md5) | le démon `munge` des nœuds tournait encore avec la clé générée à l'installation du paquet, la copie de celle du master n'ayant pas été suivie d'un `restart` | `systemctl restart munge`, puis `slurmd`, puis `slurmctld` |
| 4 | Boucle `Failed to query local AF_VSOCK CID` au redémarrage de `sshd` | socket `AF_VSOCK` indisponible dans la VM | interrompre, corriger la config SSH du nœud |
| 5 | `./bin/matrix_serial: Aucun fichier…` | nom de binaire erroné dans le script de soumission | aligner sur `matrix_mult_serial` |
| 6 | `mpirun` refuse de démarrer | exécution en root | `--allow-run-as-root` dans `submit_mpi.sbatch` |
| 7 | `nfs-kernel-server` : `Transaction order is cyclic` | boucle de dépendances systemd entre `shared.mount` (généré) et `nfs-server.service` | `systemctl mask shared.mount`, puis démarrer `rpcbind` et `nfs-server` |

Messages exacts et diagnostics : [`docs/reseau-et-depannage.md`](docs/reseau-et-depannage.md).

## Sécurité

- Aucun mot de passe, jeton ou clé dans le dépôt ; `.gitignore` exclut `.env`, `*.key` et les clés SSH. [`.env.example`](.env.example) ne contient que des valeurs d'exemple.
- Les adresses IP sont privées (RFC 1918).
- La configuration SSH des nœuds autorise `PasswordAuthentication yes` : pour un usage exposé, passer à l'authentification par clé.
- Le partage `/shared` doit rester sur le réseau de cluster isolé (`192.168.1.0/24`). Il est exporté avec `no_root_squash` parce que les jobs s'exécutent en root : acceptable en laboratoire, à remplacer par un utilisateur dédié sinon.

## Limites et prochaines étapes

- [ ] Mesurer la scalabilité : N = 512 / 1024 / 2048 / 4096, 1, 2, 4 processus, plusieurs répétitions, puis tracer *speedup* et efficacité.
- [ ] Mesurer aussi le temps total MPI (communications comprises).
- [ ] Automatiser le déploiement des nœuds (modèle de VM Proxmox ou Ansible).
- [ ] Durcir SSH (`PasswordAuthentication no` + clés) et utiliser un utilisateur non privilégié pour MPI.
- [ ] Comparer à une implémentation BLAS optimisée.

## Structure

```text
HomeLab/
├── configs/        # slurm.conf, hosts.example
├── scripts/        # setup MUNGE et NFS
├── validation/     # tests MPI multi-nœuds (avec et sans NFS)
├── docs/           # installation, architecture, Slurm, NFS, réseau et dépannage
└── mini_project/   # produit matriciel C / MPI (+ reference/ : mesures réelles)
```

## Auteur

**Komlan GAGNON** — Master 1 CHPS, UPVD. Démarche « apprendre en construisant » : chaque brique est montée, cassée, diagnostiquée puis documentée.

## Licence

[MIT](LICENSE)
