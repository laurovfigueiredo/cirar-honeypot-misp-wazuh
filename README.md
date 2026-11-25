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

 Threat Hunting → Events → filtre rule.id:100601 ou rule.id:100602
- MISP → https://192.168.0.10/events

**Log Wazuh**
<img width="938" height="842" alt="image" src="https://github.com/user-attachments/assets/af671d2e-79c1-4f4b-9732-b55651ab0e2d" /> 

{"session":"a926bfca50db","protocol":"ssh","src_ip":"192.168.0.20","src_port":48682,"dst_ip":"192.168.0.10","dst_port":2222,"username":"root","password":"teste","eventid":"cowrie.login.failed","sensor":"misp","uuid":"d74128ea-e868-11f1-8749-000c2980792f","timestamp":"2026-11-25T12:16:32.699121Z","message":"login attempt [root/teste] failed"}
{"session":"6d2fb371bc4d","protocol":"ssh","src_ip":"192.168.0.20","src_port":48678,"dst_ip":"192.168.0.10","dst_port":2222,"username":"root","password":"teste5","eventid":"cowrie.login.failed","sensor":"misp","uuid":"d74128ea-e868-11f1-8749-000c2980792f","timestamp":"2026-11-25T12:16:30.307195Z","message":"login attempt [root/teste5] failed"}
{"session":"fcca092b4b2e","protocol":"ssh","src_ip":"192.168.0.20","src_port":48672,"dst_ip":"192.168.0.10","dst_port":2222,"username":"root","password":"teste2","eventid":"cowrie.login.failed","sensor":"misp","uuid":"d74128ea-e868-11f1-8749-000c2980792f","timestamp":"2026-11-25T12:16:27.532927Z","message":"login attempt [root/teste2] failed"}
{"session":"6d5215d94aa7","protocol":"ssh","src_ip":"192.168.0.20","src_port":50228,"dst_ip":"192.168.0.10","dst_port":2222,"username":"root","password":"teste","eventid":"cowrie.login.failed","sensor":"misp","uuid":"d74128ea-e868-11f1-8749-000c2980792f","timestamp":"2026-11-25T12:16:23.993200Z","message":"login attempt [root/teste] failed"}

**Log Misp**
<img width="1026" height="192" alt="image" src="https://github.com/user-attachments/assets/7a845b86-027c-4906-9fda-2129667500a2" />

Wazuh 100602 | agente misp | usuario=root | senha=teste6 | {"session":"ade7f11905a1","protocol":"ssh","src_ip":"192.168.0.20","src_port":55680,"dst_ip":"192.168.0.10","dst_port":2222,"username":"root","password":"teste6","eventid":"cowrie.login.failed","sensor":"misp","uuid":"d74128ea-e868-11f1-8749-000c2980792f","timestamp":"2026-11-25T12:16:36.268480Z","message":"login attempt [root/teste6] failed"}
