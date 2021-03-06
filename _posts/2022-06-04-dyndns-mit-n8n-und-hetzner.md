---
layout: post
title: DynDNS mit n8n und Hetzner
date: 2022-06-04 15:07 +0200
---

Die meisten von uns haben wechselnde IP-Adressen und sind auf DynDNS-Dienste angewiesen, die einen DNS Namen immer auf die richtige IP-Adresse lenken. Dazu gibt es diverse Dienste, die zum Teil kostenpflichtig sind - oder man nutzt normale DNS Provider, die Änderungen mittels eine API erlauben. Ich persönlich nutze dazu Hetzner als Provider meines Vertrauens und für das regelmäßige updaten des DNS-Eintrages n8n, ein graphisches Workflow-Tool.

## Voraussetzungen
Ich gehe in diesem Artikel davon aus, dass sowohl n8n eingerichtet und lauffähig ist, sowie die gewünschte Domain bereits von Hetzner DNS verwaltet wird. Zu beidem gibt es im großen weiten Internet genügend Anleitungen.

## Einrichtung von Hetzner DNS
### Anlegen des Dummy-Eintrages
Damit ich mir nicht allzu viel Mühe bei der Nutzung der API geben muss, lege ich bereits einen Eintrag für die DynDNS Zone an. In meinem Fall ist das ein A-Eintrag für eine Subdomain (bspw. dyndns), die ich zunächst einfach auf 127.0.0.1 zeigen lasse. Die IP wird später automatisch aktualisiert.

Später braucht man außerdem auch noch die Zone-ID der DNS Zone bei Hetzner. Diese ist Teil der URL für die Zone. Lautet diese bspw. `https://dns.hetzner.com/zone/NVLoKmnhJikKhzzuee`, dann ist die Zone-ID `NVLoKmnhJikKhzzuee`.

### API Zugang
Für die Nutzung der API ist ein API Token notwendig. Diesen kann man unter https://dns.hetzner.com/settings/api-token generieren. Den geniererten Token sollte man sich irgendwo zwischenspeichern, da man ihn sonst nie wieder zu Gesicht bekommt.

## Der n8n Workflow
![img-description](/assets/img/n8n-workflow.png)
_Der gesamte Workflow_

Der n8n Workflow beginnt mit einem Cron-Trigger, damit der Workflow regelmäßig ausgeführt wird. Bei mir ist das alle 10 Minuten der Fall.

### Get IP
Um meine eigene externe IP Adresse zu erfahren nutze ich die API von ipify.org. Ruft man `https://api.ipify.org?format=json` auf, erhält man ein JSON-Dokument mit einem einzigen Attribut zurück. In diesem steht die eigene IP. Im weiteren Workflow kann man dieses Ergebnis dann in Expressions oder Code-Nodes nutzen, indem man auf die Variable `$node["Get IP"].json["ip"]` zurückgreift.

### Set parameters
Im Set Task setze ich Variablen:
* **zoneID** ist die zuvor ermittelte ID der DNS Zone bei Hetzner
* **record** ist die Bezeichnung des Records, den wir als DynDNS Eintrag nutzen (bspw. dyndns)

### Get records
Um den Eintrag bei Hetzner zu aktualisieren brauchen wir seine ID. Diese kann man mit der API ermitteln, indem ein GET-Request gegen `https://dns.hetzner.com/api/v1/records` durchgeführt wird. Als Parameter wird die zoneID benötigt. Dazu wird als Query Parameter ein Key-Value Paar mit dem Namen _zone___id_ und einer Expression als Inhalt angelegt. Die Expression lautet `{{ '{{$node["Set parameters"].json["zoneID"]'}}}}`, ruft also den zuvor gesetzten Parameter ab.

### Get target record
Die Record-Abfrage enthält als Ergebnis alle Einträge in dieser DNS-Zone. Um den richtigen Eintrag zu finden, nutze ich eine Function-Node mit einem kleinen Javascript. Das war für mich am einfachsten. Das Script sucht anhand des Namens des Records den entsprechenden Record und legt diesen in einer später abrufbaren Variable ab.
```javascript
items[0].json.record = items[0].json.records.find(rec => rec.name === 'dyndns');

return items;
```

### Update record
Mit den in den vorangegangen Schritten eingesammelten Daten kann ich nun den DNS-Eintrag entsprechend Update. Dazu sende ich ein PUT-Requenst. Die URL dieses Requests setzt sich aus einem statischen Teil und der zuvor ermittelten Record-ID zusammen. Hier bietet sich also die Nutzung einer Expression an. `https://dns.hetzner.com/api/v1/records/{{ '{{$json["record"]["id"]'}}}}` resultiert in der richtigen URL, wenn in der vorangegangenen Function-Node die Variable entsprechend gesetzt wurde.

Als Body-Parameter werden folgende benötigt:
* **zone_id** ist die ID der DNS Zone bei Hetzner, ich nutze als Expression `{{ '{{$node["Get target record"].json["record"]["zone_id"]'}}}}`.
* **name** ist der Name des Records; er soll gleich bleiben, daher nutze ich `{{ '{{$node["Get target record"].json["record"]["name"]'}}}}`.
* **ttl** ist das TTL-Value des Records; auch dieses soll gleich bleiben und wird daher mit `{{ '{{$node["Get target record"].json["record"]["ttl"]'}}}}` befüllt.
* **type** ist der Record Type, in der Regel sollte das der Typ A sein.
* **value** ist die IP-Adresse auf die der Record zeigen soll. Diese habe ich zuvor mit der API von ipify.org ermittelt und kann durch die Expression `{{ '{{$node["Get IP"].json["ip"]'}}}}` abgerufen werden.

## Fazit
Hetzner bietet mit seine API ein gute Möglichkeit, den DNS Dienst auch für DynDNS-Funktionen zu nutzen. Mit n8n, mit dem sich auch diverse andere Tasks automatisieren lassen, lässt sich eine solche Funktion umsetzen, ohne eine zusätzliche Software oder gar einen Docker-Container nur für DynDNS zu installieren.