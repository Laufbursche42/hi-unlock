# Anleitung

Diese Seite redet per Web Bluetooth mit einem Hitway H11, der über die Tuya-White-Label-App (com.hitway.ytd) läuft. Wähle dein Modell, dann verbinde. Sie ist eine Machbarkeitsstudie und bleibt ehrlich dabei, was heute schon geht und was noch das reale Gerät braucht.

## Was du brauchst

- Einen Hitway H11, der sich über die Tuya-App verbindet. Er wirbt mit dem Tuya-BLE-Dienst 0xfd50.
- Einen Browser mit Web Bluetooth: Chrome oder Edge auf Android oder Desktop, auf dem iPhone die App Bluefy.
- Den geräteindividuellen localKey deines Scooters (aus der Tuya-Cloud) und die numerischen dpIds der Funktionen, die du nutzen willst (aus dem Cloud-Schema oder einem BLE-Mitschnitt). Beides steht unten.

## Die ehrliche Lage in einem Absatz

Der Bluetooth-Transport ist verstanden und in dieser Seite umgesetzt. Der H11 spricht das generische Tuya-Datenpunkt-Protokoll (DP). Aber vier Dinge sind cloud- oder geräteseitig und stehen nicht im App-Paket: (1) der localKey, (2) die numerischen dpIds samt Wertebereich und Skalierung, (3) die genaue Sitzungsschlüssel-Ableitung (das Hitway-SDK verschleiert die KDF-Rechnung, belegt sind nur die Eingaben) und (4) ob ein DP-Schreibvorgang die Geschwindigkeit überhaupt ändert. Solange du die ersten beiden nicht lieferst, bleibt jeder Schreibvorgang und jede Telemetrie-Skalierung mit genanntem Grund ausgegraut. Punkt vier ist der wichtigste: **ein Geschwindigkeits-Unlock über BLE ist am H11 nicht bewiesen.**

## Warum ein BLE-Unlock nicht bewiesen ist (bitte lesen)

Die harte Geschwindigkeitskappe des H11 liegt nicht in einem Tuya-Datenpunkt. Sie liegt in der Controller-Firmware als `MOTOR[0x3e]`, gespeist vor allem aus der 1200-Baud-UART-Anzeige (`SpeedLevel * 2400 + 12000`). Ein BLE-Schreibvorgang landet am MCU des Tuya-Moduls, nicht automatisch auf der Displayleitung, und der Controller hat nur einen QS-S4-Display-Parser - keine Tuya-Frame-Verarbeitung. Also:

- Ein BLE-Schreibvorgang kann höchstens eine Prozent-Drossel lösen, **nicht** den harten Cap.
- Das Anheben der realen Höchstgeschwindigkeit über 22 km/h braucht die controllerseitige Kappe (Wege UART / Programmer / SWD), nicht BLE allein.
- Ob das Setzen von `speed_limit_enum` oder `max_speed` per BLE etwas bewirkt, ist weder bewiesen noch widerlegt; der erste Schreibtest nach der localKey-Extraktion klärt es.

Deshalb zeigt dieses Werkzeug Telemetrie und bietet einen gesperrten Schreibweg, tut aber nicht so, als hätte es einen Unlock-Knopf.

## Schritt 1: Modell wählen

Nutze das Modell-Dropdown oben.

- **Auto / Generischer Hitway (Tuya)** - die vollständige belegte DP-Menge, alles gesperrt bis bestätigt.
- **Hitway H11 (Dual-Drive)** - dieselbe Menge, für den H11 beschriftet. Der H11 hat zwei Controller (B Master, A Slave); die App-Verbindung ist eine einzige Tuya-BLE-Verbindung.

## Schritt 2: Verbinden

1. Scooter einschalten.
2. Auf Verbinden tippen, deinen Scooter aus der Liste wählen. Die Seite filtert auf den Tuya-Dienst 0xfd50 und den klassischen 0x1910 (das SDK bringt beide Profile mit und welches der H11 zeigt, ist offline nicht belegt).
3. Nach dem Verbinden erscheinen im Log die rohen Notifications. Die unverschlüsselte Device-Info-Antwort (Opcode 0x00) enthält den srand (6 Byte), den die Seite für den Sitzungsschlüssel braucht.

