Setup
RPi 4 Board mit RaspiOS 5.4 Lite
Qt5.15.2 Laufzeitbibliotheken aus Docker-Build mit EGLFS-Support
Kernel-Overlay für KMS-GPU-Treiber aktiv
Xenomai-Kernel Patch  installiert, Core 0/1 reserviert für Echtzeittasks
W5500-NIC per Jumper Wire an den SPI-Anschlüssen am RPi-GPIO
EK1100 Buskoppler, EL1014 (4 Inputs), EL2004 (4 Outputs) in dieser Reihenfolge direkt per Patchkabel verbunden mit dem W5500-NIC
Schalter an EL1014 Kanal 1 zum manuellen Test
Rückkopplung von EL2004 Kanal 2 auf EL1014 Kanal 2 per Drahtbrücke
modifizierte Testsoftware auf Grundlage des SOEM-Projekts 
Evaluierung
Zur Evaluierung wurde eine EtherCAT-Master Testsoftware vom SOEM-Projekt abgeleitet, die die C-Schickt kapselt, und zyklisch die I/Os der beiden ausliest bzw. schreibt. EL2004 Kanal 3 wird in jedem Zyklus getoggelt, Kanal 2 auf den inversen Wert von EL1014 Kanal 2 gesetzt. In einem langsameren Ausgabetask werden alle I/Os in der Konsole ausgegeben.

Ablauf je Zyklus:

Manipulation der Ausgänge
Versand des EtherCAT-Frames
Empfang des rücklaufenden EtherCAT-Frames
Ergebnisse
Funktion
Alle drei eingesetzten Kanäle zeigen das gewünschte Soll-Verhalten:

Input 1 zeigt die Stellung des angeschlossenen Schalters aus Zyklus n - 1
Input/Output 2: da das Ausgangsmodul dem Eingangsmodul nachgelagert ist, und die Eingangsinformation aus dem Zyklus n-1 stammt, wird jede Änderung des Ausgangs um zwei Zyklen verzögert rückgemeldet, so dass sich eine Schaltfrequenz von 1/4 der Buszyklusfrequenz ergibt
Input 3 wird in jedem Zyklus unabhängig getoggelt, so dass sich eine Schaltfrequenz entsprechend der Buszyklusfrequenz ergibt
--> Die Soll-Funktion ist fehlerfrei erfüllt.

Timing
Bis zu einer Zykluszeit von ~200µs besteht bei der relativ geringen Last eine stabile Kommunikation bei einer CPU-Last von ca. 2% des beanspruchten Cores. Der SPI-Clock entspricht mit 12,5MHz = 1/40 Core-Clock. Die Angabe eines anderen SPI-Prescalers im wiznet-Treiber blieb ohne Wirkung. Dieser sollte zumindest eine SPI-Clock von 15,625 Mhz bzw. 31,250MHz, was offensichtlich nicht geschieht.

Die SPI-Auslastung liegt bei dieser Zykluszeit bei ca. 75%, und teilt sich etwa symmetrisch in den Frame-Versand und Frame-Empfang auf. Bei Zykluszeiten unterhalb von 200µs steigt die CPU-Last rapide an, da der rücklaufende Frame erst nach Abschluss des Versands erfolgen kann, und sowohl Versand und Empfang den Core mit Rechenzeit beaufschlagen. Unterhalb von ~190µs blockiert das System vollständig, da das Echtzeitsystem dann ohne jede Unterbrechung im Einsatz ist, und auch die beiden Applikationscores nicht mehr zum Zuge kommen.

--> die angestrebte Zykluszeit von 100µs ist unter diesen Gegebenheiten nicht realisierbar. Während die eigentliche CPU-Last recht gering ausfällt, verhindert die vergleichsweise langsame, nicht parallelisierbare SPI-Kommunikation eine weitere Beschleunigung.

Das ausführen einer relativ aufwändigen OpenGL-Anwendung auf den verbleibenden Applikationscores hat keinen wesentlichen Einfluss auf das Echtzeitverhalten der im Xenomai-Subsystem reservierten Echtzeitcores.

Erkenntnisse weitere Schritte
Grundsätzlich kann die SPI-Clock laut Datenblatt des eingesetzten SoC auf bis zu 250MHz gesteigert werden, während der W5500 NIC für 80MHz spezifiziert ist, und somit ein stabiler Betrieb bei 62MHz und geeigneten PCB-Layout durchaus realisierbar sein sollte. Das Zustandekommen der SPI-Taktrate von nur 12,5MHz als Haupthindernis mit einem Teilungsfaktor von 40 bleibt unklar. Der W5500-Echtzeittreiber liegt lediglich als Binary vor. Zugang zum Quellcode wurde beim Entwickler angefragt, der jedoch aus nicht weiter bekannten Gründen nicht geneigt erscheint, diesen zu veröffentlichen.

Update
Der SPI-Clock wird vom RPi Core-Clock abgeleitet, der in der Default-Konfiguration von der Obergrenze von 500MHz automatisch auf 200MHz abgesenkt wird, wenn das System nicht unter Last steht. Mit der Reduktion um Faktor 2,5 reduziert sich entsprechend auch die SPI-Clock.

