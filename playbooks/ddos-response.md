# Playbook: Resposta Rápida a DDoS

## Contatos Imediatos
- SOC Lead: (11) 99999-9999
- Cloudflare: support@cloudflare.com
- ISP: atendimento@claro.com.br

## Ações em 5 Minutos
1. Verificar tráfego: `nload eth0`
2. Confirmar DDoS: `tshark -i eth0 -Y "icmp" -c 100`
3. Ativar Cloudflare: Login → Security → DDoS Protection → ON
4. Comunicar equipe: Slack #incidentes

## Mitigação
- Rate limiting no firewall
- Blackhole de IPs maliciosos
- Contatar ISPs dos atacantes
