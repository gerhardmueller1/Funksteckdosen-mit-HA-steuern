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

Das Dumme nur, bei jedem Tastendruck erschienen andere Codes... 

