---
date: 2016-01-02T16:13:00+01:00
lastmod: 2016-01-02T16:13:00+01:00
title: "Die Architektur der Galeria.de Plattform im Kontext der Produktentwicklungs-Organisation"
description: "Dieser Beitrag erläutert die wichtigsten Eckpfeiler der Software- und System-Architektur der Galeria.de Online Plattform, und zeigt das Verhältnis der Architektur zu Vorgehensmodell und Aufbauorganisation auf."
authors: ["manuelkiessling"]
slug: 2016/01/02/die-architektur-der-galeria-de-plattform-im-kontext-der-produktentwicklungsorganisation
lang: de
---

<h2 id="ber-diesen-artikel">Über diesen Artikel</h2>

<p>Im Kontext des <a href="http://galeria-kaufhof.github.io">GALERIA Kaufhof Technology Blogs</a> stellt der vorliegende Artikel ein Update und eine Erweiterung des <a href="http://galeria-kaufhof.github.io/general/2014/09/20/jump-ein-technologiesprung-bei-galeria-kaufhof/">Beitrags von September 2014</a> dar.</p>

<p>Schon damals existierte ein klar definiertes Set an Vorgaben, welches den Rahmen für Makro- und Mikroarchitekturfragen
gesteckt hat und das Projekt in Hinblick auf Fragen der System- und Softwarearchitektur leitet.</p>

<p>In den vergangenen Tagen haben wir begonnen, ausgehend von den Erfahrungen bis heute und unserer jetzigen Perspektive,
einige der Grundlagen unserer Architektur noch einmal neu aufzuschreiben.</p>

<p>Anstoß hierzu lieferte unter anderem der Launch von <a href="http://scs-architecture.org/">http://scs-architecture.org/</a>, einem Portal,
welches das Konzept der Self-contained Systems, die den zentralen Baustein auch unserer Architektur bilden, präsentiert.</p>

<p>Inhaltlich haben wir das Konzept SCS seit langem gelebt, aber semantisch war der Ansatz auf unserer Architekturlandkarte
nicht klar verortet. Mit der Überarbeitung haben nun alle zentralen Bausteine einen klaren Platz und einen klaren Namen.</p>

<p>Bei einem Projekt, welches seit nunmehr fast 2 Jahren läuft und seit fast einem Jahr im Betriebs- und
Weiterentwicklungsmodus ist, wird auch die Frage des fachlichen Wachstums spannend. So klar die bestehende Struktur ist
- was sind die architektonischen Leitlinien, wenn der fachliche Themenumfang wächst und die Plattform sich
inhaltich weiterentwickelt? Schliesslich stehen wir noch am Anfang unserer Mission, MCR-Marktführer in Europa zu werden.</p>

<p>Zusätzlich klingt in diesem Dokument auch das Verhältnis zwischen Produktarchitektur und
Produktentwicklungsorganisation stärker an (ohne dabei den Anspruch zu erheben, die Aufbau- und Ablauforganisation der
Galeria.de Produktentwicklung vollumfänglich aufzuzeigen – dies muss im Zuge anderer Beiträge erfolgen).</p>

<h2 id="die-visualisierung">Die Visualisierung</h2>

<p>Einen Überblick über die verschiedenen Architekturkomponenten und ihr Verhältnis zueinander soll das folgende Schaubild
ermöglichen:</p>

<p>
<a href="/images/Ueberblick_Architektur_GALERIA_Kaufhof_Online_Plattform.svg"><img src="http://manuel.kiessling.net/images/Ueberblick_Architektur_GALERIA_Kaufhof_Online_Plattform.svg" width="100%"></a>
<br>
(Zum Vergößern klicken)
</p>

<h2 id="grundlagen-der-architektur">Grundlagen der Architektur</h2>

<p>Zwei Grundideen bilden das Fundament der architektonischen Strukturierung: Eine vertikale Orientierung der High-Level
Komponenten in sogenannten Self-contained Systems, und eine fachliche motivierte Trennung und Gruppierung dieser
Komponenten in sogenannten Domänen.</p>

<p>Das Verhältnis von Domänen zu Systemen ist wie folgt: eine Domäne liegt immer dann vor, wenn ein oder
mehrere Systeme einen logisch zusammenhängenden Ausschnitt der fachlichen Use-Cases eines Benutzers vollumfänglich
abbilden. Konkretes Beispiel: die Domäne SEARCH bei Galeria.de umfasst diejenigen Systeme, welche von der
Benutzeroberfläche bis zur Datenhaltung das Suchen und Finden von Produkten für den Benutzer von Galeria.de ermöglichen.</p>

<p>Der Domäne SEARCH ist also mindestens ein System zugeordnet, welches sowohl die Weboberflächen-Elemente (wie zum
Beispiel die Suchbox mit Auto-Complete, Suchergebnisseite usw.) bereitstellt, als auch den Import von Produktdaten und
deren Überführung in eine spezialisierte Such-Datenbank implementiert.</p>

