# TP1 : SOAP et WSDL (Parties A et B)

Binôme : [24201] et [24093]

Remarque : certains fichiers de `xml/` contiennent des erreurs voulues pour les tests (`04-erreurs.xml`, `06-rendre-invalide.xml`). `05-corrige.xml` est la version corrigée du message à cinq erreurs.

---

# Partie A : code-first

## Étape 1 : Créer et lancer le service

**Q1.** `@WebService` fait de `EmpruntService` un service web. `@WebFault` fait de `LivreIndisponibleException` une erreur déclarée dans le contrat : elle apparaît dans le WSDL comme `wsdl:fault`.

**Q2.** JAXB crée les objets par réflexion en lisant le XML : il appelle le constructeur vide, puis remplit les champs. Sans constructeur vide, il ne peut pas instancier la classe.

Résultat : `mvn compile exec:java` affiche `WSDL : http://localhost:8080/bib/EmpruntService?wsdl` et le WSDL s'affiche dans le navigateur.

## Étape 2 : Lire le WSDL généré

**Q1.** URL du service : `http://localhost:8080/bib/EmpruntService` (`soap:address`).

| Opération | Message d'entrée | Message de sortie | Fault |
|---|---|---|---|
| `emprunterLivre` | `emprunterLivre` | `emprunterLivreResponse` | `LivreIndisponibleException` |
| `rendreLivre` | `rendreLivre` | `rendreLivreResponse` | aucun |

Le `portType` s'appelle `EmpruntService` et le `binding` `EmpruntPortBinding`.

**Q2.** `style="document"`, `use="literal"`. SOAP 1.1 : le binding utilise l'espace de noms `http://schemas.xmlsoap.org/wsdl/soap/` (SOAP 1.2 serait `.../wsdl/soap12/`).

**Q3.** `isbn` : `xs:string`. `dateRetour` : `xs:string`. La règle de format de l'ISBN n'apparaît pas : la regex n'existe que dans le code Java (aucun `xs:pattern` dans le schéma). Le schéma n'a pas `elementFormDefault="qualified"`.

**Q4.** Non. Seul `<bib:emprunterLivre>` porte un préfixe. `<idAdherent>` et `<isbn>` n'en ont pas : les éléments enfants sont non qualifiés.

**Q5.** Dans l'onglet Raw de SoapUI :
- **HTTP** : la ligne `POST /bib/EmpruntService HTTP/1.1`, les en-têtes (`Content-Type`, `SOAPAction`) et la réponse `200 OK`. Il transporte le message.
- **XML** : le format textuel du contenu, avec espaces de noms.
- **SOAP** : l'`Envelope` avec `Header` et `Body`. Il structure le message et les Faults.
- **WSDL** : le contrat qui décrit où appeler, les opérations et le format des messages. Il ne circule pas dans l'appel.

Résultat dans SoapUI (`A-117`, `978-1-234-5678-9`) : `200 OK`, `numero` = `E-2026-0812`. Cet ISBN respecte le motif, donc pas de Fault.

## Étape 3 : Envoyer des messages SOAP

Fichiers : `01-emprunter.xml`, `02-emprunter-12.xml`, `04-erreurs.xml`, `05-corrige.xml`.

Test SOAP 1.1 avec curl : `200 OK`, `numero` = `E-2026-0813`.

**Q1.** SOAP est un protocole de messages : l'opération et les données sont dans le corps XML, avec une seule URL. Un GET n'a pas de corps et sert à identifier une ressource par son URL. On utilise donc POST, même pour lire.

**Q2.** Non, le bloc `jeton` n'a pas empêché le traitement (200). Le service ne le comprend pas, mais sans `mustUnderstand="1"` il est ignoré.

**Q3.** Le message SOAP 1.2 donne HTTP 500 avec une Fault SOAP 1.1 : `faultcode` = `S:VersionMismatch`. Le serveur attendait une `Envelope` dans l'espace de noms `http://schemas.xmlsoap.org/soap/envelope/` mais a reçu `http://www.w3.org/2003/05/soap-envelope`. Le binding lu à l'étape 2 est SOAP 1.1 uniquement, et l'espace de noms de l'enveloppe identifie la version.

**Q4.** Les cinq erreurs, testées une par une :

