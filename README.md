# 🧪 Laboratório de Redes Corporativas no EVE-NG

Laboratório prático para preparação técnica para uma vaga de redes, simulando um ambiente corporativo com switches Cisco, firewall FortiGate, roteadores MikroTik e hosts Linux. O projeto é dividido em **sub-laboratórios**, porque o hardware disponível (16 GB de RAM) não comporta um ambiente único com todos os conceitos ao mesmo tempo.

---

## 💻 Ambiente

### Virtualização

- **Host:** Windows com Hyper-V, VBS e Integridade de Memória desativados (`bcdedit /set hypervisorlaunchtype off`)
- **BIOS:** SVM Mode (AMD-V) habilitado
- **Hipervisor:** VMware Workstation Pro
- **VM do EVE-NG Community:** 12 GB de RAM, 8 vCPUs, 200 GB de disco, opção *Virtualize AMD-V/RVI* habilitada

### Distribuição de RAM

| Destino | RAM |
|---|---|
| Windows (host) | ~4 GB |
| VM EVE-NG | 12 GB |
| VM de monitoramento (quando em uso) | 2–3 GB, reduzindo o EVE para 9–10 GB |

### Imagens utilizadas

Diretório: `/opt/unetlab/addons/qemu/`

| Pasta | Função | RAM por nó |
|---|---|---|
| `fortinet-FGT-v7.4` | Firewall de borda, SD-WAN, IPsec | 2 GB / 1 vCPU |
| `vios-adventerprisek9-m.SPA.159-3.M6` | Roteador Cisco (OSPF, BGP, IPsec, multicast) | 512 MB |
| `viosl2-adventerprisek9-m.ssa.high_iron_20200929` | Switch Cisco L2/L3 | 768 MB–1 GB |
| `mikrotik-6.44.5` | ISP, filial, peer BGP e IPsec | 128–256 MB |
| `linux-redesbrasil` | Hosts e servidores | depende da distro |
| VPCS (nativo do EVE) | Hosts para testes simples | ~0 |

> ⚠️ O EVE-NG reconhece o tipo de nó pelo **prefixo da pasta**. A pasta do FortiGate precisa começar com `fortinet-FGT-`. Depois de qualquer alteração em imagens:
> ```bash
> /opt/unetlab/wrappers/unl_wrapper -a fixpermissions
> ```

---

## 🗺️ Roadmap dos laboratórios

| # | Laboratório | Requisitos cobertos | Status |
|---|---|---|---|
| 1 | Switching de campus | VLAN, STP, LACP, FHRP, troubleshooting L2 | 🔄 Em andamento |
| 2 | OSPF + rotas estáticas | Roteamento, troubleshooting L3 | ⏳ Pendente |
| 3 | BGP com ISPs | eBGP/iBGP, políticas de roteamento | ⏳ Pendente |
| 4 | Borda FortiGate + SD-WAN | Firewall, SD-WAN, failover automático | ⏳ Pendente |
| 5 | IPsec multi-vendor | IPsec e troubleshooting | ⏳ Pendente |
| 6 | Híbrido com AWS | IPsec on-prem ↔ cloud, BGP, AWS | ⏳ Pendente |
| 7 | Multicast | PIM, IGMP | ⏳ Pendente |
| 8 | MLAG no data center | MLAG (conceitos de vPC) | ⏳ Pendente (requer Arista vEOS) |
| 9 | Monitoramento | Zabbix, Grafana, SNMP, syslog | ⏳ Pendente |

---

- 1 vCPU e 2 GB de RAM
- Máximo de **3 interfaces, 3 políticas de firewall e 3 rotas**
- Criptografia baixa apenas: as propostas IPsec podem ficar restritas a cifras fracas; verificar com `set proposal ?`
- Sem FortiGuard e sem FortiCare
- Uma cópia por conta FortiCloud

**Consequências:** o FortiGate é usado como firewall de borda (WAN/LAN/DMZ ou WAN1/WAN2/LAN para SD-WAN), com o roteamento entre VLANs feito nos switches L3. Para VPN com a AWS, que exige AES, o *customer gateway* deve ser MikroTik ou Cisco.

### MikroTik no lugar de um segundo FortiGate

- Leve (128–256 MB), com licença gratuita limitada apenas em velocidade (1 Mbps de upload por interface), sem impacto em testes de protocolo
- Permite treinar **IPsec multi-vendor**, onde aparecem os problemas clássicos: propostas de fase 1/fase 2, traffic selectors, NAT-T, DPD
- Imagens atuais são RouterOS 6; a sintaxe de BGP da versão 7 é diferente

### vPC e MLAG

- vPC é exclusivo do Cisco Nexus; o Nexus 9000v pede 8–10 GB por nó, inviável com 16 GB
- Alternativa: **Arista vEOS-lab** (~2 GB por nó) para praticar MLAG, com conceitos equivalentes (peer-link, keepalive, domínio, port-channel multi-chassis)
- vPC estudado pela documentação Cisco e, se disponível, pelo Cisco DevNet Sandbox

### AWS

- Site-to-Site VPN cobrada por hora de conexão: configurar **AWS Budgets** e remover VPN, VGW/TGW e customer gateway ao final
- Verificar **CGNAT** na conexão residencial; se houver, usar IP público da operadora ou um CHR/strongSwan em EC2 como alternativa

### Boas práticas no EVE-NG

- Salvar com `write memory`; **não usar Wipe** em nós configurados
- Exportar configurações ao final (*Export all CFGs*)
- Ligar os nós aos poucos para poupar CPU

---