<p>Ein System wiederum ist eine Sammlung von Anwendungen mit gemeinsamer Datenhaltung, welche in unserem Fall auch alle
innerhalb desselben Code Repositories liegen und auch gemeinsam deployed werden – bei einer Scala Domäne könnte es sich
also konkret um ein <a href="http://www.scala-sbt.org/0.13/tutorial/Multi-Project.html">sbt Multiprojekt</a> bestehend aus einer
Play2 Anwendung für das Webinterface und zusätzlich Anwendungen auf Basis von Akka für die Hintergrundverarbeitung von
Daten handeln. Und unsere Ruby Domäne CONTROL wiederum betreibt ein System, welches sich intern stark in Richtung einer
Microservice-basierten Struktur entwickelt hat und als gemeinsame Datenhaltung unter anderem einen
MessageBus-orientierten Ansatz verfolgt.</p>

<p>Miteinander sprechen diese Systeme – innerhalb einer Domänengrenze und darüber hinaus – nur über definierte
Schnittstellen (da sie keine Daten teilen dürfen), und dies unter Vermeidung von verteilten Callstacks.</p>

<p>In gewissem Sinne wird hier das bekannte Paradigma von loser Kopplung und hoher Kohäsion, welches klassischerweise auf
Ebene eines einzelnen Softwaresystems betrachtet wird, auf einer höheren Ebene fortgesetzt.</p>

<p>Die Kohäsion entsteht, weil fachlich verwandte Themen vereint werden in den Self-contained Systems einer Domäne. Die
lose Kopplung wird abgebildet dadurch, dass die verschiedenen Systeme nur über Schnittstellen miteinander kommunizieren.</p>

<p>Damit gilt für das Gesamtsystem dieselbe Eigenschaft, die auch innerhalb eines Softwaresystems gilt, welches nach diesem
Paradigma entworfen wurde: Änderungen in einer Komponente bedingen nur dann Änderungen in einer anderen Komponente,
wenn die Änderungen die Schnittstelle betreffen.</p>

<p>Dies sorgt für hohe Robustheit des Gesamtsystems (ohne verteilte Callstacks und dank Replikation von Daten können andere
Systeme weiter operieren, auch wenn ein angebundenes System nicht-verfügbar wird), ermöglicht weitgehend autarkes
Arbeiten pro Domäne (nicht zuletzt in Hinblick auf die Releasefrequenz), bietet die Möglichkeit, Systeme nach ihren
unterschiedlichen Anforderungen auch unterschiedlich zu skalieren, und erlaubt eine in Hinblick auf die erforderliche
Funktionalität passgenaue Wahl der Technologien pro System. Weitere Informationen hierzu liefert
<a href="http://scs-architecture.org">scs-architecture.org</a>.</p>

<p>Das Konzept der Domäne ist weiterhin der Brückenschlag zwischen Architektur und Aufbauorganisation. Optimalerweise steht
hinter jeder Domäne ein Team – in unserem Fall ein Scrum-Team – welches von Anforderungsmanagent über
Software-Entwicklung und QA bis hin zum Betrieb die Domäne mit ihren Systemen fachlich und technisch vollumfänglich
“owned”.</p>

<h2 id="domnen-und-ihre-systeme">Domänen und ihre Systeme</h2>

<p>Warum dann noch die Unterscheidung zwischen Domäne und System? Warum nicht 1 Domäne gleich 1 System? An dieser Stelle
findet derzeit eine Evolution unseres bisherigen Modells statt, in dem die Begriffe bisher oft synonym verwendet
wurden.</p>

<p>Die Grund für eine Unterscheidung ist, dass ein Komponentenschnitt einerseits fachlich motiviert sein kann, andererseits
technisch. Eine rein technische Motivation führt hierbei zu einem neuen System, eine fachliche Motivation zu einer neuen
Domäne. Der Begriff der “technischen Motivation” ist allerdings recht weit gefasst – es muss nicht zwangsläufig die
Einführung einer neuen Technologie (Programmiersprache, Framework, Datenbanksystem usw.) vorliegen: selbst bei
gleichbleibendem Technologiestack kann es die technische Motivation geben, eine weiterhin saubere Codebase gewährleisten
zu wollen oder feingranularer releasen zu wollen.</p>

<p>Für beide Motivationen gibt es derzeit Beispiele im Projekt. Das Team der bestehenden Domäne EXPLORE, welches sich
bisher vornehmlich um Teaser und Störer im Shop kümmert, soll in Zukunft die Verantwortung übernehmen für die
Infrastruktur von Inhalten auf Galeria.de, die nicht direkt mit dem Shopping-Erlebnis des Kunden zu tun haben,
beispielsweise Presseseiten und Unternehmensinformationen sowie Inhalte rund um das Recruiting.</p>

<p>Abgesehen davon, dass hier auf Basis eines Open Source CMS auch technisch eine neue Lösung entsteht, wird schnell klar,
dass die bisherige EXPLORE Fachlichkeit verlassen wird. Daher begründet das Team derzeit eine neue Domäne CONTENT. Auch
wenn hier vorerst dieselben handelnden Personen an Bord sind, sehen wir das zu behandelnde Thema aufgrund der
andersartigen fachlichen Ausrichtung als neue fachliche Einheit.</p>

