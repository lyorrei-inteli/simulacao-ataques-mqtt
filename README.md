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

## Análise de Vulnerabilidades

### Confiabilidade

**Cenário:** Reutilização do ClientID em sessões diferentes.

**Evidência de Vulnerabilidade:**

Ao conectar-se com o mesmo ClientID em uma nova sessão, a sessão anterior é desconectada automaticamente pelo broker, o que pode ser explorado para interromper conexões legítimas.

**Comando Simulado:**

```bash
mosquitto_sub -h localhost -t 'test/topic' -i "ClientID"
# Em outra sessão
mosquitto_sub -h localhost -t 'test/topic' -i "ClientID"
```

### Integridade

**Cenário:** Publicação não autorizada em um tópico.

**Evidência de Vulnerabilidade:**

A configuração `allow_anonymous true` permite que qualquer cliente publique em qualquer tópico, o que pode levar à disseminação de mensagens maliciosas ou incorretas.

**Comando Simulado:**

```bash
mosquitto_pub -h localhost -t 'critical/topic' -m 'Malicious payload'
```

### Disponibilidade

**Cenário:** Ataque de negação de serviço (DoS) através do esgotamento de recursos.

**Evidência de Vulnerabilidade:**

Os limites de recursos definidos no `docker-compose.yml` podem ser insuficientes para proteger contra ataques de DoS, onde um atacante cria múltiplas conexões ou envia um volume alto de mensagens para sobrecarregar o broker.

**Comando Simulado:**

Simulação de múltiplas conexões ou publicações em massa para esgotar os recursos do broker.

```bash
# Executado várias vezes para simular múltiplas conexões
mosquitto_pub -h localhost -t 'flood/topic' -m 'Payload'
```

## Recomendações de Segurança

### Confiabilidade

- Implementar autenticação e autorização robustas para clientes, utilizando tokens ou certificados digitais.
- Limitar o número de conexões simultâneas por ClientID.

### Integridade

- Desabilitar `allow_anonymous` e configurar um sistema de autenticação baseado em credenciais ou certificados.
- Utilizar ACLs (Access Control Lists) para restringir tópicos a clientes específicos.

### Disponibilidade

- Ajustar a alocação de recursos no `docker-compose.yml` para lidar com cargas elevadas.
- Implementar monitoramento para detectar e mitigar ataques de DoS rapidamente.

## Conclusão

A análise de segurança revelou várias vulnerabilidades potenciais no setup MQTT que podem afetar os pilares da CIA. As recomendações fornecidas visam fortalecer a segurança e a resiliência do sistema MQTT em uso, garantindo a proteção adequada contra ataques maliciosos.