# Funksteckdosen mit HomeAssistant steuern

Seit Jahren benutzen wir Funksteckdosen um Lichter ein- und wieder auszuschalten. 
Will heißen, die waren schon lange vorhanden, da habe ich noch nicht an Heimautomatisierung gedacht. 
-wobei, im Prinzip ist das ja auch eine Art von Automatisierung, zwar nicht mit so vielen Möglichkeiten, aber dennoch-
Nachdem ich dann irgendwann mit HA angefangen habe, stellte sich die Herausforderung auch die Funktsteckdosen mit HA
zu steuern. Mal eben geht da aber leider gar nichts. Mein erstes Unterfangen, 433MHz-Sender und Empfänger am Raspberry 
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



