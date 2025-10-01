# TRF3 Events

A aplicação TRF3 Events é um sistema automatizado de busca e processamento de eventos de processos do Tribunal Regional Federal da 3ª Região, que extrai dados processuais, documenta eventos e realiza identificação de participantes com enriquecimento de CPF através de múltiplos providers.

## **Pré-requisitos**

- Node.js versão 20.x
## Setup

1. **Criar arquivo .npmrc**
> [Gere o token em Settings → Developer settings → Personal access tokens → Tokens (classic)](https://github.com/settings/tokens)
   >
   > Substitua o token gerado em TOKEN

```bash
@precato:registry=https://npm.pkg.github.com/
@precatoorg:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=TOKEN
```

2. **Instalar dependências**

   ```bash
   npm install
   ```

3. **Criar arquivo .env**

> Copie as variáveis de ambiente do arquivo _.env.example_

4. **Iniciar containers necessários**

   - Ligar container do _rabbit_
   - Ligar a aplicação do _database-server_

5. **Executar aplicação**
   ```bash
   npm run dev
   ```
```mermaid
%% Fluxo: Busca de Eventos de Processos (UPDATE_TRF3_PROCESS_EVENTS)

  

sequenceDiagram

    participant queue

    participant consumer

    participant service

    participant provider

    participant postgres

  

    %% 1) Mensagem chega na fila e é consumida

    Note over queue: Mensagem aguardando consumo

    queue-->>consumer: UPDATE_TRF3_PROCESS_EVENTS<br/>JSON { processNumber: string }

  

    %% 2) consumer chama o serviço principal

    consumer->>service: SearchProcessEventsService.execute({ processNumber })

  

    %% 3) Serviço busca dados do processo no TRF3 (PJe)

    service->>provider: Trf3PjeProvider.getProcessData({ processNumber })

    provider-->>service: [ processData: {...}, process: {...} ]

  

    %% 4) Verificar eventos já existentes

    service->>postgres: findManyByProcessId<br/>(Busca dados já registrados)

    postgres-->>service: processEvents<br/>(eventos já existentes)

  

    %% 5) Registrar novos eventos

    service->>postgres: createMany<br/>(registrar novos eventos)

    postgres-->>service: ok

  

    %% 6) Fim

    service-->>consumer: ok

    consumer-->>queue: ACK<br/>(Fila pode remover a mensagem)
```
### Fluxo: Identificação de Participantes de Processo

```mermaid
%% Fluxo: Identificação de Participantes de Processo com Fallbacks
sequenceDiagram
    participant Queue
    participant Consumer
    participant Service
    participant EscavadorProvider
    participant NovaVidaProvider
    participant Trf3ProcessProvider
    participant Trf3HttpSender
    participant PostgresDB

    %% ETAPA 1: Recebimento e disparo do serviço
    Queue-->>Consumer: SEARCH_PROCESS_PARTICIPANT_IDENTIFICATIONS<br/>{ process_number, active_pole }
    Consumer->>Service: SearchProcessParticipantIdentificationsService.execute()
    Note over Consumer: active_pole = [nome completo - 123.XXX.XXX-XX]<br/>CPF somente com os 3 primeiros digitos

    %% ETAPA 2: Primeira tentativa - Escavador
    Note over Service: Tentativa 1: Busca com Escavador
    Service->>EscavadorProvider: escavadorProcessSearchProvider
    EscavadorProvider-->>Service: CPFs encontrados
    alt Escavador encontrou participantes com CPF
        Service->>Trf3HttpSender: sendToProcessualSearchByHttp<br/>{ process_number: "", cpf: "" }
        Trf3HttpSender-->>Service: dados enviados com sucesso
    end

    %% ETAPA 3: Verificação de lacunas (nomes sem CPF no DB)
    Service->>PostgresDB: findManyByProcessoOriginario<br/>(verifica nomes já no banco)
    PostgresDB-->>Service: processos existentes

    %% ETAPA 4: Segunda tentativa - Enriquecimento Nova Vida
    Note over Service: Tentativa 2: Enriquecimento com Nova Vida
    alt Ainda tem CPFs faltando
        Service->>NovaVidaProvider: getCpfsByFullName<br/>(busca CPF por nome completo)
        NovaVidaProvider-->>Service: CPFs encontrados por nome
        Service->>Trf3HttpSender: sendToProcessualSearchByHttp<br/>{ process_number: "", cpf: "" }
        Trf3HttpSender-->>Service: dados enviados com sucesso
    end

    %% ETAPA 5: Terceira tentativa - Busca logada TRF3 (fallback final)
    Note over Service: Tentativa 3: Busca Logada no TRF3
    alt Nova Vida não encontrou todos os CPFs
        Service->>Trf3ProcessProvider: getParticipantIdentifications<br/>(busca logada direta)
        Trf3ProcessProvider-->>Service: identificações completas
        Service->>Trf3HttpSender: sendToProcessualSearchByHttp<br/>{ process_number: "", cpf: "" }
        Trf3HttpSender-->>Service: dados enviados com sucesso
    end

    %% ETAPA 6: Uso de CPFs parciais (se houver)
    Note over Service: Usa CPFs parciais
    alt Sobrou CPFs parciais
        Service->>Trf3HttpSender: sendToProcessualSearchByHttp<br/>{ process_number: "", cpf: "" }
        Trf3HttpSender-->>Service: dados parciais enviados
    end

    %% ETAPA 7: Desfecho (sucesso ou erro)
    alt Nenhuma tentativa funcionou
        Service-->>Consumer: Erro: Não foi possível identificar todos os CPFs
    else Enriquecimento concluído
        Service-->>Consumer: Processamento concluído
    end

    %% ETAPA 8: Finalização da mensagem
    Consumer-->>Queue: ACK<br/>(Fila pode remover a mensagem)
```
