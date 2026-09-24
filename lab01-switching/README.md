## 🔌 Lab 1: Switching de campus

### Objetivo

Montar uma rede de campus com redundância total na camada 2 e no gateway, aplicando VLANs, trunks 802.1Q, Rapid-PVST+, EtherChannel com LACP, roteamento entre VLANs e HSRP.

### Nós

[[<img width="836" height="524" alt="image" src="https://github.com/user-attachments/assets/33f94d90-27e4-4479-a86c-9ced8f08f5ed" />]]

| Nó | Imagem | Função |
|---|---|---|
| CORE1, CORE2 | vIOS L2 | Core (collapsed core), switch L3, gateway |
| ACC1, ACC2 | vIOS L2 | Switches de acesso |
| PC1–PC4 | VPCS | Hosts de teste |

Consumo estimado: ~4 GB de RAM.

### Ligações

| Origem | Destino |
|---|---|
| CORE1 Gi0/0 | CORE2 Gi0/0 (Po1) |
| CORE1 Gi0/1 | CORE2 Gi0/1 (Po1) |
| CORE1 Gi0/2 | ACC1 Gi0/0 |
| CORE1 Gi0/3 | ACC2 Gi0/0 |
| CORE2 Gi0/2 | ACC1 Gi0/1 |
| CORE2 Gi0/3 | ACC2 Gi0/1 |
| ACC1 Gi1/0 / Gi1/1 | PC1 / PC2 |
| ACC2 Gi1/0 / Gi1/1 | PC3 / PC4 |

### Endereçamento

| VLAN | Nome | Rede | Gateway (VIP HSRP) | CORE1 | CORE2 |
|---|---|---|---|---|---|
| 10 | USUARIOS | 10.10.10.0/24 | .1 | .2 | .3 |
| 20 | SERVIDORES | 10.10.20.0/24 | .1 | .2 | .3 |
| 99 | GERENCIA | 10.10.99.0/24 | .1 | .2 | .3 |
| 999 | NATIVA-NAO-USADA | nenhuma | nenhum | nenhum | nenhum |

| Dispositivo | IP |
|---|---|
| ACC1 (gerência) | 10.10.99.11 |
| ACC2 (gerência) | 10.10.99.12 |
| PC1 (VLAN 10) | 10.10.10.101 |
| PC2 (VLAN 20) | 10.10.20.101 |
| PC3 (VLAN 10) | 10.10.10.102 |
| PC4 (VLAN 20) | 10.10.20.102 |

### Princípio de desenho

**O root bridge de cada VLAN é o mesmo switch que é HSRP Active dessa VLAN.** CORE1 responde pelas VLANs 10 e 99, CORE2 pela VLAN 20. O tráfego sobe pelo caminho não bloqueado direto até o gateway, e a carga fica dividida entre os dois cores.

---

## 📚 Conceitos aplicados no Lab 1

### Modelo hierárquico (acesso, distribuição e core)

Redes corporativas são organizadas em camadas:

- **Acesso:** onde usuários, impressoras e servidores se conectam.
- **Distribuição:** agrega os switches de acesso e aplica políticas.
- **Core:** backbone de alta velocidade.

Em redes pequenas e médias, distribuição e core são unificados no **collapsed core**, o modelo usado neste lab. Cada switch de acesso tem um uplink para cada core, garantindo redundância.

### VLAN, trunk e native VLAN

- **VLAN:** divide um switch físico em vários domínios de broadcast isolados.
- **Trunk (802.1Q):** porta que transporta várias VLANs, marcando cada quadro com uma *tag* que identifica a VLAN.
- **Native VLAN:** VLAN que trafega **sem tag** no trunk. Por segurança (mitigação de *VLAN hopping*), usa-se uma VLAN vazia e sem uso (999). A native precisa ser igual nas duas pontas do trunk.
- **Allowed VLANs:** restringe quais VLANs podem passar em cada trunk.

### Loops de camada 2 e STP

Links redundantes formam anéis. Como quadros Ethernet **não têm TTL**, um broadcast circula para sempre e causa uma **tempestade de broadcast**, além de instabilidade da tabela MAC.

O **STP (Spanning Tree Protocol)** faz os switches trocarem **BPDUs**, detectarem os anéis e **bloquearem logicamente** portas, transformando a malha em uma árvore sem loops. Se um caminho ativo cair, a porta reserva é desbloqueada.

### Root bridge

É a raiz da árvore do STP. Vence a eleição o switch com o **menor Bridge ID** (prioridade + MAC). A prioridade padrão é 32768, então, sem configuração, vence o menor MAC, o que é aleatório e pode eleger um switch inadequado.

