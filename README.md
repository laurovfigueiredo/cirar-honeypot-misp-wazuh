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
