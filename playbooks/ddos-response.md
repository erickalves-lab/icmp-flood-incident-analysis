# Playbook de Resposta a Incidentes DDoS: ICMP Flood

## Metadados do Playbook
- **ID:** IR-PB-004
- **Versao:** 2.0
- **Ataque Coberto:** DDoS Volumetrico (ICMP Flood, DNS Amplification)
- **Nivel de Severidade:** ALTA (Critico)
- **Tempo Estimado de Execucao:** 15-45 minutos
- **Ultima Revisao:** 15/03/2024
- **Proxima Revisao:** 15/06/2024

---

## Fase 1: Deteccao e Confirmacao (0-5 minutos)

### 1.1. Sinais de Alerta
- SIEM alerta: "ICMP Traffic > 500% baseline"
- Dashboard Grafana: Pico de trafego inbound
- Clientes reportam lentidao ou indisponibilidade

### 1.2. Comandos de Confirmacao

#### 1. Verificar interface de rede
nload -u m -i 1000 -o 1000 eth0

#### 2. Capturar amostra de trafego ICMP
timeout 30 tshark -i eth0 -f "icmp" -c 1000 -w /tmp/ddos-capture-$(date +%s).pcap

#### 3. Analisar origem diversificada (sinal de DDoS)
tshark -r /tmp/ddos-capture-*.pcap -T fields -e ip.src | sort | uniq -c | sort -nr | head -20

#### 4. Verificar taxa de pacotes
netstat -i eth0 | grep "icmp"  # ou usar iftop, bmon

### 1.3. Criterios de Confirmacao
**Confirmar DDoS se:**
- > 10.000 pacotes ICMP/segundo
- > 1.000 IPs de origem unicos
- Trafego > 80% da capacidade do link

---

## Fase 2: Ativacao e Comunicacao (5-10 minutos)

### 2.1. Cadeia de Escalonamento
- SOC L1 Detecta -> Verifica Severidade -> Se ALTA: SOC L2 + NOC -> Gerente de Seguranca -> Diretoria TI -> Cloudflare/ISP

### 2.2. Contatos Prioritarios
| Funcao | Nome | Contato 1 | Contato 2 |
|--------|------|-----------|-----------|
| **SOC Lead** | Joao Silva | Telegram: @joaosoc | (11) 99999-0001 |
| **NOC Lead** | Maria Santos | Slack: @marianoc | (11) 99999-0002 |
| **Cloudflare** | Suporte Enterprise | support@cloudflare.com | +1 (888) 993-5273 |
| **ISP Principal** | Claro  | claro-empresas@claro.com.br | 0800 722 4242 |

### 2.3. Template de Comunicacao Inicial
**Assunto:** [SEVERITY: HIGH] Incidente DDoS em andamento - Acao Imediata Requerida

- INCIDENTE: DDoS ICMP Flood
- HORA: [HH:MM]
- SERVICOS AFETADOS: [Listar]
- TRAFEGO ATUAL: [X] Gbps, [Y] pps
- ACOES EM CURSO: [Listar]
- PROXIMA ATUALIZACAO: [HH:MM]

---

## Fase 3: Mitigacao Tecnica (10-30 minutos)

### 3.1. Mitigacao Imediata (On-Premise)

#### 1. Rate limiting no firewall (iptables)
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 10/second -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

#### 2. Bloqueio de faixas IP maliciosas
for ip in $(cat /tmp/attackers-ips.txt); do
    iptables -A INPUT -s $ip -j DROP
done

#### 3. Habilitar protecao SYN cookies
sysctl -w net.ipv4.tcp_syncookies=1

##### 3.2. Ativacao de Servicos Externos
**Cloudflare:**
1. Login em dashboard.cloudflare.com
2. Navegar para: Security -> DDoS -> HTTP DDoS Protection
3. Configurar:
   - Sensitivity: High
   - Action: Challenge/Block