- `spanning-tree vlan X root primary`: baixa a prioridade para o switch vencer.
- `spanning-tree vlan X root secondary`: deixa o switch como o próximo da fila.

### Papéis de porta (RSTP)

| Papel | Significado | Estado |
|---|---|---|
| **Root port** | Melhor caminho até o root bridge (só existe em switches não-root) | Encaminha |
| **Designated port** | Porta que representa o segmento; todas as portas do root são designated | Encaminha |
| **Alternate port** | Caminho reserva até o root | Bloqueada |

> *Root bridge* é o switch; *root port* é uma porta de um switch não-root.

### Rapid-PVST+

- **Rapid (RSTP, 802.1w):** converge em ~1 s, contra 30–50 s do STP 802.1D.
- **PVST (Per-VLAN):** uma árvore independente por VLAN, cada uma com seu root. Isso permite **dividir a carga**: o cabo bloqueado para uma VLAN encaminha a outra.

### PortFast e BPDU Guard

- **PortFast:** a porta de host sobe imediatamente, sem esperar a convergência do STP.
- **BPDU Guard:** se chegar um BPDU em uma porta de host (ou seja, alguém ligou um switch ali), a porta vai para **err-disabled**.

### EtherChannel, Port-channel (Po1) e LACP

- **EtherChannel:** agrupa vários links físicos em uma **interface lógica única**, o **Port-channel**. "Po1" é o Port-channel número 1.
- Para o STP, o Port-channel é **um único link**: nada é bloqueado. Ganha-se banda somada e redundância sem reconvergência.
- **LACP (IEEE 802.3ad/802.1AX):** protocolo de negociação do agrupamento.

| Modo | Comportamento |
|---|---|
| `active` | Inicia a negociação LACP |
| `passive` | Só responde; passive + passive não forma canal |
| `on` | Força o canal sem negociação (risco de loop) |

- O balanceamento é feito **por fluxo** (hash de MAC/IP/portas), não por pacote. Um fluxo único nunca passa da velocidade de um link.
- Todas as portas membros precisam ter **configuração idêntica**, senão ficam suspensas.

### SVI e roteamento entre VLANs

- **SVI (`interface vlan X`):** interface virtual L3 que representa a VLAN dentro do switch.
- Com `ip routing`, o switch roteia entre as SVIs e vira um **switch L3**, atuando como gateway dos hosts.
- Uma SVI só fica *up* se a VLAN existe e há pelo menos uma porta ativa nela.

### HSRP e FHRP

Um host aceita só um gateway. Se ele cair, o host perde acesso a outras redes. O **HSRP (Hot Standby Router Protocol)**, proprietário da Cisco, é um **FHRP (First Hop Redundancy Protocol)**:

- Os roteadores têm IPs reais (.2 e .3) e compartilham um **IP virtual** (.1), usado como gateway pelos hosts, e um MAC virtual (`0000.0c07.acXX`, onde XX é o número do grupo em hexadecimal).
- **Active:** responde pelo IP virtual. **Standby:** monitora e assume em caso de falha.
- Hellos a cada 3 s, *hold time* de 10 s.
- **Priority:** padrão 100; o maior vence.
- **Preempt:** permite retomar o papel de Active ao voltar de uma queda.

| Protocolo | Tipo | Papéis |
|---|---|---|
| HSRP | Cisco | Active / Standby |
| VRRP | Padrão aberto (usado por FortiGate e MikroTik) | Master / Backup |
| GLBP | Cisco, com balanceamento | AVG / AVF |

### Alinhamento root bridge + HSRP Active

Se o root da VLAN e o Active HSRP forem o mesmo switch, o tráfego sobe pelo caminho mais curto até o gateway. Desalinhados, o tráfego atravessa o link entre os cores desnecessariamente.

---

## 🧱 Construção do Lab 1 em camadas

A construção é incremental: cada camada é configurada, observada e entendida antes da próxima.

| Camada | O que é configurado | O que observar |
|---|---|---|
| 1 | VLANs, trunks, portas de acesso | Quem virou root **sozinho** (menor MAC) e qual cabo entre os cores ficou bloqueado |
| 2 | Root primary/secondary | Root correto por VLAN; porta bloqueada diferente na VLAN 10 e na VLAN 20 |
| 3 | Po1 com LACP | `Po1(SU)`; as portas físicas somem do STP e dão lugar ao Po1, encaminhando |
| 4 | SVIs + `ip routing`, PCs com gateway .2 | Ping entre VLANs funciona, **mas** parar o CORE1 derruba tudo |
| 5 | HSRP, PCs com gateway .1 | Parar o CORE1 causa só alguns segundos de perda; preempt devolve o papel |

