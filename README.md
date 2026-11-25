# cirar-honeypot-misp-wazuh
 Integração entre um Honeypot, o Wazuh e o MISP para detectar ataques reais antecipadamente, coletar evidências e automatizar o bloqueio de ameaças na rede.

Cada ferramenta desempenha um papel específico nessa arquitetura de defesa ativa:

1. O Papel de Cada Ferramenta

• Honeypot (Isca): É um sistema ou serviço vulnerável propositalmente exposto (como um SSH falso com Cowrie ou um servidor web). Ele não tem tráfego legítimo; qualquer interação com ele é maliciosa.
• Wazuh (SIEM / XDR): Coleta os logs do honeypot, correlaciona os eventos (como tentativas de força bruta, comandos executados ou arquivos baixados pelo invasor) e gera alertas estruturados.
• MISP (Threat Intelligence): Funciona como um banco de dados centralizado de inteligência de ameaças. Ele recebe, armazena, enriquece e compartilha os Indicadores de Comprometimento (IoCs), como IPs maliciosos, hashes de malware e domínios.


2. Como o Fluxo Funciona na Prática

1. Captura: Um invasor encontra o honeypot e tenta explorá-lo.
2. Análise e Alerta: O agente do Wazuh instalado no honeypot envia esses logs para o servidor Wazuh, que identifica a técnica de ataque (ex: mapeando para o framework MITRE ATT&CK).
3. Alimentação do MISP: Através de um script de integração, o Wazuh envia automaticamente os dados do ataque (como o IP de origem do invasor e o hash do payload baixado) para criar um novo evento no MISP.
4. Defesa Proativa: O MISP distribui esses novos IoCs para outros sistemas de segurança da empresa (Firewalls, Proxies, EDRs e até de volta para o próprio Wazuh via listas CDB), bloqueando o atacante antes que ele tente invadir os servidores reais de produção.

Principais Benefícios

• Zero Falsos Positivos: Como usuários reais nunca acessam o honeypot, todo alerta gerado é um ataque real confirmado.
• Inteligência Própria: Em vez de depender apenas de listas públicas de ameaças, a empresa gera inteligência baseada nos ataques direcionados especificamente à sua infraestrutura.
• Resposta Automatizada: Reduz o tempo entre a detecção de um invasor e o bloqueio dele em toda a rede corporativa.


Como gerar eventos de honeypot “na mão” (Cowrie → Wazuh → MISP)
0. Pré-check (opcional)
systemctl is-active cowrie wazuh-manager
ss -tlnp | grep -E ':(2222|2223)\b'
Esperado: active, active e as duas portas escutando.
1. Abra os logs para acompanhar em tempo real
Use um terminal só para isso:
tail -f /home/cowrie/cowrie/var/log/cowrie/cowrie.json
Em outro terminal:
tail -f /var/ossec/logs/alerts/alerts.log
E, se quiser ver o MISP sendo alimentado:
tail -f /var/ossec/logs/integrations.log
2. Gerar senha incorreta → regra 100601
De outro host da rede (recomendado, para o IP de origem aparecer real, ex.: 192.168.0.227):
ssh -p 2222 \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -o PubkeyAuthentication=no \
  -o PreferredAuthentications=password \
  -o NumberOfPasswordPrompts=1 \
  root@192.168.0.10
Quando pedir a senha, digite uma errada:
senhaerrada
✅ Resultado esperado:
- No cowrie.json:
{"eventid":"cowrie.login.failed",...,"username":"root","password":"senhaerrada","src_ip":"192.168.0.227",...}
- No alerts.log:
Rule: 100601 (level 7) -> 'Cowrie: senha incorreta para o usuario root - IP 192.168.0.227'
- No integrations.log:
[custom-misp] ... OK: evento MISP <n> criado (alerta Wazuh 100601)
3. Gerar força bruta → regra 100602
Repita 5 vezes seguidas em menos de 120 segundos, pelo mesmo IP:
for i in 1 2 3 4 5; do
  ssh -p 2222 \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/dev/null \
    -o PubkeyAuthentication=no \
    -o PreferredAuthentications=password \
    -o NumberOfPasswordPrompts=1 \
    root@192.168.0.10
done
Digite uma senha errada em cada tentativa (admin, 123456, root, teste, senhaerrada).
✅ Resultado esperado no alerts.log:
Rule: 100602 (level 12) -> 'Cowrie: possivel forca bruta SSH a partir de 192.168.0.227'
E também vira evento no MISP, pois nível 12 ≥ 7.
Se rodar de dentro do próprio servidor, o src_ip aparecerá como 192.168.0.10 ou 127.0.0.1. Para apresentação, use outro PC.
4. Gerar login bem-sucedido → regra 100603
ssh -p 2222 \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -o PubkeyAuthentication=no \
  -o PreferredAuthentications=password \
  root@192.168.0.10
Senha correta (liberada no userdb):
password
✅ Resultado:
Rule: 100603 (level 12) -> 'Cowrie: login bem-sucedido - root de 192.168.0.227'
5. Gerar comandos do atacante → regra 100604
Já dentro da sessão bem-sucedida, digite:
whoami
uname -a
ls /
cat /etc/passwd
✅ Resultado no alerts.log:
Rule: 100604 (level 5) -> 'Cowrie: comando executado pelo atacante: whoami'
Rule: 100604 (level 5) -> 'Cowrie: comando executado pelo atacante: uname -a'
Esses comandos viram alerta no Wazuh, mas não viram evento no MISP, porque a regra é nível 5 e o integrador só envia alertas ≥ 7.
6. Telnet (opcional)
telnet 192.168.0.10 2223
- login: root
- senha errada → 100601
- senha password → 100603
7. Conferir a prova final
Log bruto do honeypot
tail -10 /home/cowrie/cowrie/var/log/cowrie/cowrie.json
Alertas do Wazuh
grep -i 'cowrie' /var/ossec/logs/alerts/alerts.log | tail -12
Integrador MISP
tail -8 /var/ossec/logs/integrations.log
Eventos no MISP
KEY=$(cat /var/ossec/etc/certs/misp_api.key)
curl -sk -H "Authorization: $KEY" -H 'Accept: application/json' \
  https://192.168.0.10/events \
| python3 -c 'import sys,json; d=json.load(sys.stdin); l=d if isinstance(d,list) else d.get("response",[]); print([(e["Event"]["id"],e["Event"]["info"]) for e in l][:8])'
Nos painéis
- Wazuh Dashboard → Threat Hunting → Events → filtre rule.id:100601 ou rule.id:100602
- MISP → https://192.168.0.10/events