4. Ativar Network-layer DDoS Protection
5. Configurar Rate Limiting Rules:
   - If: ICMP Traffic
   - Then: Block
   - Duration: 1 hour

**AWS Shield Advanced (se aplicavel):**
aws shield create-protection --name "DDoS-Mitigation-$(date +%s)" --resource-arn arn:aws:cloudfront::123456789:distribution/ABCDEFG

#### 3.3. Redirecionamento de Trafego (BGP)
##### Comando para roteador (exemplo Cisco)
- router bgp 64512
- network 200.100.50.0 mask 255.255.255.0
- route-map SCRUBBING-OUT out
- !
- route-map SCRUBBING-OUT permit 10
- match ip address prefix-list TO-SCRUB
 - set ip next-hop 192.0.2.1  # IP do scrubbing center

---

## Fase 4: Monitoramento e Validacao (30-45 minutos)

### 4.1. Comandos de Verificacao

#### 1. Verificar eficacia da mitigacao
- vnstat -l -i eth0  # trafego em tempo real
- iftop -i eth0 -f "icmp"

#### 2. Monitorar servicos
- curl -I https://seusite.com --max-time 5
- nmap -p 80,443,53 seusite.com

#### 3. Logs de firewall
- tail -f /var/log/iptables.log | grep DROP

### 4.2. Dashboard de Monitoramento
- Grafana: [Link para dashboard DDoS]
- Cloudflare Radar: [Link para analytics]
- Status Page: [Link para pagina de status]

### 4.3. Criterios de Resolucao
**Incidente resolvido quando:**
- Trafego ICMP < 100 pps por 15 minutos
- Servicos respondendo em < 200ms
- 0% de packet loss para clientes legitimos

---

## Fase 5: Documentacao e Licoes Aprendidas

### 5.1. Coleta de Evidencias

### Coletar dados forenses
- tshark -r captura.pcap -Y "icmp" -T json > /evidence/icmp-analysis.json
- netstat -s -p icmp > /evidence/icmp-stats.txt
- grep "DROP.*icmp" /var/log/iptables.log > /evidence/firewall-drops.log

### 5.2. Template de Relatorio Pos-Incidente
## Incidente: [ID]
- **Data:** [DD/MM/YYYY]
- **Duracao:** [X] minutos
- **Impacto:** [Servicos afetados]
- **Causa Raiz:** [Descricao tecnica]
- **Acoes Tomadas:** [Lista numerada]
- **Licoes Aprendidas:** [Pontos-chave]

---

## Apendices

### A. Scripts Automatizados

- #!/bin/bash
- #detect-ddos.sh
- THRESHOLD=10000
- CURRENT=$(netstat -s | grep "icmp" | head -1 | awk '{print $1}')

- if [ $CURRENT -gt $THRESHOLD ]; then
- echo "ALERTA: Possivel DDoS ICMP detectado"
- systemctl start ddos-mitigation.service
- fi

### B. Referencias Rapidas
- NIST SP 800-61: Computer Security Incident Handling Guide
- CERT.br: Tratamento de Incidentes de Seguranca
- Cloudflare Docs: DDoS Protection

### C. Glossario
- pps: Packets Per Second
- BGP: Border Gateway Protocol
- Anycast: Tecnica de roteamento
- Scrubbing: Limpeza de trafego DDoS

---

## Checklist de Execucao
- [ ] Confirmacao do ataque (comandos executados)
- [ ] Escalonamento realizado (contatos notificados)
- [ ] Mitigacao ativada (firewall + cloud)
- [ ] Comunicacao enviada (clientes + diretoria)
- [ ] Monitoramento intensivo (dashboards)
- [ ] Evidencias coletadas (logs + captures)
- [ ] Relatorio iniciado (template preenchido)

---

**ID do Playbook:** IR-PB-004 | **Versao:** 2.0 | **Ultima Atualizacao:** 15/03/2024