---

## ✅ Validação

| Comando | Onde | Resultado esperado |
|---|---|---|
| `show vlan brief` | Todos | VLANs 10, 20, 99, 999; portas de acesso nas VLANs corretas |
| `show interfaces trunk` | Todos | Trunks ativos, native 999, allowed 10,20,99 |
| `show etherchannel summary` | Cores | `Po1(SU)` com membros `(P)` |
| `show lacp neighbor` | Cores | Vizinho LACP nas duas portas |
| `show spanning-tree root` | Todos | CORE1 root das VLANs 10/99; CORE2 root da VLAN 20 |
| `show spanning-tree vlan 10` | ACC1 | Gi0/0 Root/FWD; Gi0/1 Altn/BLK |
| `show spanning-tree vlan 20` | ACC1 | Gi0/1 Root/FWD; Gi0/0 Altn/BLK |
| `show standby brief` | Cores | CORE1 Active nas VLANs 10/99; CORE2 Active na VLAN 20 |

Testes de conectividade a partir do PC1:

```
ping 10.10.10.102     # L2 na mesma VLAN, entre switches diferentes
ping 10.10.20.102     # roteamento entre VLANs
trace 10.10.20.102    # caminho percorrido
```

---

## 🔁 Testes de failover

Com `ping 10.10.20.102 -t` rodando no PC1:

| Teste | Ação | Comportamento esperado |
|---|---|---|
| Queda de membro do Po1 | `shutdown` na Gi0/0 do CORE1 | Po1 continua *up*; perda zero ou quase |
| Queda de uplink | `shutdown` na Gi0/0 do ACC1 | Gi0/1 passa de Alternate para Root em ~1 s (RSTP) |
| Queda de core | Parar o nó CORE1 | CORE2 assume HSRP e root; alguns segundos de perda |
| Retorno do core | Religar o CORE1 | Preempt devolve o papel de Active ao CORE1 |

---

## 🛠️ Troubleshooting: defeitos para injetar

Injetar **um defeito por vez**, diagnosticar pelos sintomas e só depois corrigir.

| # | Defeito | Como injetar | Como diagnosticar |
|---|---|---|---|
| 1 | Native VLAN divergente | `switchport trunk native vlan 1` só no ACC2 Gi0/0 | Logs de CDP de *native VLAN mismatch*; `show interfaces trunk` nas duas pontas |
| 2 | VLAN fora do allowed list | `switchport trunk allowed vlan remove 10` na Gi0/2 do CORE1 | Falha parcial; coluna *allowed and active* do `show interfaces trunk` |
| 3 | LACP passive/passive | `channel-group 1 mode passive` nos dois cores | Po1 não sobe; `show etherchannel summary`, `show lacp neighbor` |
| 4 | Membro suspenso | Allowed VLAN diferente em só uma porta do Po1 | Flag `(s)` ou `(I)` no `show etherchannel summary` |
| 5 | Root sequestrado | `spanning-tree vlan 10 priority 4096` no ACC2 | `show spanning-tree root`; caminho subótimo no `trace`. Prevenção: **root guard** |
| 6 | BPDU Guard em ação | Ligar ACC1 Gi1/3 ↔ ACC2 Gi1/3 com portfast + bpduguard | `show interfaces status err-disabled`; recuperar com `shutdown` / `no shutdown` |
| 7 | HSRP desalinhado do root | `standby 10 priority 120` no CORE2 | Tráfego da VLAN 10 atravessa o Po1; `trace` e contadores de interface |
| 8 | SVI down/down | `no vlan 20` no CORE1 | `show ip interface brief`; SVI cai com configuração intacta |

---

## 🎤 Perguntas de entrevista

- Por que usar uma native VLAN que não carrega tráfego?
- Como funciona a eleição do root bridge? O que acontece se ninguém configurar?
- Qual a diferença entre root bridge e root port?
- Quais são os papéis de porta no RSTP e o que cada um faz?
- Por que o RSTP converge tão mais rápido que o STP 802.1D?
- Qual a vantagem do PVST sobre uma árvore única para todas as VLANs?
- Por que LACP em vez de `mode on`?
- Por que um único download não fica mais rápido em um Port-channel de 2 links?
- O que faz uma SVI ficar down/down?
- Qual a diferença entre HSRP e VRRP? Para que serve o preempt?
- Por que alinhar o root bridge com o HSRP Active?
- Como proteger a rede contra um switch indevido virando root? (root guard, BPDU guard)
---