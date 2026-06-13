# Parte do Rodrigo — Datacenter + Core + ACLs Ed.A + IA

Configs partidas por equipamento. A ordem dos ficheiros numerados (`01-`, `02-`…)
é a ordem em que devem ser colados em cada dispositivo no GNS3.

```
src/
  Datacenter/
    R-DC/        base, interfaces, rotas, DHCP (exclusoes + pools), ACL-WAN-IN
    SW-DC/       01-vlans-trunk  +  02-port-security (SRS)
    servidores/  HTTP institucional, HTTP interno, FTP
  Core/
    R-Central/   nucleo de routing
    R-ISP/       01-03 saida Internet  +  04-nat-pat.txt (IA)
  EdificioA/
    R-Ed-A/      00-02 ACLs do Rodrigo  +  03-floating-route.txt (IA)
    SW-BA1/      port-security.txt (SRS, tarefa do Rodrigo nos switches do Romeu)
  EdificioB/
    R-Ed-B/      floating-route.txt (IA do Rodrigo, aplica no router do Ricardo)
```

Fonte autoritativa dos IPs: Google Drive → `Plano/PBL_Planeamento_RRR`.

## Tabela de endereçamento (resumo)

### Datacenter — 10.30.0.0/24
| VLAN | Segmento | Sub-rede | Gateway |
|---|---|---|---|
| 90 | Gestao DC | 10.30.0.0/29 | 10.30.0.1 |
| 100 | Servidores DC | 10.30.0.8/29 | 10.30.0.9 (R-DC/DHCP) |

Servidores (IP fixo): HTTP institucional `10.30.0.10` · HTTP interno `10.30.0.11` · FTP `10.30.0.12`.

Gestao DC (IP fixo): R-DC/gateway `10.30.0.1` · SW-DC (SVI VLAN 90) `10.30.0.2` · livres `.3`–`.6`.

**Segurança do DC (zona protegida):** o SW-DC e o R-DC **não** transportam a VLAN 60
(Convidados). No trunk SW-DC↔R-DC: `native vlan 999` (Parking Lot, VLAN morta) +
`allowed vlan 90,100`. **Porquê 999 e não 60 nem 1:** deixar a native na **VLAN 1**
seria um erro de segurança ainda pior do que desviar da convenção do enunciado
(Convidados=native); a 999 é uma VLAN morta cujas portas têm port-security e estão em
shutdown. A VLAN 60 não está no allowed e o R-DC não tem sub-interface `.60` → o
tráfego de Convidados **morre** no DC. Portas não usadas → VLAN 999 + shutdown + port-security.

### Links WAN — 10.99.0.0/24 (/30 cada)
| Ligacao | Sub-rede | IP local | IP remoto |
|---|---|---|---|
| DC ↔ Central | 10.99.0.0/30 | R-DC .1 | R-C .2 |
| Central ↔ ISP | 10.99.0.4/30 | R-C .5 | R-ISP .6 |
| Central ↔ Ed.A | 10.99.0.8/30 | R-C .10 | R-EdA .9 |
| Central ↔ Ed.B | 10.99.0.12/30 | R-C .14 | R-EdB .13 |
| Ed.A ↔ Ed.B | 10.99.0.16/30 | R-EdA .17 | R-EdB .18 (floating backup) |

**Mapa de interfaces físicas (cablagem GNS3 — espelhado: mesmo nº de porto nos 2 lados):**
- R-C: `e0/0`→R-Ed-B · `e0/1`→R-ISP · `e0/2`→R-Ed-A · `e0/3`→R-DC
- R-DC: `e0/3`→R-C (WAN) · `e0/1`→SW-DC (trunk)
- R-ISP: `e0/1`→R-C (interno) · `e0/0`→Internet (externo)
- R-Ed-A: `e0/2`→R-C · `e2/1`→R-Ed-B (link direto) · `e0/0` e `e0/1`→LANs/switches
- R-Ed-B: `e0/0`→R-C · `e2/1`→R-Ed-A (link direto)

### Ed.A — 10.10.0.0/20
V10 `10.10.0.0/25` · V20 `10.10.0.128/28` · V50 `10.10.0.144/29` ·
V60 `10.10.0.152/29` (native) · V70 `10.10.0.160/27` · V80 `10.10.0.192/28` · V90 `10.10.0.208/28`.