## Schritt 3: Schlüssel eintragen

1. Trage den localKey deines Scooters ein (16 Zeichen oder als Hex).
2. Trage den srand ein, den du aus der Device-Info-Notification im Log abliest.
3. Die Seite zeigt daraufhin secretKey5 = MD5(localKey verkettet mit srand).

**Ehrlicher Vorbehalt zur Ableitung:** im dekompilierten Hitway-SDK ist die genaue Schlüsselableitung verschleiert. Belegt sind nur die *Eingaben*: der Cloud-localKey, der 6-Byte-srand und ein 32-Byte-authKey. Das hier gezeigte secretKey5 = MD5(localKey || srand) ist der Tuya-Standard-Kandidat; ob der H11 genau das nutzt, ist unbestätigt. Wird ein korrekt gebauter, scharfer Frame abgelehnt, liegt es vermutlich an der KDF oder am DP-Kanal-Weg, nicht an einem Fehler des Werkzeugs.

### Woher kommt der localKey

Der localKey wird beim Erst-Pairing an dein Tuya-Konto vergeben und liegt in der Cloud. Er reist nie über Bluetooth, deshalb kann man ihn nicht aus einem BLE-Mitschnitt lesen, und er steht nicht im Firmware-Dump (die dortigen 0x2000-Base64-Blöcke sind Provisioning-Keys, nicht der Cloud-localKey). Er ist kontogebunden: eine Neuzuweisung an ein anderes Konto erzeugt einen neuen localKey.

Zwei Wege liefern ihn:

1. **Frida auf einem gerooteten Telefon.** Hooke `DeviceBean.getLocalKey`; das Skript liefert devId, localKey, productId und die schemaMap auf einmal. Braucht Root oder eine debugbare App.
2. **Tuya-IoT-Plattform (iot.tuya.com).** Kostenloses Konto anlegen, Cloud-Projekt erstellen, unter Devices das App-Konto per QR verknüpfen, dann im API Explorer `GET /v1.0/devices/{device_id}` für `local_key` und `.../specifications` fürs Schema abrufen. OEM-White-Label-Konten lassen sich manchmal nicht direkt verknüpfen; dann den Scooter zusätzlich in Tuyas generischer App Smart Life koppeln und diese verknüpfen.

Der localKey ist ein Geheimnis deines Geräts, gib ihn nicht weiter.

## Schritt 4: Geräteschema laden (dpIds)

Die numerische dpId einer Funktion steht nicht in der App und ist in keinem Modell fest verdrahtet - sie liegt im Tuya-Cloud-Produktschema. Die bekannten Code-Namen des H11 sind unter anderem `speed_limit_enum`, `max_speed`, `work_mode`, `boost`, `zero_start`, `cruise_switch`, `assist` sowie die Telemetrie-Codes `speed`, `battery_percentage`, `voltage`, `current`, `mileage_total`, `endurance_mileage`, `temp_value`, `fault`. Ihre Zahlen sind offline unbekannt.

Trage im Feld Geräteschema als JSON ein, was du kennst, code auf dpId:

```
{ "speed_limit_enum": 104, "work_mode": 108, "battery_percentage": 106 }
```

Mit der langen Form kannst du je Eintrag auch den Wire-Typ und die Protokollversion setzen:

```
{ "work_mode": { "dpId": 108, "type": "enum", "pv": 3 } }
```

Jede Zeile, deren dpId du einträgst, schaltet frei (sobald auch die Schlüssel drin sind). Zeilen ohne dpId bleiben ausgegraut mit dem Abzeichen "Schema nötig". Vertraue nicht den Beispielwerten aus Abschnitt 4.4 des Berichts (`max_speed` id 104, `speed_limit_enum` id 105) - das sind ausdrücklich Platzhalter, nicht das echte Geräteschema.

## Schritt 5: Bluetooth-Log auswerten (btsnoop)

Die Schlüssel-Karte hat einen Bluetooth-Log-Upload. Lade ein btsnoop `.log`, ein `.gz`, einen Android-Fehlerbericht `.zip` oder einen Hex-Text-Dump. Alles wird rein lokal ausgewertet (die CSP verbietet jeden Upload).