| # | Erreur | Réaction du serveur | Correction |
|---|---|---|---|
| 1 | `<soap:envelope>` (casse) | 500, `S:Client` : attendu `Envelope`, obtenu `envelope` | `<soap:Envelope>` |
| 2 | Espace de noms `http://supnum.mr/bibliotheque` | 500, `S:Client` : méthode de répartition introuvable pour `{http://supnum.mr/bibliotheque}rendreLivre` | `http://supnum.mr/bib` |
| 3 | `</rendreLivre>` ne ferme pas `<bib:rendreLivre>` | 500, `S:Server` : Unexpected close tag `</rendreLivre>`; expected `</bib:rendreLivre>` | `</bib:rendreLivre>` |
| 4 | `<bib:numero>` au lieu de `<numeroEmprunt>` | **aucune erreur** : 200, `penalite` = 0 | `<numeroEmprunt>` (sans préfixe) |
| 5 | `Header` après `Body` | **aucune erreur** : toléré | `Header` avant `Body` |

Les erreurs 1, 2 et 3 déclenchent une Fault. Les erreurs 4 et 5 ne déclenchent rien :
- **Erreur 4** : l'élément inconnu est ignoré, `numeroEmprunt` vaut `null`, et sans validation de schéma rien ne le signale.
- **Erreur 5** : Metro lit le Body sans se plaindre de l'ordre, bien que la spécification impose Header avant Body.

## Étape 4 : Faults

| Test | HTTP | `faultcode` | `faultstring` |
|---|---|---|---|
| 1. ISBN `978-2-10` | 500 | `S:Client` | `ISBN mal formé : 978-2-10` |
| 2. ISBN `978-2-10-074444-4` | 500 | `S:Server` | `Livre indisponible` |
| 3a. `mustUnderstand="1"` | 500 | `S:MustUnderstand` | `MustUnderstand headers:[{http://supnum.mr/bib}jeton] are not understood` |
| 3b. `mustUnderstand="0"` | 200 | aucune | emprunt créé, `E-2026-0814` |
| 4. `rendreLivre` avec `XYZ` | 500 | `S:Client` | `Numéro d'emprunt inconnu : XYZ` |

`rendreLivre` avec `E-2026-0812` : [à compléter si testé : 200 OK, `penalite` = 0].

**Q1.** Cas 1 : 500, `S:Client`, « ISBN mal formé : 978-2-10 ». Cas 2 : 500, `S:Server`, « Livre indisponible ».

**Q2.** `<detail>` contient :
```xml
<ns2:LivreIndisponible xmlns:ns2="http://supnum.mr/bib">
  <isbn>978-2-10-074444-4</isbn>
  <retourPrevu>2026-12-02</retourPrevu>
</ns2:LivreIndisponible>
```
Sa structure est décrite dans le WSDL par le `wsdl:fault` de `emprunterLivre` (portType), le `message` `LivreIndisponibleException` (part `fault`, élément `tns:LivreIndisponible`) et l'élément `LivreIndisponible` du schéma XSD (`?xsd=1`), de type `livreIndisponibleInfo`.

**Q3.** JAX-WS a choisi `S:Server`. Pourtant la requête est valide et c'est l'état métier (livre déjà emprunté) qui bloque : ce n'est ni une panne serveur ni une erreur de format. J'aurais plutôt choisi `Client` (ou une Fault métier distincte), car le client peut agir (autre livre, ou attendre la date de retour). `Server` suggère à tort un incident technique.