Das Xenomai-Kernel umfasst keinen CPU-Governors, die eine dynamische Taktung übernehmen, da eine dynamische Änderung der Systemfrequenz verständlicherweise nicht mit harter Echtzeit vereinbar ist. Der Core-Clock muss daher per Kernelparameter in der Datei /boot/config.txt statisch auf 500MHz angehoben werden:

core_freq=500
force_turbo=1
Die durch den Wiznet-Treiber einstellbare SPI-Clock entspricht dann in der Folge der erwarteten Frequenz, so dass sich bei einem Teiler von 16 ein SPI-Clock von 31,5MHz ergibt.

Ein stabiler Betrieb ist bei fliegendem Aufbau mit Patchkabeln bei 16,25MHz noch gegeben, bei 31,5MHz besteht jedoch keine ausreichende, elektrische Signalqualität mehr für eine Kommunikation mit dem Ethernet-Controller. Mit einer zu Testzwecken manuell gefertigten, besser geschirmten Verbindung erweisen sich auch die 31,5MHz als funktionsfähig. Die SPI-Auslastung beläuft sich im beschriebenen Aufbau und 100µs Zykluszeit auf ca. 80%, bei kaum messbarer CPU-Last, so dass auch hier die zu geringe SPI-Frequenz das maßgebliche Hindernis darstellt.

Höhere Taktraten bzw. geringere Vorteiler werden vom Wiznet-Echtzeittreiber des SOEM-Projekts jedoch generell nicht akzeptiert, auch wenn durch ein spezialisiertes Hardware-Layout die notwendigen Voraussetzungen geschaffen werden, und das SPI-Peripheral bis zu 250MHz ermöglicht, und die Einschränkung höchstwahrscheinlich alleine der elektrischen Signalübertragung auf dem RPi-Board geschuldet ist.

Lösungsansätze
der angestrebte RPi-EtherCAT-Master wird voraussichtlich nur für Tischgeräte bei der VMP zum Einsatz kommen. Je nach erreichbarem Datendurchsatz ist ggf. abzuklären, ob eine geringere Zykluszeit, die sich in der Latenzzeit spiegelt, ausreichend ist.

Entwicklung eines eigenen W5500-Echtzeittreibers. Zu realisieren ist dabei lediglich die Raw-Socket Kommunikation ohne Ethernet-Stack. Eine Aufwandsabschätzung wird erschwert, da der von simplerobot bereitgestellte Treiber nicht als Quellcode vorliegt. Für Embedded-Targets existieren jedoch diverse Treiberimplementierung, deren Raw-Socket-Schicht ggf. als Vorlage dienen kann. Eine Einarbeitung in die Xenomai-API wird jedoch unumgänglich sein.

Auslagerung der EtherCAT-Masterfunktionalität in ein Embedded-System
Ein auf harte Echtzeit ausgelegtes Embedded-System wird ohne große Schwierigkeiten in der Lage sein, das angestrebte Timing zu erfüllen, bei deutlich geringeren Jitter als mit einem Applikationsprozessor realisierbar.

Zu den Aufgabe eines solchen Systems gehört bei Minimalauslegung die Abwicklung der zyklischen Prozessdatenkommunikation, und der schnelle Austausch der Prozessdaten mit dem Rpi, die jedoch ebenfalls einen Echtzeittreiber und ein Protokoll auf Seiten des Applikationssystems erforderlich macht.

Bei einer erweiterten Auslegung kann ein solches System folgende Aufgaben übernehmen:

Übernahme der gesamten EtherCAT-Masterfunktionalität inkl. des Master-Stacks zum zyklischen PDO-Austausch und zur Realisierung aller erforderlichen EtherCAT-Dienste wie FoE.
logische Kombination der vom jeweiligen Prüfprogramm konfigurierten Signale, insbesondere das Triggern von PDO-Signalen und ggf. die Berechnung und Bereitstellung von Soll- und Istwerten aus den im EtherCAT-Antwortframe gewonnenen Messwerten.
Applikationsschnittstelle zur Konfiguration und Beobachtung des Busgeschehens ohne harte Echtzeitanforderung.
Als Vorlage für letztere Option kann grundsätzlich das SOEM-Projekt dienen, das insgesamt jedoch qualitativ nicht den besten Eindruck erweckt, und spärlich dokumentiert ist. Bei diversen, kommerziellen Anbieter sind Masterstacks erhältlich, die sowohl für Emebedded-Linux, als auch bereits für Arm-Cortex-Derivate optimiert einsetzbar sind, und die erforderlichen Ethernet-Linklayers beinhalten, u.a.:

https://www.ibv-augsburg.de/icnet/ethercat-master/ 
https://www.acontis.com/de/ethercat-master.html 
Hier ist ggf. der Aufwand einer Eigenentwicklung unter Einbeziehung des OpenSource-Projektes gegen die Kosten und Bedingungen der kommerziellen Softwarelizenzen abzuwägen.