Was ein Mitschnitt verrät und was nicht:

- **Verrät:** die unverschlüsselte Device-Info-Antwort (Opcode 0x00), daraus zeigt das Werkzeug `srand` (6 Byte) und `authKey` (32 Byte) als Kandidaten samt aller Frame-Metadaten; und die verschlüsselten DP-Frames als Chiffretext (nur gezählt).
- **Verrät nicht:** den localKey (geht nie über die Luft), den Sitzungs-AES-Schlüssel (verschleierte KDF über localKey + srand + authKey) und damit den Klartext jedes DP-Werts. Du kannst srand und authKey ernten, aber ohne localKey bleiben die DP-Nutzdaten undurchsichtig.

Das Log hilft also, `srand` einzutragen und `dpId`-Kandidaten vorzuschlagen; den localKey liefert es nie. Schreiben bleibt danach gesperrt.

## Schritt 6: Telemetrie, Einstellungen und erweiterte Einstellungen

- **Live-Werte** werden aus dem DP-Bericht gefüllt, sobald eine passende dpId im Schema steht. Werte erscheinen roh mit dem Abzeichen "unbestätigt", weil Skalierung und Einheit cloud-seitig und hier nicht bestätigt sind.
- **Einstellungen** und **Erweiterte Einstellungen** zeigen je DP eine beschriftete Zeile. Jede Zeile ist ausgegraut, bis ihre dpId im Schema steht und die Sitzung bereit ist (localKey plus srand plus verbunden). Riskante Schreibvorgänge (Tempolimit, `work_mode`, Sperren) fragen zusätzlich nach.
- **Roh-DP schreiben** ist die Notluke: du lieferst die dpId selbst, deshalb geht sie schon unter dem Sitzungs-Gate allein. Vorschau rechnet und loggt den Frame nur; Scharf sendet ihn wirklich.

Ehrlicher Vorbehalt: selbst ein korrekt gebauter scharfer Frame kann vom Gerät abgelehnt werden (KDF/DP-Kanal), und selbst wenn er angenommen wird, ändert er womöglich die Geschwindigkeit nicht (der harte Cap sitzt im Controller). Das ist kein Fehler des Werkzeugs.

## Schritt 7: Fahrmodus-Sequenzer

Anders als bei manchen anderen Scootern ist beim H11 **kein Boost-Trick belegt**. Diese Karte schaltet nur den Fahrmodus-DP (`work_mode`) per BLE, mit optionaler Sprachansage, damit beide Hände am Lenker bleiben. Ob ein BLE-Modus-Wechsel am H11 etwas bewirkt, ist unbestätigt, und er kann den harten Firmware-Cap nicht anheben. Betrachte es als Experiment, nur auf privatem Gelände.

## Diagnose und Selbsttest

- Öffentliches Log (an per Voreinstellung) maskiert sensible Stellen und lange Hex-Ketten (localKey, Seriennummer, srand) vor dem Kopieren oder Speichern, damit ein geteiltes Log nichts verrät.
- Diagnose zeigt jeden decodierten Frame (zusammengesetzt, Kontrollbyte, seq/cmd/flag, je DP dpId/Typ/Wert) und zapft zusätzliche Notify-Kanäle an. Genau dieser Mitschnitt ist es, den du brauchst, um die dpIds zu lernen.
- Diagnose: alle Geräte listet die GATT-Dienste eines beliebig gewählten Geräts.
- Selbsttest prüft die Krypto-Bausteine gegen bekannte Testvektoren. Muss grün sein.
- Log kopieren beziehungsweise Log speichern gibt dir den ganzen Mitschnitt zum Behalten oder Schicken.

## Recht und Sicherheit

Nur am eigenen Fahrzeug auf privatem Gelände. Das Anheben der Geschwindigkeit hebt die Drossel auf, die ABE erlischt und der Betrieb auf öffentlichen Wegen ist dann nicht erlaubt. Wer ein verändertes Fahrzeug im öffentlichen Verkehr fährt, begeht mit hoher Wahrscheinlichkeit Straftaten (Fahren ohne die nötige Fahrerlaubnis sowie einen Verstoß gegen das Pflichtversicherungsgesetz). Alles auf eigenes Risiko.
