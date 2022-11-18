
Fonctionnement avec chunk de données flushé sur disque (recommandé si on ne veut pas tout maintenir en mémoire)

# Rotate chunk

Exemple de commande: 

`-XX:StartFlightRecording=delay=5s,disk=true,maxage=1m,maxsize=2M,filename=C:\dev\workspaces\java-kie -XX:FlightRecorderOptions=repository=C:\tmp\jfr_tmp`

Description de la commande:

On demande de démarrer un enregistrement JFR 5s après le démarrage de la VM. L'âge maximal des chunk conservé sera de 1 minute et on tolère jusqu'à 2MB de chunk. Les chunk seront flushé sur disque (disk=true) dans le répertoire C:\tmp\jfr_tmp

Le maxage et le maxsize sont vérifié au moment où un chunk est marqué comme final (finishChunk).

```java
private void finishChunk(RepositoryChunk chunk, Instant time, PlatformRecording ignoreMe) {
        chunk.finish(time);
        for (PlatformRecording r : getRecordings()) {
            if (r != ignoreMe && r.getState() == RecordingState.RUNNING) {
                r.appendChunk(chunk);
            }
        }
        FilePurger.purge();
    }
```

A ce moment le chunk est ajouté à la liste des chunks :

```java
    void appendChunk(RepositoryChunk chunk) {
        if (!chunk.isFinished()) {
            throw new Error("not finished chunk " + chunk.getStartTime());
        }
        synchronized (recorder) {
            if (!toDisk) {
                return;
            }
            if (maxAge != null) {
                trimToAge(chunk.getEndTime().minus(maxAge));
            }
            chunks.addLast(chunk);
            added(chunk);
            trimToSize();
        }
    }
```

Avec chunk.getEndTime().minus(maxAge) on détermine le endtime minimal d'un chunk pour être conservé (REFEND).
Dans le trimToAge, on prend la head de la queue de chunk (donc théoriquement le plus ancien), si ce chunk est plus vieux que REFEND on le supprime et on poursuit le nettoyage avec le suivante etc...

le trimToSize on supprime les chunks du plus ancien au plus récent jusqu'à ce que la taille total des chunks soit < maxsize

Quand on dump via la commande JFR.dump, on effectue un snapshot du record en cours. Ce snapshot prend la forme d'un clone

```java

private PlatformRecording newSnapShot(PlatformRecorder recorder, Recording recording, Boolean pathToGcRoots) throws DCmdException, IOException {
        if (recording == null) {
            // Operate on all recordings
            PlatformRecording snapshot = recorder.newTemporaryRecording();
            recorder.fillWithRecordedData(snapshot, pathToGcRoots);
            return snapshot;
        }

        PlatformRecording pr = PrivateAccess.getInstance().getPlatformRecording(recording);
        return pr.newSnapshotClone("Dumped by user", pathToGcRoots);
    }
```

On fait repointé le clone sur les chunks du record en cours et on stop le record du clone. Le stop du record entraine le finish du chunk en cours et donc l'eventuel trim des chunk (maxage et ou maxsize).

Un dump correspond à la concaténation des chunks post snapshot (donc ceux qui correpondent à la fenêtre temporelle maxage si on en a mis une).

# Chunk size 

La chunk size est par défaut à 12MB. La rotation des chunks en fonction de la chunk size est piloté par une periodic task java (toutes les secondes) et check shouldRotate via un appel natif. Cela va derrière déclencher un finish du chunk et les mécanismes de maxage/maxsize

```cpp

void JfrChunkRotation::evaluate(const JfrChunkWriter& writer) {
  assert(threshold > 0, "invariant");
  if (rotate) {
    // already in progress
    return;
  }
  if (writer.size_written() > threshold) {
    rotate = true;
    notify();
  }
}

bool JfrChunkRotation::should_rotate() {
  return rotate;
}

```

On peut donc à un moment avoir une occupation disque > maxsize, qui correpond à la taille maximal d'un chunk.

# Usages de dump periodique

Si on veut faire des dumps périodiques, il faut s'outiller soit en externe via jcmd, soit tenter de les déclencher en interne à nos JVM (en cours d'analyse) et/ou peut être JMX.

L'inconvénient des outils de JVM c'est qu'avec notre tmpfs, et donc un mount namespace, ces commandes sont difficilement utilisables sans les droits root car il faut basculer dans le namespace mount du process java, pour que la socket d'appel se crée dans le bon tmp.

A noter, la commande `jfr assemble chunkDir dest` qui permet de produire un jfr à partir de l'ensemble des fichiers de chunk dans chunkDir. Du moment qu'on définit un repertoire repository qui ne soit pas dans un namespace isolé, on pourrait peut être obtenir nos jfr plus facilement (à creuser)
