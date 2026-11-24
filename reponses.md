# TP1 : SOAP et WSDL, Partie A (code-first)

Binôme : [Nom1] et [Nom2]

---

## Étape 1 : Créer et lancer le service

**Q1.** `@WebService` fait de `EmpruntService` un service web.
`@WebFault` fait de `LivreIndisponibleException` une erreur déclarée dans le contrat : elle apparaît dans le WSDL comme `wsdl:fault`.

**Q2.** JAXB (la liaison Java ↔ XML) crée les objets par réflexion en lisant le XML : il appelle le constructeur vide, puis remplit les champs. Sans constructeur vide, il ne peut pas instancier la classe.

Résultat : `mvn compile exec:java` affiche `WSDL : http://localhost:8080/bib/EmpruntService?wsdl` et le WSDL s'affiche dans le navigateur.

---

## Étape 2 : Lire le WSDL généré

**Q1.** URL du service : `http://localhost:8080/bib/EmpruntService` (élément `soap:address` du `service`).

| Opération | Message d'entrée | Message de sortie | Fault |
|---|---|---|---|
| `emprunterLivre` | `emprunterLivre` | `emprunterLivreResponse` | `LivreIndisponibleException` |
| `rendreLivre` | `rendreLivre` | `rendreLivreResponse` | aucun |

Le `portType` s'appelle `EmpruntService` et le `binding` `EmpruntPortBinding`.

**Q2.** `style="document"`, `use="literal"`. C'est du SOAP 1.1 : le binding utilise l'espace de noms `http://schemas.xmlsoap.org/wsdl/soap/` (pour SOAP 1.2, ce serait `.../wsdl/soap12/`).

**Q3.** `isbn` est de type `xs:string` et `dateRetour` de type `xs:string`. La règle de format de l'ISBN n'apparaît pas : la regex n'existe que dans le code Java, pas dans le contrat (aucun `xs:pattern` dans le schéma). Le schéma n'a pas `elementFormDefault="qualified"`.

**Q4.** Non. Dans la requête SoapUI, seul `<bib:emprunterLivre>` porte un préfixe. `<idAdherent>` et `<isbn>` n'en ont pas, car les éléments enfants sont non qualifiés.

**Q5.** Dans l'onglet Raw de SoapUI :
- **HTTP** : la ligne `POST /bib/EmpruntService HTTP/1.1`, les en-têtes (`Content-Type: text/xml`, `SOAPAction: ""`) et la réponse `HTTP/1.1 200 OK`. Il transporte le message.
- **XML** : le format textuel du contenu, bien formé et avec des espaces de noms.
- **SOAP** : l'`Envelope` avec son `Header` et son `Body`. Il définit la structure du message et des Faults.
- **WSDL** : le contrat qui décrit où appeler, quelles opérations existent et quel est le format des messages. Il ne circule pas dans l'appel.

Résultat observé dans SoapUI (`A-117`, `978-1-234-5678-9`) : `200 OK`, `numero` = `E-2026-0812`, `dateRetour` = `2026-12-08`. Cet ISBN respecte le motif, donc il n'y a pas de Fault.

---

## Étape 3 : Envoyer des messages SOAP

Fichiers créés : `xml/01-emprunter.xml`, `xml/02-emprunter-12.xml`, `xml/04-erreurs.xml`, `xml/05-corrige.xml`.

Test du point 2 (SOAP 1.1 avec curl) : `HTTP/1.1 200 OK`, `emprunterLivreResponse` avec `numero` = `E-2026-0813`.

**Q1.** SOAP est un protocole de messages : l'opération et les données sont dans le corps XML, et il n'y a qu'une seule URL. Un GET n'a pas de corps et sert à identifier une ressource par son URL. On utilise donc POST, même pour lire une information.

**Q2.** Non, le bloc `jeton` du Header n'a pas empêché le traitement (réponse 200). Le service ne le comprend pas, mais comme il n'a pas `mustUnderstand="1"`, il l'ignore.

**Q3.** Le message SOAP 1.2 donne `HTTP 500` avec une Fault SOAP 1.1 :
- `faultcode` = `S:VersionMismatch`
- `faultstring` : le serveur attendait une `Envelope` dans l'espace de noms `http://schemas.xmlsoap.org/soap/envelope/` mais a reçu `http://www.w3.org/2003/05/soap-envelope`.

Explication : le binding lu à l'étape 2 est SOAP 1.1 uniquement. L'espace de noms de l'enveloppe identifie la version, donc l'enveloppe 1.2 est rejetée. Le service répond lui-même en SOAP 1.1.

**Q4.** Les cinq erreurs du message, testées une par une :

