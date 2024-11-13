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

Für Der Taster für A 

Das -ich nenne es mal so- Bescheuerte an der Sache ist, dass der Code nicht einfach aus der Stellung der DIP-Schalter und der Tasten 
abzulesen, bzw. in einen Binär-Code umzusetzen ist. Der Code "überspringt" jedes zweite Bit!

![grafik](https://github.com/user-attachments/assets/6677726f-6166-4d3e-b567-8780c79c27e5)


Damit alleine ist der Code aber noch nicht berechnet. Nachdem 
