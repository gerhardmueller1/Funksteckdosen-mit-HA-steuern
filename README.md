# RFCode entschlüsselt

Seit Jahren benutzen wir Funksteckdosen um Lichter ein- und wieder auszuschalten. 
Will heißen, die waren schon lange vorhanden, da habe ich noch nicht an Heimautomatisierung gedacht
-wobei, im Prinzip ist das ja auch eine Art von Automatisierung, zwar nicht mit so vielen Möglichkeiten, aber dennoch-.
Das schöne an den Dingern ist.. die funktionieren einfach. Da muss nix programmiert werden, einfach ein paar Dip-Schalter
von null auf eins stellen und gut ist. Keinen Account bei irgendeinem Heinopei einrichten, keine Daten irgendwo hinterlegen 
und irgendwo was programmieren oder konfigurieren. Kein Rechner, kein Handy, kein Internet notwendig. Einfach den Handsender 
nehmen und Lichter schalten. So einfach kann es sein. 

# Warum Integration in Heimautomatisierung?

Nachdem ich dann irgendwann mit HA angefangen habe, stellte sich die Herausforderung auch die Funktsteckdosen mit HA
zu steuern. Nicht das die nicht mehr funktionierten, einfach der Bequemlichkeit wegen. Einfache Szenen oder Automatisierungen 
um z.B. ein Licht einzuschalten, wenn der Fernseher an ist oder schlichtweg einfach alle Lichter Abends irgenwann auszuschalten. 
Keine Komplizierten Dinge, die Handsender werden ja weiterhin genutzt. 

Mal eben geht da aber leider gar nichts. Mein erstes Unterfangen, 433MHz-Sender und Empfänger am Raspberry 
anzuschließen scheiterten, nicht weil es nicht funktionierte, erste Erfolge und Fortschritte waren da, aber das was mir alles
zu kompliziert. Daher habe ich mir dann einen Sonoff-RF-Sender/Empfänger zugelegt und diesen, erst mit Tasmota, dann mit 
ESPHome geflasht. Und siehe da, der RF-Empfänger liefert Codes beim Drücken der Tasten der verschiedenen Handsender. Super, 
einfach die Codes notieren und die Codes senden. 

Hmmm.. nächstes Problem: Für jede Taste des Senders muss ein Helfer, also ein ```input_button``` und eine passende 
Automatisierung angelegt werden: 

```
alias: RF Wohnzimmer A Off
description: ""
triggers:
  - entity_id:
      - input_button.rf_wohnzimmer_a_off
    from: null
    trigger: state
conditions: []
actions:
  - data:
      low: 1010
      high: 340
      sync: 10340
      code: 13781727
    action: esphome.rf_bridge_send_rf_code
mode: single
```

Block kopieren, etwas anpassen und mit ein wenig Aufwand ist die Fernbedienung "nachgebaut": 