<p>Anders im Team ORDER, welches sich um alle Fachlichkeiten rund um das Thema Bestellungen kümmert. Hier zeichnet sich ab,
dass es sinnvoll sein könnte, die Webshop-orientierten Aspekte des Bestellens von der nachgelagerten Bestellverarbeitung
zu trennen – sinnvoll zum Beispiel vor dem Hintergrund der möglicherweise sehr unterschiedlichen
Skalierungsanforderungen für den Shop einerseits und die nachgelagerte Verarbeitung von Bestellungen andererseits;
weiterhin ist zu erwarten, dass eine Entkopplung dieser beiden Aspekte auch in Hinblick auf Releasezyklen und Sauberkeit
der lokalen Architektur Vorteile bringt. Self-contained Systems sind stets Monolithen – das ist nicht per se negativ,
aber jeder Monolith kann irgendwann zu groß werden; “zu groß” ist eine sehr subjektive Eigenschaft, aber ein Maßstab ist
sicherlich die mittlerweile weitverbreitete Formulierung “passt nicht mehr vollständig in den Kopf eines Teammitglieds”.</p>

<h2 id="systeme-und-umgebungen">Systeme und Umgebungen</h2>

<p>Das Schaubild spricht weiterhin von Umgebungen. Gemeint ist damit der Kontext, in dem die Anwendungen von Systemen
laufen können. Aufgrund der vertikalen Orientierung von Systemen ist der Ausführungskontext ihrer Anwendungen potentiell
verteilt: Eine Play2-basierte Scala Anwendung wird auf den Servern von Galeria.de ausgeführt, also in der sogenannten
Plattformumgebung – diese Backend-Anwendung liefert aber vielleicht eine Single-Page Application oder eine andere Form
von JavaScript-Anwendung aus; diese läuft dann im Browser des Benutzers, also in dessen Umgebung.</p>

<p>Weiterhin sprechen viele Systeme der Plattformumgebung mit Fremdsystemen, im Falle von Galeria.de beispielsweise mit
Systemen der Warenwirtschaft. Diese werden ausgeführt in einer Fremdumgebung, also einer Umgebung die technisch und
organisatorisch außerhalb der Kontrolle von Galeria.de liegt.</p>

<h2 id="schnittstellen">Schnittstellen</h2>

<p>Der Verzicht auf eine gemeinsame Datenhaltung bedingt klar definierte und zuverlässig arbeitende Schnittstellen.
Leitbild ist für uns das World Wide Web, weshalb wir auf HTTP als Transportprotokoll setzen und unsere Schnittstellen
nach den Prinzipien von REST gestalten.</p>

<p>Wir unterscheiden hierbei 4 Typen von Schnittstellen:</p>

<ul>
  <li>Typ “Web”
    <ul>
      <li>baut auf HTTP auf</li>
      <li>RESTful</li>
      <li>HATEOASful</li>
      <li>benutzerorientierter Media-Type</li>
      <li>klassischerweise die von einer Backend-Anwendung generierte, HTML-basierte Webseite (oder Webseiten-Elemente, welche
durch die Frontend-Integration per SSI eingebunden werden)</li>
    </ul>
  </li>
  <li>Typ “REST”
    <ul>
      <li>baut auf HTTP auf</li>
      <li>RESTful</li>
      <li>HATEOASful</li>
      <li>maschinenorientierter Media-Type</li>
      <li>klassischerweise der durch eine Backend-Anwendung bereitgestellte JSON-Webservice, welcher von JavaScript
Anwendungen, Mobile Apps, oder Fremdanwendungen genutzt wird</li>
    </ul>
  </li>
  <li>Typ “Multipart”
    <ul>
      <li>baut auf HTTP auf</li>
      <li>RESTful</li>
      <li>maschinenorientierter Media-Type</li>
      <li>Auf HTTP multipart basierende Feed- oder Snapshot-Schnittstelle für die Synchronisation von Massendaten und für
Event-Sourcing</li>
    </ul>
  </li>
  <li>Typ “Other”
    <ul>
      <li>Schnittstellen, die nicht den anderen Typen entsprechen: FTP, SOAP, usw. Unterschiedlichste Transportprotokolle und
Medientypen sind denkbar.</li>
    </ul>
  </li>
</ul>

<h2 id="zusammenfassung">Zusammenfassung</h2>

<p>Der Architekturansatz von vertikal orientierten Self-contained Systems gepaart mit fachlich geschnittenen Domänen und
HTTP-basierten Schnittstellen bildet den Rahmen für alle Produktentwicklungsinitiativen von Galeria.de.</p>

<p>Dieser Rahmen ermöglicht optimale Kundenorientierung durch fachliche Spezialisierung in den Domänenteams, ein robustes
Gesamtprodukt dank der Entkopplung von Datenhaltung und Callstacks, und eine effektive Weiterentwicklung dank autarker
Releaseprozesse und einem auf die Schnittstellen beschränkten Abstimmungsprozess in den Softwareentwicklungsteams.</p>
