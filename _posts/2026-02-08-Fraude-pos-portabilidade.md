---
layout: post
title:  "Fraude Pós-Portabilidade: quando o ataque acontece em minutos (e o SIEM ainda está dormindo)"
date:   2026-08-02
---
# Fraude Pós-Portabilidade: quando o ataque acontece em minutos (e o SIEM ainda está dormindo)

Antes de qualquer coisa… não, isso aqui **não é teoria**.
Isso acontece **todo santo dia** em banco, fintech, marketplace e qualquer sistema que ainda confia em SMS como fator forte de autenticação.

Se você acha que SIM Swap é algo sofisticado, sinto dizer: **não é**.
É rápido, repetível e deixa rastro — o problema é que quase ninguém olha para os rastros certos.

A ideia desse post é simples: **mostrar como detectar fraude pós-portabilidade usando SIEM de verdade**, com regra que funciona no mundo real, não em slide de vendor.

---

## A cadeia de ataque (não tem mistério)

O fluxo quase sempre é esse:

1. Portabilidade ou troca de SIM
2. Linha cai
3. Reset de MFA via SMS
4. Account recovery
5. Login de origem estranha
6. Conta sequestrada

Isso tudo costuma acontecer **em menos de 30 minutos**.
Quem trata esses eventos de forma isolada já perdeu.

---

## O erro clássico dos SOCs

O SIEM até vê tudo isso, mas:

* Portabilidade fica no log de telecom
* Reset de MFA fica no IdP
* Recovery fica na aplicação
* Login anômalo vira alerta genérico

Ou seja: **ninguém correlaciona**.

E fraude não acontece em eventos isolados.
Ela acontece em **cadeia**.

---

## Vamos ao que interessa: regras de SIEM

Vou usar campos genéricos. Ajusta para o teu parser e segue o jogo.

---

## Splunk – correlação que realmente importa

### Portabilidade ou SIM Swap (evento raiz)

```spl
index=telecom
(event_type="port_request" OR event_type="sim_swap" OR event_type="iccid_change")
| stats earliest(_time) as port_time by user_id phone
```

Isso **não é alerta**.
Isso é **contexto de risco**.

---

### Reset de MFA até 24h após portabilidade

```spl
index=auth
(event_type="mfa_reset" OR event_type="mfa_disabled")
| join user_id
    [ search index=telecom event_type="port_request"
      | eval port_time=_time ]
| where _time >= port_time AND _time <= port_time + 86400
```

Aqui já começa a feder.
Reset de MFA depois de portabilidade **não é normal**.

---

### Account Recovery + Portabilidade + MFA Reset

```spl
index=auth
(event_type="account_recovery" OR event_type="password_reset")
| join user_id
    [ search index=auth event_type="mfa_reset"
      | eval mfa_time=_time ]
| join user_id
    [ search index=telecom event_type="port_request"
      | eval port_time=_time ]
| where _time >= port_time AND _time <= port_time + 86400
```

Se isso aqui disparou e ninguém bloqueou a conta, **o estrago já aconteceu**.

---

### Login fora do padrão (ASN / país)

```spl
index=auth event_type="login"
| where src_asn NOT IN ("AS_INTERNO_1","AS_INTERNO_2")
| where geo_country!="BR"
```

Sozinho é ruído.
Com o resto da cadeia, é confirmação.

---

## Elastic / OpenSearch – EQL simples e eficiente

### Cadeia completa em 24h

```eql
sequence by user_id with maxspan=24h
  [ authentication where event_type == "port_request" ]
  [ authentication where event_type in ("mfa_reset","mfa_disabled") ]
  [ authentication where event_type in ("account_recovery","password_reset") ]
```

Isso aqui deveria ser **CRITICAL por padrão**.

---

### Troca de IMEI após portabilidade

```eql
sequence by user_id with maxspan=24h
  [ authentication where event_type == "port_request" ]
  [ authentication where event_type == "login" and imei != previous_imei ]
```

Golpista raramente mantém o mesmo device.

---

## QRadar – AQL direto ao ponto

```sql
SELECT username, MIN(starttime) AS port_time
FROM events
WHERE eventname='port_request'
GROUP BY username
```

Agora cruza com:

```sql
SELECT username, starttime
FROM events
WHERE eventname IN ('mfa_reset','account_recovery')
AND starttime > port_time
AND starttime < port_time + 86400000
```

Não tem milagre.
Tem correlação.

---

## Microsoft Sentinel – KQL sem frescura

```kql
let PortEvents =
SecurityEvent
| where EventType == "port_request"
| project UserId, PortTime=TimeGenerated;

let MFAEvents =
SecurityEvent
| where EventType in ("mfa_reset","mfa_disabled");

let RecoveryEvents =
SecurityEvent
| where EventType in ("account_recovery","password_reset");

PortEvents
| join MFAEvents on UserId
| join RecoveryEvents on UserId
| where TimeGenerated between (PortTime .. PortTime + 24h)
```

Se isso aqui vira só alerta e não ação, o problema não é técnico.

---

## Score antifraude (simples e funcional)

| Evento                   | Score |
| ------------------------ | ----- |
| Portabilidade / SIM Swap | +50   |
| Reset de MFA             | +40   |
| Account Recovery         | +40   |
| ASN estranho             | +15   |
| País diferente           | +15   |

Passou de **80 pontos**?
Não discute — **bloqueia**.

---

## O que deveria acontecer automaticamente

Quando a regra dispara:

* Revoga token
* Bloqueia reset de senha
* Mata sessões
* Exige verificação fora de banda
* Notifica antifraude

Qualquer coisa menos que isso é **teatro de segurança**.

---

## Conclusão (sem romantizar)

Fraude por SIM Swap **não é APT**.
É oportunismo com telecom mal protegida e autenticação fraca.

Quem ainda depende de alerta isolado já perdeu.
Quem correlaciona evento **ganha tempo** — e tempo é o que separa incidente de prejuízo.

Se o teu SIEM ainda trata portabilidade como “evento informativo”, o atacante agradece.