![grafik](https://github.com/user-attachments/assets/15d07dbf-389f-415b-a85e-11f1371e5219)


## High, Low, Sync und Code ... 

Damit kam ich dann einige Zeit hin, bzw. konnte damit arbeiten. Lichter ein und ausschalten ... läuft!

```
type: horizontal-stack
cards:
  - type: vertical-stack
    cards:
      - type: vertical-stack
        cards:
          - show_name: false
            show_icon: true
            type: button
            tap_action:
              action: toggle
            entity: input_button.rf_wohnzimmer_a_on
          - show_name: false
            show_icon: true
            type: button
            tap_action:
              action: toggle
            entity: input_button.rf_wohnzimmer_a_off
    title: A
  - type: vertical-stack
    cards:
      ....
    title: B
  ....
title: Lichter Wohnzimmer
```

Und läuft bis heute. Nachdem dann alles so gut funktionierte haben wir irgendwann ein weiteres Set an Funksteckdosen 
gekauft. Erstmal ohne die im HA einzubinden, irgendwann dann doch. Nun kam aber der Zeitpunkt, zu dem ich mich mit 
dem Sniffen der Codes mal wieder beschäftigen musste. So weit, so gut. Handsender genommen, die Konsole des 
RF-Sender/Empfängers aufgemacht, Tasten gedrückt und die Codes notiert. 

Das Dumme nur, bei jedem Tastendruck erschienen andere Codes .... WTF!

```
13:22:27	[I]	[rf_bridge:057]	Received RFBridge Code: sync=0x28BE low=0x0168 high=0x0410 code=0x151454
13:22:27	[D]	[text_sensor:064]	'Rf Bridge sync': Sending state '10430'
13:22:27	[D]	[text_sensor:064]	'Rf Bridge low': Sending state '360'
13:22:28	[D]	[text_sensor:064]	'Rf Bridge high': Sending state '1040'
13:22:28	[D]	[text_sensor:064]	'Rf Bridge code': Sending state '1381460'
```

Keine Sorge, ich werde euch hier nicht mit langen Zahlenreihen belästigen!

Low, High, Sync und Code scheinen wahllos zu variieren; nun vielleicht nicht ganz wahllos. Wenn ich eine Taste
lange drücke ist zumindest der "Code" recht stabil, nur der erste und der letzte Wert, also beim Drücken und beim
Wiederloslassen, weichen ab. Die anderen drei Werte, Low, High, Sync, schwanken. 

Faszinierend ist auch, das bei jedem weiteren Versuch die drei Werte Low, High und Sync in einem anderen Bereich liegen. 
Das einzige was stabil bleibt ist der Code (mit Ausnahme des ersten und letzen Wertes). 

## Analyse der Codes

Auf die Schnelle ist da keine Systematik zu erkennen. Das große Googeln begann: "RfCode berechnen", "433MHz Codec", 
"Elro Funksteckdosen Codes" und was weiß ich nicht alles. Ehrlich gesagt, so richtig schlau bin ich aus all dem nicht geworden. 
Das einzige was ich gelertn habe ist, dass die drei Werte *Low, High und Sync nicht wirklich relevant* sind. Diese drei Werte 
beschreiben das Zeitverhalten zum Senden, bzw. beim Empfang der Codes, also die Puls-Länge bzw. Dauer zum senden einer Eins bzw. Null 
und des Syncronisations-Signals. (Siehe hier [Github, Tamota Issue 1387](https://github.com/arendst/Tasmota/issues/1387)). 

Wie sich der eigentliche Code berechnet ist dort auch nicht wirklich beschrieben. Auch auf der  
[FHEM Seite zu Intertechno Funksendern](https://wiki.fhem.de/wiki/Intertechno_Code_Berechnung) ist -für mich- keine klare Beschreibung
des Codes vorhanden. Aber auf dieser Basis hatte ich den Ansatz zur Berechnung der Codes. Dieser berechnet sich aus dem Binärcode der 
DIP-Schalter, der "Schalt-Gruppe" (also A,B,C,D oder E, welches es beim Elro-Handsender nicht gibt) und je einem Bit für Ein und Aus. 

```1	2	3	4	5	A	B	C	D	E	On	Off``` 

Bei einem meiner DIP-Schalter sind die Bits 1 und 4 gesetzt. D.h. der "Basis-Code" für diese Funksteckdosen lautet 

```1	0	0	1	0	0	0	0	0	0	0	0``` 

Wenn nun die Steckdose A eingeschaltet werden soll, wird noch das sechste Bit und das Bit für "On" gesetzt: 

```1	0	0	1	0	1	0	0	0	0	1	0``` 

Das -ich nenne es mal so- Bescheuerte an der Sache ist, dass der Code nicht einfach aus der Stellung der DIP-Schalter und der Tasten 
abzulesen, bzw. in einen Binär-Code umzusetzen ist. Der Code "überspringt" jedes zweite Bit!

![grafik](https://github.com/user-attachments/assets/6677726f-6166-4d3e-b567-8780c79c27e5)

Und das nicht alleine! Jedes Bit des o.g. Codes wird erst noch negiert. Somit ergibt sich für meinen Funksender für "Steckdose A an" erstmal

```0	1	1	0	1	0	1	1	1	1	0	1``` 

und mit den "zwischengeschalteten" Nullen

```00	01	01	00	01	00	01	01	01	01	00	01``` .

Somit lässt sich der RFCode einfach als Hex-Wert des o.g. Binär-Codes ablesen. Für meine "Steckdose A an" lautet der Code somit 1328465. 
Wie oben schon erwähnt sind Low, High und Sync nur für das zeitliche Verhalten beim Senden der Codes relevant. Diese setze ich somit beim 
Senden von RFCodes nun auf einen fixen Wert. 

## Berechnung der Codes zum Senden

Für die Berechnung der RFCodes (egal ob empfangen oder zu senden) habe ich mir dann eine Excel-Tabelle zusammengebastelt. Mit der
kann ich nun einfach für die DIP-Schalter meinen Code eingeben und schematisch die Tasten der Schalt-Gruppen (A-E) und Ein oder Aus einsezten
und somit den Code berechnen, den ich senden muss. Somit reduziert sich der Aufwand für mich darauf, die Codes in der Aktion 
```esphome.rf_bridge_send_rf_code``` nutzen. Als Low, High, Sync verwende ich einfach einen Standard (330, 1000, 10290) der sich be mir als
praktikabel herausgestellt hat. 

Das Tool zur Berechnung des Codes habe ich nun auch als Google-Tabelle umgesetzt, Du findest es
[hier](https://docs.google.com/spreadsheets/d/1-JK3X-01phvdzJit6nTt2Sel1AQAIu--iug18V1fK_A/edit?usp=sharing).
Alternativ kannst Du dir Berechnung des Dezimal-Codes über folgendes Python-Skript ermitteln:

```
#!/usr/bin/env python3

import sys

arg = sys.argv[1] if len(sys.argv) > 1 else None
if arg is None:
    raise ValueError("usage: " + __file__ + " [12345][ABDCE][XY]\n where X=on,Y=off")

chars = list(arg.upper())
val = {char: 1 for char in chars}
sendcode = 0
hexcode = 0

for char in "12345ABCDEXY":
    hexcode <<= 1
    hexcode += 1 if char in val else 0
    sendcode <<= 2
    sendcode += 1 if char not in val else 0

print(f"{hexcode:24b} {sendcode}")
```

Als Argument einfach die Codes der gesetzten DIP-Schalter übergeben und welche Tastengruppe gedrückt wird (A-E)
sowie Ein (X) oder Aus (Y). 

```
~/py > ./sample.py 15AX
            100011000010 1377617
```

## Schlußbemerkung

Ob man hiermit mehrere Steckdosen gleichzeitig anschalten kann (also z.B. "A und B an") hängt wohl von den
Geräten ab, bei mir funktioniert es nicht!