### Ed.B — 10.20.0.0/20
V10 `10.20.0.0/24` · V20 `10.20.1.0/26` · V30 `10.20.1.64/28` · V40 `10.20.1.96/27` ·
V60 `10.20.1.128/29` (native) · V70 `10.20.1.160/27` · V80 `10.20.1.192/28` · V90 `10.20.1.224/27`.

## Configuracao inicial dos hosts (antes dos IPs fixos)

- **VPCs (PCs de teste, NAO servidores):** IP estatico com
  `ip [IP_DO_PC] /[MASCARA] [IP_DA_GATEWAY]` e a seguir **obrigatorio** `save`.
  (Em rede normal os PCs recebem IP por DHCP; isto e so para testes com IP fixo.)
- **Servidores (Ubuntu):** IP fixo **persistente** via **netplan** (NAO `ip addr add`,
  que se perde no reboot). Cada ficheiro em `Datacenter/servidores/` escreve um
  `/etc/netplan/01-static.yaml` com `dhcp4: false` + endereco + default via `10.30.0.9`,
  e desativa o cloud-init para nao reverter. IPs: `.10` institucional · `.11` interno
  · `.12` FTP. O `dhcp4: false` para tambem os discovers parasitas na VLAN 100.

## Implementacao Avancada (IA) — parte do Rodrigo

- **NAT/PAT** (`Core/R-ISP/04-nat-pat.txt`): PAT overload das redes internas +
  port-forward TCP 80 para o website institucional (`10.30.0.10`).
- **Redundancia L3 / floating routes** (`EdificioA/R-Ed-A/03-floating-route.txt` e
  `EdificioB/R-Ed-B/floating-route.txt`): backup Ed.A↔Ed.B pelo link direto
  `10.99.0.16/30` com AD=200. Requer a rota primaria /20 via R-Central (AD 1) para
  flutuar corretamente.
- (Os outros 2 itens IA — LACP e Private VLAN — sao do Romeu.)

### Teste de failover L3 (defesa)
1. Em R-Ed-A: `show ip route 10.20.0.0` → next-hop `10.99.0.10` (via R-Central).
2. Desligar o link principal: `interface <link p/ R-C>` → `shutdown`.
3. `ping` de um host Ed.A para um host Ed.B → continua a funcionar.
4. `show ip route 10.20.0.0` → next-hop passa a `10.99.0.18` (link direto, AD 200).
5. `no shutdown` para restaurar.

## Itens em aberto (coordenacao com o grupo)

1. **helper-address = 10.30.0.9** — os routers Ed.A e Ed.B têm de apontar o
   `ip helper-address` para o R-DC (`10.30.0.9`). O config do Ed.B (Ricardo) usava
   `10.30.0.2` no plano antigo → tem de passar a `10.30.0.9`. O `Romeu/Config_ACL_Edificio_B`
   ainda usa os IPs antigos dos servidores (`.2/.3/.4`) → atualizar para `.9/.10/.11/.12`.
2. **Black-hole routes** (evitar loop na sumarizacao /20):
   - R-Ed-A (Romeu): `ip route 10.10.0.0 255.255.240.0 Null0`
   - R-Ed-B (Ricardo): `ip route 10.20.0.0 255.255.240.0 Null0`
3. **Sub-interfaces R-Ed-A** — confirmar com o Romeu se são `e0/0.X` (grelha) ou
   `e0/1.X` (config antigo). A aplicacao das ACLs (`EdificioA/R-Ed-A/02-acls-aplicar.txt`)
   tem de usar a mesma interface.
4. **Interfaces do link direto Ed.A↔Ed.B** — confirmar com Romeu/Ricardo a interface
   fisica (assumi `Eth0/3` no R-Ed-A e `Eth0/1` no R-Ed-B).
5. **Nome R-ISP / R-SP** — hostname mantido `R-ISP`; o no no GNS3 chama-se `R-SP`.
6. **Port Security (SRS)** — SW-DC já configurado (`Datacenter/SW-DC/02-port-security.txt`).
   BA1 (`EdificioA/SW-BA1/port-security.txt`) é template: ajustar o range para as
   portas access reais dos switches do Romeu antes de aplicar.
