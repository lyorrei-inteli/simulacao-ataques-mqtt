# Relatório de Análise de Segurança MQTT

## Sumário

Este relatório detalha uma análise de segurança focada no protocolo MQTT, explorando vulnerabilidades em um cenário composto por um broker MQTT local e um remoto. A análise segue os pilares da CIA - Confiabilidade, Integridade e Disponibilidade - para identificar e documentar potenciais riscos de segurança e recomendar medidas de mitigação.

## Configuração do Ambiente de Testes

### Broker MQTT Remoto

Utilizamos um broker MQTT público para a configuração remota, acessado via HiveMQ, que permite uma conexão rápida e fácil sem a necessidade de configuração adicional.

### Broker MQTT Local

O ambiente local foi configurado utilizando Docker e o Mosquitto como broker MQTT, seguindo os passos abaixo:

1. **Criação do Ambiente Docker:**

    ```bash
    mkdir mqtt5
    cd mqtt5
    mkdir config data log
    ```

2. **Configuração do Mosquitto:**

    Arquivo `config/mosquitto.conf`:

    ```
    allow_anonymous true
    listener 1883
    listener 9001 protocol websockets
    persistence true
    password_file /mosquitto/config/pwfile
    persistence_file mosquitto.db
    persistence_location /mosquitto/data/
    ```

3. **Docker Compose:**

    Arquivo `docker-compose.yml`:

    ```yaml
    version: "3.7"
    services:
      mqtt5:
        image: eclipse-mosquitto
        container_name: mqtt5
        ports:
          - "1883:1883" # Default MQTT port
          - "9001:9001" # MQTT port for WebSockets
        volumes:
          - ./config:/mosquitto/config:rw
          - ./data:/mosquitto/data:rw
          - ./log:/mosquitto/log:rw
        restart: unless-stopped
        deploy:
          resources:
            limits:
              cpus: '0.01'
              memory: 100M
    ```

    O comando para iniciar o ambiente:

    ```bash
    docker-compose -p mqtt5 up -d
    ```

## Análise Aprofundada das Vulnerabilidades do MQTT

### Confiabilidade

**Cenário de Ataque: Reutilização do ClientID**

A reutilização de ClientIDs em diferentes sessões é uma vulnerabilidade significativa no MQTT. Quando um novo cliente se conecta usando um ClientID que já está em uso, o broker MQTT desconecta automaticamente a sessão anterior que estava usando esse ID. Essa funcionalidade pode ser explorada por um atacante para interromper serviços críticos, desautorizando dispositivos legítimos de suas funções regulares.

**Evidência de Vulnerabilidade:**

1. **Conexão Inicial com Cliente Legítimo:**
   ```bash
   mosquitto_sub -h localhost -t 'test/topic' -i "ClientID"
   ```
   Este comando conecta um cliente legítimo ao broker MQTT, subscrevendo a um tópico 'test/topic' com um ClientID específico.

2. **Interrupção pela Reutilização do ClientID:**
   ```bash
   mosquitto_sub -h localhost -t 'test/topic' -i "ClientID"
   ```
   Executar este comando em uma nova sessão, utilizando o mesmo ClientID, resultará na desconexão do cliente legítimo anterior. Este comportamento pode ser abusado para desestabilizar a comunicação MQTT, afetando a confiabilidade do sistema.

### Integridade

**Cenário de Ataque: Publicação Não Autorizada em Tópicos**

A permissão de publicação anônima em tópicos é uma séria ameaça à integridade dos dados no MQTT. Com a configuração `allow_anonymous true`, qualquer cliente pode publicar mensagens em qualquer tópico sem autenticação, possibilitando a inserção de dados falsificados ou mal-intencionados.

**Evidência de Vulnerabilidade:**

1. **Publicação Anônima:**
   ```bash
   mosquitto_pub -h localhost -t 'critical/topic' -m 'Malicious payload'
   ```
   Este comando ilustra como um atacante pode facilmente publicar uma mensagem maliciosa em um tópico crítico, sem necessidade de autenticação. A mensagem pode ser projetada para causar disrupção operacional, manipular dispositivos IoT ou disseminar informações falsas.

### Disponibilidade

**Cenário de Ataque: Ataque de Negação de Serviço (DoS)**

A limitação de recursos no ambiente Docker é uma preocupação para a disponibilidade do MQTT. Atacantes podem explorar essas limitações, lançando ataques de DoS que sobrecarregam o broker com requisições excessivas, seja em volume de mensagens ou número de conexões, resultando em degradação de serviço ou inoperância total.

**Evidência de Vulnerabilidade:**

1. **Simulação de Carga Excessiva:**
   A simulação de um ataque de DoS pode ser feita através de scripts ou ferramentas que geram um grande número de conexões ou publicações em massa, visando esgotar os recursos do broker. Um exemplo simplificado seria o uso de um loop para publicar mensagens repetidamente:
   ```bash
   while true; do mosquitto_pub -h localhost -t 'flood/topic' -m 'Payload'; done
   ```
   Este comando cria um fluxo contínuo de mensagens para o broker, o que, em um cenário real de ataque, poderia ser amplificado por vários atacantes simultâneos, causando uma sobrecarga significativa no sistema.

## Recomendações Detalhadas

Para cada uma das vulnerabilidades identificadas, as seguintes medidas de mitigação são recomendadas:

- **Confiabilidade:** Implementar um sistema robusto de gerenciamento de sessões que inclua mecanismos de autenticação e autorização mais sofisticados. Utilizar tokens de sessão únicos e de curta duração pode ajudar a prevenir a reutilização maliciosa de ClientIDs.
- **Integridade:** Desabilitar completamente a opção `allow_anonymous` e estabelecer um sistema de controle de acesso baseado em listas de controle de acesso (ACL) e certificados digitais para autenticação. Além disso, a adoção de criptografia TLS/SSL para comunicações MQTT pode proteger contra a interceptação e modificação de mensagens.
- **Disponibilidade:** Alocar recursos de forma dinâmica e implementar soluções de escalabilidade automática para o broker MQTT pode ajudar a mitigar ataques de DoS. Ferramentas de monitoramento e detecção de anomalias também são essenciais para identificar e responder rapidamente a ataques potenciais.

Estas recomendações, quando implementadas, podem ajudar significativamente a fortalecer a segurança de ambientes que utilizam o protocolo MQTT, protegendo-os contra uma ampla gama de ameaças.