| # | Erreur | Réaction du serveur | Correction |
|---|---|---|---|
| 1 | `<soap:envelope>` (casse) | HTTP 500, `S:Client` : attendu `Envelope`, obtenu `envelope` | `<soap:Envelope>` |
| 2 | Espace de noms `http://supnum.mr/bibliotheque` | HTTP 500, `S:Client` : « Méthode de répartition introuvable pour {http://supnum.mr/bibliotheque}rendreLivre » | `http://supnum.mr/bib` |
| 3 | `</rendreLivre>` ne ferme pas `<bib:rendreLivre>` | HTTP 500, `S:Server` : « Unexpected close tag </rendreLivre>; expected </bib:rendreLivre> » | `</bib:rendreLivre>` |
| 4 | `<bib:numero>` au lieu de `<numeroEmprunt>` | **Aucune erreur** : 200 OK, `penalite` = 0 | `<numeroEmprunt>` (sans préfixe) |
| 5 | `Header` placé après le `Body` | **Aucune erreur** : toléré | `Header` avant `Body` |

Les erreurs 1, 2 et 3 déclenchent une Fault. Les erreurs 4 et 5 ne déclenchent aucune erreur côté serveur :
- **Erreur 4** : l'élément inconnu est ignoré, le paramètre `numeroEmprunt` vaut `null`, et comme aucune validation de schéma n'est active, rien ne le signale. Le service répond avec une pénalité de 0.
- **Erreur 5** : Metro lit le Body sans se plaindre de l'ordre, bien que la spécification SOAP impose le Header avant le Body.

---

## Étape 4 : Provoquer et construire des Faults

| Test | HTTP | `faultcode` | `faultstring` |
|---|---|---|---|
| 1. ISBN `978-2-10` (format invalide) | 500 | `S:Client` | `ISBN mal formé : 978-2-10` |
| 2. ISBN `978-2-10-074444-4` (déjà emprunté) | 500 | `S:Server` | `Livre indisponible` |
| 3a. `mustUnderstand="1"` sur `jeton` | 500 | `S:MustUnderstand` | `MustUnderstand headers:[{http://supnum.mr/bib}jeton] are not understood` |
| 3b. `mustUnderstand="0"` | 200 | (aucune Fault) | emprunt créé, `E-2026-0814` |
| 4. `rendreLivre` avec `XYZ` | 500 | `S:Client` | `Numéro d'emprunt inconnu : XYZ` |

Test 4 avec un numéro valide (`E-2026-0812`) : [à compléter : 200 OK, `penalite` = 0].

**Q1.** Cas 1 : HTTP 500, `faultcode` = `S:Client`, `faultstring` = « ISBN mal formé : 978-2-10 ».
Cas 2 : HTTP 500, `faultcode` = `S:Server`, `faultstring` = « Livre indisponible ».

**Q2.** Dans le cas 2, `<detail>` contient :
```xml
<ns2:LivreIndisponible xmlns:ns2="http://supnum.mr/bib">
  <isbn>978-2-10-074444-4</isbn>
  <retourPrevu>2026-12-02</retourPrevu>
</ns2:LivreIndisponible>
```
Sa structure est décrite dans le WSDL par :
- le `wsdl:fault` de l'opération `emprunterLivre` (dans le `portType`),
- le `message` `LivreIndisponibleException` (part `fault`, élément `tns:LivreIndisponible`),
- l'élément `LivreIndisponible` du schéma XSD (`?xsd=1`), de type `livreIndisponibleInfo` (`isbn`, `retourPrevu`).

**Q3.** JAX-WS a choisi `S:Server` pour le livre indisponible : toute exception déclarée par `@WebFault` est rangée côté serveur par défaut. Pourtant la requête est valide et c'est l'état métier (livre déjà emprunté) qui la rend impossible. Ce n'est ni une panne du serveur ni une erreur de format du client. J'aurais plutôt choisi `Client` (ou une Fault métier distincte), car le client peut agir : choisir un autre livre ou attendre la date de retour. Un code `Server` suggère à tort un incident technique à réessayer plus tard.

**Q4.** Avec `mustUnderstand="1"`, le serveur renvoie HTTP 500 avec une Fault `S:MustUnderstand`. Le bloc d'en-tête est déclaré obligatoire mais le service ne sait pas le traiter, donc il refuse tout le message et n'exécute pas l'opération (aucun emprunt créé). Avec `mustUnderstand="0"` (ou sans l'attribut), le bloc est facultatif : le serveur l'ignore et répond 200 OK.

**Q5.** Différences entre ma Fault SOAP 1.2 (`xml/03-fault-sender-12.xml`) et les Faults 1.1 reçues :
- `faultcode` (`S:Client` / `S:Server`) devient `Code/Value`, avec `Sender` / `Receiver`.
- `faultstring` devient `Reason/Text`, avec un attribut `xml:lang` obligatoire.
- `detail` devient `Detail`. En 1.2, ces éléments sont qualifiés dans l'espace de noms `http://www.w3.org/2003/05/soap-envelope` (préfixe `soap:`), alors qu'en 1.1 `faultcode`, `faultstring` et `detail` ne sont pas qualifiés.
- L'enveloppe change d'espace de noms (1.1 : `http://schemas.xmlsoap.org/soap/envelope/`, 1.2 : `http://www.w3.org/2003/05/soap-envelope`).

Vérification du fichier `03-fault-sender-12.xml` : [à compléter : `xmllint --noout` ou ouverture dans Chrome, sans erreur].

Modification du code : dans `rendreLivre`, ajout du contrôle `if (numeroEmprunt == null || !numeroEmprunt.startsWith("E-2026-")) throw faultClient(...)`. Il remplace le silence de l'erreur 4 de l'étape 3 (paramètre manquant, puis `null`, puis pénalité 0) par une Fault `Client` explicite.