**Q4.** Avec `mustUnderstand="1"`, le serveur renvoie 500 avec `S:MustUnderstand` : le bloc est obligatoire mais non compris, donc tout le message est refusé et l'opération n'est pas exécutée. Avec `"0"` (ou sans l'attribut), le bloc est ignoré et la réponse est 200.

**Q5.** Ma Fault 1.2 (`xml/03-fault-sender-12.xml`) diffère des Faults 1.1 reçues :
- `faultcode` (`S:Client`/`S:Server`) devient `Code/Value` avec `Sender`/`Receiver`.
- `faultstring` devient `Reason/Text` avec `xml:lang` obligatoire.
- `detail` devient `Detail`. Ces éléments sont qualifiés dans l'espace de noms SOAP 1.2, alors qu'en 1.1 ils ne le sont pas.
- L'enveloppe change d'espace de noms.

Vérification du fichier `03-fault-sender-12.xml` : [à compléter : ouvert dans Chrome / xmllint, sans erreur].

Code modifié : dans `rendreLivre`, ajout du contrôle `numeroEmprunt == null || !startsWith("E-2026-")` qui lève une Fault `Client`, à la place du silence de l'erreur 4 de l'étape 3.

---

# Partie B : contract-first

## Étape 5 : Générer le code depuis le WSDL

`mvn generate-sources` : `BUILD SUCCESS`, code généré dans `target/generated-sources/wsimport/mr/supnum/bib/contrat/`.

**Q1.** Opération : `emprunterLivre`. Message de fault : `LivreIndisponibleFault` (le `wsdl:fault` s'appelle `LivreIndisponible`). Action SOAP : `http://supnum.mr/bib/emprunterLivre`, définie dans `soap:operation` du binding.

**Q2.** Signature dans `EmpruntPortType` : `EmpruntType emprunterLivre(String idAdherent, String isbn) throws LivreIndisponibleFault`. Le style document/literal « wrapped » déplie les paramètres.

**Q3.** `dateRetour` est un `XMLGregorianCalendar` (type Java de `xs:date`). En partie A c'était un `String`.

**Q4.** La classe d'exception est `LivreIndisponibleFault`. Son nom vient du nom du `wsdl:message` (`LivreIndisponibleFault`), et non du nom du `wsdl:fault`. L'élément XSD `LivreIndisponible` devient la classe qui porte le détail (`faultInfo`).

## Étape 6 : Implémenter et comparer

`mvn compile exec:java` démarre le serveur (`WSDL : http://localhost:8080/bib/EmpruntService?wsdl`).

Tests curl avec `xml/07-emprunter-contractfirst.xml` et `SOAPAction: "http://supnum.mr/bib/emprunterLivre"` :

| Test | HTTP | Résultat |
|---|---|---|
| ISBN `978-2-07-036822-8` | 200 | `numero` = `E-2026-0812`, `dateRetour` = `2026-12-08` |
| ISBN `978-2-10-074444-4` | 500 | `S:Server`, « Livre indisponible », `<detail>` avec `LivreIndisponible` (`isbn`, `retourPrevu` = `2026-12-02`) |
| ISBN `978-2-10` avec `@SchemaValidation` | 500 | `S:Client`, `SAXParseException; cvc-pattern-valid : La valeur '978-2-10' n'est pas un facet valide par rapport au modèle '97[89]-\d-\d{2,5}-\d{2,6}-\d' pour le type 'IsbnType'` |

**Q1.** Le WSDL publié est celui fourni grâce à `wsdlLocation` : [à vérifier dans Chrome : présence de `xs:pattern` sur `IsbnType` et de `xs:date`]. En partie A, ni le motif ni `xs:date` n'existaient.

**Q2.** Oui, il a fallu modifier le XML : le schéma a `elementFormDefault="qualified"`, donc `idAdherent` et `isbn` doivent porter le préfixe (`<bib:idAdherent>`, `<bib:isbn>`), alors qu'en partie A ils étaient non qualifiés. L'en-tête HTTP `SOAPAction` doit aussi valoir `"http://supnum.mr/bib/emprunterLivre"` (vide en partie A). Dans la réponse, les éléments enfants sont aussi qualifiés (espace de noms par défaut).

**Q3.** Sans `@SchemaValidation`, [à confirmer par votre test : `978-2-10` n'est pas refusé, le service ne valide pas le motif]. Avec `@SchemaValidation`, il est refusé par une Fault `S:Client` (HTTP 500) : `cvc-pattern-valid`, la valeur ne correspond pas au motif de `IsbnType`. Le contrat est donc appliqué automatiquement.

**Q4.** Sans `wsdlLocation` : [à tester ; attendu : Metro régénère le WSDL à partir des annotations, donc le motif de l'ISBN, `xs:date` et le `soapAction` du contrat disparaissent, comme en partie A].

---

# Conclusion : choisir et rendre

**Q1.** Deux différences observées entre les WSDL :
- **Types** : partie A, `isbn` et `dateRetour` sont `xs:string` sans motif. Partie B, `IsbnType` avec `xs:pattern` et `xs:date`.
- **Éléments qualifiés et SOAPAction** : partie A, paramètres non qualifiés et `soapAction=""`. Partie B, paramètres qualifiés et `soapAction` explicite.
- Aussi : le fault s'appelle `LivreIndisponibleException` en A et `LivreIndisponibleFault` en B ; le contrat B ne contient que `emprunterLivre`.

**Q2.** Si on